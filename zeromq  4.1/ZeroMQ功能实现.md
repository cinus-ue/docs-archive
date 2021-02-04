# ZeroMQ功能实现

## ZeroMQ概述
ZeroMQ（也称为ØMQ、0MQ或ZMQ）是一个异步消息网络库，旨在分布式或并发应用程序中使用。  

ZeroMQ通过各种传输（TCP、进程内、进程间、多播、WebSocket等）支持常见的消息传递模式（发布-订阅、请求/应答、任务分发、扇出等），使得消息传递非常简单。其异步I/O模型能够为多核消息系统提供足够的扩展性，性能足以用来构建集群产品。  

ZeroMQ常用的通信模式有三类：  
- 1.请求-应答（request-reply）  
> 由请求端发起请求，并等待回应端回应请求。从请求端来看，一定是一对对收发配对的；反之，在回应端一定是发收对。请求端和回应端都可以是1:N的模型。ZeroMQ可以很好的支持路由功能（实现路由功能的组件叫作Device），把1:N扩展为N:M（只需要加入若干路由节点）。  

- 2.发布-订阅(pub-sub)  
> 这个模型里，发布端是单向只发送数据的，且不关心是否把全部的信息都发送给订阅端。如果发布端开始发布信息的时候，订阅端尚未连接上来，这些信息直接丢弃。不过一旦订阅端连接上来，中间会保证没有信息丢失。同样，订阅端则只负责接收，而不能反馈。如果发布端和订阅端需要交互（比如要确认订阅者是否已经连接上），则使用额外的socket采用请求回应模型满足这个需求。   

- 3.推送-拖拉（push-pull）  
> 这个模型里，管道是单向的，从PUSH端单向的向PULL端单向的推送数据流。  

```cpp
/*  Socket types. */
#define ZMQ_PAIR 0
#define ZMQ_PUB 1
#define ZMQ_SUB 2
#define ZMQ_REQ 3
#define ZMQ_REP 4
#define ZMQ_DEALER 5
#define ZMQ_ROUTER 6
#define ZMQ_PULL 7
#define ZMQ_PUSH 8
#define ZMQ_XPUB 9
#define ZMQ_XSUB 10
#define ZMQ_STREAM 11
```

## 通信协议
ZeroMQ支持tcp(网络通信),inproc(进程内通信),ipc(进程间通信),pgm和epgm(多路广播)等通信协议。  
通信协议配置简单，用类似于URL形式的字符串指定即可，格式分别为inproc://、ipc://、tcp://、pgm://。  
ZeroMQ会自动根据指定的字符串解析出协议、地址、端口号等信息。
```cpp
// 代码socket_base.cpp
int zmq::socket_base_t::parse_uri (const char *uri_,
                        std::string &protocol_, std::string &address_)
{
    zmq_assert (uri_ != NULL);

    std::string uri (uri_);
    std::string::size_type pos = uri.find ("://");
    if (pos == std::string::npos) {
        errno = EINVAL;
        return -1;
    }
    protocol_ = uri.substr (0, pos);
    address_ = uri.substr (pos + 3);

    if (protocol_.empty () || address_.empty ()) {
        errno = EINVAL;
        return -1;
    }
    return 0;
}

int zmq::socket_base_t::check_protocol (const std::string &protocol_)
{
    //  First check out whether the protcol is something we are aware of.
    if (protocol_ != "inproc"
    &&  protocol_ != "ipc"
    &&  protocol_ != "tcp"
    &&  protocol_ != "pgm"
    &&  protocol_ != "epgm"
    &&  protocol_ != "tipc"
    &&  protocol_ != "norm") {
        errno = EPROTONOSUPPORT;
        return -1;
    }
    ...
}
```

### 线程与通信
ZeroMQ几乎所有的I/O操作都是异步的，主线程不会被阻塞。ZeroMQ会根据用户调用zmq_init函数时传入的接口参数，创建对应数量的I/O Thread。  
每个I/O Thread都有与之绑定的Poller，Poller采用经典的Reactor模式实现，Poller根据不同操作系统平台使用不同的网络I/O模型（select、poll、epoll、devpoll、kequeue等）。    
主线程与I/O线程通过Mail Box传递消息来进行通信。服务端开始监听或者客户端发起连接时，在主线程中创建zmq_connecter或zmq_listener，通过Mail Box发消息的形式将其绑定到I/O线程，I/O线程会把zmq_connecter或zmq_listener添加到Poller中用以侦听读/写事件。  
服务端与客户端在第一次通信时，会创建zmq_init来发送identity，用以进行认证。认证结束后，双方会为此次连接创建Session，以后双方就通过Session进行通信。每个Session都会关联到相应的读/写管道，主线程收发消息只是分别从管道中读/写数据。Session并不实际跟kernel交换I/O数据，而是通过plugin到Session中的Engine来与kernel交换I/O数据。  

1.创建Context，负责保存Socket实例和其他各种资源，如线程io_thread和它的mailbox。    
```cpp
// 代码ctx.hpp
//  I/O threads.
typedef std::vector <zmq::io_thread_t*> io_threads_t;
io_threads_t io_threads;

//  Array of pointers to mailboxes for both application and I/O threads.
uint32_t slot_count;
mailbox_t **slots;

// 代码ctx.cpp
zmq::ctx_t::ctx_t () :
    tag (ZMQ_CTX_TAG_VALUE_GOOD),
    starting (true),
    terminating (false),
    reaper (NULL),
    slot_count (0),
    slots (NULL),  //保存mailbox
    max_sockets (clipped_maxsocket (ZMQ_MAX_SOCKETS_DFLT)),
    io_thread_count (ZMQ_IO_THREADS_DFLT),
    ipv6 (false),
    thread_priority (ZMQ_THREAD_PRIORITY_DFLT),
    thread_sched_policy (ZMQ_THREAD_SCHED_POLICY_DFLT)
```

创建Socket对象，由Context负责管理。
```cpp
// 代码 ctx.cpp
zmq::socket_base_t *zmq::ctx_t::create_socket (int type_)
{
    slot_sync.lock ();
    if (unlikely (starting)) {

        starting = false;
        //  Initialise the array of mailboxes. Additional three slots are for
        //  zmq_ctx_term thread and reaper thread.
        opt_sync.lock ();
        int mazmq = max_sockets;
        int ios = io_thread_count;
        opt_sync.unlock ();
        slot_count = mazmq + ios + 2;
        slots = (mailbox_t **) malloc (sizeof (mailbox_t*) * slot_count);
        alloc_assert (slots);

        //  Initialise the infrastructure for zmq_ctx_term thread.
        slots [term_tid] = &term_mailbox;

        //  Create the reaper thread.
        reaper = new (std::nothrow) reaper_t (this, reaper_tid);
        alloc_assert (reaper);
        // 保存reaper的mailbox，用于以后通信 
        // 保存的是回收对象的Mailbox,保存完毕后就会启动回收对象轮询线程。
        slots [reaper_tid] = reaper->get_mailbox ();
        // 启动reaper的工作线程
        reaper->start ();

        //  Create I/O thread objects and launch them.
        for (int i = 2; i != ios + 2; i++) {
            io_thread_t *io_thread = new (std::nothrow) io_thread_t (this, i);
            alloc_assert (io_thread);
            io_threads.push_back (io_thread);
            slots [i] = io_thread->get_mailbox ();
            io_thread->start ();
        }

        //  In the unused part of the slot array, create a list of empty slots.
        for (int32_t i = (int32_t) slot_count - 1;
              i >= (int32_t) ios + 2; i--) {
            empty_slots.push_back (i);
            slots [i] = NULL;
        }
    }

    //  Once zmq_ctx_term() was called, we can't create new sockets.
    if (terminating) {
        slot_sync.unlock ();
        errno = ETERM;
        return NULL;
    }

    //  If max_sockets limit was reached, return error.
    if (empty_slots.empty ()) {
        slot_sync.unlock ();
        errno = EMFILE;
        return NULL;
    }

    //  Choose a slot for the socket.
    uint32_t slot = empty_slots.back ();
    empty_slots.pop_back ();

    //  sid是生成并递增的唯一的socket ID
    //  Generate new unique socket ID.
    int sid = ((int) max_socket_id.add (1)) + 1;

    //  Create the socket and register its mailbox.
    socket_base_t *s = socket_base_t::create (type_, this, slot, sid);
    if (!s) {
        empty_slots.push_back (slot);
        slot_sync.unlock ();
        return NULL;
    }
    sockets.push_back (s);
    slots [slot] = s->get_mailbox ();

    slot_sync.unlock ();
    return s;
}

```

2.Socket为每个连接创建一个Session。  
Socket根据连接类型，创建tcp_conneter(客户端)或tcp_listener(服务器)，后者将完成连接或接受连接的过程，并创建stream_egine。
engine被直接挂接到session_base上，后续的数据读写由engine直接处理，不再经过tcp_connnter或tcp_listener。  

```cpp
int zmq::socket_base_t::bind (const char *addr_)
{
    ...
    if (protocol == "tcp") {
        // 创建tcp_listener
        tcp_listener_t *listener = new (std::nothrow) tcp_listener_t (
            io_thread, this, options);
        alloc_assert (listener);
        // Create a listening socket.
        int rc = listener->set_address (address.c_str ());
        if (rc != 0) {
            delete listener;
            event_bind_failed (address, zmq_errno());
            return -1;
        }

        // Save last endpoint URI
        listener->get_address (last_endpoint);

        add_endpoint (last_endpoint.c_str (), (own_t *) listener, NULL);
        return 0;
    }
    ...
}

void zmq::socket_base_t::add_endpoint (const char *addr_, own_t *endpoint_, pipe_t *pipe)
{
    //  Activate the session. Make it a child of this socket.
    launch_child (endpoint_);
    endpoints.insert (endpoints_t::value_type (std::string (addr_), endpoint_pipe_t (endpoint_, pipe)));
}

void zmq::own_t::launch_child (own_t *object_)
{
    //  Specify the owner of the object.
    object_->set_owner (this);

    //  Plug the object into the I/O thread.
    send_plug (object_);

    //  Take ownership of the object.
    send_own (this, object_);
}

void zmq::tcp_listener_t::process_plug ()
{
    //  Start polling for incoming connections.
    handle = add_fd (s);   //通过add_fd将tcp_listener传如作为poll_entry的events，当事件通知时调用in_event
    set_pollin (handle);
}

zmq::io_object_t::handle_t zmq::io_object_t::add_fd (fd_t fd_)
{
    return poller->add_fd (fd_, this);
}

void zmq::io_object_t::set_pollin (handle_t handle_)
{
    poller->set_pollin (handle_);
}
```

当poller中有事件通知时，调用tcp_listener的in_event。  
此时会创建stream_engine，选择IO线程，创建session_base。
```cpp
void zmq::tcp_listener_t::in_event ()
{
    fd_t fd = accept ();

    //  If connection was reset by the peer in the meantime, just ignore it.
    //  TODO: Handle specific errors like ENFILE/EMFILE etc.
    if (fd == retired_fd) {
        socket->event_accept_failed (endpoint, zmq_errno());
        return;
    }

    tune_tcp_socket (fd);
    tune_tcp_keepalives (fd, options.tcp_keepalive, options.tcp_keepalive_cnt, options.tcp_keepalive_idle, options.tcp_keepalive_intvl);

    // remember our fd for ZMQ_SRCFD in messages
    socket->set_fd(fd);

    //  Create the engine object for this connection.
    stream_engine_t *engine = new (std::nothrow)
        stream_engine_t (fd, options, endpoint, !options.raw_sock);
    alloc_assert (engine);

    //  Choose I/O thread to run connecter in. Given that we are already
    //  running in an I/O thread, there must be at least one available.
    io_thread_t *io_thread = choose_io_thread (options.affinity);
    zmq_assert (io_thread);

    //  Create and launch a session object.
    session_base_t *session = session_base_t::create (io_thread, false, socket,
        options, NULL);
    errno_assert (session);
    session->inc_seqnum ();
    launch_child (session); 
    send_attach (session, engine, false);    // 向session_base发送ATTACH命令
    socket->event_accepted (endpoint, fd);
}

// 代码object.cpp
void zmq::object_t::send_attach (session_base_t *destination_,
    i_engine *engine_, bool inc_seqnum_)
{
    if (inc_seqnum_)
        destination_->inc_seqnum ();

    command_t cmd;
    cmd.destination = destination_;
    cmd.type = command_t::attach;
    cmd.args.attach.engine = engine_;
    send_command (cmd);
}

//代码session_base.cpp
void zmq::session_base_t::process_attach (i_engine *engine_)
{
    zmq_assert (engine_ != NULL);
    zmq_assert (!engine);
    engine = engine_;

    if (!engine_->has_handshake_stage ())
        engine_ready ();    //创建pipe，用于与主线程的通信

    //  Plug in the engine.
    engine->plug (io_thread, this);
}


void zmq::session_base_t::engine_ready ()
{
    //  Create the pipe if it does not exist yet.
    if (!pipe && !is_terminating ()) {
        ...
        pipe_t *pipes [2] = {NULL, NULL};
        ...
        int hwms [2] = {conflate? -1 : options.rcvhwm,
            conflate? -1 : options.sndhwm};
        bool conflates [2] = {conflate, conflate};
        int rc = pipepair (parents, pipes, hwms, conflates);
        errno_assert (rc == 0);
        //  Plug the local end of the pipe.
        pipes [0]->set_event_sink (this);   // 将session_base与pipe关联

        //  Remember the local end of the pipe.
        zmq_assert (!pipe);
        pipe = pipes [0];

        //  Ask socket to plug into the remote end of the pipe.
        send_bind (socket, pipes [1]);
    }
}

// stream_engine通过session_base读写消息，
int zmq::stream_engine_t::push_msg_to_session (msg_t *msg_)
{
    return session->push_msg (msg_);
}

int zmq::session_base_t::push_msg (msg_t *msg_)
{
    if (pipe && pipe->write (msg_)) {
        int rc = msg_->init ();
        errno_assert (rc == 0);
        return 0;
    }

    errno = EAGAIN;
    return -1;
}
```

## 管道事件
```cpp
    struct i_pipe_events
    {
        virtual ~i_pipe_events () {}
        /*
        表示管道可读，管道实际调用socket_base或session_base的read_activated方法，而socket_base实际会调用xread_activated方法
        */
        virtual void read_activated (zmq::pipe_t *pipe_) = 0;
        /*
        表示管道可写，管道实际调用socket_base或session_base的write_activated方法，而socket_base实际会调用xwrite_activated方法。
        */
        virtual void write_activated (zmq::pipe_t *pipe_) = 0;
        // 当连接突然中断时调用
        virtual void hiccuped (zmq::pipe_t *pipe_) = 0;
        // 表示管道终止
        virtual void pipe_terminated (zmq::pipe_t *pipe_) = 0;
    };
```


## 发送消息
客户端连接服务端，创建session和pipe，并调用attach_pipe。
```cpp
int zmq::socket_base_t::connect (const char *addr_)
{
    ...
    //  Choose the I/O thread to run the session in.
    io_thread_t *io_thread = choose_io_thread (options.affinity);
    if (!io_thread) {
        errno = EMTHREAD;
        return -1;
    }

    address_t *paddr = new (std::nothrow) address_t (protocol, address);
    alloc_assert (paddr);
    ...
    //  Create session.
    session_base_t *session = session_base_t::create (io_thread, true, this,
        options, paddr);
    errno_assert (session);

    //  PGM does not support subscription forwarding; ask for all data to be
    //  sent to this pipe. (same for NORM, currently?)
    bool subscribe_to_all = protocol == "pgm" || protocol == "epgm" || protocol == "norm";
    pipe_t *newpipe = NULL;

    if (options.immediate != 1 || subscribe_to_all) {
        //  Create a bi-directional pipe.
        object_t *parents [2] = {this, session};
        pipe_t *new_pipes [2] = {NULL, NULL};

        bool conflate = options.conflate &&
            (options.type == ZMQ_DEALER ||
             options.type == ZMQ_PULL ||
             options.type == ZMQ_PUSH ||
             options.type == ZMQ_PUB ||
             options.type == ZMQ_SUB);

        int hwms [2] = {conflate? -1 : options.sndhwm,
            conflate? -1 : options.rcvhwm};
        bool conflates [2] = {conflate, conflate};
        rc = pipepair (parents, new_pipes, hwms, conflates);
        errno_assert (rc == 0);

        //  Attach local end of the pipe to the socket object.
        attach_pipe (new_pipes [0], subscribe_to_all);
        newpipe = new_pipes [0];

        //  Attach remote end of the pipe to the session object later on.
        session->attach_pipe (new_pipes [1]);
    }

    //  Save last endpoint URI
    paddr->to_string (last_endpoint);

    add_endpoint (addr_, (own_t *) session, newpipe);
    return 0;
}
```

创建2个pipe，相互引用。
```cpp
int zmq::pipepair (class object_t *parents_ [2], class pipe_t* pipes_ [2],
    int hwms_ [2], bool conflate_ [2])
{
    //   Creates two pipe objects. These objects are connected by two ypipes,
    //   each to pass messages in one direction.

    typedef ypipe_t <msg_t, message_pipe_granularity> upipe_normal_t;
    typedef ypipe_conflate_t <msg_t> upipe_conflate_t;

    pipe_t::upipe_t *upipe1;
    if(conflate_ [0])
        upipe1 = new (std::nothrow) upipe_conflate_t ();
    else
        upipe1 = new (std::nothrow) upipe_normal_t ();
    alloc_assert (upipe1);

    pipe_t::upipe_t *upipe2;
    if(conflate_ [1])
        upipe2 = new (std::nothrow) upipe_conflate_t ();
    else
        upipe2 = new (std::nothrow) upipe_normal_t ();
    alloc_assert (upipe2);

    pipes_ [0] = new (std::nothrow) pipe_t (parents_ [0], upipe1, upipe2,
        hwms_ [1], hwms_ [0], conflate_ [0]);
    alloc_assert (pipes_ [0]);
    pipes_ [1] = new (std::nothrow) pipe_t (parents_ [1], upipe2, upipe1,
        hwms_ [0], hwms_ [1], conflate_ [1]);
    alloc_assert (pipes_ [1]);

    pipes_ [0]->set_peer (pipes_ [1]);
    pipes_ [1]->set_peer (pipes_ [0]);

    return 0;
}
```

调用Socket实例的xsend，向current_out(pipe)中写入消息，调用flush发送activate_read命令。
```cpp
int zmq::router_t::xsend (msg_t *msg_)
{
    ...
    //  Push the message into the pipe. If there's no out pipe, just drop it.
    if (current_out) {

        // Close the remote connection if user has asked to do so
        // by sending zero length message.
        // Pending messages in the pipe will be dropped (on receiving term- ack)
        if (raw_sock && msg_->size() == 0) {
            current_out->terminate (false);
            int rc = msg_->close ();
            errno_assert (rc == 0);
            rc = msg_->init ();
            errno_assert (rc == 0);
            current_out = NULL;
            return 0;
        }

        bool ok = current_out->write (msg_);
        if (unlikely (!ok)) {
            // Message failed to send - we must close it ourselves.
            int rc = msg_->close ();
            errno_assert (rc == 0);
            current_out = NULL;
        } else {
          if (!more_out) {
              current_out->flush ();
              current_out = NULL;
          }
        }
    }
    ...
}

void zmq::pipe_t::flush ()
{
    //  The peer does not exist anymore at this point.
    if (state == term_ack_sent)
        return;

    if (outpipe && !outpipe->flush ())
        send_activate_read (peer);
}
```

pipe处理activate_read命令。
当出管道有数据可读时，会调用session_base的read_activated事件。
```cpp
void zmq::pipe_t::process_activate_read ()
{
    if (!in_active && (state == active || state == waiting_for_delimiter)) {
        in_active = true;
        sink->read_activated (this); // sink为session对象
    }
}
```

```cpp
void zmq::session_base_t::read_activated (pipe_t *pipe_)
{
    ...
    if (likely (pipe_ == pipe))
        engine->restart_output ();
    else
        engine->zap_msg_available ();
}

```
然后会调用对应engine的restart_output
```cpp
void zmq::stream_engine_t::restart_output ()
{
    ...
    //  Speculative write: The assumption is that at the moment new message
    //  was sent by the user the socket is probably available for writing.
    //  Thus we try to write the data to socket avoiding polling for POLLOUT.
    //  Consequently, the latency should be better in request/reply scenarios.
    out_event ();
}

void zmq::stream_engine_t::out_event ()
{
    zmq_assert (!io_error);

    //  If write buffer is empty, try to read new data from the encoder.
    if (!outsize) {

        //  Even when we stop polling as soon as there is no
        //  data to send, the poller may invoke out_event one
        //  more time due to 'speculative write' optimisation.
        if (unlikely (encoder == NULL)) {
            zmq_assert (handshaking);
            return;
        }

        outpos = NULL;
        outsize = encoder->encode (&outpos, 0);

        while (outsize < out_batch_size) {
            if ((this->*next_msg) (&tx_msg) == -1)
                break;
            encoder->load_msg (&tx_msg);
            unsigned char *bufptr = outpos + outsize;
            size_t n = encoder->encode (&bufptr, out_batch_size - outsize);
            zmq_assert (n > 0);
            if (outpos == NULL)
                outpos = bufptr;
            outsize += n;
        }

        //  If there is no data to send, stop polling for output.
        if (outsize == 0) {
            output_stopped = true;
            reset_pollout (handle);
            return;
        }
    }

    //  If there are any data to write in write buffer, write as much as
    //  possible to the socket. Note that amount of data to write can be
    //  arbitrarily large. However, we assume that underlying TCP layer has
    //  limited transmission buffer and thus the actual number of bytes
    //  written should be reasonably modest.
    const int nbytes = tcp_write (s, outpos, outsize);
    ...
}
```

## 接收消息
socket, session, connecter, listener和engine之间的command传送，通过mailbox完成。  
socket_base与session_base之间的message传送，通过pipe完成。
session_base与engine之间直接引用，不需要特别处理。

服务端接收消息流程：
主线程通过调用socket_base的recv接收消息, 轮询mailbox中的命令, 处理ACTIVATE_READ命令，将pipe放入fq中，通过xrecv读取pipe的数据。
```cpp
int zmq::socket_base_t::recv (msg_t *msg_, int flags_)
{
    ...
    //  In blocking scenario, commands are processed over and over again until
    //  we are able to fetch a message.
    bool block = (ticks != 0);
    while (true) {
        // 处理命令
        if (unlikely (process_commands (block ? timeout : 0, false) != 0)) 
            return -1;
        // 读取消息
        rc = xrecv (msg_); 
        if (rc == 0) {
            ticks = 0;
            break;
        }
        ...
    }
    ...
}

//  mailbox中的处理命令
int zmq::socket_base_t::process_commands (int timeout_, bool throttle_)
{
    int rc;
    command_t cmd;
    if (timeout_ != 0) {
        //  If we are asked to wait, simply ask mailbox to wait.
        rc = mailbox.recv (&cmd, timeout_);
    }
    else {
        ...
        rc = mailbox.recv (&cmd, 0);
    }
    //  Process all available commands.
    while (rc == 0) {
        // 处理ACTIVATE_READ命令，destination即session_base创建的pipe
        cmd.destination->process_command (cmd);
        rc = mailbox.recv (&cmd, 0);
    }
    ...
}

void zmq::pipe_t::process_activate_read ()
{
    if (!in_active && (state == active || state == waiting_for_delimiter)) {
        in_active = true;
        sink->read_activated (this);  //sink为rep，最终实际调用父类router的xread_activated
    }
}

void zmq::socket_base_t::read_activated (pipe_t *pipe_)
{
    xread_activated (pipe_);
}

// 将pipe加入到fq中，之后可以通过fq读取pipe中的消息
void zmq::router_t::xread_activated (pipe_t *pipe_)
{
    std::set <pipe_t*>::iterator it = anonymous_pipes.find (pipe_);
    if (it == anonymous_pipes.end ())
        fq.activated (pipe_);
    else {
        bool identity_ok = identify_peer (pipe_);
        if (identity_ok) {
            anonymous_pipes.erase (it);
            fq.attach (pipe_);
        }
    }
}

// 最终调用router的xrecv从pipe中读取数据
int zmq::router_t::xrecv (msg_t *msg_)
{
    ...
    pipe_t *pipe = NULL;
    int rc = fq.recvpipe (msg_, &pipe);
    //  It's possible that we receive peer's identity. That happens
    //  after reconnection. The current implementation assumes that
    //  the peer always uses the same identity.
    while (rc == 0 && msg_->is_identity ())
        rc = fq.recvpipe (msg_, &pipe);

    if (rc != 0)
        return -1;
    zmq_assert (pipe != NULL);
    ...
}
```

## I/O线程
I/O线程是ZeroMQ异步处理网络IO的后台线程。  
io_thread_t实现继承object_t，并实现i_poll_events接口，其内部包含一个mailbox和一个poller对象。
```cpp
zmq::io_thread_t::io_thread_t (ctx_t *ctx_, uint32_t tid_) :
    object_t (ctx_, tid_)
{
    poller = new (std::nothrow) poller_t (*ctx_);
    alloc_assert (poller);
    /* poller_t是从不同操作系统提供的事件通知机制中抽象出来的概念，用来通知描述符和计时器事件，poller_t通过typedef定义为操作系统首选的通知机制(select_t/poll_t/epoll_t等)。 */ 
    mailbox_handle = poller->add_fd (mailbox.get_fd (), this);
    poller->set_pollin (mailbox_handle);
}
```
所有运行在io_thread_t上的对象都继承自辅助类io_object_t，该类实现了向io_thread_t注册/删除文件描述符(add_fd/rm_fd)和计时器(add_timer/cancel_timer)事件的功能，同时io_object_t 还继承了i_poll_events 接口来实现事件回调功能。  i_poll_events 接口定义了文件描述符和计时器事件就绪时的回调处理函数（in_event/out_event/timer_event）。io_thread_t 实现此接口(in_event)来处理来自mailbox的事件。  

创建监听对象或创建连接时可以通过多个IO线程进行负载均衡  
```cpp
// affinity表示哪些IO线程有资格,默认为0表示所有IO线程都可以处理。
zmq::io_thread_t *zmq::ctx_t::choose_io_thread (uint64_t affinity_)
{
    if (io_threads.empty ())
        return NULL;
    //  Find the I/O thread with minimum load.
    int min_load = -1;
    io_thread_t *selected_io_thread = NULL;
    for (io_threads_t::size_type i = 0; i != io_threads.size (); i++) {
        if (!affinity_ || (affinity_ & (uint64_t (1) << i))) {
             // 获取IO线程socket载入次数
            int load = io_threads [i]->get_load ();
             // 这里对IO线程进行负载均衡
            if (selected_io_thread == NULL || load < min_load) {
                min_load = load;
                selected_io_thread = io_threads [i];
            }
        }
    }
    return selected_io_thread;
}
```

## Mail Box
mailbox_t本质是一个具有就绪通知功能的存储命令的队列。就绪通知机制由signaler_t提供的文件描述符实现。  
队列是由ypipe_t实现的无锁无溢出队列。当mailbox_t事件触发时，io线程从mailbox中获取命令，并让命令的接收者进行处理。
```cpp
// 命令类型
enum type_t
        {
            stop,    //  发送给IO线程表示当前对象需要停止
            plug,    //  发送给IO线程表示当前对象需要注册到IO线程中
            own,     //  将创建的对象Session的加入到当前Socket的所属集合中
            attach,  //  附加engine到Session中
            bind,    //  建立session到Socket之间的管道,调用者事先调用了inc_seqnum发送命令（增加接收方中的计数 sent_seqnum）.
            activate_read,  //  通过写管道发送通知给读管道多少信息可读
            activate_write,   //  通过读管道发送通知给写读管道多少信息可写
            hiccup,     //  创建一个新的管道后通过读管道发送给写管道，参数是管道类型，然而他的目的地是私有的,因此我们必须用void指针
            pipe_term,  //  通过读管道发送到写管道告诉他中止所有管道
            pipe_term_ack,  //  写管道对PipeTerm命令响应
            term_req,   //  通过IO对象发送给socket请求终端IO对象
            term,       //  通过socket发送给IO对象他自己开始关闭
            term_ack,   //  通过IO对象发送给socket让它知道已经关闭
            reap,       //  将关闭套接字的所有权转移给回收线程.
            reaped,     //  关闭套接字通知回收线程他已经释放
            inproc_connected,
            done        //  当所有socket都被释放通过回收线程发送给 term 线程
        } type;


// 代码object.cpp
void zmq::object_t::send_command (command_t &cmd_)
{
    ctx->send_command (cmd_.destination->get_tid (), cmd_);
}

// 代码ctx.cpp
void zmq::ctx_t::send_command (uint32_t tid_, const command_t &command_)
{
    slots [tid_]->send (command_);
}

// 代码mailbox.cpp
void zmq::mailbox_t::send (const command_t &cmd_)
{
    sync.lock ();
    // 向管道写入命令
    cpipe.write (cmd_, false);
    const bool ok = cpipe.flush ();
    sync.unlock ();
    if (!ok)
        // 传递信号
        signaler.send ();
}

// io_thread_t实现此接口(in_event)来处理来自mailbox的事件
// 所以当有命令进入mailbox的时候，poller会被唤醒，并调用io_thread的in_event()函数
void zmq::io_thread_t::in_event ()
{
    command_t cmd;
    int rc = mailbox.recv (&cmd, 0);

    while (rc == 0 || errno == EINTR) {
        if (rc == 0)
            cmd.destination->process_command (cmd);
        rc = mailbox.recv (&cmd, 0);
    }
    errno_assert (rc != 0 && errno == EAGAIN);
}
// 根据不同命令类型进行处理,处理方式由具体的Socket子类去重载。
void zmq::object_t::process_command (command_t &cmd_)
{
    switch (cmd_.type) {

    case command_t::activate_read:
        process_activate_read ();
        break;

    case command_t::activate_write:
        process_activate_write (cmd_.args.activate_write.msgs_read);
        break;

    case command_t::stop:
        process_stop ();
        break;

    case command_t::plug:
        process_plug ();
        process_seqnum ();
        break;
    ...
}
```
## 回收线程
回收线程由类reaper_t实现。socket通过（send_reap）向回收线程发送回收命令，回收线程收到命令后会将socket从应用线程迁移到回收线程上，这样socket就可以在回收线程上处理命令(term/term_ack)，直到socket的所有子对象都成功销毁时，socket就会在回收线程上销毁。实际上回收线程只是待回收对象驻留的线程，对象的处理逻辑仍然由对象自身处理。
```cpp
// 向回收线程的邮箱发送当前SocketBase的回收命令
int zmq::socket_base_t::close ()
{
    //  Mark the socket as dead
    tag = 0xdeadbeef;

    //  Transfer the ownership of the socket from this application thread
    //  to the reaper thread which will take care of the rest of shutdown
    //  process.
    send_reap (this);

    return 0;
}
// 接收到释放命令进行处理
void zmq::reaper_t::process_reap (socket_base_t *socket_)
{
    //  Add the socket to the poller.
    socket_->start_reaping (poller);

    ++sockets;
}

void zmq::socket_base_t::start_reaping (poller_t *poller_)
{
    //  Plug the socket to the reaper thread.
    poller = poller_;
    handle = poller->add_fd (mailbox.get_fd (), this);
    poller->set_pollin (handle);

    //  Initialise the termination and check whether it can be deallocated
    //  immediately.
    // 当前Socket终止处理
    terminate ();
    // 最后确认释放
    check_destroy ();
}
void zmq::own_t::terminate ()
{
    //  If termination is already underway, there's no point
    //  in starting it anew.
    if (terminating)
        return;

    //  As for the root of the ownership tree, there's noone to terminate it,
    //  so it has to terminate itself.
    if (!owner) {
        process_term (options.linger);
        return;
    }

    //  If I am an owned object, I'll ask my owner to terminate me.
    send_term_req (owner, this);
}

void zmq::socket_base_t::check_destroy ()
{
    //  If the object was already marked as destroyed, finish the deallocation.
    if (destroyed) {

        //  Remove the socket from the reaper's poller.
        poller->rm_fd (handle);

        //  Remove the socket from the context.
        destroy_socket (this);

        //  Notify the reaper about the fact.
        send_reaped ();

        //  Deallocate.
        own_t::process_destroy ();
    }
}

void zmq::ctx_t::destroy_socket (class socket_base_t *socket_)
{
    slot_sync.lock ();

    //  Free the associated thread slot.
    uint32_t tid = socket_->get_tid ();
    empty_slots.push_back (tid); //重新加入到可用socket栈中
    slots [tid] = NULL; //清空引用

    //  从当前使用socket集合中移除
    //  Remove the socket from the list of sockets.
    sockets.erase (socket_);
    // 若当前接收到中止信号且当前socket全部已释放时停止回收线程
    //  If zmq_ctx_term() was already called and there are no more socket
    //  we can ask reaper thread to terminate.
    if (terminating && sockets.empty ())
        reaper->stop ();

    slot_sync.unlock ();
}
```


