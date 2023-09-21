https://blog.csdn.net/qq_14996421/article/details/115982546

# 基本使用

如果只是退出当前的连接，可以用`quit`或者`exit`命令。Attach 到目标进程上的 arthas 还会继续运行，端口会保持开放，下次连接时用`telnet 127.0.0.1 3658`可以重新连接上。如果想完全退出 arthas，可以执行`stop`命令。

