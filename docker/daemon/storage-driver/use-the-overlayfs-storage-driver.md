# Use the OverlayFS storage driver

OverlayFS is a modern _union filesystem_ that is similar to AUFS, but faster and with a simpler implementation. Docker provides a storage driver for OverlayFS.

This topic refers to the Linux kernel driver as `OverlayFS` and to the Docker storage driver as `overlay` or `overlay2`.

overlayFS有两种实现即：overlay2 和 overlay。

> **Note**: If you use OverlayFS, use the `overlay2` driver rather than the `overlay` driver, because it is more efficient in terms of inode utilization. To use the new driver, you need version 4.0 or higher of the Linux kernel.
>
> For more information about differences between `overlay` vs `overlay2`, refer to [Select a storage driver](https://docs.docker.com/engine/userguide/storagedriver/selectadriver/#overlay-vs-overlay2).

**How the overlay2 driver works**

要分析：/var/lib/docker/overlay2/ 和 /var/lib/docker/image/overlay2（记录镜像之间的关系即如何组成rootfs）

The `overlay2` driver natively supports up to 128 lower OverlayFS layers. This capability provides better performance for layer-related Docker commands such as `docker build` and `docker commit`, and consumes fewer inodes on the backing filesystem.

**overlayFS**

overlayFS也是一种Unionfs，它的核心功能是，将宿主机上多个目录下的内容整合成在一个目录下，用于集中访问。一些术语如下：

* layers：所有用于联合挂载的目录都被叫做layer.
* union mount point: 联合挂载，最终统一显示layers中的所有内容
* lowerdir: lower directory
* upperdir: upper directory ，上层目录与下层目录文件相同时，上层目录中的文件会屏蔽下层目录中的相同文件。

**Image and container layers on-disk**

因为docker1.10开始，使用基于内容的寻址，因此目录名和镜像层的id不一致，这点会在image部分详细介绍。

The image layer IDs do not correspond to the directory IDs.

镜像和容器的layers都存放在`<Docker Root Dir>/overlay2/`目录中.

```text
# 统计拉取镜像前layers number
$ ls -l /var/lib/docker/overlay2/ |wc -l
64
# pull images
$ docker pull nginx
Using default tag: latest
latest: Pulling from library/nginx
68ced04f60ab: Pull complete 
c4039fd85dcc: Pull complete 
c16ce02d3d61: Pull complete 
Digest: sha256:380eb808e2a3b0dd954f92c1cae2f845e6558a15037efefcabc5b4e03d666d03
Status: Downloaded newer image for nginx:latest
​
# 统计拉取镜像后layers number，增加了三层即image layers number
$ ls -l /var/lib/docker/overlay2/ |wc -l
67
## 因为docker1.10开始，使用基于内容的寻址，因此目录名和镜像层的id不一致
## The image layer IDs do not correspond to the directory IDs.
## 查找出刚刚新建的目录
$ find /var/lib/docker/overlay2/ -maxdepth 1 -type d  -ctime -1
/var/lib/docker/overlay2/
/var/lib/docker/overlay2/l
/var/lib/docker/overlay2/c00d18aff05c71a045b727267076c606b2311c33dece0a4bf08c26c57e4b7397
/var/lib/docker/overlay2/e52349985914b5cb38ccd181a5c1b1ac895b528030ef198932eed52db4d7e675
/var/lib/docker/overlay2/e4c770efa8e4a2b227ba444d036beb5bf77b7cefe18848a48d3f615275d3ae7f
​
## 各个目录中的内容
$ ls /var/lib/docker/overlay2/c00d18aff05c71a045b727267076c606b2311c33dece0a4bf08c26c57e4b7397
diff link lower work 
## The layer is lowest layer
$ ls /var/lib/docker/overlay2/e52349985914b5cb38ccd181a5c1b1ac895b528030ef198932eed52db4d7e675
diff  link
## mild layer
$ ls /var/lib/docker/overlay2/e4c770efa8e4a2b227ba444d036beb5bf77b7cefe18848a48d3f615275d3ae7f
diff  link  lower  work
​
## run a container
$ docker run -itd a1523e859360
14879c18bc421081212bedd6b977b127a4129c9ca4a43792ff5d5cb2462d54bb
​
## 69 - 67 = 2， 又多了two layers
## container layer： writeable layer
## container layer-init：docker自动生成的最上层只读层，主要放置了容器运行所必要的文件系统
$ ls -l /var/lib/docker/overlay2/ |wc -l
69
$ find /var/lib/docker/overlay2/ -maxdepth 1 -type d  -ctime -1
/var/lib/docker/overlay2/
/var/lib/docker/overlay2/l
/var/lib/docker/overlay2/c00d18aff05c71a045b727267076c606b2311c33dece0a4bf08c26c57e4b7397
/var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d
/var/lib/docker/overlay2/e52349985914b5cb38ccd181a5c1b1ac895b528030ef198932eed52db4d7e675
/var/lib/docker/overlay2/e4c770efa8e4a2b227ba444d036beb5bf77b7cefe18848a48d3f615275d3ae7f
/var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d-init
​
## container layers
$ ls /var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d
diff  link  lower  merged  work
$ ls /var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d-init/
diff  link  lower  work
​
## MergedDir
$ docker inspect 14879c18bc|grep -i merged
                "MergedDir": "/var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d/merged",
​
```

这几个文件和目录的作用：

* link ：该文件记录了 shortened layer identifiers， 在使用mount进行联合挂载时就是使用这个identity

  避免mount参数过长，超过页面大小。

  ```text
  $ cat /var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d/link  
  I6Z6WSDKRZXYY4U746VZKVT6EC
  $ ls -l /var/lib/docker/overlay2/l/I6Z6WSDKRZXYY4U746VZKVT6EC                               
  lrwxrwxrwx 1 root root 72 Mar  3 20:45 /var/lib/docker/overlay2/l/I6Z6WSDKRZXYY4U746VZKVT6EC -> ../9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d/diff
  ​
  ```

* diff：a directory called `diff` which contains the layer’s contents. 该目录包含了layer中的所有file.

  ```text
  # 不同layers里面包含的file也是不同的，通常the lowest layer中包含内容比较多
  $ ls /var/lib/docker/overlay2/e52349985914b5cb38ccd181a5c1b1ac895b528030ef198932eed52db4d7e675/diff/
  bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
  ​
  # 呐，the layer olny has a few files.
  $ tree /var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d/diff/
  /var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d/diff/
  |-- run
  |   `-- nginx.pid
  `-- var
      `-- cache
          `-- nginx
              |-- client_temp
              |-- fastcgi_temp
              |-- proxy_temp
              |-- scgi_temp
              `-- uwsgi_temp
  ​
  9 directories, 1 file
  ​
  ```

* lower: 记录了该layer下面所依赖的全部lower layers

  ```text
  ## 越靠上层的layer，它的lower文件中包含的内容越多
  $ cat /var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d/lower 
  l/PLVAAASBWIQO2YONUYSVV5PKBD:l/WPL4JA7J43PH3PNKK2NZPAKHK4:l/IHG5Z4I4HWCMEU2I2GRHXM4BC4:l/T2YBV4KKYOCGEZCFQYUJSYD3WRr
  ```

* work: work目录是用来完成如copy-on\_write的操作.

  a `work` directory which is used internally by OverlayFS.

  so,一般不要操作这个目录。

  ```text
  ## 没有文件 这个work是/var/lib/docker/overlay2/SHA-256-init/work/下的目录
  $ for i in $(find /var/lib/docker/overlay2/ -type d |grep "/work$"|grep -v diff); do ls $i;done
  work
  work
  work
  work
  work
  work
  work
  work
  ​
  ```

* merged: a `merged` directory, which contains the unified contents of its parent layer and itself。

  这个目录很好理解就是overlayFS的mount point. 它整合和显示所有layers\(container & image\)中的files.

  ```text
  ## 查看 the lowest layers中一个文件的内容
  $ cat /var/lib/docker/overlay2/e52349985914b5cb38ccd181a5c1b1ac895b528030ef198932eed52db4d7e675/diff/etc/host.conf
  multi on
  ​
  # 进入容器修改它
  $ docker exec -it 148 bash
  # 清空
  root@14879c18bc42:/# > /etc/host.conf 
  root@14879c18bc42:/# exit
  ​
  # image layers中该文件没有变化
  $ cat /var/lib/docker/overlay2/e52349985914b5cb38ccd181a5c1b1ac895b528030ef198932eed52db4d7e675/diff/etc/host.conf
  multi on
  ​
  # 观察容器layer中做了刚刚修改的host.conf文件
  $ ls /var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d/merged/etc/host
  host.conf  hostname   hosts    
  # 查看内容是, 空白
  $ cat /var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d/merged/etc/host.conf
  root@fishong:~# 
  ​
  ## 例子二： 在merged中创建文件，在容器中观察是否也显示
  $ ls /var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d/merged/tmp/
  $ touch  /var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d/merged/tmp/a.txt 
  $ echo "test merged" > /var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d/merged/tmp/a.txt
  $ docker exec -it 148 cat /tmp/a.txt
  test merged
  ​
  ## the container layer的diff 和 merged 给人的感觉是 -- "同步的"
  ## 0.0 
  $ touch /var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d/diff/tmp/b.txt
  $ ls /var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d/merged/tmp/   
  a.txt  b.txt
  ​
  ```

**container\_layer-init**

`container_layer-init`是docker自动生成的最上层只读层，**主要放置了容器运行所必要的文件系统**.

例如 `/proc` 进程信息，`/etc/hostname`，`/resolve.conf`等。

```text
$ tree /var/lib/docker/overlay2/60476e584bdd9a2e028cf3f315fa6fb084b82885a07f466c546ddc139a981e1a-init/
/var/lib/docker/overlay2/60476e584bdd9a2e028cf3f315fa6fb084b82885a07f466c546ddc139a981e1a-init/
|-- diff
|   |-- dev
|   |   |-- console
|   |   |-- pts
|   |   `-- shm
|   |-- etc
|   |   |-- hostname
|   |   |-- hosts
|   |   |-- mtab -> /proc/mounts
|   |   `-- resolv.conf
|   |-- proc
|   `-- sys
|-- link
|-- lower
`-- work
    `-- work
​
9 directories, 7 files
```

“6abc47……”为**读写层**，“6abc47…..-init”为**初始层**。初始层中大多是初始化容器环境时，与容器相关的环境信息，如容器主机名，主机host信息以及域名服务文件等。**所有对容器做出的改变都记录在读写层**

删除了`container_layer-init`目录后，不会影响容器的"运行",因为它依赖的实际文件在`container_layer 及其指向的image_layers`，但是影响容器中的显示。

 删除了`container_layer-init`目录后 ， `docker exec -it container_id ls /`显示空白-- \(但是cd /var\)还是可以，因为文件没有消失，只是少了一个显示的入口。

but，由于文件还是存在的，所以依然可以运行。

Note： docker restart 不会触发remount操作，所以丢失了init-layer还可以restart。但是docker stop之后再进行start操作就挂不上去了。

```text
​
$ mount |grep merged|grep f4114
overlay on /var/lib/docker/overlay2/f4114ceb335cfb7d807d0cb28bd90762c09ffec8b06cad72efda6ef7e9e1a83f/merged type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay2/l/Z434XCFK4HMQKS5T23XIUVM5UK:/var/lib/docker/overlay2/l/4XRGHECIVYZWBM5I2NNDSMHGQA:/var/lib/docker/overlay2/l/QPZLV3N6ZOE6RNZDCSC75M546E:/var/lib/docker/overlay2/l/6PT67736YLFUUEN5JTRVNNENVS,upperdir=/var/lib/docker/overlay2/0126f3499108f446a3571db15f40d251b2c78d130b0bf8fed13de6f1ebd10dc8/diff,workdir=/var/lib/docker/overlay2/0126f3499108f446a3571db15f40d251b2c78d130b0bf8fed13de6f1ebd10dc8/work)
​
$ docker run -d nginx:1.12
4d7a9411c107157e99f9b07bd77e131d9a1c75f8e744dc8014e99092fda98d80
$ docker inspect 4d7 |grep -i merged
                "MergedDir": "/var/lib/docker/overlay2/f4114ceb335cfb7d807d0cb28bd90762c09ffec8b06cad72efda6ef7e9e1a83f/merge ",
## 删除container init layer directory
$ \rm -r /var/lib/docker/overlay2/f4114ceb335cfb7d807d0cb28bd90762c09ffec8b06cad72efda6ef7e9e1a83f-init/
## restart
$ docker restart 4d7
4d7
## 在容器中执行ls / 显示空白
$ docker exec -it 4d7 ls /
​
## 但这些文件其实还是都存在的
$ ls /var/lib/docker/overlay2/f4114ceb335cfb7d807d0cb28bd90762c09ffec8b06cad72efda6ef7e9e1a83f/merged/etc/hosts
/var/lib/docker/overlay2/f4114ceb335cfb7d807d0cb28bd90762c09ffec8b06cad72efda6ef7e9e1a83f/merged/etc/hosts
​
## 关闭容器后，无法再启动容器
$ docker stop 809
809
## overlay层丢失，导致启动失败
## 丢失 container init layer directory 后，容器无法进行正常挂载和启动了。
$ docker start 809
Error response from daemon: error creating overlay mount to /var/lib/docker/overlay2/8bdd72f57eb6166b731f21610ffb8650bcf1473c238c90a0b483fdc4ae89e3ec/merged: no such file or directory
Error: failed to start containers: 809
​
​
```

end

**l \(lowercase L\) directory**

The new `l` \(lowercase `L`\) directory contains shortened layer identifiers as symbolic links. **These identifiers are used to avoid hitting the page size limitation on arguments to the `mount` command.**

> `Docker Root Dir/overlay2/l` 这个目录下存了好多软链接
>
> ”l“目录包含一些符号链接作为缩短的层标识符. 这些缩短的标识符用来避免挂载时超出页面大小的限制

```text
## 缩短的层标识符，就是用在这里避免挂载时超出页面大小的限制
## 如果mount时，lowerdir参数使用sha256的全长，则如果layers过多就会超出页面大小-- 
$ mount |grep "type overlay"|head -n 1
overlay on /var/lib/docker/overlay2/b7fad7052ba9d05cd44bf901fe51ba47651ae66ba440fc8b649036eb64039be3/merged type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay2/l/P6RUAT6UM2Z3RJDMEXS
CVNKRZH:/var/lib/docker/overlay2/l/JAGFKZBLXZDUY3PGYCMRYXFMC7:/var/lib/docker/overlay2/l/IY6NEYTCHNFKSQ7YGUYTXGKJEC:/var/lib/docker/overlay2/l/STSVDUQICGN67AVTESFLCR5TJ4:/var/lib/docker/overlay2/l/RIYO5JKTBQTQP3VMYKN5ZSMC72:/var/lib/docker/overlay2/l/7ITYJCWF2RFVMWI4OHEKYSHGB2:/var/lib/docker/overlay2/l/JSN333VOASAHJI22UJKQDNVTM7:/var/lib/docker/overlay2/l/LG7KYRPVVDPT2LZ2GV6ZMIT3LB,upperdir=/var/lib/docker/overlay2/b7fad7052ba9d05cd44bf901fe51ba47651ae66ba440fc8b649036eb64039be3/diff,workdir=/var/lib/docker/overlay2/b7fad7052ba9d05cd44bf901fe51ba47651ae66ba440fc8b649036eb64039be3/work)
​
​
$ ls -l /var/lib/docker/overlay2/l
​
total 20
lrwxrwxrwx 1 root root 72 Jun 20 07:36 6Y5IM2XC7TSNIJZZFLJCS6I4I4 -> ../3a36935c9df35472229c57f4a27105a136f5e4dbef0f87905b2e506e494e348b/diff
lrwxrwxrwx 1 root root 72 Jun 20 07:36 B3WWEFKBG3PLLV737KZFIASSW7 -> ../4e9fa83caff3e8f4cc83693fa407a4a9fac9573deaf481506c102d484dd1e6a1/diff
lrwxrwxrwx 1 root root 72 Jun 20 07:36 JEYMODZYFCZFYSDABYXD5MF6YO -> ../eca1e4e1694283e001f200a667bb3cb40853cf2d1b12c29feda7422fed78afed/diff
lrwxrwxrwx 1 root root 72 Jun 20 07:36 NFYKDW6APBCCUCTOUSYDH4DXAT -> ../223c2864175491657d238e2664251df13b63adb8d050924fd1bfcdb278b866f7/diff
lrwxrwxrwx 1 root root 72 Jun 20 07:36 UL2MW33MSE3Q5VYIKBRN4ZAGQP -> ../e8876a226237217ec61c4baf238a32992291d059fdac95ed6303bdff3f59cff5/diff
​
```

Note：上面可以很明显的看出overlayFS的挂载点是`merged`目录。

**The lowest layer、mild layers and The container layer**

The lowest layer： 位于OverlayFS的最底层，只有`link`和`diff`

Mild layers: 除了最底层的images layers，都用`link`、`diff`、`work`和`lower`

The container layer: 作为容器目录，提供联合挂载点，包含的内容有：`link`、`diff`、`work`、`lower`和`merged`

Note： 只有running状态的的container才会有对应的merged目录。

```text
$ docker stop 148
148
$ ls /var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d/merged/ 
ls: cannot access '/var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d/merged/': No such file or directory
​
## with starting the  container , the merged directory appears.
$ docker start 148
148
$ ls /var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d/merged/tmp/
/var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d/merged/tmp/:
a.txt
​
​
```

**/var/lib/docker/image/overlay2**

前面讨论了镜像和容器是怎么存储，这里讨论下overlayFS怎么组织images的即如何将多个layers正确的组成rootfs为容器提供完整的运行环境。

```text
## 目录结构
$ tree  -L 2 /var/lib/docker/image/overlay2/
/var/lib/docker/image/overlay2/
|-- distribution
|   |-- diffid-by-digest
|   `-- v2metadata-by-diffid
|-- imagedb
|   |-- content （镜像详细信息：CPU architecture、configuration、the layers of rootfs等）
|   `-- metadata （空目录不讨论）
|-- layerdb （这个比较有意思，记录rootfs到/var/lib/docker/overlays/中的对应关系）
|   |-- mounts
|   |-- sha256
|   `-- tmp
`-- repositories.json
​
10 directories, 1 file
​
```

下面详细介绍下这些目录的作用：

* `imagedb/content/`: 保存了镜像ID命名的镜像配置文件，该配置记录的镜像的详细信息，包括rootfs的由哪些layers组成。

  **The image ID is a hash of the local image JSON configuration.**

  ```text
  ## 查看Nginx image ID
  $ docker images |grep nginx
  nginx     latest              a1523e859360        6 days ago          127MB
  ​
  ##  每个镜像都以一个已镜像id命名的配置文件来描述该镜像的信息和rootfs的组成。
  $ ls /var/lib/docker/image/overlay2/imagedb/content/sha256/a1523e859360df9ffe2b31a8270f5e16422609fe138c1636383efdc34b9ea2d6 
  /var/lib/docker/image/overlay2/imagedb/content/sha256/a1523e859360df9ffe2b31a8270f5e16422609fe138c1636383efdc34b9ea2d6
  $ cat /var/lib/docker/image/overlay2/imagedb/content/sha256/a1523e859360df9ffe2b31a8270f5e16422609fe138c1636383efdc34b9ea2d6|python -m json.tool |tail
      "os": "linux",
      "rootfs": {
          "diff_ids": [
              "sha256:f2cb0ecef392f2a630fa1205b874ab2e2aedf96de04d0b8838e4e728e28142da",
              "sha256:fe08d5d042ab93bee05f9cda17f1c57066e146b0704be2ff755d14c25e6aa5e8",
              "sha256:318be7aea8fc62d5910cca0d49311fa8d95502c90e2a91b7a4d78032a670b644"
          ],
          "type": "layers"
      }
  }
  ​
  ## The image ID is a hash of the local image JSON configuration.
  $ sha256sum /var/lib/docker/image/overlay2/imagedb/content/sha256/a1523e859360df9ffe2b31a8270f5e16422609fe138c1636383efdc34b9ea2d6
  a1523e859360df9ffe2b31a8270f5e16422609fe138c1636383efdc34b9ea2d6  /var/lib/docker/image/overlay2/imagedb/content/sha256/a1523e859360df9ffe2b31a8270f5e16422609fe138c1636383efdc34b9ea2d6
  ```

* `layerdb/mounts` : 容器挂载目录，记录了一个容器启动时的

  ```text
  ## 都是以容器ID命名的目录
  $ ls /var/lib/docker/image/overlay2/layerdb/mounts/
  101d536509fc4321387b2b98bd04ee0cd79847d31bdbc68bb02c2c7319875358  b184047a71fc2b2faafcee06f9951b1283ee8db36ab5e38dbaf94f2b2a0f9fe2
  14879c18bc421081212bedd6b977b127a4129c9ca4a43792ff5d5cb2462d54bb  ca670526971e0964f030e7aa34d98281d5b753e17e552d2fae416f748d90cef8
  3dfabc245faa7d7f229145d22e3a95f95d4000753b44a2164b967c6cfadb5177
  docker ps -qa --no-trunc
  b184047a71fc2b2faafcee06f9951b1283ee8db36ab5e38dbaf94f2b2a0f9fe2
  14879c18bc421081212bedd6b977b127a4129c9ca4a43792ff5d5cb2462d54bb
  ca670526971e0964f030e7aa34d98281d5b753e17e552d2fae416f748d90cef8
  101d536509fc4321387b2b98bd04ee0cd79847d31bdbc68bb02c2c7319875358
  3dfabc245faa7d7f229145d22e3a95f95d4000753b44a2164b967c6cfadb5177
  ​
  # 目录结构
  $ tree /var/lib/docker/image/overlay2/layerdb/mounts/14879c18bc421081212bedd6b977b127a4129c9ca4a43792ff5d5cb2462d54bb/
  /var/lib/docker/image/overlay2/layerdb/mounts/14879c18bc421081212bedd6b977b127a4129c9ca4a43792ff5d5cb2462d54bb/
  |-- init-id
  |-- mount-id
  `-- parent
  ​
  0 directories, 3 files
  ​
  ## init-id 即创建容器时docker自动生成的包含初始化文件的目录，删除init-id指向的目录后，容器无法启动
  $ cd  /var/lib/docker/image/overlay2/layerdb/mounts/14879c18bc421081212bedd6b977b127a4129c9ca4a43792ff5d5cb2462d54bb/
  $ cat init-id 
  9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d-ini
  ​
  ## 查看init-id指向的layers中包含的文件内容
  $ tree  /var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d-init/
  /var/lib/docker/overlay2/9409fe76eb5a9318a8ab2ecf93f0d3b5153d7f9c6d53d15a513b6316f669af3d-init/
  |-- diff
  |   |-- dev
  |   |   |-- console
  |   |   |-- pts
  |   |   `-- shm
  |   `-- etc
  |       |-- hostname
  |       |-- hosts
  |       |-- mtab -> /proc/mounts
  |       `-- resolv.conf
  |-- link
  |-- lower
  `-- work
      `-- work
  ​
  7 directories, 7 files
  ​
  ```

* `layerdb/sha256/`: 所有image layers的元数据信息

  ```text
  $ ls -l  /var/lib/docker/overlay2/|wc -l
  69
  $ docker ps -qa
  14879c18bc42
  ca670526971e
  101d536509fc
  3dfabc245faa
  #  60 = 69 - 2*4 -1
  #  layerdb下的layers和image layers是一致的。
  $ ls -l  /var/lib/docker/image/overlay2/layerdb/sha256/|wc -l
  60
  ​
  ## 目录内容
  $ ls /var/lib/docker/image/overlay2/layerdb/sha256/f2cb0ecef392f2a630fa1205b874ab2e2aedf96de04d0b8838e4e728e28142da/
  cache-id  diff  size  tar-split.json.gz
  $  ls /var/lib/docker/image/overlay2/layerdb/sha256/faa5b0b261bf657b7449f908818f1868baaf048dfc6e336d7e5c3dda438720b2/
  cache-id  diff  parent  size  tar-split.json.gz
  ​
  ## 文件说明：
  cache-id：指向/var/lib/docker/overlay2/image_layer_id
  diff: 该layer本身的sha256 diff_id，若是第一层，则diff_id == ChainID
  size: 该layer大小，单位Byte
  parent: 指向上层ChainID,最上层的layer没有parent文件
  tar-split.json.gz:
  ​
  ​
  ## 如下，parent指向上层chainID
  $ grep -inr 'f2cb0ecef392f2a630fa1205b874ab2e2aedf96de04d0b8838e4e728e28142da' /var/lib/docker/image/overlay2/layerdb/sha256/
  /var/lib/docker/image/overlay2/layerdb/sha256/f2cb0ecef392f2a630fa1205b874ab2e2aedf96de04d0b8838e4e728e28142da/diff:1:sha256:f2cb0ecef392f2a630fa1205b874ab2e2aedf96de04d0b8838e4e728e28142da
  /var/lib/docker/image/overlay2/layerdb/sha256/4245b7ef9b70e3b2975ed908c7d68ce5f03972d8be702b0ed491e32445b42b8f/parent:1:sha256:f2cb0ecef392f2a630fa1205b874ab2e2aedf96de04d0b8838e4e728e28142da
  ```

**image layers ChainID**

layer的元数据信息都放在了`Docker Root Dir`/image/`Storage Driver`/layerdb/sha256目录下，目录名称是layer的chainid，由于最底层的layer的chainid和diffid相同，所以`chainid` 就是本身即Image Config中记录的diff\_id就是它的目录名。

**chainID:**docker内容寻址机制采用的索引ID，其值根据当前层和所有Parent层的diffID算得：

* 若该镜像层是最底层，那么其chainID 和 diffID 相同
* 否则，chainID=sha256\(父层chainID+" "+本层diffID\)

  `echo -n "sha256:Parent_chain_id sha256:diff_id" |sha256sum`

计算chainid时，用到了所有parent layer的信息，从而能保证根据chainid得到的rootfs是唯一的。

```text
$ docker info|grep  'Storage Driver'
Storage Driver: overlay2
$ docker info|grep  'Root'
Docker Root Dir: /var/lib/docker
​
​
1. 找到镜像的Image Config 文件,拿到镜像的RootFS由哪些layers组成
$ docker images nginx          
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               <none>              568c4670fa80        5 days ago          109MB
$ cat /var/lib/docker/image/overlay2/imagedb/content/sha256/568c4670fa800978e08e4a51132b995a54f8d5ae83ca133ef5546d092b864acf |jq
...
  "rootfs": {
    "type": "layers",
    # 从上到下依次是从底层到顶层
    # 确实是3层，docker pull nginx也可以看到只下载了三层。
    "diff_ids": [
      "sha256:ef68f6734aa485edf13a8509fe60e4272428deaf63f446a441b79d47fc5d17d3",  ##最底层
      "sha256:ad5345cbb119f7c720123e3adf28b164143e4157ca6e46a629ca694e75f7825f",  ## 第二层
      "sha256:ece4f9fdef598687f23d39643bacbf2c609201b087b93bbae81b931da72d2a64"   ## 第三层
    ]
2. 根据chainid目录下cache-id找到具体的layer data
## 底层的chainID = diffid 即目录名
$ cd /var/lib/docker/image/overlay2/layerdb/sha256
# cache-id是docker下载layer的时候在本地生成的一个随机uuid，
# cache-id文件内容指向真正存放layer文件的地方
$ cat ef68f6734aa485edf13a8509fe60e4272428deaf63f446a441b79d47fc5d17d3/cache-id
b1acb2ed8732e3d4d07aa44444bd98b4863df2634212c390ecdc337821376f17
# the image lowest layer directory
$ cd /var/lib/docker/overlay2/b1acb2ed8732e3d4d07aa44444bd98b4863df2634212c390ecdc337821376f17
$ ls
diff  link
$ du -sh ./
62M ./
3. 求出第二层的ChainID即第二层的目录名
# 用上层的ChainID + 本层的 DiffID求出
$ echo  -n "sha256:ef68f6734aa485edf13a8509fe60e4272428deaf63f446a441b79d47fc5d17d3 sha256:ad5345cbb119f7c720123e3adf28b164143e4157ca6e46a629ca694e75f7825f" | sha256sum  
6b9d35d8d75115937cd78da275f527cccef672cbd71f34062dffe2e930fd7e13  -
$ cd /var/lib/docker/overlay2/layerdb/sha256
​
## 第二层中的parent记录的就是第一层的chainid
$ cat sha256/6b9d35d8d75115937cd78da275f527cccef672cbd71f34062dffe2e930fd7e13/parent   
sha256:ef68f6734aa485edf13a8509fe60e4272428deaf63f446a441b79d47fc5d17d3
​
$ cat sha256/6b9d35d8d75115937cd78da275f527cccef672cbd71f34062dffe2e930fd7e13/cache-id 
7d407e11dcbde25b1685eb5a5fde08a653ba525a29d0ebc82e39ed98bfbe5a1f
## 实际layers 大小
$ du -sh /var/lib/docker/overlay2/7d407e11dcbde25b1685eb5a5fde08a653ba525a29d0ebc82e39ed98bfbe5a1f/ 
54M /var/lib/docker/overlay2/7d407e11dcbde25b1685eb5a5fde08a653ba525a29d0ebc82e39ed98bfbe5a1f/
​
--- 第三层的cache-id和parent
## 计算第三层的chainid： 即第二层的chainid 与 第三层diffid
$ echo  -n "sha256:6b9d35d8d75115937cd78da275f527cccef672cbd71f34062dffe2e930fd7e13 sha256:ece4f9fdef598687f23d39643bacbf2c609201b087b93bbae81b931da72d2a64" | sha256sum -
ac0442c0fafd48e24a96fa3099ea7ad20012c8759e1dd03dd387dbfbe382984c  -
$ ls /var/lib/docker/image/overlay2/layerdb/sha256/ac0442c0fafd48e24a96fa3099ea7ad20012c8759e1dd03dd387dbfbe382984c/
cache-id  diff  parent  size  tar-split.json.gz
 
 0.0 yes
```

end

**How the overlay driver works**

OverlayFS layers two directories on a single Linux host and presents them as a single directory. These directories are called _layers_ and the unification process is referred to as a _union mount_. OverlayFS refers to the lower directory as `lowerdir` and the upper directory a `upperdir`. The unified view is exposed through its own directory called `merged`.

> overlay 具体怎么组织镜像的就不研究了--

overlay只提供两个目录的联合挂载，如下图，`lowerdir`表示image layer，`upperdir`表示container layer, 而`merged`就是union mount point.

![](../../../.gitbook/assets/overlay_constructs.jpg)

lowerdir聚合了image layers中的所有文件，作为只读层提供给容器使用，当修改lowerdir目录中的文件时（eg：file2），同样是通过COW机制将file2复制一份到upperdir供于容器修改。

```text
## 可以对比overlay2的挂载，overlay的lowerdir只有一个目录
$ mount |grep overlay|grep merge|head -1
overlay on /var/lib/docker/overlay/a775e191f740e049ad5d14f8fb052f15e80dfc66462436b575ea1cb737383980/merged type overlay (rw,relatime,context="system_u:object_r:svirt_lxc_file_t:s0:c66,c424",lowerdir=/var/lib/docker/overlay/1c69ae68e4c887e573c3681afa9f053a3c5b9c0fe6e38606062b48ee687aa510/root,upperdir=/var/lib/docker/overlay/a775e191f740e049ad5d14f8fb052f15e80dfc66462436b575ea1cb737383980/upper,workdir=/var/lib/docker/overlay/a775e191f740e049ad5d14f8fb052f15e80dfc66462436b575ea1cb737383980/work)
​
## overlay: the container layer
$ ls -l /var/lib/docker/overlay/<directory-of-running-container>
​
total 16
-rw-r--r-- 1 root root   64 Jun 20 16:39 lower-id
drwxr-xr-x 1 root root 4096 Jun 20 16:39 merged
drwxr-xr-x 4 root root 4096 Jun 20 16:39 upper
drwx------ 3 root root 4096 Jun 20 16:39 work
​
```

* `lower-id` : The `lower-id` file contains the ID of the top layer of the image the container is based on, which is the OverlayFS `lowerdir`.
* merged: The `merged` directory is the union mount of the `lowerdir` and `upperdir`, which comprises the view of the filesystem from within the running container.

  视图目录即挂载点

* upper： The `upper` directory contains the contents of the container’s read-write layer, which corresponds to the OverlayFS `upperdir`.
* work： OverlayFS用于COW的目录

**hard link**

用于`lowerdir`包含了image layers中的全部文件，而且它不是Union FS实现的，那么它是怎么聚合全部镜像中的文件呢？

答案就是：硬链接

The image layer directories contain the files unique to that layer as well as hard links to the data that is shared with lower layers. This allows for efficient use of disk space.

```text
$ docker pull ubuntu
​
Using default tag: latest
latest: Pulling from library/ubuntu
​
5ba4f30e5bea: Pull complete
9d7d19c9dc56: Pull complete
ac6ad7efd0f9: Pull complete
e7491a747824: Pull complete
a3ed95caeb02: Pull complete
Digest: sha256:46fb5d001b88ad904c5c732b086b596b92cfb4a4840a3abd0e35dbb6870585e4
Status: Downloaded newer image for ubuntu:latest
​
## 镜像几层就会产生几个layers
## The image layer IDs do not correspond to the directory IDs.
$ ls -l /var/lib/docker/overlay/
​
total 20
drwx------ 3 root root 4096 Jun 20 16:11 38f3ed2eac129654acef11c32670b534670c3a06e483fce313d72e3e0a15baa8
drwx------ 3 root root 4096 Jun 20 16:11 55f1e14c361b90570df46371b20ce6d480c434981cbda5fd68c6ff61aa0a5358
drwx------ 3 root root 4096 Jun 20 16:11 824c8a961a4f5e8fe4f4243dab57c5be798e7fd195f6d88ab06aea92ba931654
drwx------ 3 root root 4096 Jun 20 16:11 ad0fe55125ebf599da124da175174a4b8c1878afe6907bf7c78570341f308461
drwx------ 3 root root 4096 Jun 20 16:11 edab9b5e5bf73f2997524eebeac1de4cf9c8b904fa8ad3ec43b3504196aa3801
​
$ ls -i  merged/bin/less 
1571468 merged/bin/less
# 查看它的lowerdir
$ cat lower-id  
6556a9754ef4b63428a29323451a242b63c470d76fa60c9b6d666c6839244c2d
# 是同一个文件，所以不会浪费磁盘空间
$ ls -i ../6556a9754ef4b63428a29323451a242b63c470d76fa60c9b6d666c6839244c2d/root/bin/less 
1571468 ../6556a9754ef4b63428a29323451a242b63c470d76fa60c9b6d666c6839244c2d/root/bin/less
```

但是，通过硬链接方式实现文件共享有个问题。

由于hard link共享inode，所以如果只修改了inode中的信息，文件会被认为没有修改的。

比如，在容器中执行`chown test:test /etc/hosts` 那么这个操作会穿透到镜像层，因为宿主信息是包含在inode里面的... overlay会认为你没有对文件进行修改，但现实是你不但修改了容器中的这个文件，连本机镜像的文件也被修改了。

**overlay 和 overlay2下编译镜像的区别**

Dockerfile大致如下：

```text
centos:7.5
   ↑
base-os (创建了一系列目录, 安装了部分命令工具)
   ↑
open-jdk (下载并部署 jdk 至 /opt/soft 下, 335MB)
   ↑
application (应用部署, 且执行了 chown user:user /opt)
​
```

`chown` 命令在`overlay`下不会引起镜像体积增大，但是在`overlay2`中会引起镜像体积增大。

> chown只修改文件的 metadata 属性,即inode中包含的属性有变化

`overlay` 和 `overlay2`的本质区别是，`overlay`使用硬链接来实现文件共享、

简单来说, 硬链接是有着相同 inode 号仅文件名不同的文件, 因此硬链接存在以下几点特性:

* 文件有相同的 inode 及 data block
* 只能对已存在的文件进行创建
* 不能交叉文件系统进行硬链接的创建
* 不能对目录进行创建，只可对文件创建
* 删除一个硬链接文件并不影响其他有相同 inode 号的文件

**由于 overlayfs 的硬链接引用的原理, chown 只更改 metadata 数据, 而 metadata 是作用在 inode 节点上的, 一个硬链接文件的属性被修改, 同步的, 所有指向这个 inode 节点的文件的属性都会变化. 也就是说, overlayfs 通过硬链接将文件挂载到新的镜像层之后, 对里面已存在的文件做 chown 操作, 意味着底层\(只读层\)的文件属性也会被修改, docker 就会认为这个操作没有引发任何文件的变化, 所以就不会再拷贝那335MB 的文件. 而 overlayfs2 的每层都是独立的, 即使文件属性的变化, 也会导致整个文件被拷贝, 所以在 overlayfs2 下, 会产生335MB 的空间浪费**。

厉害！

**How container reads and writes work with overlay or overlay2**

这部分都是官网上的内容，我就不翻译了，作了些补充。

核心：OverlayFS是文件层面的文件系统，COW时，需要cop the entire file,有性能随后。

 Union FS的read、write、delete和rename.其中rename需要自己捕获异常，从而完成不同layers之间的copy.

 如果mv的文件在同层则执行传统的`rename()`,如果不在同层则需要借助`-EXDEV`先copy然后再`rename()`

**Reading files**

Consider three scenarios where a container opens a file for read access with overlay.

* **The file does not exist in the container layer**: If a container opens a file for read access and the file does not already exist in the container \(`upperdir`\) it is read from the image \(`lowerdir)`. This incurs very little performance overhead.
* **The file only exists in the container layer**: If a container opens a file for read access and the file exists in the container \(`upperdir`\) and not in the image \(`lowerdir`\), it is read directly from the container.

  > 要的文件不包含在lowerdir，则直接读容器里面的。

* **The file exists in both the container layer and the image layer**: If a container opens a file for read access and the file exists in the image layer and the container layer, the file’s version in the container layer is read. Files in the container layer \(`upperdir`\) obscure files with the same name in the image layer \(`lowerdir`\).

**Modifying files or directories**

Consider some scenarios where files in a container are modified.

* **Writing to a file for the first time**: The first time a container writes to an existing file, that file does not exist in the container \(`upperdir`\). The `overlay`/`overlay2` driver performs a _copy\_up_ operation to copy the file from the image \(`lowerdir`\) to the container \(`upperdir`\). The container then writes the changes to the new copy of the file in the container layer.

  > copy-on-write。

  However, OverlayFS works at the file level rather than the block level. This means that all OverlayFS copy\_up operations copy the entire file, even if the file is very large and only a small part of it is being modified. This can have a noticeable impact on container write performance. However, two things are worth noting:

  * The copy\_up operation only occurs the first time a given file is written to. Subsequent writes to the same file operate against the copy of the file already copied up to the container.

    > The copy\_up operation only occurs the first time a given file is written to.
    >
    > 之后对这个文件的所有写入操作就不会copy了，因为已经在upperdir了.
    >
    > OverlayFS copy\_up 操作是文件层面的即将整个文件copy到upperdir。so，当一个很大的文件做了很小的修改的操作也会copy the entire file，会对容器的写性能有noticeable impact.
    >
    > 因为OverlayFS是file level，而不是block level.

  * OverlayFS only works with two layers. This means that performance should be better than AUFS, which can suffer noticeable latencies when searching for files in images with many layers. This advantage applies to both `overlay` and `overlay2` drivers. `overlayfs2` will be slightly less performant than `overlayfs` on initial read, because it has to look through more layers, but it caches the results so this is only a small penalty。

    > OverlayFS只工作在两层目录，所有搜索文件很快，延迟低。

* **Deleting files and directories**:
  * When a _file_ is deleted within a container, a _whiteout_ file is created in the container \(`upperdir`\). The version of the file in the image layer \(`lowerdir`\) is not deleted \(because the `lowerdir` is read-only\). However, the whiteout file prevents it from being available to the container.

    > 删除文件：在uppdedir创建一个同名的_空白文件_ ，来显性的掩盖住lowerdir下的文件。因为lowerdir是只读的镜像层。此外， the whiteout file prevents it from being available to the container.
    >
    > OvarlayFS在容器层也会控制这个空白文件使其在容器中不可用。
    >
    > ```text
    > # docker inspect c87e1847a|grep overlay -A 4
    >       "Data": {
    >           "LowerDir": "/var/lib/docker/overlay/fb3e401c172667d4e8aef6b81c8769eaa427b1d8a5c0336974e3d2ee288c851a/root",
    >           "MergedDir": "/var/lib/docker/overlay/116f46eeacd8708edccd93746019d836adc7e27ab99796f1593f0e87d3b731d6/merged",
    >           "UpperDir": "/var/lib/docker/overlay/116f46eeacd8708edccd93746019d836adc7e27ab99796f1593f0e87d3b731d6/upper",
    >           "WorkDir": "/var/lib/docker/overlay/116f46eeacd8708edccd93746019d836adc7e27ab99796f1593f0e87d3b731d6/work"
    >       }
    > ​
    > ###  实验开始
    > $ cd /var/lib/docker/overlay/116f46eeacd8708edccd93746019d836adc7e27ab99796f1593f0e87d3b731d6
    > $ find ./ -name "ls" 
    > ./merged/usr/bin/ls
    > $ ls -l ./merged/usr/bin/ls 
    > -rwxr-xr-x. 36 root root 117656 Nov  6  2016 ./merged/usr/bin/ls
    > ​
    > ​
    > ### 进入容器删除/usr/bin/ls
    > $ docker exec -it c87e1847a /bin/bash
    > $ ll /usr/bin/ls
    > ls        lsblk     lscpu     lsinitrd  lsipc     lslocks   lslogins  lsns      
    > $ ll /usr/bin/ls
    > -rwxr-xr-x. 36 root root 117656 Nov  5  2016 /usr/bin/ls
    > $ \rm /usr/bin/ls
    > rm: cannot remove '/usr/bin/ls': No such file or directory
    > $ exit
    > ​
    > ###再次查看OverlayFS
    > # 显而易见，ls此时在upperdir。自动掩盖了lowerdir中的ls
    > $ find ./ -name "ls"  
    > ./upper/usr/bin/ls
    > $ ls -l ./upper/usr/bin/ls
    > c---------. 1 root root 0, 0 Nov 16 22:53 ./upper/usr/bin/ls
    > $ cat ./upper/usr/bin/ls
    > cat: ./upper/usr/bin/ls: No such device or address
    > ​
    > ​
    > # docker history ec4 
    > IMAGE           CREATED BY                                      SIZE                
    > ec4c1ab7be63    /bin/sh -c rm /tmp/test.lol                     0 B                 
    > 9a57afeef51f    /bin/sh -c #(nop) ADD dir:8b2e149859d73cfaeb8   10.49 MB            
    > ff915470bdc3    /bin/sh -c #(nop) ENV LC_ALL=zh_CN.GBK          0 B    
    > <truncate>
    > PS：西津说：RUN rm 时，会使镜像变大。嗯哼，我验证了下是错的谢谢。（2017-12-07）
    > 哈哈，西津是对的，你看上面/tmp/test.lol并没有被删除，反而增加了一个whiteout文件的大小啊
    > 哈哈哈，虽然这个whiteout很小~~ 但是还是正确的不减反增。（2017-12-08）
    > ```
    >
    > 卧槽，c开头的是什么文件呢，不会确实如上所述，在upperdir创建一个同名的空白文件，表示该文件在container中已被删除。

  * When a _directory_ is deleted within a container, an _opaque directory_ is created within the container \(`upperdir`\). This works in the same way as a whiteout file and effectively prevents the directory from being accessed, even though it still exists in the image \(`lowerdir`\).

    > ```text
    > ## 同上咯
    > $ find ./  -type d -name "var"
    > ./merged/var
    > $ docker exec -it c87e1847a /bin/bash
    > [root@centos-1223253240-cdsll /]# \rm -r /var/                                      
    > [root@centos-1223253240-cdsll /]# exit
    > $ find ./  -type d -name "var"  //--已经没有var命名的目录了
    > $ find ./   -name "var"
    > ./upper/var
    > # //哈哈哈，也是变成文件了,whiteout
    > $ ls -l ./upper/var    
    > c---------. 1 root root 0, 0 Nov 16 23:05 ./upper/var
    > ​
    > PS：c开头的是字符（character）设备
    > ```
    >
    > white-outs
    >
    > Deletion of files is handled by a special file type called white-outs. The white-out file type is similar to negative dentries: they describe a filename which isn't there. This is used to mark a file in the lower read-only filesystem as deleted, since only the topmost layer can be modified. However, white-outs would require support from all the filesystems, to store and recognize such a special file type. Currently, there is a special type, `DT_WHT` defined in `include/linux/fs.h` which defines a white-out, but is not in use.
* **Renaming directories**: Calling `rename(2)` for a directory is allowed only when both the source and the destination path are on the top layer. Otherwise, it returns `EXDEV` error \(“cross-device link not permitted”\). Your application needs to be designed to handle `EXDEV` and fall back to a “copy and unlink” strategy.

  > ```text
  > $ docker exec -it 78e9f6 /bin/bash
  > $ mv /root/ /lol        
  > $ ls -ld lol
  > dr-xr-x---.   3 root root  4096 Jul 11 11:53 lol
  > ​
  > $ cd /var/lib/docker/overlay/b5f93fd9804cca94def1a4c318fe44b8b169d4046d86248e658c50a2ea42e495
  > # 这个视图层的lol该是upperdir的--因为lowerdir is read-only.
  > $ find ./ -type d -name "lol"
  > ./upper/lol    
  > ./merged/lol   
  > ​
  > 嗷嗷，上面意思是rename如果操作不同挂载层的文件，则会报错。即
  > -EXDEV is returned in a rename() operation if the source and destination file paths are on different mounted filesystems.
  > 对于这个问题，它说你的APP要能捕捉并处理这个异常,通过fall back（倒退）执行cp和unlink策略
  > Your application needs to be designed to handle EXDEV and fall back to a “copy and unlink” strategy.
  > ​
  > ```
  >
  > Calling `rename(2)` for a directory is allowed only when both the source and the destination path are on the top layer. Otherwise, it returns `EXDEV` error \(“cross-device link not permitted”\). Your application needs to be designed to handle `EXDEV` and fall back to a “copy and unlink” strategy.
  >
  > 答案在这里？
  >
  > Rename on union mounts is handled through `-EXDEV`. `-EXDEV` is returned in a `rename()` operation if the source and destination file paths are on different mounted filesystems. In such a case, the application, such as `mv`, resorts to a copy operation, and **unlinks the file from which the filesystem moved** . On union mounts, since any writes are performed in the topmost layer, a move operation to directories in the lower layers returns `-EXDEV`, which means the application must copy the file to the new directory. If both the source and destination of the `rename()` operation are in the topmost later, the traditional rename method is used.
  >
  > [https://lwn.net/Articles/312641/](https://lwn.net/Articles/312641/)
  >
  > 啧啧，一言以蔽之。
  >
  > OverlayFS不完全支持rename即遇到不同层之前的rename时，会报错，需要你自己捕获异常并处理它
  >
  > OverlayFS does not fully support the `rename(2)` system call. Your application needs to detect its failure and fall back to a “copy and unlink” strategy.
  >
  > 来来，再看看“copy and unlink”strategy
  >
  > ```text
  > ## 进入容器目录--
  > $ cd /var/lib/docker/overlay/b5f93fd9804cca94def1a4c318fe44b8b169d4046d86248e658c50a2ea42e495
  > # lowerdir 中的 cat
  > $ find ./ -name "cat"   
  > ./merged/usr/bin/cat
  > ​
  > ## 执行mv操作，将lowerdir中的cat命令，重名到topmost layer
  > $ docker exec -it 78e9f614608 /bin/bash
  > bash-4.2# mv  /usr/bin/cat  /usr/bin/lol
  > bash-4.2# ls -l /usr/bin/lol 
  > -rwxr-xr-x. 1 0 0 54080 Nov  5  2016 /usr/bin/lol
  > bash-4.2# exit
  > ​
  > ## 开始了开始了，好玩的开始了~~
  > # //fall back的 copy 加正常的 rename()
  > $ find ./ -name "lol"   
  > ./upper/usr/bin/lol
  > ./merged/usr/bin/lol
  > # //unlink 操作。建个同名的whiteout文件，遮蔽住lowerdir中的cat文件
  > # //实现删除效果，哈哈哈
  > $ find ./ -name "cat"   
  > ./upper/usr/bin/cat    
  > $ ls -l ./upper/usr/bin/cat
  > c---------. 1 root root 0, 0 Nov 21 08:55 ./upper/usr/bin/cat
  > ​
  > # 咦，有个好玩的想法。直接在container中创建c开头的字符设备文件会发生什么呢？？？
  > ​
  > [root@centos-0s56q tmp]# mknod test c 0 0      //哈哈哈，容器里面创建0，0 ，权限不足
  > mknod: 'test': Operation not permitted
  > [root@centos-0s56q tmp]# mknod test.test c 1 0 
  > [root@centos-0s56q tmp]# ls -l /tmp/test.test 
  > crw-r--r--. 1 root root 1, 0 Dec  6 11:24 /tmp/test.test
  > ​
  > # 宿主机上执行没有问题
  > [root@hz-hsyqzs-nginx-97-54 ~]# mknod /tmp/test.x c 0 0
  > [root@hz-hsyqzs-nginx-97-54 ~]# ll /tmp/test.x    
  > crw-r----- 1 root root 0, 0 Dec  6 19:18 /tmp/test.x
  > ​
  > # man mknod
  > NAME
  >  mknod - make block or character special files
  > SYNOPSIS
  >  mknod [OPTION]... NAME TYPE [MAJOR MINOR]  //后面两个数字是主次设备号
  >  -m, --mode=MODE
  >         set file permission bits to MODE, not a=rw - umask
  >  Both  MAJOR and MINOR must be specified when TYPE is b, c, or u, and they must be omitted when TYPE is p.  If MAJOR or MINOR begins with 0x or 0X, it is interpreted as hexadecimal;
  >  otherwise, if it begins with 0, as octal; otherwise, as decimal.  TYPE may be:
  > ​
  >  b      create a block (buffered) special file
  > ​
  >  c, u   create a character (unbuffered) special file
  > ​
  >  p      create a FIFO
  > ​
  > ```

**OverlayFS and Docker Performance\(mark\)**

Both `overlay2` and `overlay` drivers are more performant than `aufs` and `devicemapper`. In certain circumstances, `overlay2` may perform better than `btrfs` as well. However, be aware of the following details.

* **Page Caching**. OverlayFS supports page cache sharing. Multiple containers accessing the same file share a single page cache entry for that file. This makes the `overlay` and `overlay2` drivers efficient with memory and a good option for high-density use cases such as PaaS.

  > 多个容器可以通过同一个page cache来访问同一个文件。
  >
  > Linux Page Caching -- 好吧这里是一大块东西
  >
  > OverlayFS支持共享页面缓存，也就是说多个容器访问相同的文件，会共享相同的页面缓存。这使得overlay/overlay2能够有效使用内存，对于PaaS平台和高密度环境是很好的选择。

* **copy\_up**. As with AUFS, OverlayFS has to perform copy-up operations whenever a container writes to a file for the first time. This can add latency into the write operation, especially for large files. However, once the file has been copied up, all subsequent writes to that file occur in the upper layer, without the need for further copy-up operations.

  The OverlayFS `copy_up` operation is faster than the same operation with AUFS, because AUFS supports more layers than OverlayFS and it is possible to incur far larger latencies if searching through many AUFS layers. `overlay2` supports multiple layers as well, but mitigates any performance hit with caching.

  > 说了下copy\_up：意思是第一次执行copy时虽然会产生点延时，但是后面就快很多了。
  >
  > 而且比起来AUFS快了很多因为AUFS的多层会导致在search上面产生的延迟。
  >
  > overlay2对多层支持也会很好，但是会减少缓存带来的性能。

* **Inode limits**. Use of the `overlay` storage driver can cause excessive inode consumption. This is especially true in the presence of a large number of images and containers on the Docker host. The only way to increase the number of inodes available to a filesystem is to reformat it. To avoid running into this issue, it is highly recommended that you use `overlay2` if at all possible.

  > if at all possible，如果可能的话highly recommend 使用`overylay2` .因为`overlay` 在大量镜像和容器的情况下回导致inode被耗尽。
  >
  > ```text
  > # df -ih
  > Filesystem              Inodes IUsed IFree IUse% Mounted on
  > /dev/mapper/centos-root    36M  111K   36M    1% /
  > devtmpfs                  976K   418  975K    1% /dev
  > tmpfs                     978K     1  978K    1% /dev/shm
  > tmpfs                     978K   578  978K    1% /run
  > tmpfs                     978K    16  978K    1% /sys/fs/cgroup
  > /dev/vda1                 500K   350  500K    1% /boot
  > /dev/rbd0                 300M  526K  300M    1% /data
  > tmpfs                     978K     1  978K    1% /run/user/0
  > ​
  > 查看系统中inode的使用情况
  > ```

**Performance best practices**

The following generic performance best practices also apply to OverlayFS.

* **Use fast storage**: Solid-state drives \(SSDs\) provide faster reads and writes than spinning disks.
* **Use volumes for write-heavy workloads**: Volumes provide the best and most predictable performance for write-heavy workloads. This is because they bypass the storage driver and do not incur any of the potential overheads introduced by thin provisioning and copy-on-write. Volumes have other benefits, such as allowing you to share data among containers and persisting even when no running container is using them.

  > 卷为高负载工作提供了最佳和最可预测的性能。 这是因为它们绕过了存储驱动程序，不会产生精简配置和写入时复制引入的任何潜在开销。 卷还有其他好处，例如允许您在容器之间共享数据，并且即使在没有运行的容器的情况下保持持久化。（PVC - PersistentVolumeClaim）

**Limitations on OverlayFS compatibility**

To summarize the OverlayFS’s aspect which is incompatible with other filesystems:

* **open\(2\)**: OverlayFS only implements a subset of the POSIX standards. This can result in certain OverlayFS operations breaking POSIX standards. One such operation is the _copy-up_ operation. Suppose that your application calls `fd1=open("foo", O_RDONLY)` and then `fd2=open("foo", O_RDWR)`. In this case, your application expects `fd1` and `fd2` to refer to the same file. However, due to a copy-up operation that occurs after the second calling to `open(2)`, the descriptors refer to different files. The `fd1` continues to reference the file in the image \(`lowerdir`\) and the `fd2` references the file in the container \(`upperdir`\). A workaround for this is to `touch` the files which causes the copy-up operation to happen. All subsequent `open(2)` operations regardless of read-only or read-write access mode will be referencing the file in the container \(`upperdir`\).

  `yum` is known to be affected unless the `yum-plugin-ovl` package is installed. If the `yum-plugin-ovl`package is not available in your distribution such as RHEL/CentOS prior to 6.8 or 7.2, you may need to run `touch /var/lib/rpm/*` before running `yum install`. This package implements the `touch`workaround referenced above for `yum`.

  > 太多了，懒得看了
  >
  > OverlayFS的open操作仅仅实现了POSIX标准的一个子集。这必然导致OverlayFS的某些操作违反POSIX标准。必比如一个这样的复制操作。假设你的应用调用`fd1=open("foo", O_RDONLY)` and then `fd2=open("foo", O_RDWR)`.这导致你的应用期望得到指向同一个文件的两个文件描述符即`fd1` and `fd2` 然而，由于是copy-up操作导致调用 `open(2)` 后返回的`fd1` and `fd2` 指向不同的文件。**`fd1` 指向lowerdir `fd2`指向upperdir**.一个解决方式是 `touch` the files当copy\_cp操作发生时。这样所以的子`open(2)` 操作无论以read-only or read-write方式访问文件都会指向到upperdir。

* **rename\(2\)**: OverlayFS does not fully support the `rename(2)` system call. Your application needs to detect its failure and fall back to a “copy and unlink” strategy.

  > 下面是个实际的例子：不同layers之间进行mv即调用rename\(2\).docker fall back to a “copy and unlink” strategy.so，导致浪费了空间。

  ```text
   # docker history 6bbfc909fd2c
  ...                
  <missing>           6 months ago   /bin/sh -c mv /usr/local/jdk1.7.0_80 /usr/loc   306.3 MB            
  <missing>           6 months ago   /bin/sh -c mv /usr/local/apache-tomcat-7.0.77   13.75 MB            
  <missing>           6 months ago   /bin/sh -c #(nop) ADD file:335d94b63fdef5f578   13.75 MB            
  <missing>           6 months ago  /bin/sh -c #(nop) ADD file:3c034af1c455b5d64f   306.3 MB   ...
  呐，如上两次智障的mv操作，拜拜消耗了320MB的空间
  ```

参考：

[https://docs.docker.com/storage/storagedriver/overlayfs-driver/](https://docs.docker.com/storage/storagedriver/overlayfs-driver/)

[https://docs.lvrui.io/2018/12/11/overlayfs%E4%B8%8Eoverlayfs2%E4%B8%8B%E7%BC%96%E8%AF%91%E9%95%9C%E5%83%8F%E7%9A%84%E5%8C%BA%E5%88%AB/](https://docs.lvrui.io/2018/12/11/overlayfs%E4%B8%8Eoverlayfs2%E4%B8%8B%E7%BC%96%E8%AF%91%E9%95%9C%E5%83%8F%E7%9A%84%E5%8C%BA%E5%88%AB/)

[https://arkingc.github.io/2017/05/05/2017-05-05-docker-filesystem-overlay/](https://arkingc.github.io/2017/05/05/2017-05-05-docker-filesystem-overlay/)

