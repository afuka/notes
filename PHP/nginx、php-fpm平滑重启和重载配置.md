## nginx
相对而言，nginx前后版本对于重启以及重新加载配置的改变不大，其进程分为master主进程和worker工作进程。master进程通过信号通知worker进程执行工作。通过简单操作命令 `nginx -s reload` 即可快速完成重新加载配置操作，也可以`nginx restart` 进行重启。

其中，在有请求未完成的时。`restart`，请求会直接断掉。`reload` 平滑重启的条件是在php-fpm.conf文件中设置`process_control_timeout`参数 (这个参数默认值为0)，这个参数表示在php-fpm reload 后，旧的worker的最大存活时间。

需要注意的是当 `nginx reload` 的时候 master 进程ID是没有变化的，worker 进程ID是有变化的。而`nginx restart` 的时候 master，worker的进程ID都有变化。可以使用  `watch` 命令进行实时监控
```
watch -n1 'ps -ef | grep nginx | grep -v grep'
```
`reload` 时，旧的worker进程和新的worker进程是共存的，旧的worker进程在处理完请求后会被杀掉。而 `restart `时，不会存在新旧共存的情况（master，worker 都是）。如果有请求未完成时。旧的master，worker 会有一段时间处理请求（超时时间）。然后被杀掉，创建新的master，worker 进程。所以 restart 的时候nginx 会有卡顿的现象。

## php-fpm
php-fpm进程也分为master主进程和worker工作进程。可以与nginx一样通过命令实现平滑重
```
php-fpm reload 重载配置（平滑重启）
php-fpm restart 重启
```
但是要注意的是php 5.3.3 以后的 php-fpm 不再支持 php-fpm 以前具有的 /usr/local/php/sbin/php-fpm (start|stop|reload)等命令，需要使用信号控制：
```
INT、TERM、QUIT：退出Fpm,在master收到退出信号后将向所有的 worker 进程发送退出信号，通知 worker 退出，然后 master 退出
USR1: 重新加载日志文件，生产环境中通常会根据时间对日志进行切割，切割后会生成一个新的日志文件，如果进程不重新加载文件，则无法继续写入日志，这时就需要向 master 发送一个 USR1 的信号，告诉 master 重新加载日志文件。
USR2: 重启 Fpm，首先 master 也是会向所有的 worker进程发送退出信号，等全部 worker 成功退出后，master 会调用 execvp() 重新启动一个新的 Fpm，最后旧的 master 退出。
```

例如一个简单的重启办法：
```
kill -USR2 「master进程号」  
```
这种方案一般是没有生成php-fpm.pid文件时使用，如果要生成php-fpm.pid，使用下面这种方案：
打开配置文件 `cat /usr/local/php/etc/php-fpm.conf` 发现：
```
[global]
; Pid file
; Note: the default prefix is /usr/local/php/var
; Default Value: none
;pid = run/php-fpm.pid
```
pid文件路径应该位于/usr/local/php/var/run/php-fpm.pid，由于注释掉，所以没有生成，我们把注释去除，再`kill -USR2 {master 进程} `重启php-fpm,便会生成pid文件，下次就可以使用以下命令重启,关闭php-fpm了：
```
# php-fpm 关闭：
kill -INT 'cat /usr/local/php/var/run/php-fpm.pid'
# php-fpm 重启：
kill -USR2 'cat /usr/local/php/var/run/php-fpm.pid'
```