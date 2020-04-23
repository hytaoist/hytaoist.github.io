---
layout: post
title: Alamofire源码导读
date: 2020-04-08 10:22
tags: [源码导读]
categories: [iOS]
---

本文基于Alamofire v5.0.5，地址：https://github.com/Alamofire/Alamofire

<!-- more-->
系统的API发起网络请求一般是通过获取一个URLSessionTask的实例，调用resume()方法即可发起一个网络请求；在iOS中构建网络应用，实际是使用的URLSessionTask的子类（URLSessionDataTask，URLSessionDownloadTask，URLSessionUploadTask，URLSessionStreamTask，URLSessionWebsocketTask）。

执行一次网络请求的过程：创建会话（**URLSession**）-> 生成请求任务（**URLSessionTask**）-> 开启请求（**resume()**）-> 响应了之后回调。Alamofire这个库在于其简洁的API，使用者不用再去自己创建会话及实现其代理，只需要传入请求的URL及响应的回调，就可以完成一次请求。

##  结构
Alamofire中定义的类图
![](https://hy-blog-pic.oss-cn-shanghai.aliyuncs.com/uPic/Alamofire的类图.png)
整个Alamofire的设计非常简洁，其中定义的类除了需要进行缓存的请求，会话是使用class类型，其余几乎都是采用协议与结构体实现。完整的将网络请求，URL校验，失败重试机制，请求缓存，错误机制，响应解析等功能模块融入其中。
![](https://hy-blog-pic.oss-cn-shanghai.aliyuncs.com/uPic/Alamofire_Process.png)



## 请求任务的发送
在Alamofire中，对系统的网络操作的类都有进行封装，会话也是如此 - Session。Session如同系统的UISession一样，被设计成单例。如果你使用AF.request来请求，便是使用了其单例。在一次会话中，可能会涉及到多次请求任务的场景。这些请求里可能某些请求会失败，需要对这些请求进行重试或者丢弃等操作；要完成这样的操作，必定会涉及到如何存储这些请求，如何区分这些请求成功与否，如何移除这些请求？

Session这个类中使用了Set<Request>这个集合来存储被激活的请求。
```swift
 /// Set of currently active `Request`s.
    var activeRequests: Set<Request> = []
```
在执行这些操作的时候，为避免主线程阻塞，都是采用GCD将任务分发到异步队列里。在Session中，也是持有了内部**rootQueue**。
```swift
	/// Root `DispatchQueue` for all internal callbacks and state update. **MUST** be a serial queue.
   public let rootQueue: DispatchQueue

```

响应了之后如何操作？如：

1.DataTask请求任务开启后，得到了响应，回调事情的触发
2.下载任务开启之后，下载进度随下载的数据更新等

在Alamofire中，采用了协议与通知机制来进行不同场景下事件的监听与调用。定义了**EventMonitor**协议与**AlamofireNotifications**。这些事件的监听又与每个会话息息相关，因此Session内部持有EventMonitor。
```swift
    /// `CompositeEventMonitor` used to compose Alamofire's `defaultEventMonitors` and any passed `EventMonitor`s.
    public let eventMonitor: CompositeEventMonitor
    /// `EventMonitor`s included in all instances. `[AlamofireNotifications()]` by default.
    public let defaultEventMonitors: [EventMonitor] = [AlamofireNotifications()]

```

至此，创建一个Session实例需要：

1.URLSession实例
2.SessionDelegate实例
3.rootQueue
4.eventMonitor

创建了Session实例之后，随即执行request方法。层层调用之后会执行`perform(request)`，这个方法会异步地将request加入到activeRequests中，并且执行设置操作`performSetupOperations`。这个设置操作做了哪些事情？
```swift
    func performSetupOperations(for request: Request, convertible: URLRequestConvertible) {
        let initialRequest: URLRequest
		// 验证request是否是合法的URLRequest	
        do {
            initialRequest = try convertible.asURLRequest()
            try initialRequest.validate()
        } catch {
            rootQueue.async { request.didFailToCreateURLRequest(with: error.asAFError(or: .createURLRequestFailed(error: error))) }
            return
        }
		// 完成创建初始的URLRequest
        rootQueue.async { request.didCreateInitialURLRequest(initialRequest) }

        guard !request.isCancelled else { return }

        guard let adapter = adapter(for: request) else {
            rootQueue.async { self.didCreateURLRequest(initialRequest, for: request) }
            return
        }
		// 请求适配设置
        adapter.adapt(initialRequest, for: self) { result in
            do {
                let adaptedRequest = try result.get()
                try adaptedRequest.validate()

                self.rootQueue.async {
                		// 完成了适配初始的URLRequest
                    request.didAdaptInitialRequest(initialRequest, to: adaptedRequest)
                    // 完成了创建URLRequest
                    self.didCreateURLRequest(adaptedRequest, for: request)
                }
            } catch {
                self.rootQueue.async { request.didFailToAdaptURLRequest(initialRequest, withError: .requestAdaptationFailed(error: error)) }
            }
        }
    }

```
这个设置过程中，比较重要的是**request.didCreateInitialURLRequest**与**didCreateURLRequest**。一个是完成创建初始的URLRequest实例，另一个是完成创建URLRequest实例。
```swift
// Request - didCreateInitialURLRequest
	func didCreateInitialURLRequest(_ request: URLRequest) {
		// 确保任务是在underlyingQueue中
        dispatchPrecondition(condition: .onQueue(underlyingQueue))
		// 将request添加到MutableState的requests数组中
        protectedMutableState.write { $0.requests.append(request) }
		// 事件监听者的初始化URLRequest方法
        eventMonitor?.request(self, didCreateInitialURLRequest: request)
    }
```

**MutableState**是一个存储各种状态（如：请求状态，任务，请求，重试次数，下载/重定向处理等）的结构体，这个结构体内的元素都会随着请求状态的改变而改变。请求任务是在Session中的**updateStatesForTask**里开启的，updateStatesForTask方法是在didCreateURLRequest中被调用，如果请求实例是resumed状态，就获取这个请求实例的请求任务，调用resume()开启请求。
```swift
	// Session - didCreateURLRequest
    func didCreateURLRequest(_ urlRequest: URLRequest, for request: Request) {
    
        request.didCreateURLRequest(urlRequest)
		// 判断请求是否已被取消
        guard !request.isCancelled else { return }
		// 从request中，获取任务
        let task = request.task(for: urlRequest, using: session)
        // 将任务存储在Session的requestTaskMap中
        requestTaskMap[request] = task
        request.didCreateTask(task)
		// 改变任务的状态，开启请求（resumed -> resume()）
        updateStatesForTask(task, request: request)
    }
    
    // Session - updateStatesForTask
    func updateStatesForTask(_ task: URLSessionTask, request: Request) {
        request.withState { state in
            switch state {
            case .initialized, .finished:
                // Do nothing.
                break
            case .resumed:
                task.resume()
                rootQueue.async { request.didResumeTask(task) }
            case .suspended:
                task.suspend()
                rootQueue.async { request.didSuspendTask(task) }
            case .cancelled:
                // Resume to ensure metrics are gathered.
                task.resume()
                task.cancel()
                rootQueue.async { request.didCancelTask(task) }
            }
        }
    }

```
至此，一个网络请求在Alamofire中，经历了会话 - 会话任务 - 事件监听 - 请求 / 任务存储等过程，发送出去了。

## 请求任务的响应
请求在发出去之后，什么时候会有响应 / 失败，时间是不确定的。因此在一个请求响应了之后，系统API提供给开发者调用的回调操作有两种。

1.通过代理（URLSessionDelegate）回调。

2.通过对应任务的completionHandler回调。如果未设置completionHandler，则会通过代理的方法回调。具体参见：[URLSession](https://developer.apple.com/documentation/foundation/urlsession)


### 会话代理
这个类（SessionDelegate）如其名，是会话的代理对象。用于处理请求发出后，响应会话的代理方法。主要涉及**URLSessionDataDelegate**，**URLSessionDownloadDelegate**，**URLSessionTaskDelegate**。

一个数据请求（DataRequest），响应的过程会分为两个步骤；在获得服务器响应之后，
**一**：会先调用URLSessionDataDelegate.didReceive代理方法。
```swift
	open func urlSession(_ session: URLSession, dataTask: URLSessionDataTask, didReceive data: Data) {
        eventMonitor?.urlSession(session, dataTask: dataTask, didReceive: data)
		// 确保能获取到本次请求的request
        guard let request = request(for: dataTask, as: DataRequest.self) else {
            assertionFailure("dataTask did not find DataRequest.")
            return
        }
		// 接受数据
        request.didReceive(data: data)
    }

```
didReceive的逻辑是将元数据拼接 / 添加在protectedData中，并计算与更新数据下载的进度。
```swift
    // Request - didReceive
    func didReceive(data: Data) {
        if self.data == nil {
            protectedData.directValue = data
        } else {
            protectedData.append(data)
        }

        updateDownloadProgress()
    }

```

**二**：响应完成了之后，会调用**URLSessionTaskDelegate**的didCompleteWithError代理方法。
```swift
    open func urlSession(_ session: URLSession, task: URLSessionTask, didCompleteWithError error: Error?) {
        eventMonitor?.urlSession(session, task: task, didCompleteWithError: error)
		// 从状态提供者里获取本次的请求
        let request = stateProvider?.request(for: task)
		// 调用完成请求任务
        stateProvider?.didCompleteTask(task) {
            request?.didCompleteTask(task, with: error.map { $0.asAFError(or: .sessionTaskFailed(error: $0)) })
        }
    }

```

在didCompleteWithError代理方法里会获取**状态提供者**，并调用它的didCompleteTask。状态提供者实际是遵循**SessionStateProvider**的Session实例。所以，这里也就是调用了Session.didCompleteTask。
```swift
	// Session - didCompleteTask
    func didCompleteTask(_ task: URLSessionTask, completion: @escaping () -> Void) {
        dispatchPrecondition(condition: .onQueue(rootQueue))
		// 请求任务完成了，需要清理requestTaskMap中存储的request与task。
        let didDisassociate = requestTaskMap.disassociateIfNecessaryAfterCompletingTask(task)
		
        if didDisassociate {
            completion()
        } else {
            waitingCompletions[task] = completion
        }
    }

```

**RequestTaskMap**中的操作完成后，会调用请求的**didCompleteTask**，继而调用**retryOrFinish**。**retryOrFinish**会将根据错误和代理方法来决定本次请求是需要重试还是结束。
```swift
	//Request - didCompleteTask
    func didCompleteTask(_ task: URLSessionTask, with error: AFError?) {
        dispatchPrecondition(condition: .onQueue(underlyingQueue))

        self.error = self.error ?? error

        protectedValidators.directValue.forEach { $0() }

        eventMonitor?.request(self, didCompleteTask: task, with: error)
		// 决定完成本次请求，还是重试本次请求
        retryOrFinish(error: self.error)
    }

```

如何决定结束本次请求或者重试本次请求？结束满足的条件：**整个请求过程中，没有出现错误** 或  **没有retrier**。

```swift
	// Request - retryOrFinish
    func retryOrFinish(error: AFError?) {
        dispatchPrecondition(condition: .onQueue(underlyingQueue))
		// 没有错误，结束本次请求
        guard let error = error, let delegate = delegate else { finish(); return }
		// 处理结果，决定是否重试
        delegate.retryResult(for: self, dueTo: error) { retryResult in
            switch retryResult {
            case .doNotRetry:
                self.finish()
            case let .doNotRetryWithError(retryError):
                self.finish(error: retryError.asAFError(orFailWith: "Received retryError was not already AFError"))
            case .retry, .retryWithDelay:
                delegate.retryRequest(self, withDelay: retryResult.delay)
            }
        }
    }

```

结束本次请求，处理请求的结果（包括：处理响应序列，执行回调，移除activeRequests中的Request）。
```swift
    /// 处理响应序列，并且回调.
    func processNextResponseSerializer() {
    	// 能够获取到回调处理序列，就直接在回调队列里异步执行处理序列；如果没有获取到回调处理序列，表示本次请求已经完成，调用序列处理完成的闭包
        guard let responseSerializer = nextResponseSerializer() else {
            // 暂存回调
            var completions: [() -> Void] = []

            protectedMutableState.write { mutableState in
                completions = mutableState.responseSerializerCompletions

               // 移除可变状态管理者里的响应回调
                mutableState.responseSerializers.removeAll()
                mutableState.responseSerializerCompletions.removeAll()

                if mutableState.state.canTransitionTo(.finished) {
                    mutableState.state = .finished
                }

                mutableState.responseSerializerProcessingFinished = true
                mutableState.isFinishing = false
            }

            completions.forEach { $0() }

            // 从activeRequests中移除目前完成的请求
            cleanup()

            return
        }

        serializationQueue.async { responseSerializer() }
    }

```
至此，一个请求在经历创建会话，发送请求，服务器响应，处理响应结果，执行回调等阶段后，退出了会话。

## 响应的元数据处理
元数据处理是一个将服务器返回的元数据处理为DataResponse类型的过程。在Alamofire中，无论当前请求是成功与否，执行fisnish方法的时候，都会去执行
processNextResponseSerializer，在这个方法中会通过当前请求实例的nextResponseSerializer方法从请求实例的状态管理器里获取responseSerializer闭包，进行异步调用处理，元数据的处理也包含在其中。

![](https://hy-blog-pic.oss-cn-shanghai.aliyuncs.com/uPic/AF_响应处理序列.png)

请求实例的responseSerializer闭包是如何被添加进状态管理器的responseSerializers数组中的？

### 添加响应处理闭包
无论当前请求是一个DataRequest还是一个DownloadRequest，处理响应元数据的闭包都是通过它们各自的拓展的responseString()来传入的。
```swift
// DataRequest - responseString
extension DataRequest {
 	
    @discardableResult
    public func responseString(queue: DispatchQueue = .main,
                               encoding: String.Encoding? = nil,
                               completionHandler: @escaping (AFDataResponse<String>) -> Void) -> Self {
        return response(queue: queue,
                        responseSerializer: StringResponseSerializer(encoding: encoding),
                        completionHandler: completionHandler)
    }
}

// DownloadRequest - responseString
extension DownloadRequest {
    @discardableResult
    public func responseString(queue: DispatchQueue = .main,
                               encoding: String.Encoding? = nil,
                               completionHandler: @escaping (AFDownloadResponse<String>) -> Void)
        -> Self {
        return response(queue: queue,
                        responseSerializer: StringResponseSerializer(encoding: encoding),
                        completionHandler: completionHandler)
    }
}

```

以DataRequest为例，被传入的completionHandler，是在response方法中被添加进请求的responseSerializer数组中。response方法内部调用**appendResponseSerializer**添加闭包。这个方法除了添加闭包外，还执行了请求实例的processNextResponseSerializer方法。
```swift
	//Request - appendResponseSerializer，参数是闭包
    func appendResponseSerializer(_ closure: @escaping () -> Void) {
        protectedMutableState.write { mutableState in
        	// 将闭包添加进状态管理器的responseSerializers数组
            mutableState.responseSerializers.append(closure)

            if mutableState.state == .finished {
                mutableState.state = .resumed
            }
			// 当状态管理器中响应序列处理完成，又调用调用下一个响应序列处理
            if mutableState.responseSerializerProcessingFinished {
                underlyingQueue.async { self.processNextResponseSerializer() }
            }

            if mutableState.state.canTransitionTo(.resumed) {
                underlyingQueue.async { if self.delegate?.startImmediately == true { self.resume() } }
            }
        }
    }

```

### 响应处理闭包的职责
上面已经讲述响应处理闭包的添加过程。这个被传入的闭包，在调用的时候，具体会做哪些事情？
```swift
public func response<Serializer: DataResponseSerializerProtocol>(queue: DispatchQueue = .main,
                                                                     responseSerializer: Serializer,
                                                                     completionHandler: @escaping (AFDataResponse<Serializer.SerializedObject>) -> Void)
        -> Self {
        appendResponseSerializer {
            // 获取开始的绝对时间
            let start = CFAbsoluteTimeGetCurrent()
            let result: AFResult<Serializer.SerializedObject> = Result {
            		// 将返回的Data类型的元数据转为String类型, 再封装成为Result类型
                try responseSerializer.serialize(request: self.request,
                                                 response: self.response,
                                                 data: self.data,
                                                 error: self.error)
            }.mapError { error in
                error.asAFError(or: .responseSerializationFailed(reason: .customSerializationFailed(error: error)))
            }
			  // 获取结果处理后的结束时间
            let end = CFAbsoluteTimeGetCurrent()
            
            self.underlyingQueue.async {
            		// 生成DataRespo实例
                let response = DataResponse(request: self.request,
                                            response: self.response,
                                            data: self.data,
                                            metrics: self.metrics,
                                            serializationDuration: end - start,
                                            result: result)

                self.eventMonitor?.request(self, didParseResponse: response)

                guard let serializerError = result.failure, let delegate = self.delegate else {
                    self.responseSerializerDidComplete { queue.async { completionHandler(response) } }
                    return
                }
				   // 询问代理，调用retryResult。
                delegate.retryResult(for: self, dueTo: serializerError) { retryResult in
                    var didComplete: (() -> Void)?

                    defer {
                        if let didComplete = didComplete {
                            self.responseSerializerDidComplete { queue.async { didComplete() } }
                        }
                    }

                    switch retryResult {
                    case .doNotRetry:
                        didComplete = { completionHandler(response) }

                    case let .doNotRetryWithError(retryError):
                        let result: AFResult<Serializer.SerializedObject> = .failure(retryError.asAFError(orFailWith: "Received retryError was not already AFError"))

                        let response = DataResponse(request: self.request,
                                                    response: self.response,
                                                    data: self.data,
                                                    metrics: self.metrics,
                                                    serializationDuration: end - start,
                                                    result: result)

                        didComplete = { completionHandler(response) }

                    case .retry, .retryWithDelay:
                        delegate.retryRequest(self, withDelay: retryResult.delay)
                    }
                }
            }
        }

        return self
    }

```
由此可以看到，响应处理闭包会将服务器返回的Data类型的数据转为String类型，与**请求实例**，**原响应数据**，**整个操作时间**，及**Result实例**组装一个DataRespons的实例；最后询问代理执行retryResult，来判定结束本次请求还是需要重试本次请求。

## 线程安全
Alamofire没有直接使用NSLock来保证线程安全。而是使用了**os_unfair_lock_t**（一种低级的锁，https://developer.apple.com/documentation/os/synchronization?language=objc ）。然后对锁的操作进行封装（详细查看**UnfairLock**这个类）。UnfairLock对外暴露两个around操作，一个有返回类型，一个无返回类型。
```swift
final class UnfairLock {
    private let unfairLock: os_unfair_lock_t

    init() {
        unfairLock = .allocate(capacity: 1)
        unfairLock.initialize(to: os_unfair_lock())
    }

    deinit {
        unfairLock.deinitialize(count: 1)
        unfairLock.deallocate()
    }

    private func lock() {
        os_unfair_lock_lock(unfairLock)
    }

    private func unlock() {
        os_unfair_lock_unlock(unfairLock)
    }

    /// Executes a closure returning a value while acquiring the lock.
    ///
    /// - Parameter closure: The closure to run.
    ///
    /// - Returns:           The value the closure generated.
    func around<T>(_ closure: () -> T) -> T {
        lock(); defer { unlock() }
        return closure()
    }

    /// Execute a closure while acquiring the lock.
    ///
    /// - Parameter closure: The closure to run.
    func around(_ closure: () -> Void) {
        lock(); defer { unlock() }
        return closure()
    }
}

```

同时，作者设计了Protector类；使用Protector来对外暴露read / write方法来进行线程安全的数据访问。
```swift
/// A thread-safe wrapper around a value.
final class Protector<T> {
    private let lock = UnfairLock()
    private var value: T

    init(_ value: T) {
        self.value = value
    }

    /// The contained value. Unsafe for anything more than direct read or write.
    var directValue: T {
        get { return lock.around { value } }
        set { lock.around { value = newValue } }
    }

    /// Synchronously read or transform the contained value.
    ///
    /// - Parameter closure: The closure to execute.
    ///
    /// - Returns:           The return value of the closure passed.
    func read<U>(_ closure: (T) -> U) -> U {
        return lock.around { closure(self.value) }
    }

    /// Synchronously modify the protected value.
    ///
    /// - Parameter closure: The closure to execute.
    ///
    /// - Returns:           The modified value.
    @discardableResult
    func write<U>(_ closure: (inout T) -> U) -> U {
        return lock.around { closure(&self.value) }
    }
}

```


## 结束语
本文分析了Alamofire中，执行一个请求会经历的过程；并讲解了其中大部分细节的实现。总体来说，Alamofire的逻辑很值得读一读。

## 链接
1. 自旋锁与互斥锁 https://www.zhihu.com/question/66733477
2. Swift下标语法 https://docs.swift.org/swift-book/LanguageGuide/Subscripts.html
