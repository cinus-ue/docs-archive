# Supervisor功能实现
- supervisor生成主进程并成为守护进程，根据配置依次生成子进程
- supervisor与supervisorctl基于rpc通信
- 子进程与主进程基于管道通信

## supervisord
- ServerOptions类主要进行了配置文件的解析，创建和关闭HTTP服务等工作。
- Supervisor类实现了基于异步IO的服务运行，接收子进程、rpc客户端的通信处理等工作。

```python
# Main program
def main(args=None, test=False):
    assert os.name == "posix", "This code makes Unix-specific assumptions"
    # if we hup, restart by making a new Supervisor()
    first = True
    while 1:
        options = ServerOptions()
        options.realize(args, doc=__doc__)
        options.first = first
        options.test = test
        if options.profile_options:
            sort_order, callers = options.profile_options
            profile('go(options)', globals(), locals(), sort_order, callers)
        else:
            go(options) # 加载配置开始运行
        options.close_httpservers()
        options.close_logger()
        first = False
        if test or (options.mood < SupervisorStates.RESTARTING):
            break

def go(options): # pragma: no cover
    d = Supervisor(options) # 实例化一个Supervisor对象
    try:
        d.main() # 运行main()函数
    except asyncore.ExitNow:
        pass

``` 

d.main()中调用self.run()执行相应的配置，并且作为守护进程运行。
```python
def run(self):
    self.process_groups = {} # clear
    self.stop_groups = None # clear
    events.clear()
    try:
        # 根据配置进行添加process
        for config in self.options.process_group_configs:
            self.add_process_group(config)
        self.options.process_environment() # 进程环境
        self.options.openhttpservers(self) # 打开http web
        self.options.setsignals()  # 用于捕获信号
        # 主进程是否成为守护进程
        if (not self.options.nodaemon) and self.options.first:
            self.options.daemonize()
        # writing pid file needs to come *after* daemonizing or pid
        # will be wrong
        self.options.write_pidfile()
        # 运行异步io服务器
        self.runforever()
    finally:
        # 异常退出，清理工作
        self.options.cleanup()
```

在runforever中通过while循环处理读写事件并检测进程状态是否发生变化。
```python
    def runforever(self):
        # 事件通知机制
        events.notify(events.SupervisorRunningEvent())
        timeout = 1 # this cannot be fewer than the smallest TickEvent (5)
        # 获取已经注册的句柄
        socket_map = self.options.get_socket_map()
        while 1:
            # 保存运行信息等
            combined_map = {}
            combined_map.update(socket_map)
            combined_map.update(self.get_process_map())
            # 进程信息
            pgroups = list(self.process_groups.values())
            pgroups.sort()
            # 根据进程配置开启或关闭进程
            if self.options.mood < SupervisorStates.RUNNING:
                if not self.stopping:
                    # first time, set the stopping flag, do a
                    # notification and set stop_groups
                    self.stopping = True
                    self.stop_groups = pgroups[:]
                    events.notify(events.SupervisorStoppingEvent())

                self.ordered_stop_groups_phase_1()

                if not self.shutdown_report():
                    # if there are no unstopped processes (we're done
                    # killing everything), it's OK to shutdown or reload
                    raise asyncore.ExitNow

            for fd, dispatcher in combined_map.items():
                if dispatcher.readable():
                    self.options.poller.register_readable(fd)
                if dispatcher.writable():
                    self.options.poller.register_writable(fd)
            # poll操作
            r, w = self.options.poller.poll(timeout)

            for fd in r:
                if fd in combined_map:
                    try:
                        dispatcher = combined_map[fd]
                        self.options.logger.blather(
                            'read event caused by %(dispatcher)r',
                            dispatcher=dispatcher)
                        dispatcher.handle_read_event()
                        if not dispatcher.readable():
                            self.options.poller.unregister_readable(fd)
                    except asyncore.ExitNow:
                        raise
                    except:
                        combined_map[fd].handle_error()

            for fd in w:
                if fd in combined_map:
                    try:
                        dispatcher = combined_map[fd]
                        self.options.logger.blather(
                            'write event caused by %(dispatcher)r',
                            dispatcher=dispatcher)
                        dispatcher.handle_write_event()
                        if not dispatcher.writable():
                            self.options.poller.unregister_writable(fd)
                    except asyncore.ExitNow:
                        raise
                    except:
                        combined_map[fd].handle_error()

            for group in pgroups:
                group.transition()
            # 获取已经死亡的子进程信息
            self.reap()
            # 处理信号
            self.handle_signal()
            # tick时钟
            self.tick()

            if self.options.mood < SupervisorStates.RUNNING:
                self.ordered_stop_groups_phase_2()

            if self.options.test:
                break
```

创建进程组（ProcessGroup继承ProcessGroupBase），初始化时调用ProcessConfig的make_process完成进程创建。
```python
def add_process_group(self, config):
    name = config.name
    if name not in self.process_groups:
        config.after_setuid()
        # 根据初始化后的配置文件生成相应的子进程实例
        self.process_groups[name] = config.make_group()
        events.notify(events.ProcessGroupAddedEvent(name))
        return True
    return False

    def make_group(self):
        from supervisor.process import ProcessGroup
        return ProcessGroup(self)

class ProcessGroup(ProcessGroupBase):
    def transition(self):
        for proc in self.processes.values():
            proc.transition()

class ProcessGroupBase(object):
    def __init__(self, config):
        self.config = config
        self.processes = {}
        for pconfig in self.config.process_configs:
            self.processes[pconfig.name] = pconfig.make_process(self)
        
def make_process(self, group=None):
    from supervisor.process import Subprocess
    process = Subprocess(self)
    process.group = group
    return process
```
Subprocess调用spawn创建进程，最终是通过os.execve(filename, argv, env)来运行用户配置的程序。

## supervisorctl

supervisorctl初始化配置并创建Controller，然后根据参数来判断使用哪一种交互方式。
```python
def main(args=None, options=None):
    if options is None:
        options = ClientOptions() # 配置

    # 判断是否包含参数，不含参数设置interactive = 1
    options.realize(args, doc=__doc__)
    c = Controller(options)

    if options.args:
        c.onecmd(" ".join(options.args))
        sys.exit(c.exitstatus)

    if options.interactive:
        c.exec_cmdloop(args, options)
        sys.exit(0)  # exitstatus always 0 for interactive mode


# supervisor/options.py
from supervisor.supervisorctl import DefaultControllerPlugin
default_factory = ('default', DefaultControllerPlugin, {})
# we always add the default factory. If you want to a supervisorctl
# without the default plugin, please write your own supervisorctl.
self.plugin_factories = [default_factory]
```

Controller在初始化时根据options.plugin_factories，创建plugin。
```python
def __init__(self, options, completekey='tab', stdin=None,
             stdout=None):
    self.options = options
    self.prompt = self.options.prompt + '> '
    self.options.plugins = []
    self.vocab = ['help']
    self._complete_info = None
    self.exitstatus = LSBInitExitStatuses.SUCCESS
    cmd.Cmd.__init__(self, completekey, stdin, stdout)
    for name, factory, kwargs in self.options.plugin_factories:
        plugin = factory(self, **kwargs)
        for a in dir(plugin):
            if a.startswith('do_') and callable(getattr(plugin, a)):
                self.vocab.append(a[3:])
        self.options.plugins.append(plugin)
        plugin.name = name
```

通过_get_do_func查找命令并执行，当执行命令错误码为401时表示未通过身份验证，此时需要输入用户名和密码。
```python
do_func = self._get_do_func(cmd)
if do_func is None:
    return self.default(line)
    try:
        try:
            return do_func(arg)
        except xmlrpclib.ProtocolError as e:
            if e.errcode == 401:
                if self.options.interactive:
                    self.output('Server requires authentication')
                    username = raw_input('Username:')
                    password = getpass.getpass(prompt='Password:')
                    self.output('')
                    self.options.username = username
                    self.options.password = password
                    return self.onecmd(line)
                else:
                    self.output('Server requires authentication')
                    self.exitstatus = LSBInitExitStatuses.GENERIC
                else:
                    self.exitstatus = LSBInitExitStatuses.GENERIC
                    raise
            do_func(arg)
    except Exception:
        (file, fun, line), t, v, tbinfo = asyncore.compact_traceback()
        error = 'error: %s, %s: file: %s line: %s' % (t, v, file, line)
        self.output(error)
        self.exitstatus = LSBInitExitStatuses.GENERIC

def _get_do_func(self, cmd):
    func_name = 'do_' + cmd
    func = getattr(self, func_name, None)
    if not func:
        for plugin in self.options.plugins:
            func = getattr(plugin, func_name, None)
            if func is not None:
                break
    return func
```