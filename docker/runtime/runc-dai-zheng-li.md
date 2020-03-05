# runc\(带整理\)

runC -- runC is a CLI tool for spawning and running containers according to the OCI specification.

runc list  == \# docker ps

容器标准格式也要求容器把自身运行时的状态持久化到磁盘中，这样便于外部的其它工具对此信息使用和演绎。该运行时状态以JSON格式编码存储。 推荐把运行时状态的JSON文件存储在临时文件系统中以便系统重启后会自动移除。

基于Linux内核的操作系统，该信息应该统一地存储在/run/opencontainer/containers目录，该目录结构下以容器ID命名的文件夹（/run/opencontainer/containers//state.json）中存放容器的状态信息并实时更新。有了这样默认的容器状态信息存储位置以后，外部的应用程序就可以在系统上简便地找到所有运行着的容器了。 But:CoreOS 在/run/docker/libcontainerd/Container\_id下

docker 1。18后目录变了\(/run/docker/containerd/C\_ID\)

