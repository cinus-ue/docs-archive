# Django auto-reloader机制

在用runserver启动服务的开发者模式下，Django提供了一种auto-reloader的机制，用户修改代码后不需要手动重启服务就能自动加载新的代码。该功能提高了开发调试的效率。

## runserver命令

runserver中存在关于use_reloader的判断。如果在启动命令中没有--noreload，程序就会执行autoreload.run_with_reloader这个函数，否则，就会执行self.inner_run，直接启动应用。
```python
# django/ore/management/commands/runserver.py
def run(self, **options):
    """Run the server, using the autoreloader if needed."""
    use_reloader = options['use_reloader']

    if use_reloader:
        autoreload.run_with_reloader(self.inner_run, **options)
    else:
        self.inner_run(None, **options)
```

## 自动重载模块

当第一次运行run_with_reloader时RUN_MAIN环境变量尚未设置，因此必然执行restart_with_reloader函数。当子进程检测到文件变化，exit_code为3时，重新创建子进程，此时RUN_MAIN环境变量变量为true，则会执行start_django函数。
```python
# django/utils/autoreload.py
def run_with_reloader(main_func, *args, **kwargs):
    signal.signal(signal.SIGTERM, lambda *args: sys.exit(0))
    try:
        # 判断是否存在环境变量RUN_MAIN，且值被设置为true
        if os.environ.get(DJANGO_AUTORELOAD_ENV) == 'true':
            reloader = get_reloader()
            logger.info('Watching for file changes with %s', reloader.__class__.__name__)
            start_django(reloader, main_func, *args, **kwargs)
        else:
            # 第一次必然会进入这个分支，因为没有地方设置过RUN_MAIN
            # 创建一个新的子进程来运行服务
            exit_code = restart_with_reloader()
            sys.exit(exit_code)
    except KeyboardInterrupt:
        pass
```
restart_with_reloader通过while循环判断exit_code!=3。如果子进程以exit_code=3退出（检测到了文件修改），就再启动一遍子进程，修改的代码自然就生效了；如果子进程以exit_code!=3退出，主进程也结束。
```python
def restart_with_reloader():
    # 设置环境变量RUN_MAIN，且值为true
    new_environ = {**os.environ, DJANGO_AUTORELOAD_ENV: 'true'}
    args = get_child_arguments()
    while True:
        p = subprocess.run(args, env=new_environ, close_fds=False)
        if p.returncode != 3:
            return p.returncode

def trigger_reload(filename):
    logger.info('%s changed, reloading.', filename)
    sys.exit(3)
```

## 监测文件变动

Django提供两种文件监测方式：
- WatchmanReloader 需安装Watchman，适用于大型项目，可降低响应的时间
- StatReloader     通过时间来判断文件是否修改
```python
def get_reloader():
    """Return the most suitable reloader for this environment."""
    try:
        WatchmanReloader.check_availability()
    except WatchmanUnavailable:
        return StatReloader()
    return WatchmanReloader()
```

StatReloader实现：
```python
def tick(self):
    mtimes = {}
    while True:
        for filepath, mtime in self.snapshot_files():
            old_time = mtimes.get(filepath)
            mtimes[filepath] = mtime
            if old_time is None:
                logger.debug('File %s first seen with mtime %s', filepath, mtime)
                continue
            elif mtime > old_time:
                logger.debug('File %s previous mtime: %s, current mtime: %s', filepath, old_time, mtime)
                self.notify_file_changed(filepath)

        time.sleep(self.SLEEP_TIME)
        yield
```