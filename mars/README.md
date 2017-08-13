
# mars
mars common library walkthrough

## comm
### thread

```cpp
// comm/thread/thread.h
#ifdef _WIN32
#include "../windows/thread/thread.h"
#else
#include "../unix/thread/thread.h"
#endif
```

声明：comm/unix/thread/thread.h  
实现：boost/libs/thread/src/pthread/thread.cpp  
设施：Condition，Mutex，BaseScopedLock(基于Mutex或SpinLock)，

**线程使用参考**：

- GetProxyInfo；DNS::GetHostByName，__Coro_Poll；  
- Alarm、ShortLink/ShortLinkTaskManager、TcpClient/TcpServer、UdpClient/UdpServer；  

#### class ThreadUtil
1. `yield`：sched_yield()  
2. `sleep`/`usleep`  
3. `currentthreadid`：pthread_self() // mach_thread_self()?   
4. `isruning`：pthread_kill(_id, 0)  
5. `join`：pthread_join(_id, 0);  

#### class Thread
```obj-c
class Thread {
private:
    class RunnableReference;

private:
    RunnableReference*  runable_ref_;
    pthread_attr_t attr_;
    bool outside_join_;
};
```

#### class RunnableReference
核心封装了 `Runnable* target` 及所属线程状态。
Reference 支持引用计数 `AddRef`/`RemoveRef`，当 `RemoveRef` 至 `count--==0` 时，执行 `delete this` 自杀。相当于实现了 Runnable 类型的智能指针。

```CPP
class RunnableReference {

public:
    Runnable* target;
    thread_tid tid;
    int count; // 引用计数
    bool isinthread;
    bool isjoined;
    bool isended;
}
```

`Runnable` 抽象了可执行的对象。  
`RunnableFunctor:Runnable` 封装了类 std::function 函数对象。  
`run() { func_(); }` 无参数，说明传入的模板实参应为已绑定的函数对象，且将调用抽象为 `operator()`，隐藏了函数原型及带参调用细节。

```CPP
struct Runnable {
    virtual ~Runnable() {}
    virtual void run() = 0;
};

namespace detail {

    template <class T>
    class RunnableFunctor : public Runnable {
    public:
        RunnableFunctor(const T& f) : func_(f) {}
        virtual void run() { func_(); }
    private:
        T func_;
    };

}
```

##### init & cleanup
```obj
// 初始化 RunnableReference
static void init(void* arg) 
{
    // ((RunnableReference*)arg).isinthread = true 表示已经在当前线程调度（entry: start_routine）。

    // 如果 killsig<0 || killsig>32，则调用 pthread_kill 发送异常sig信号给当前线程。

}

// 清理 RunnableReference
static void cleanup(void* arg) {
    // 复位 RunnableReference 的状态
    runableref->isinthread = false;
    killsig = 0;
    runableref->isended = true;
    
    // 坚持引用计数（当refCount==0时自杀）
    runableref->RemoveRef(lock);
}

```

##### start
Thread 提供了 2 个构造接口，提供了 3 个启动接口。  
构造好 Thread 实例后，即可调用 `start*` 创建（启动）线程。

1. 创建一个线程

```CPP
start
{
    if (_newone) *_newone = false;
    pthread_create(&runable_ref_->tid, &attr_, start_routine, runable_ref_);
    if (_newone) *_newone = true;
}

```

2. 创建一个线程，延迟运行

```CPP
int start_after(long after) 
{
    runable_ref_->aftertime = after;

    int ret =  pthread_create(&runable_ref_->tid, &attr_, start_routine_after, runable_ref_);

}

void cancel_after() {
    ScopedSpinLock lock(runable_ref_->splock);

    if (!isruning()) return;

    runable_ref_->iscanceldelaystart = true;
    runable_ref_->condtime.notifyAll(true);
}
```

3. 创建一个线程，延迟运行（有超时限制）

```CPP
int start_periodic(long after, long periodic) { // ms

    runable_ref_->aftertime = after;
    runable_ref_->periodictime = periodic;

    int ret = pthread_create(&runable_ref_->tid, &attr_, start_routine_periodic, runable_ref_);

}

void cancel_periodic() {
    ScopedSpinLock lock(runable_ref_->splock);

    if (!isruning()) return;

    runable_ref_->iscanceldelaystart = true;
    runable_ref_->condtime.notifyAll(true);
}
```

##### start_routine
`start*` -> `pthread_create` 的入口函数为 `start_routine*`，传入参数为 `RunnableReference* Thread::runable_ref_`。

1. 立即执行 `Runnable::run()`

```CPP
static void* start_routine(void* arg) {
    init(arg);
    volatile RunnableReference* runableref = static_cast<RunnableReference*>(arg);
    pthread_cleanup_push(&cleanup, arg);
    runableref->target->run();
    pthread_cleanup_pop(1);
    return 0;
}
```

2. 等待指定时间 aftertime 后才执行 `Runnable::run()`（如果尚未 cancel_after）

```CPP
static void* start_routine_after(void* arg) {
    init(arg);
    volatile RunnableReference* runableref = static_cast<RunnableReference*>(arg);
    pthread_cleanup_push(&cleanup, arg);

    if (!runableref->iscanceldelaystart) {
        (const_cast<RunnableReference*>(runableref))->condtime.wait(runableref->aftertime);

        if (!runableref->iscanceldelaystart)
            runableref->target->run();
    }

    pthread_cleanup_pop(1);
    return 0;
}
```

3. periodic 在 after 的基础上，引入 while loop，调用 cancel_periodic 退出循环。

```CPP
static void* start_routine_periodic(void* arg) {
    init(arg);
    volatile RunnableReference* runableref = static_cast<RunnableReference*>(arg);
    pthread_cleanup_push(&cleanup, arg);

    if (!runableref->iscanceldelaystart) {
        (const_cast<RunnableReference*>(runableref))->condtime.wait(runableref->aftertime);

        while (!runableref->iscanceldelaystart) {
            runableref->target->run();

            if (!runableref->iscanceldelaystart)
                (const_cast<RunnableReference*>(runableref))->condtime.wait(runableref->periodictime);
        }
    }

    pthread_cleanup_pop(1);
    return 0;
}
```

##### pthread_cleanup
Thread 在构造函数中动态创建 `runable_ref_`：

```CPP
runable_ref_ = new RunnableReference(NULL);
```

借助 pthread cleanup 机制，在函数体（RunnableFunctor）执行之前将清理函数（及其参数）注册 push 到清理栈中。在函数体 return 退出之前 pop 弹出执行清理程序。

这样，保证在线程函数 **`start_routine*`**（RunnableReference(Runnable)::run 或其 while loop） 结束 return 之前调用 `cleanup -> runableref->RemoveRef` 解除引用（销毁释放） `runable_ref_` 对象。

### MessageQueue

- `message_queue.h` 
- `message_queue.cc`  

#### MessageQueueCreater
```CPP
class MessageQueueCreater {
public:

    MessageQueue_t GetMessageQueue();
    MessageQueue_t CreateMessageQueue();

private:
    Thread                              thread_;
    Mutex                               messagequeue_mutex_;
    MessageQueue_t                      messagequeue_id_;
    boost::shared_ptr<RunloopCond>      breaker_;
};
```

##### MessageQueueCreater()
thread_ 构造传入 MessageQueueCreater 的成员函数 `__ThreadRunloop()` 作为模板实参 T& op，进而创建 RunnableReference(Runnable* _target)。

```
thread_(boost::bind(&MessageQueueCreater::__ThreadRunloop, this), _msg_queue_name)
```

```CPP
MessageQueueCreater::MessageQueueCreater(bool _iscreate, const char* _msg_queue_name)
    : MessageQueueCreater(boost::shared_ptr<RunloopCond>(), _iscreate, _msg_queue_name)
{}
    
MessageQueueCreater::MessageQueueCreater(boost::shared_ptr<RunloopCond> _breaker, bool _iscreate, const char* _msg_queue_name)
    : thread_(boost::bind(&MessageQueueCreater::__ThreadRunloop, this), _msg_queue_name)
	, messagequeue_id_(KInvalidQueueID), breaker_(_breaker) {
	if (_iscreate)
		CreateMessageQueue();
}

```

##### CreateMessageQueue
CreateMessageQueue 并没有创建（Create）消息队列（MessageQueue），而是调用 thread_.start 创建启动线程——Thread。

```CPP
MessageQueue_t MessageQueueCreater::GetMessageQueue() {
    return messagequeue_id_;
}

MessageQueue_t MessageQueueCreater::CreateMessageQueue() {
    ScopedLock lock(messagequeue_mutex_);

    if (thread_.isruning()) return messagequeue_id_;

    if (0 != thread_.start()) { return KInvalidQueueID;}
    messagequeue_id_ = __CreateMessageQueueInfo(breaker_, thread_.tid());
    xinfo2(TSF"create messageqeue id:%_", messagequeue_id_);
    
    return messagequeue_id_;
}
```

再来看看静态函数 *`__CreateMessageQueueInfo`* ，自解释为创建消息队列信息。

```CPP
static MessageQueue_t __CreateMessageQueueInfo(boost::shared_ptr<RunloopCond>& _breaker, thread_tid _tid) {
    ScopedLock lock(sg_messagequeue_map_mutex);

    MessageQueue_t id = (MessageQueue_t)_tid;

    if (sg_messagequeue_map.end() == sg_messagequeue_map.find(id)) {
        MessageQueueContent& content = sg_messagequeue_map[id];
        HandlerWrapper* handler = new HandlerWrapper(&__AsyncInvokeHandler, false, id, __MakeSeq());
        content.lst_handler.push_back(handler);
        content.invoke_reg = handler->reg;
        if (_breaker)
            content.breaker = _breaker;
        else
            content.breaker = boost::make_shared<Cond>();
    }

    return id;
}

```

由 `MessageQueue_t id = (MessageQueue_t)_tid;` 可知，消息队列  id 实际上就是线程 _tid。
这里为每个 Thread 创建一个 MessageQueueContent，并记录到以 id 为键的全局字典  sg_messagequeue_map 中。

> **MessageQueue** ≈ Thread + MessageQueueContent

其次，创建默认的 HandlerWrapper(`MessageHandler=__AsyncInvokeHandler`) 插入链表 lst_handler 中。

##### MessageQueueContent
**`MessageQueueContent`** 为具体的消息队列内容，主要包含3个队列。  

```CPP
struct MessageQueueContent {
    MessageQueueContent(): breakflag(false) {}

    MessageHandler_t invoke_reg;
    bool breakflag;
    boost::shared_ptr<RunloopCond> breaker;
    std::list<MessageWrapper*> lst_message;
    std::list<HandlerWrapper*> lst_handler;
    
    std::list<RunLoopInfo> lst_runloop_info;
    
};

```

- **lst_message**：AsyncInvoke-PostMessage，异步调用列表。  
- **lst_handler**：Message Handler List，消息回调列表。  

***sg_messagequeue_map*** 为全局的 `map<MessageQueue_t, MessageQueueContent>`，由 tid 映射线程消息队列。

```CPP
#define sg_messagequeue_map messagequeue_map()

static std::map<MessageQueue_t, MessageQueueContent>& messagequeue_map() {
    static std::map<MessageQueue_t, MessageQueueContent>* mq_map = new std::map<MessageQueue_t, MessageQueueContent>;
    return *mq_map;
}
```

#### MessagePost & MessageHandler

- struct MessageHandler_t（封装 MessageQueue_t，`Handler2Queue` 取 queue）  
- struct MessagePost_t（记录 MessageHandler_t 和 seq，`Post2Handler` 取 reg）  
- struct MessageTitle_t（message ID？）  
- struct Message（body1=AsyncInvokeFunction）   
- struct MessageTiming  

```obj-c
typedef boost::function<void (const MessagePost_t& _id, Message& _message)> MessageHandler;
```

##### AsyncInvoke-PostMessage
`AsyncInvoke*` 函数调用 `PostMessage` 将异步调用 `Message(_title, _func)` 插入 `sg_messagequeue_map[MessageQueue_t]=MessageQueueContent` 的消息列表 `lst_message` 中。

```CPP
MessagePost_t PostMessage(const MessageHandler_t& _handlerid, const Message& _message, const MessageTiming& _timing) {
    ScopedLock lock(sg_messagequeue_map_mutex);
    const MessageQueue_t& id = _handlerid.queue;

    std::map<MessageQueue_t, MessageQueueContent>::iterator pos = sg_messagequeue_map.find(id);

    MessageQueueContent& content = pos->second;

    MessageWrapper* messagewrapper = new MessageWrapper(_handlerid, _message, _timing, __MakeSeq());

    content.lst_message.push_back(messagewrapper);
    content.breaker->Notify(lock);
    return messagewrapper->postid;
}
```

###### InstallAsyncHandler-InstallMessageHandler
```CPP
//AsyncInvoke
MessageHandler_t InstallAsyncHandler(const MessageQueue_t& id);
```

`InstallMessageHandler` 函数调用 `InstallMessageHandler` 将异步回调  `__AsyncInvokeHandler` 插入 `sg_messagequeue_map[MessageQueue_t]=MessageQueueContent` 的消息列表 `lst_handler` 中。

```CPP
MessageHandler_t InstallAsyncHandler(const MessageQueue_t& id) {
    ASSERT(0 != id);
    return InstallMessageHandler(__AsyncInvokeHandler, false, id);
}

MessageHandler_t InstallMessageHandler(const MessageHandler& _handler, bool _recvbroadcast, const MessageQueue_t& _messagequeueid) {

    ScopedLock lock(sg_messagequeue_map_mutex);
    const MessageQueue_t& id = _messagequeueid;

    std::map<MessageQueue_t, MessageQueueContent>::iterator pos = sg_messagequeue_map.find(id);

    HandlerWrapper* handler = new HandlerWrapper(_handler, _recvbroadcast, _messagequeueid, __MakeSeq());
    pos->second.lst_handler.push_back(handler);
    return handler->reg;
}
```

静态函数 *`__AsyncInvokeHandler`* 仅仅是个壳，执行 `_message.body1` 对应的 AsyncInvokeFunction。

```
static void __AsyncInvokeHandler(const MessagePost_t& _id, Message& _message) {
    (*boost::any_cast<boost::shared_ptr<AsyncInvokeFunction> >(_message.body1))();
}
```

##### Async Invoke Block

```CPP
// 获取 MessageQueue::ScopeRegister 的 MessageHandler_t
#define AYNC_HANDLER asyncreg_.Get()
```

```CPP
#define ASYNC_BLOCK_START MessageQueue::AsyncInvoke([=] () {
#define ASYNC_BLOCK_END }, AYNC_HANDLER);

#define SYNC2ASYNC_FUNC(func) \
if (MessageQueue::CurrentThreadMessageQueue() != MessageQueue::Handler2Queue(AYNC_HANDLER)) \
{ MessageQueue::AsyncInvoke(func, AYNC_HANDLER); return; } \

#define RETURN_SYNC2ASYNC_FUNC(func, ret) \
if (MessageQueue::CurrentThreadMessageQueue() != MessageQueue::Handler2Queue(AYNC_HANDLER)) \
{ MessageQueue::AsyncInvoke(func, AYNC_HANDLER); return ret; } \

#define RETURN_SYNC2ASYNC_FUNC_TITLE(func, title, ret) \
if (MessageQueue::CurrentThreadMessageQueue() != MessageQueue::Handler2Queue(AYNC_HANDLER)) \
{ MessageQueue::AsyncInvoke(func, title, AYNC_HANDLER); return ret; } \

#define RETURN_WAIT_SYNC2ASYNC_FUNC(func, ret) \
if (MessageQueue::CurrentThreadMessageQueue() != MessageQueue::Handler2Queue(AYNC_HANDLER)) \
{ MessageQueue::MessagePost_t postId = MessageQueue::AsyncInvoke(func, AYNC_HANDLER);MessageQueue::WaitMessage(postId); return ret; } \

#define WAIT_SYNC2ASYNC_FUNC(func) \
\
if (MessageQueue::CurrentThreadMessageQueue() != MessageQueue::Handler2Queue(AYNC_HANDLER)) \
{\
return MessageQueue::WaitInvoke(func, AYNC_HANDLER);\
}

```

#### RunLoop
##### class RunLoop
```CPP
class RunLoop {

private:
    boost::function<bool ()> breaker_func_;
    boost::function<void ()> duty_func_;
};

```

MessageQueueCreater 构造函数指定 thread_ 的入口参数为 `::__ThreadRunloop`。

```CPP
void MessageQueueCreater::__ThreadRunloop() {
    ScopedLock lock(messagequeue_mutex_);
    lock.unlock();
    
    RunLoop().Run();
    
}
```

在 **`RunLoop::Run()`** 中轮询 lst_message 和 lst_handler。

##### class RunloopCond

### socket
#### unix_socket

#### ipstack
local_ipstack.h / local_ipstack.cc

#### ComplexConnect

#### socketselect
```CPP
// comm/socket/socketselect.h
#ifdef _WIN32
#include "../windows/socketselect/socketselect2.h"
#else
#include "../unix/socket/socketselect.h"
#endif

```

#### socketpoll
```CPP
// comm/socket/socketpoll.h
#ifdef _WIN32
#include "../windows/SocketSelect/socketselect2.h"
#else
#include "../unix/socket/socketpoll.h"
#endif

```

#### TcpClient/TcpServer
- event_：MTcpEvent  
- observer_：MTcpServer  

```CPP
TcpClient::TcpClient(const char* _ip, uint16_t _port, MTcpEvent& _event, int _timeout)
    : ip_(strdup(_ip)) , port_(_port) , event_(_event)
    , socket_(INVALID_SOCKET) , have_read_data_(false) , will_disconnect_(false) , writedbufid_(0)
    , thread_(boost::bind(&TcpClient::__RunThread, this))
    , timeout_(_timeout), status_(kTcpInit) {
    if (!pipe_.IsCreateSuc()) status_ = kTcpInitErr;
}
```

```CPP
TcpServer::TcpServer(const char* _ip, uint16_t _port, MTcpServer& _observer, int _backlog)
    : observer_(_observer)
    , thread_(boost::bind(&TcpServer::__ListenThread, this))
    , listen_sock_(INVALID_SOCKET), backlog_(_backlog) {
    memset(&bind_addr_, 0, sizeof(bind_addr_));
    bind_addr_ = *(struct sockaddr_in*)(&socket_address(_ip, _port).address());
}
```

#### UdpClient/UdpServer
- event_：IAsyncUdpClientEvent  
- event_：IAsyncUdpServerEvent  

```CPP
UdpClient::UdpClient(const std::string& _ip, int _port, IAsyncUdpClientEvent* _event)
:fd_socket_(INVALID_SOCKET)
, event_(_event)
, selector_(breaker_, true)
{
    thread_ = new Thread(boost::bind(&UdpClient::__RunLoop, this));
    
    __InitSocket(_ip, _port);
}
```

```CPP
UdpServer::UdpServer(int _port, IAsyncUdpServerEvent* _event)
    : fd_socket_(INVALID_SOCKET)
    , event_(_event)
    , selector_(breaker_, true) {
    thread_ = new Thread(boost::bind(&UdpServer::__RunLoop, this));

    __InitSocket(_port);
    thread_->start();
}
```

### coroutine(coro_socket)

- class Coroutine/class Wrapper：Start/Join/Notify/Wait  
- class SocketPoll：Poll  
- class SocketSelect：Select  
- class Multiplexing：Add/Run  

> SocketSelect 包含成员 `SocketPoll socket_poll_`;  
> Multiplexing 包含成员 `::SocketPoll poll_`;  

macOS/iOS 不支持 epoll，可支持 select、poll 和 kqueue 等 I/O 模型。  

- 网络诊断模块 `sdt/dnsquery.cc` 中的 RecvWithinTime() 使用了 select 模型。  
- macOS 版 `comm/unix/socket/socketselect.cc` 中实现的 SocketSelect 基于 ***kqueue/kevent*** 模型完成对 kevent(ident为fd) 的轮询处理。  
- iOS 版 `coroutine/coro_socket.h` 中定义的 SocketSelect 和 SocketPoll 基于 ***poll*** 模型完成对 pollfd 的 PollEvent 轮询处理。  

```CPP
int SocketSelect::Select(int _msec) {
    __Coro_Poll(_msec, Poll());
    return Ret();
}
    
int SocketPoll::Poll(int _msec) {
    __Coro_Poll(_msec, *this);
    return Ret();
}
```

`SocketSelect::Select` 和 `SocketPoll::Poll` 均调用 `__Coro_Poll` 静态函数。

**__Coro_Poll** callstack:  

> static void __Coro_Poll(int _msec, ::SocketPoll& _socket_poll)  
>> void Multiplexing::Run(int32_t _timeout); // in Thread  
>>> int SocketPoll::Poll(int _msec);  
>>>> int poll (struct pollfd *, nfds_t, int);  

static Multiplexing coro select in static Thread:

```CPP
static void __Coro_Poll(int _msec, ::SocketPoll& _socket_poll) {
    
    ::SocketBreaker breaker;
    ::SocketPoll selector(breaker);
    selector.Consign(_socket_poll);
    int ret = selector.Poll(0);
    
    if (0 != ret) {
        selector.ConsignReport(_socket_poll, 1);
        return;
    }
    
    boost::shared_ptr<mq::RunloopCond> cond = mq::RunloopCond::CurrentCond();
    TaskInfo task(coroutine::RunningCoroutine(), _socket_poll);
    if (0 <= _msec) { task.abs_timeout_.gettickcount() += _msec;};
    
    //messagequeue for coro select
    if (cond && cond->type() == boost::typeindex::type_id<coroutine::RunloopCond>()) {
        static_cast<coroutine::RunloopCond*>(cond.get())->Add(task);
        coroutine::Yield();
        return;
    }
    
    //new thread for coro select
    static Multiplexing s_multiplexing;
    static Thread s_thread([&](){
        while (true) { 				// loop
            s_multiplexing.Run(10*60*1000);	// ②
        }
    }, XLOGGER_TAG"::coro_multiplexing");
    s_thread.start();				// ①
    s_multiplexing.Add(task);
    coroutine::Yield();
    return;
}

```

#### Multiplexing
在 __Coro_Poll 的 s_thread 中 `while (true) { s_multiplexing.Run{ SocketPoll::Poll } }` 执行 I/O 轮询。

```CPP
class Multiplexing {
public:
    Multiplexing()
    : poll_(breaker_, true)
    {}
    
public:
    void Add(TaskInfo& _task) {
        ScopedLock lock(mutex_);
        tasks_.push_back(&_task);
        poll_.Breaker().Break();
    }
    
    ::SocketBreaker& Breaeker() {
        return poll_.Breaker();
    }
    
    void Run(int32_t _timeout) {
        
        int32_t timeout = _timeout;
        
        __Before(timeout);
        poll_.Poll(timeout);
        __After();		// coroutine::Resume
    }

private:
    Mutex                                   mutex_;
    ::SocketBreaker                         breaker_;
    ::SocketPoll                            poll_;
    std::list<TaskInfo*>                    tasks_;
};

```

### http
依赖 class AutoBuffer

- class RequestLine  
- class StatusLine  
- class HeaderFields  
- class BufferBodyProvider : public IBlockBodyProvider  
- class IStreamBodyProvider  
- class Builder  
- class MemoryBodyReceiver : public BodyReceiver  
- class Parser  

## stn
### ShortLink

```CPP
ShortLink::ShortLink(MessageQueue::MessageQueue_t _messagequeueid, NetSource& _netsource, const Task& _task, bool _use_proxy)
    : asyncreg_(MessageQueue::InstallAsyncHandler(_messagequeueid))
	, net_source_(_netsource)
	, task_(_task)
	, thread_(boost::bind(&ShortLink::__Run, this), XLOGGER_TAG "::shortlink")
    , use_proxy_(_use_proxy)
    , tracker_(shortlink_tracker::Create())
    {
    xinfo2(TSF"%_, handler:(%_,%_)",XTHIS, asyncreg_.Get().queue, asyncreg_.Get().seq);
    xassert2(breaker_.IsCreateSuc(), "Create Breaker Fail!!!");
}

```

#### ShortLinkTaskManager

### LongLink

#### LongLinkTaskManager

#### LongLinkConnectMonitor

### NetCore
#### StartTask
`ASYNC_BLOCK_START` 和 `ASYNC_BLOCK_END` 将其间代码**闭包**成 Lambda，并调用 `PostMessage` 投入消息队列的异步调用列表 `lst_message` 中。

```CPP
void NetCore::StartTask(const Task& _task) {
    
    ASYNC_BLOCK_START

    case Task::kChannelLong:
        start_ok = longlink_task_manager_->StartTask(task);
        break;

    case Task::kChannelShort:
        start_ok = shortlink_task_manager_->StartTask(task);
        break;
        
    ASYNC_BLOCK_END
}
```