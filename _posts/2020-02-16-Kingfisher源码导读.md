---
layout: post
title: Kingfisher源码导读
date: 2020-02-16 20:32
tags: [源码导读]
categories: [iOS]
---

## 前言
在使用第三方库的时候，往往只用其功能，很少去关注背后的实现细节。本文从源码（Kingfisher v5.13.0）角度，学习与研究Kingfisher背后实现细节。

<!-- more-->
### kf属性
kf是计算属性，用以返回包裹系统类UIIMageView，UIButton，NSButton的KingfisherWrapper，也就是用struct装饰泛型Base。
```swift
public struct KingfisherWrapper<Base> {
    public let base: Base
    public init(_ base: Base) {
        self.base = base
    }
}

extension KingfisherCompatible {
    /// Gets a namespace holder for Kingfisher compatible types.
    public var kf: KingfisherWrapper<Self> {
        get { return KingfisherWrapper(self) }
        set { }
    }
}

extension KingfisherCompatibleValue {
    /// Gets a namespace holder for Kingfisher compatible types.
    public var kf: KingfisherWrapper<Self> {
        get { return KingfisherWrapper(self) }
        set { }
    }
}
```

### setImage()
在设置图片的时候，无论是UIImageView，UIButton，NSButton都调用了`setImage()` 。这个方法核心的功能，在于使用KingfisherManager的`retrieveImage`来生成下载任务，并且将下载任务存储到当前实例的KingfisherWrapper的imageTask属性。但是在KingfisherWrapper中并没有定义imageTask属性，这个怎么做到的？使用了**关联对象**来动态存取。
```swift
private var imageTask: DownloadTask? {
    get { return getAssociatedObject(base, &imageTaskKey) }
    set { setRetainedAssociatedObject(base, &imageTaskKey, newValue)}
}

public private(set) var taskIdentifier: Source.Identifier.Value? {
    get {
        let box: Box<Source.Identifier.Value>? = getAssociatedObject(base, &taskIdentifierKey)
        return box?.value
    }
    set {
        let box = newValue.map { Box($0) }
        setRetainedAssociatedObject(base, &taskIdentifierKey, box)
    }
}
```

每个下载任务都有自己的Identifier，这个Identifier也是通过关联对象动态设置为KingfisherWrapper的属性。被关联的对象必须是具有内存管理的类型，因此，这里将UInt类型的Identifier封装在了Box类里，间接完成Identifier的设置。
```swift
class Box<T> {
    var value: T
    
    init(_ value: T) {
        self.value = value
    }
}

```

setImage()方法是整个Kingfisher提供给外部对象导入使用加载网络图片。
```swift
@discardableResult
public func setImage(
    with source: Source?,
    placeholder: Placeholder? = nil,
    options: KingfisherOptionsInfo? = nil,
    progressBlock: DownloadProgressBlock? = nil,
    completionHandler: ((Result<RetrieveImageResult, KingfisherError>) -> Void)? = nil) -> DownloadTask?
{
    var mutatingSelf = self
    // source判空
    guard let source = source else {
        mutatingSelf.placeholder = placeholder
        mutatingSelf.taskIdentifier = nil
        completionHandler?(.failure(KingfisherError.imageSettingError(reason: .emptySource)))
        return nil
    }
    // 加载配置
    var options = KingfisherParsedOptionsInfo(KingfisherManager.shared.defaultOptions + (options ?? .empty))
    let noImageOrPlaceholderSet = base.image == nil && self.placeholder == nil
    if !options.keepCurrentImageWhileLoading || noImageOrPlaceholderSet {
        // Always set placeholder while there is no image/placeholder yet.
        mutatingSelf.placeholder = placeholder
    }

    let maybeIndicator = indicator
    maybeIndicator?.startAnimatingView()
    // 任务Identifier自增加，以确保每次调用此方法，都是一个新的加载任务。
    let issuedIdentifier = Source.Identifier.next()
    mutatingSelf.taskIdentifier = issuedIdentifier

    if base.shouldPreloadAllAnimation() {
        options.preloadAllAnimationData = true
    }

    if let block = progressBlock {
        options.onDataReceived = (options.onDataReceived ?? []) + [ImageLoadingProgressSideEffect(block)]
    }
    // 图片如果是有Provider提供，直接获取。
    if let provider = ImageProgressiveProvider(options, refresh: { image in
        self.base.image = image
    }) {
        options.onDataReceived = (options.onDataReceived ?? []) + [provider]
    }
    
    options.onDataReceived?.forEach {
        $0.onShouldApply = { issuedIdentifier == self.taskIdentifier }
    }
    // 非以上情形，执行KingfisherManager里，加载图片。
    let task = KingfisherManager.shared.retrieveImage(
        with: source,
        options: options,
        downloadTaskUpdated: { mutatingSelf.imageTask = $0 },
        completionHandler: { result in
            // 成功获取到了图片，执行回调
            CallbackQueue.mainCurrentOrAsync.execute {
                // 指示器停止动画
                maybeIndicator?.stopAnimatingView()
                guard issuedIdentifier == self.taskIdentifier else {
                    let reason: KingfisherError.ImageSettingErrorReason
                    do {
                        let value = try result.get()
                        reason = .notCurrentSourceTask(result: value, error: nil, source: source)
                    } catch {
                        reason = .notCurrentSourceTask(result: nil, error: error, source: source)
                    }
                    let error = KingfisherError.imageSettingError(reason: reason)
                    completionHandler?(.failure(error))
                    return
                }
                
                mutatingSelf.imageTask = nil
                mutatingSelf.taskIdentifier = nil
                
                switch result {
                case .success(let value):
                    // 是否有Transition过程
                    guard self.needsTransition(options: options, cacheType: value.cacheType) else {
                        mutatingSelf.placeholder = nil
                        self.base.image = value.image
                        completionHandler?(result)
                        return
                    }
                    // 执行Transition，并更新图片
                    self.makeTransition(image: value.image, transition: options.transition) {
                        completionHandler?(result)
                    }
                    
                case .failure:
                    if let image = options.onFailureImage {
                        self.base.image = image
                    }
                    completionHandler?(result)
                }
            }
        }
    )
    mutatingSelf.imageTask = task
    return task
}

```


###  管理者 - KingfisherManager
#### 获取图片
从setImage()到是否执行请求下载图片的逻辑，是由KingfisherManager的`retrieveImage()`处理的。`retrieveImage()`的处理逻辑是：
1. 判断图片是否需要强制刷新，如果是，就执行loadAndCacheImage方法生成下载任务
2. 图片在缓存中存在且不强制刷新，并且当前retrieveImage方法返回nil，不生成新的下载任务
3. 图片在缓存中不存在且不强制刷新；生成下载任务并缓存图片。
```swift
private func retrieveImage(
    with source: Source,
    context: RetrievingContext,
    completionHandler: ((Result<RetrieveImageResult, KingfisherError>) -> Void)?) -> DownloadTask?
{
    let options = context.options
	  // 判断是否需要强制刷新，是执行loadAndCacheImage方法生成下载任务
    if options.forceRefresh {
        return loadAndCacheImage(
            source: source,
            context: context,
            completionHandler: completionHandler)?.value
        
    } else {
    	  // 从缓存中加载
        let loadedFromCache = retrieveImageFromCache(
            source: source,
            context: context,
            completionHandler: completionHandler)
        // 如果缓存中有这张图片，当前返回nil，不生成下载任务，执行结束
        if loadedFromCache {
            return nil
        }
        // 未从缓存中找到图片，但配置是只从缓存中读取图片，此时，抛出Error。
        if options.onlyFromCache {
            let error = KingfisherError.cacheError(reason: .imageNotExisting(key: source.cacheKey))
            completionHandler?(.failure(error))
            return nil
        }
        
        return loadAndCacheImage(
            source: source,
            context: context,
            completionHandler: completionHandler)?.value
    }
}
```

可以看出，无论是在哪种情况下，只需要调用kf.setImage()方法去设置图片；图片的下载，缓存，以及是否需要开启网络请求下载图片都隐藏于这行代码之下了。


#### 从缓存中获取图片 - retrieveImageFromCache
这个`retrieveImageFromCache()`包含了从内存 / 硬盘中查找图片，返回值为Bool，显示缓存中有无这张图片。
```swift
func retrieveImageFromCache(
        source: Source,
        context: RetrievingContext,
        completionHandler: ((Result<RetrieveImageResult, KingfisherError>) -> Void)?) -> Bool
    {
        let options = context.options
        // 1. 判断图片是否已经存在目标缓存中
        let targetCache = options.targetCache ?? cache
        let key = source.cacheKey
        let targetImageCached = targetCache.imageCachedType(
            forKey: key, processorIdentifier: options.processor.identifier)
        let validCache = targetImageCached.cached &&
            (options.fromMemoryCacheOrRefresh == false || targetImageCached == .memory)
        if validCache {
        	  // 获取图片
            targetCache.retrieveImage(forKey: key, options: options) { result in
                guard let completionHandler = completionHandler else { return }
                options.callbackQueue.execute {
                    result.match(
                        onSuccess: { cacheResult in
                            let value: Result<RetrieveImageResult, KingfisherError>
                            if let image = cacheResult.image {
                                value = result.map {
                                    RetrieveImageResult(
                                        image: image,
                                        cacheType: $0.cacheType,
                                        source: source,
                                        originalSource: context.originalSource
                                    )
                                }
                            } else {
                                value = .failure(KingfisherError.cacheError(reason: .imageNotExisting(key: key)))
                            }
                            completionHandler(value)
                        },
                        onFailure: { _ in
                            completionHandler(.failure(KingfisherError.cacheError(reason: .imageNotExisting(key:key))))
                        }
                    )
                }
            }
            return true
        }

        // 2. 判断原始的图片是否已经缓存，如果存在缓存图片，那么直接返回。
        let originalCache = options.originalCache ?? targetCache
        // No need to store the same file in the same cache again.
        if originalCache === targetCache && options.processor == DefaultImageProcessor.default {
            return false
        }

        // 检查是否存在未处理的图片
        let originalImageCacheType = originalCache.imageCachedType(
            forKey: key, processorIdentifier: DefaultImageProcessor.default.identifier)
        let canAcceptDiskCache = !options.fromMemoryCacheOrRefresh
        
        let canUseOriginalImageCache =
            (canAcceptDiskCache && originalImageCacheType.cached) ||
            (!canAcceptDiskCache && originalImageCacheType == .memory)
        
        if canUseOriginalImageCache {
            // 找到缓存的图片，处理为原始的数据
            var optionsWithoutProcessor = options
            optionsWithoutProcessor.processor = DefaultImageProcessor.default
            originalCache.retrieveImage(forKey: key, options: optionsWithoutProcessor) { result in

                result.match(
                    onSuccess: { cacheResult in
                        guard let image = cacheResult.image else {
                            assertionFailure("The image (under key: \(key) should be existing in the original cache.")
                            return
                        }

                        let processor = options.processor
                        (options.processingQueue ?? self.processingQueue).execute {
                            let item = ImageProcessItem.image(image)
                            guard let processedImage = processor.process(item: item, options: options) else {
                                let error = KingfisherError.processorError(
                                    reason: .processingFailed(processor: processor, item: item))
                                options.callbackQueue.execute { completionHandler?(.failure(error)) }
                                return
                            }

                            var cacheOptions = options
                            cacheOptions.callbackQueue = .untouch
                            // 回调协作器
                            let coordinator = CacheCallbackCoordinator(
                                shouldWaitForCache: options.waitForCache, shouldCacheOriginal: false)
                            // 缓存已经处理过的图片
                            targetCache.store(
                                processedImage,
                                forKey: key,
                                options: cacheOptions,
                                toDisk: !options.cacheMemoryOnly)
                            {
                                _ in
                                coordinator.apply(.cachingImage) {
                                    let value = RetrieveImageResult(
                                        image: processedImage,
                                        cacheType: .none,
                                        source: source,
                                        originalSource: context.originalSource
                                    )
                                    options.callbackQueue.execute { completionHandler?(.success(value)) }
                                }
                            }

                            coordinator.apply(.cacheInitiated) {
                                let value = RetrieveImageResult(
                                    image: processedImage,
                                    cacheType: .none,
                                    source: source,
                                    originalSource: context.originalSource
                                )
                                options.callbackQueue.execute { completionHandler?(.success(value)) }
                            }
                        }
                    },
                    onFailure: { _ in
                        // 失败回调
                        options.callbackQueue.execute {
                            completionHandler?(
                                .failure(KingfisherError.cacheError(reason: .imageNotExisting(key: key)))
                            )
                        }
                    }
                )
            }
            return true
        }

        return false
    }

```


#### 从网络中加载并缓存图片 - loadAndCacheImage

何时会执行从网络中加载并缓存图片 ？
分为两种情况：
- 设置图片时，缓存（内存 / 硬盘）中没有这张图片。
- 设置图片时，强制刷新图片数据。

`loadAndCacheImage`方法做了个判断。判断当前图片是否需要从网络中下载获取（从网络中获取，执行下载，并缓存图片），还是通过ImageProvider获取（直接获取图片并缓存），并生成对应的下载任务。
```swift
@discardableResult
func loadAndCacheImage(
    source: Source,
    context: RetrievingContext,
    completionHandler: ((Result<RetrieveImageResult, KingfisherError>) -> Void)?) -> DownloadTask.WrappedTask?
{
    let options = context.options
    // 定义内嵌函数
    func _cacheImage(_ result: Result<ImageLoadingResult, KingfisherError>) {
        cacheImage(
            source: source,
            options: options,
            context: context,
            result: result,
            completionHandler: completionHandler
        )
    }
    
    switch source {
    // 从网络中加载图片
    case .network(let resource):
        let downloader = options.downloader ?? self.downloader
        let task = downloader.downloadImage(
            with: resource.downloadURL, options: options, completionHandler: _cacheImage
        )
        return task.map(DownloadTask.WrappedTask.download)
    // 从ImageProvider中加载图片
    case .provider(let provider):
        provideImage(provider: provider, options: options, completionHandler: _cacheImage)
        return .dataProviding
    }
}

```


## 缓存细节
### loadAndCacheImage中的缓存逻辑
在`loadAndCacheImage`中，执行了下载图片任务后，都会执行`_cacheImage()`内嵌函数。`_cacheImage`内部执行`cacheImage`。cacheImage的实现：
```swift
private func cacheImage(
    source: Source,
    options: KingfisherParsedOptionsInfo,
    context: RetrievingContext,
    result: Result<ImageLoadingResult, KingfisherError>,
    completionHandler: ((Result<RetrieveImageResult, KingfisherError>) -> Void)?)
{
    switch result {
    case .success(let value):
        let needToCacheOriginalImage = options.cacheOriginalImage &&
                                       options.processor != DefaultImageProcessor.default
                                           
        let coordinator = CacheCallbackCoordinator(
            shouldWaitForCache: options.waitForCache, shouldCacheOriginal: needToCacheOriginalImage)
        // 将图片缓存
        let targetCache = options.targetCache ?? self.cache
        targetCache.store(
            value.image,
            original: value.originalData,
            forKey: source.cacheKey,
            options: options,
            toDisk: !options.cacheMemoryOnly)
        {
            _ in
            coordinator.apply(.cachingImage) {
                let result = RetrieveImageResult(
                    image: value.image,
                    cacheType: .none,
                    source: source,
                    originalSource: context.originalSource
                )
                completionHandler?(.success(result))
            }
        }
        // 判断是否需要缓存原始图片
        if needToCacheOriginalImage {
            let originalCache = options.originalCache ?? targetCache
            originalCache.storeToDisk(
                value.originalData,
                forKey: source.cacheKey,
                processorIdentifier: DefaultImageProcessor.default.identifier,
                expiration: options.diskCacheExpiration)
            {
                _ in
                coordinator.apply(.cachingOriginalImage) {
                    let result = RetrieveImageResult(
                        image: value.image,
                        cacheType: .none,
                        source: source,
                        originalSource: context.originalSource
                    )
                    completionHandler?(.success(result))
                }
            }
        }

        coordinator.apply(.cacheInitiated) {
            let result = RetrieveImageResult(
                image: value.image,
                cacheType: .none,
                source: source,
                originalSource: context.originalSource
            )
            completionHandler?(.success(result))
        }

    case .failure(let error):
        completionHandler?(.failure(error))
    }
}
```
在option配置里，获取到 targetCache / originalCache（本质都是`ImageCache`），来执行图片的缓存；缓存图片之后，执行回调是通过生成缓存回调协作器（CacheCallbackCoordinator）完成在缓存初始化/缓存原始图片/缓存中/的情形下执行缓存回调逻辑。

### 传入的配置信息 - KingfisherParsedOptionsInfo
`KingfisherParsedOptionsInfo`实际是`KingfisherOptionsInfoItem`的结构体形式，便于生成控制Kingfisher中，加载网络图片，缓存等行为的配置信息。


### 缓存回调协作器 - CacheCallbackCoordinator
缓存回调协作器内部会更根据当前的缓存状态与操作来判定，是否触发`trigger`，`trigger`是空闭包，传入缓存图片成功后的逻辑进行处理。协作器apply方法内部的Switch语句使用了元组来做判断。
```swift
    func apply(_ action: Action, trigger: () -> Void) {
        switch (state, action) {
        case (.done, _):
            break

        // From .idle
        case (.idle, .cacheInitiated):
            if !shouldWaitForCache {
                state = .done
                trigger()
            }
        case (.idle, .cachingImage):
            if shouldCacheOriginal {
                state = .imageCached
            } else {
                state = .done
                trigger()
            }
        case (.idle, .cachingOriginalImage):
            state = .originalImageCached

        // From .imageCached
        case (.imageCached, .cachingOriginalImage):
            state = .done
            trigger()

        // From .originalImageCached
        case (.originalImageCached, .cachingImage):
            state = .done
            trigger()

        default:
            assertionFailure("This case should not happen in CacheCallbackCoordinator: \(state) - \(action)")
        }
    }
```

### 图片缓存 - ImageCache
ImageCache是一个单例，内部持有内存存储（MemoryStorage）和硬盘存储（DiskStorage）的实例，担任着真正的将图片缓存在内存/硬盘的职责。在初始化的时候，内部监听三个通知
- UIApplication.didReceiveMemoryWarningNotification - 清理内存缓存
- UIApplication.willTerminateNotification - 清理过期缓存
- UIApplication.didEnterBackgroundNotification - 后台清理过期缓存

#### ImageCache中的图片存储逻辑
当ImageCache调用了store()，其内部会将该图片存储在Memory中，然后再判断是否需要存储在硬盘中，如需要，就将该张图片序列化为Data进行存储。
```swift
    open func store(_ image: KFCrossPlatformImage,
                    original: Data? = nil,
                    forKey key: String,
                    options: KingfisherParsedOptionsInfo,
                    toDisk: Bool = true,
                    completionHandler: ((CacheStoreResult) -> Void)? = nil)
    {
        let identifier = options.processor.identifier
        let callbackQueue = options.callbackQueue
        
        let computedKey = key.computedKey(with: identifier)
        // 图片存储在内存
        memoryStorage.storeNoThrow(value: image, forKey: computedKey, expiration: options.memoryCacheExpiration)
        
        guard toDisk else {
            if let completionHandler = completionHandler {
                let result = CacheStoreResult(memoryCacheResult: .success(()), diskCacheResult: .success(()))
                callbackQueue.execute { completionHandler(result) }
            }
            return
        }
        
        ioQueue.async {
        	  //序列化图片，存储在硬盘
            let serializer = options.cacheSerializer
            if let data = serializer.data(with: image, original: original) {
                self.syncStoreToDisk(
                    data,
                    forKey: key,
                    processorIdentifier: identifier,
                    callbackQueue: callbackQueue,
                    expiration: options.diskCacheExpiration,
                    completionHandler: completionHandler)
            } else {
                guard let completionHandler = completionHandler else { return }
                
                let diskError = KingfisherError.cacheError(
                    reason: .cannotSerializeImage(image: image, original: original, serializer: serializer))
                let result = CacheStoreResult(
                    memoryCacheResult: .success(()),
                    diskCacheResult: .failure(diskError))
                callbackQueue.execute { completionHandler(result) }
            }
        }
    }

```

#### 存储至内存
将图片存在内存中，调用MemoryStorage实例的storeNoThrow()。存储过程是通过**NSCache**来完成的，storage是个NSCache的实例；同时，存储对应的Key，keys是个String类型的**Set**集合，在存储的时候，只需要保证key不重复即可。同时，StorageObject是用于封装需要存储的对象。

``` swift
	// MemoryStorage中的storeNoThrow()
	func storeNoThrow(
        value: T,
        forKey key: String,
        expiration: StorageExpiration? = nil)
    {
        lock.lock()
        defer { lock.unlock() }
        let expiration = expiration ?? config.expiration
        // The expiration indicates that already expired, no need to store.
        guard !expiration.isExpired else { return }
        
        let object = StorageObject(value, key: key, expiration: expiration)
        storage.setObject(object, forKey: key as NSString, cost: value.cacheCost)
        keys.insert(key)
    }
```

#### 存储至硬盘
将图片存在硬盘中，调用DiskStorage实例的store()。实则是将图片序列化之后的Data数据写入文件保存在硬盘中。
```swift
	 // DiskStorage中的store() 
    func store(
        value: T,
        forKey key: String,
        expiration: StorageExpiration? = nil) throws
    {
        let expiration = expiration ?? config.expiration
        // 图片数据过期，不需要存储
        guard !expiration.isExpired else { return }
        
        let data: Data
        do {
            data = try value.toData()
        } catch {
            throw KingfisherError.cacheError(reason: .cannotConvertToData(object: value, error: error))
        }
		  // 图片数据写入文件
        let fileURL = cacheFileURL(forKey: key)
        do {
            try data.write(to: fileURL)
        } catch {
            throw KingfisherError.cacheError(
                reason: .cannotCreateCacheFile(fileURL: fileURL, key: key, data: data, error: error)
            )
        }
		
        let now = Date()
        let attributes: [FileAttributeKey : Any] = [
            // 更新创建日期
            .creationDate: now.fileAttributeDate,
            // 更新修改日期
            .modificationDate: expiration.estimatedExpirationSinceNow.fileAttributeDate
        ]
        do {
            try config.fileManager.setAttributes(attributes, ofItemAtPath: fileURL.path)
        } catch {
            try? config.fileManager.removeItem(at: fileURL)
            throw KingfisherError.cacheError(
                reason: .cannotSetCacheFileAttribute(
                    filePath: fileURL.path,
                    attributes: attributes,
                    error: error
                )
            )
        }
    }

```

#### 移除缓存
依据缓存图片的条件，判定从内存 / 硬盘位置移除缓存。
```swift
    open func removeImage(forKey key: String,
                          processorIdentifier identifier: String = "",
                          fromMemory: Bool = true,
                          fromDisk: Bool = true,
                          callbackQueue: CallbackQueue = .untouch,
                          completionHandler: (() -> Void)? = nil)
    {
        let computedKey = key.computedKey(with: identifier)
		  // 从内存中移除
        if fromMemory {
            try? memoryStorage.remove(forKey: computedKey)
        }
        // 从硬盘中移除
        if fromDisk {
            ioQueue.async{
                try? self.diskStorage.remove(forKey: computedKey)
                if let completionHandler = completionHandler {
                    callbackQueue.execute { completionHandler() }
                }
            }
        } else {
            if let completionHandler = completionHandler {
                callbackQueue.execute { completionHandler() }
            }
        }
    }

```

**从内存中移除**，只需要通过key在对应的NSCache中移除对象。
```swift
    // 从内存中移除
	func remove(forKey key: String) throws {
        lock.lock()
        defer { lock.unlock() }
        storage.removeObject(forKey: key as NSString)
        keys.remove(key)
    }
```

**从硬盘中移除**，通过`FileManager`移除对应的类目。
```swift
	func removeFile(at url: URL) throws {
        try config.fileManager.removeItem(at: url)
    }

```

#### 移除过期缓存
移除过期的缓存，从内存与硬盘上移除对应过期的对象。

**从内存中移除过期图片**
```swift
func removeExpired() {
    lock.lock()
    defer { lock.unlock() }
    for key in keys {
        let nsKey = key as NSString
        guard let object = storage.object(forKey: nsKey) else {
            keys.remove(key)
            continue
        }
        if object.estimatedExpiration.isPast {
            storage.removeObject(forKey: nsKey)
            keys.remove(key)
        }
    }
}

```

**从硬盘中移除过期图片**
在ImageCache中，移除过期图片，通过调用内部持有DiskStorage实例的`removeExpiredValues()`完成。
```swift
open func cleanExpiredDiskCache(completion handler: (() -> Void)? = nil) {
    ioQueue.async {
        do {
            var removed: [URL] = []
            let removedExpired = try self.diskStorage.removeExpiredValues()
            removed.append(contentsOf: removedExpired)

            let removedSizeExceeded = try self.diskStorage.removeSizeExceededValues()
            removed.append(contentsOf: removedSizeExceeded)

            if !removed.isEmpty {
                DispatchQueue.main.async {
                    let cleanedHashes = removed.map { $0.lastPathComponent }
                    NotificationCenter.default.post(
                        name: .KingfisherDidCleanDiskCache,
                        object: self,
                        userInfo: [KingfisherDiskCacheCleanedHashKey: cleanedHashes])
                }
            }

            if let handler = handler {
                DispatchQueue.main.async { handler() }
            }
        } catch {}
    }
}

```

DiskStorage中的`removeExpiredValues()`；先查找到所有缓存文件的URL，再获取到过期的文件（去除文件夹）移除。
```swift
func removeExpiredValues(referenceDate: Date = Date()) throws -> [URL] {
    let propertyKeys: [URLResourceKey] = [
        .isDirectoryKey,
        .contentModificationDateKey
    ]

    let urls = try allFileURLs(for: propertyKeys)
    let keys = Set(propertyKeys)
    let expiredFiles = urls.filter { fileURL in
        do {
            let meta = try FileMeta(fileURL: fileURL, resourceKeys: keys)
            if meta.isDirectory {
                return false
            }
            return meta.expired(referenceDate: referenceDate)
        } catch {
            return true
        }
    }
    try expiredFiles.forEach { url in
        try removeFile(at: url)
    }
    return expiredFiles
}

```

```swift
// 获取所有文件的URL
func allFileURLs(for propertyKeys: [URLResourceKey]) throws -> [URL] {
    let fileManager = config.fileManager

    guard let directoryEnumerator = fileManager.enumerator(
        at: directoryURL, includingPropertiesForKeys: propertyKeys, options: .skipsHiddenFiles) else
    {
        throw KingfisherError.cacheError(reason: .fileEnumeratorCreationFailed(url: directoryURL))
    }

    guard let urls = directoryEnumerator.allObjects as? [URL] else {
        throw KingfisherError.cacheError(reason: .invalidFileEnumeratorContent(url: directoryURL))
    }
    return urls
}
```



## 网络细节
在前面的讲述中，提到在使用`kf.setImage()`设置网络图片的时候都是通过将URL装饰成一个的下载任务（DownloadTask）。
```swift
public struct DownloadTask {

    /// The `SessionDataTask` object bounded to this download task. Multiple `DownloadTask`s could refer
    /// to a same `sessionTask`. This is an optimization in Kingfisher to prevent multiple downloading task
    /// for the same URL resource at the same time.
    ///
    /// When you `cancel` a `DownloadTask`, this `SessionDataTask` and its cancel token will be pass through.
    /// You can use them to identify the cancelled task.
    public let sessionTask: SessionDataTask

    /// The cancel token which is used to cancel the task. This is only for identify the task when it is cancelled.
    /// To cancel a `DownloadTask`, use `cancel` instead.
    public let cancelToken: SessionDataTask.CancelToken

    /// Cancel this task if it is running. It will do nothing if this task is not running.
    ///
    /// - Note:
    /// In Kingfisher, there is an optimization to prevent starting another download task if the target URL is being
    /// downloading. However, even when internally no new session task created, a `DownloadTask` will be still created
    /// and returned when you call related methods, but it will share the session downloading task with a previous task.
    /// In this case, if multiple `DownloadTask`s share a single session download task, cancelling a `DownloadTask`
    /// does not affect other `DownloadTask`s.
    ///
    /// If you need to cancel all `DownloadTask`s of a url, use `ImageDownloader.cancel(url:)`. If you need to cancel
    /// all downloading tasks of an `ImageDownloader`, use `ImageDownloader.cancelAll()`.
    public func cancel() {
        sessionTask.cancel(token: cancelToken)
    }
}

```

很显然，DownloadTask中真正核心的内容在SessionDataTask，而且DownloadTask是个结构体，SessionDataTask是个类。

#### 会话数据任务 - SessionDataTask
SessionDataTask管理每个要进行下载的URLSessionDataTask；接收下载的数据；取消下载任务时，同时取消对应的回调。SessionDataTask的属性就已说明了这些。所以真正在图片下载任务执行过程中担任主角的是SessionDataTask。
```swift
	 /// 当前任务下载的数据.
    public private(set) var mutableData: Data

    /// 当前下载任务
    public let task: URLSessionDataTask
    /// 回调容器
    private var callbacksStore = [CancelToken: TaskCallback]()

    var callbacks: [SessionDataTask.TaskCallback] {
        lock.lock()
        defer { lock.unlock() }
        return Array(callbacksStore.values)
    }
	 /// 当前任务Token，也相当于身份ID。
    private var currentToken = 0
    /// 锁
    private let lock = NSLock()
	 /// 下载任务结束Delegate
    let onTaskDone = Delegate<(Result<(Data, URLResponse?), KingfisherError>, [TaskCallback]), Void>()
    /// 取消回调Delegate。
    let onCallbackCancelled = Delegate<(CancelToken, TaskCallback), Void>()

```

取消下载任务，通过当前下载任务的CancelToken，先移除这个下载任务的回调，再调用`cancel()`方法取消。
```swift
func cancel(token: CancelToken) {
    guard let callback = removeCallback(token) else {
        return
    }
    if callbacksStore.count == 0 {
        task.cancel()
    }
    onCallbackCancelled.call((token, callback))
}
```



#### 图片下载器 - ImageDownloader
图片的下载是通过URLSession完成的。创建一个URLSession，需要指定URLSessionConfiguration。由于在每个会话数据任务中，都不是固定某种会话类型。所以这里使用URLSessionConfiguration.ephemeral（非持久化类型的URLSessionConfiguration）。创建了图片下载器后，调用downloadImage()开启下载。在这个方法里，做了如下几个事情：
- 创建URLRequest
- 设置下载任务完成后的回调
- 将下载任务DownloadTask添加进SessionDelegate进行管理，并开启下载任务
- 下载任务结果处理

```swift
open func downloadImage(
        with url: URL,
        options: KingfisherParsedOptionsInfo,
        completionHandler: ((Result<ImageLoadingResult, KingfisherError>) -> Void)? = nil) -> DownloadTask?
    {
        // 创建默认的Request
        var request = URLRequest(url: url, cachePolicy: .reloadIgnoringLocalCacheData, timeoutInterval: downloadTimeout)
        request.httpShouldUsePipelining = requestsUsePipelining

        if let requestModifier = options.requestModifier {
            guard let r = requestModifier.modified(for: request) else {
                options.callbackQueue.execute {
                    completionHandler?(.failure(KingfisherError.requestError(reason: .emptyRequest)))
                }
                return nil
            }
            request = r
        }
        
        // url判空（有可能url传入了nil）
        guard let url = request.url, !url.absoluteString.isEmpty else {
            options.callbackQueue.execute {
                completionHandler?(.failure(KingfisherError.requestError(reason: .invalidURL(request: request))))
            }
            return nil
        }

        // 装饰onCompleted / completionHandler
        let onCompleted = completionHandler.map {
            block -> Delegate<Result<ImageLoadingResult, KingfisherError>, Void> in
            let delegate =  Delegate<Result<ImageLoadingResult, KingfisherError>, Void>()
            delegate.delegate(on: self) { (_, callback) in
                block(callback)
            }
            return delegate
        }

        let callback = SessionDataTask.TaskCallback(
            onCompleted: onCompleted,
            options: options
        )

		  // 准备下载
        let downloadTask: DownloadTask
        if let existingTask = sessionDelegate.task(for: url) {
            downloadTask = sessionDelegate.append(existingTask, url: url, callback: callback)
        } else {
            let sessionDataTask = session.dataTask(with: request)
            sessionDataTask.priority = options.downloadPriority
            downloadTask = sessionDelegate.add(sessionDataTask, url: url, callback: callback)
        }

        let sessionTask = downloadTask.sessionTask

        // 由sessionTask开始下载
        if !sessionTask.started {
            sessionTask.onTaskDone.delegate(on: self) { (self, done) in
                // 下载完成后回调
                // result: Result<(Data, URLResponse?)>, callbacks: [TaskCallback]
                let (result, callbacks) = done

                // 处理下载的数据前，Delegate处理逻辑
                do {
                    let value = try result.get()
                    self.delegate?.imageDownloader(
                        self,
                        didFinishDownloadingImageForURL: url,
                        with: value.1,
                        error: nil
                    )
                } catch {
                    self.delegate?.imageDownloader(
                        self,
                        didFinishDownloadingImageForURL: url,
                        with: nil,
                        error: error
                    )
                }

                switch result {
                // 下载成功，处理下载的数据转化为图片
                case .success(let (data, response)):
                    let processor = ImageDataProcessor(
                        data: data, callbacks: callbacks, processingQueue: options.processingQueue)
                    processor.onImageProcessed.delegate(on: self) { (self, result) in
                        // result: Result<Image>, callback: SessionDataTask.TaskCallback
                        let (result, callback) = result

                        if let image = try? result.get() {
                            self.delegate?.imageDownloader(self, didDownload: image, for: url, with: response)
                        }

                        let imageResult = result.map { ImageLoadingResult(image: $0, url: url, originalData: data) }
                        let queue = callback.options.callbackQueue
                        queue.execute { callback.onCompleted?.call(imageResult) }
                    }
                    processor.process()
				   // 下载失败，抛出异常
                case .failure(let error):
                    callbacks.forEach { callback in
                        let queue = callback.options.callbackQueue
                        queue.execute { callback.onCompleted?.call(.failure(error)) }
                    }
                }
            }
            delegate?.imageDownloader(self, willDownloadImageForURL: url, with: request)
            sessionTask.resume()
        }
        return downloadTask
    }
```

#### 图片下载器的任务管理者 - SessionDelegate
这是下载任务的直接管理者。产生的下载任务均是存放在SessionDelegate实例的tasks字典中，这个字典是以url为key，DownloadTask为value。所以，当重复调用kf.setImage()，只要是URL是同一个，任务不会被重复添加。只是会更新它的回调与`CancelToken`。简而言之，同一个URL的图片下载任务是相同的，不更新；但是，它的回调可能改变了，需要更新。

SessionDelegate中将添加单个Task到tasks字典。
```swift
func add(
    _ dataTask: URLSessionDataTask,
    url: URL,
    callback: SessionDataTask.TaskCallback) -> DownloadTask
{	  
	  // 加锁
    lock.lock()
    defer { lock.unlock() }

    // 创建新的SessionDataTask
    let task = SessionDataTask(task: dataTask)
    // 配置这个sessionDataTask的取消操作
    task.onCallbackCancelled.delegate(on: self) { [unowned task] (self, value) in
        let (token, callback) = value

        let error = KingfisherError.requestError(reason: .taskCancelled(task: task, token: token))
        task.onTaskDone.call((.failure(error), [callback]))
        // No other callbacks waiting, we can clear the task now.
        if !task.containsCallbacks {
            let dataTask = task.task
            self.remove(dataTask)
        }
    }
    // 回调添加进SessionDelegate
    let token = task.addCallback(callback)
    // 添加task到tasks字典中
    tasks[url] = task
    return DownloadTask(sessionTask: task, cancelToken: token)
}

```

当tasks字典中存在某个下载任务，后续再有相同的下载任务，只更新与下载任务搭配的回调。
```swift
func append(_ task: SessionDataTask, url: URL, callback: SessionDataTask.TaskCallback) -> DownloadTask {
	 // 将回调更新到回调存储器里并获取到最新的CancelToken
    let token = task.addCallback(callback)
    return DownloadTask(sessionTask: task, cancelToken: token)
}

```

将回调添加到回调存储容器里的逻辑。
```swift
func addCallback(_ callback: TaskCallback) -> CancelToken {
    lock.lock()
    defer { lock.unlock() }
    callbacksStore[currentToken] = callback
	 // 将Token+1以保持最新
    defer { currentToken += 1 }
    return currentToken
}

```

下载任务管理者中，移除某个下载任务；直接将tasks字典中，url对应的值设置为nil。
```swift
private func remove(_ task: URLSessionTask) {
    guard let url = task.originalRequest?.url else {
        return
    }
    lock.lock()
    defer {lock.unlock()}
    tasks[url] = nil
}
```

至此，可以看出图片的下载最终是通过URLSessionDataTask，来开启。复杂的是在于，给UIImageView，UIButton，NSButton等对象设置图片的时候，统一的通过一个方法`setImage()`，就将图片的下载，缓存，重复调用等逻辑组合在一起，最终完成加载网络图片的过程。

#### 避免循环引用 - Delegate<Input, Output>
在SessionDataTask中，可以看到两个属性`onTaskDone`与`onCallbackCancelled`。它们的类型都是`Delegate<Input, Output>`。
```swift
let onTaskDone = Delegate<(Result<(Data, URLResponse?), KingfisherError>, [TaskCallback]), Void>()
let onCallbackCancelled = Delegate<(CancelToken, TaskCallback), Void>()

```

这个类型，其实定义了一个中间类型；在初始化了实例之后，通过`delegate()`将外部方法中需要完成的操作 / 回调，传入它的属性block。
```swift
/// A delegate helper type to "shadow" weak `self`, to prevent creating an unexpected retain cycle.
class Delegate<Input, Output> {
    init() {}
    
    private var block: ((Input) -> Output?)?
    
    func delegate<T: AnyObject>(on target: T, block: ((T, Input) -> Output)?) {
        // The `target` is weak inside block, so you do not need to worry about it in the caller side.
        self.block = { [weak target] input in
            guard let target = target else { return nil }
            return block?(target, input)
        }
    }
    
    func call(_ input: Input) -> Output? {
        return block?(input)
    }
}

extension Delegate where Input == Void {
    // To make syntax better for `Void` input.
    func call() -> Output? {
        return call(())
    }
}

```

在SessionDataTask的下载任务完成后，会调用onTaskDone的call()。onTaskDone的`delegate()`是在ImageDownloader实例的downloadImage()中调用的，也就是将下载任务完成后的操作，在downloadImage()中传递给了onTaskDone。
```swift
// ImageDownloader的downloadImage()
// 开始下载
if !sessionTask.started {
    // 下载完成后回调操作传递给onTaskDone。
    sessionTask.onTaskDone.delegate(on: self) { (self, done) in
        // result: Result<(Data, URLResponse?)>, callbacks: [TaskCallback]
        let (result, callbacks) = done

        // 处理下载的数据前，Delegate处理逻辑
        do {
            let value = try result.get()
            self.delegate?.imageDownloader(
                self,
                didFinishDownloadingImageForURL: url,
                with: value.1,
                error: nil
            )
        } catch {
            self.delegate?.imageDownloader(
                self,
                didFinishDownloadingImageForURL: url,
                with: nil,
                error: error
            )
        }

        switch result {
        // 下载成功，处理下载的数据转化为图片
        case .success(let (data, response)):
            let processor = ImageDataProcessor(
                data: data, callbacks: callbacks, processingQueue: options.processingQueue)
            processor.onImageProcessed.delegate(on: self) { (self, result) in
                // result: Result<Image>, callback: SessionDataTask.TaskCallback
                let (result, callback) = result

                if let image = try? result.get() {
                    self.delegate?.imageDownloader(self, didDownload: image, for: url, with: response)
                }

                let imageResult = result.map { ImageLoadingResult(image: $0, url: url, originalData: data) }
                let queue = callback.options.callbackQueue
                queue.execute { callback.onCompleted?.call(imageResult) }
            }
            processor.process()
		   // 下载失败，抛出异常
        case .failure(let error):
            callbacks.forEach { callback in
                let queue = callback.options.callbackQueue
                queue.execute { callback.onCompleted?.call(.failure(error)) }
            }
        }
    }
    delegate?.imageDownloader(self, willDownloadImageForURL: url, with: request)
    sessionTask.resume()

}

```



## 断言与错误
断言是在不期望的情形出现时，停止程序运行。
- 在CacheCallbackCoordinator的trigger()触发判断里，使用`assertionFailure`断言。

```swift
assertionFailure("This case should not happen in CacheCallbackCoordinator: \(state) - \(action)")

```

- 用了enum类型的KingfisherError，做为框架中使用的错误类型；且KingfisherError作为错误类型名空间，内部详细定义了会出现的错误类型`RequestErrorReason`，`ResponseErrorReason`，`CacheErrorReason`，`ProcessorErrorReason`，`ImageSettingErrorReason`。同时与NSError桥接，遵循了`LocalizedError`与`CustomNSError`协议。


## 值得关注的点
1. 在Kingfisher框架中没有看到继承体系。即使是要实现给系统类UIImageView，UIButton，NSButton增加额外属性或方法的时候，是通过结构体封装这些类和关联对象间接实现的。
2. 在多重闭包中，避免循环引用，可以新增一个中间类；将要完成的操作传递给中间类的实例。


## 问题
在某个图片未被成功下载前，重复调用kf.setImage()，会有什么效果？

这种情况会在UITableViewCell / UICollectionViewCell中出现，当某个图片没有被缓存时，快速上下滑动。会出现下载任务（DownloadTask）会多次创建，但是与图片相关的`SessionDataTask`，同一个URL，只会创建1次。一旦图片下载成功并缓存了以后，再调用setImage()，就不会产生新的下载任务（DownloadTask），同时也不会创建`SessionDataTask`。



## 参考链接
Error与NSError的关系：https://www.jianshu.com/p/a36047852ccc

Kingfisher：https://github.com/onevcat/Kingfisher



