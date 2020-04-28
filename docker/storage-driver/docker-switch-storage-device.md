# docker switch storage device

docker daemon  切换storage device后不会清理原来的"/var/lib/docker/存储引擎/"只会新建一个

"/var/lib/docker/存储引擎-new/"，然后docker会在"/var/lib/docker/存储引擎-new/"下面工作，就得

docker image仍然存在于"/var/lib/docker/存储引擎-old/"目录中。

