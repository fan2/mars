
## Thread

使用 std::thread 替换 Thread

	> std::thread 控制接口是否开放完备？  
	> 参考 weiyunUploadSDK 的 xpUploadThreadPool 实现。  
	> 失去透明的线程及消息队列控制管理，是否无法满足部分业务需求？    

### ThreadPool

某些业务(upload, download) 实现 threadpool？  

	> 参考 weiyunUploadSDK 的 xpUploadThreadPool 实现。  
	> 参考 [invxp/libthreadpool](https://github.com/invxp/libthreadpool) + [tonbit/coroutine](https://github.com/tonbit/coroutine)。  

## async

### AsyncInvoke

异步调用`AsyncInvoke*->PostMessage` 改为基于 future 的 [std::async](http://www.cnblogs.com/qicosmos/p/3534211.html) 实现。

	> 可参考 [whoshuu/cpr](https://github.com/whoshuu/cpr) api.h  
	> Message 直接改为 std::function 对象（lambda 表达式）  
	> std::async 会自动创建一个线程并返回一个std::future，可否 cancel？  

### Callback

异步回调改为基于 std::function 对象（lambda 表达式）。

	> 可参考 [whoshuu/cpr](https://github.com/whoshuu/cpr) api.h  
	> Message 直接改为 std::function 对象（lambda 表达式）  
	> 只需记录 map<request_id,respond_lambda> 回调上下文，无需透明的线程及消息队列？  

```CPP
typedef std::function<void(const weiyun::DiskFileDownloadRspItem& rsp, int errorcode)> CheckDownloadFileCallback;

void xpCloudDownloadBiz::checkDownloadFile(uint32_t type, const weiyun::FileItem& file, CheckDownloadFileCallback lambda);

template<typename Req, typename ReqMsgBody, typename Rsp, typename RspMsgBody>
void xpNetService::sendRecv(const std::string& cmd, Req* req, void(ReqMsgBody::*set_allocated_req)(Req *), Rsp*(RspMsgBody::*get_rsp)(), std::function<void(int errorcode, std::shared_ptr<Rsp> rsp)> lambda);
```

### Dispatch

**`std::async`** 会自动创建匿名线程执行异步调用，无法显式控制异步调用的任务调度。

**`std::thread`** 只提供了基础的线程封装，但并不支持线程切换，没有类似 iOS 的 `dispatch_queue` 接口。

如果想统一规划多线程及任务调度，还是需要为每个 std::thread 映射一个等效的消息队列 `map<tid, map<id, lambda>>`，必要时还需记录各个 lambda 的运行状态信息，以便精准控制管理 。  
也即还是要实现 runloop、业务相关的 task queue 及其对应的 task/callback scheduler。  
