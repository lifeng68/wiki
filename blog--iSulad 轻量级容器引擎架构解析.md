# iSulad 轻量级容器引擎架构解析

​ iSulad 是一种由 C/C++编程语言编写的容器引擎，当前已经在 openeuler 社区开源(https://gitee.com/openeuler/iSulad)。当前主流的容器引擎 docker、containerd、cri-o 等均是由 GO 语言编写。随着边缘计算、物联网等嵌入式设备场景的不断兴起，在资源受限环境下，业务容器化的需求越来越强烈。由高级语言编写的容器引擎在底燥占用上的劣势越来越凸显。另外由于容器引擎对外接口的标准化，因此用 C/C++重写一个容器引擎成为了可能。

​ <img src="iSulad-place.png" alt="iSulad" style="zoom:100%;" />

​ 本文介绍 iSulad 的功能特性以及对整体架构进行介绍。

## iSulad 功能特性介绍

### 提供客户端命令行进行容器操作

iSulad 是典型的 CS 架构模式。iSulad 作为 daemon 服务端，同时也提供了单独的客户端命令 isula，供用户使用。iSula 提供的命令参数覆盖了常用的大部分的应用场景。包含容器的操作接口，如运行、停止、删除、pause 等操作，也包含了镜像相关的操作，如下载、导入、删除等。

```
$ sudo isula --help
USAGE:
	isula <command> [args...]

COMMANDS:
	attach 	Attach to a running container
	cp     	Copy files/folders between a container and the local filesystem
	create 	Create a new container
	events 	Get real time events from the server
	exec   	Run a command in a running container
	export 	export container
	images 	List images
	import 	Import the contents from a tarball to create a filesystem image
	info   	Display system-wide information
	inspect	Return low-level information on a container or image
	kill   	Kill one or more  running containers
	load   	load an image from a tar archive
	login  	Log in to a Docker registry
	logout 	Log out from a Docker registry
	logs   	Fetch the logs of a container
	pause  	Pause all processes within one or more containers
	ps     	List containers
	pull   	Pull an image or a repository from a registry
	rename 	Rename a container
	restart	Restart one or more containers
	rm     	Remove one or more containers
	rmi    	Remove one or more images
	run    	Run a command in a new container
	start  	Start one or more stopped containers
	stats  	Display a live stream of container(s) resource usage statistics
	stop   	Stop one or more containers
	tag    	Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE
	top    	Display the running processes of a container
	unpause	Unpause all processes within one or more containers
	update 	Update configuration of one or more containers
	version	Display information about isula
	wait   	Block until one or more containers stop, then print their exit codes
```

### 支持 CRI 标准协议

CRI(Container Runtime Interface)是由 K8S 定义的容器引擎需要向 k8S 对外提供的容器和镜像的服务的接口,供容器引擎接入 K8S。CRI 接口基于 gRPC 实现。iSulad 遵循 CRI 接口规范，实现 CRI gRPC Server，包括 Runtime Service 和 Image Service 分别用来提供容器运行时接口和镜像操作接口。iSulad 的 gRPC Server 需要监听本地的 Unix socket，而 K8S 的组件 kubelet 则作为 gRPC Client 运行。

<img src="iSulad-cri.png" alt="iSulad" style="zoom:100%;" />

### 支持 CNI 网络标准协议

CNI(Container Network Interface) 是 google 和 CoreOS 主导制定的容器网络标准协议。通过 CNI 协议，iSulad 通过 JSON 格式的文件与具体网络插件进行通信，进而实现容器的网络功能。容器网络具体的功能均由网络插件来实现。iSulad 中使用 C 语言实现了 clibcni 接口模块，用来实现对应功能。

<img src="iSulad-cni.png" alt="iSulad" style="zoom:100%;" />

### 遵循 OCI 标准

OCI 包含两个标准规范 **容器运行时标准 （runtime spec）和 容器镜像标准（image spec）**。

#### 支持 OCI 标准镜像格式

iSulad 支持 OCI 标准镜像格式以及与 docker 兼容的镜像格式。iSulad 支持从 docker hub 等镜像仓库下载容器镜像，或者导入由 docker 导出的镜像文件。

容器运行所需的 rootfs 以及一些资源配置等信息被打包成特定的数据结构，称为容器镜像。基于容器镜像可以方便地运行容器。由于运行环境和应用被一起打包到了容器镜像中，这样就解决了应用部署时的环境依赖问题。每个容器镜像由一个或多个层数据、以及一个 config.json 配置文件组成。多个层之间有依赖关系，这种依赖关系称为父子关系（被依赖的层为父层）。运行容器之前，所有层的数据会合并挂载成一个 rootfs 供容器使用，称为容器层。合并后的数据如果有冲突，则子层的数据会覆盖父层中路径名称都相同的数据。

<img src="iSulad-img-single.png" alt="iSulad-img-single" style="zoom:100%;" />

​ 图 单镜像的结构

镜像的分层是为了解决空间占用问题。如果本层及其所有递归依赖的父层具有相同的数据，这些数据就可以复用，以减少空间占用。下图描述的是具有相同 layer0 层和 layer1 层的两个容器镜像的数据复用结构：

<img src="iSulad-img-dual.png" alt="iSulad-img-dual" style="zoom:100%;" />

​ 图 镜像层复用

#### 支持 OCI 标准 runtime

iSulad 支持标准 OCI runtime 操作接口，可以进行容器生命周期管理。iSulad 不仅支持当前主流的容器 runtime 如 runc、kata，而且针对低底噪的需求，将 C 语言编写的 lxc 进行了适配修改，使其能够作为支持 OCI 标准协议的 C 语言 runtime，进一步降低了容器引擎基础设施底噪开销。
我们可以使用 iSulad 来运行一个新的容器，通过查看运行过程中产生的进程以及持久化的配置，理解运行新容器的过程。

运行容器，首先需要下载镜像，我们使用 busybox 来作为容器镜像。

调用 isula 客户端命令下载镜像。

```bash
$ sudo isula pull busybox
Image "busybox" pulling
Image "c7c37e472d31c1685b48f7004fd6a64361c95965587a951692c5f298c6685998" pulled
```

​ 运行容器指创建一个新的容器，并启动它。

```bash
$ sudo isula run -itd busybox
42fc2595f876b5a18f7729dfb10d0def29789fb73fe0f1327e0277e6d85189a1
```

运行容器后，可以通过本地文件查看持久化的容器配置文件。

```bash
# cd /var/lib/isulad/engines/lcr/42fc2595f876b5a18f7729dfb10d0def29789fb73fe0f1327e0277e6d85189a1
# ls -al
total 92
drwxr-x--- 3 root root  4096 7月  27 16:25 .
drwxr-x--- 3 root root  4096 7月  27 16:25 ..
-rw-r----- 1 root root  4045 7月  27 16:25 config
-rw-r----- 1 root root 23878 7月  27 16:25 config.json
-rw-r----- 1 root root  2314 7月  27 16:25 config.v2.json
-rw-r----- 1 root root     0 7月  27 16:25 console.log
-rw-r----- 1 root root   101 7月  27 16:25 hostconfig.json
-rw-r--r-- 1 root root    10 7月  27 16:25 hostname
-rw-r--r-- 1 root root   183 7月  27 16:25 hosts
drwx------ 3 root root  4096 7月  27 16:25 mounts
-rw-r----- 1 root root     5 7月  27 16:25 ocihooks.json
-rw-r--r-- 1 root root   707 7月  27 16:25 resolv.conf
-rw-r----- 1 root root 26140 7月  27 16:25 seccomp
```

其中 config.json 文件为符合 OCI 标准协议的容器配置文件，内容包括启动容器所需的配置信息。

```bash
# cat config.json
{
    "ociVersion": "1.0.1",
    "hooks": {

    },
    "hostname": "localhost",
    "mounts": [
        {
            "source": "tmpfs",
            "destination": "/dev",
            "options": [
                "nosuid",
                "strictatime",
                "mode=755",
                "size=65536k"
            ],
            "type": "tmpfs"
        },
        {
            "source": "mqueue",
            "destination": "/dev/mqueue",
            "options": [
                "nosuid",
                "noexec",
                "nodev"
            ],
            "type": "mqueue"
        },
        ...
```

config.v2.json 为 iSulad 维护管理容器持久化的一些信息,包括容器的基础配置，以及创建时间，启动时间，容器的 PID、运行启动时间等信息。

```bash
# cat config.v2.json
{
    "CommonConfig": {
        "Path": "sh",
        "Config": {
            "Hostname": "localhost",
            "User": "",
            "Tty": true,
            "OpenStdin": true,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
            ],
            "Cmd": [
                "sh"
            ],
            "WorkingDir": "",
            "LogDriver": "json-file",
            "Annotations": {
                "log.console.file": "/var/lib/isulad/engines/lcr/42fc2595f876b5a18f7729dfb10d0def29789fb73fe0f1327e0277e6d85189a1/console.log",
                "log.console.filerotate": "7",
                "log.console.filesize": "30KB",
                "cgroup.dir": "/isulad",
                "log.console.driver": "json-file",
                "rootfs.mount": "/var/lib/isulad/mnt/rootfs",
                "native.umask": "secure"
            }
        },
        "Created": "2020-07-27T16:25:16.170149428+08:00",
        "Image": "busybox",
        "ImageType": "oci",
        "HostnamePath": "/var/lib/isulad/engines/lcr/42fc2595f876b5a18f7729dfb10d0def29789fb73fe0f1327e0277e6d85189a1/hostname",
        "HostsPath": "/var/lib/isulad/engines/lcr/42fc2595f876b5a18f7729dfb10d0def29789fb73fe0f1327e0277e6d85189a1/hosts",
        "ResolvConfPath": "/var/lib/isulad/engines/lcr/42fc2595f876b5a18f7729dfb10d0def29789fb73fe0f1327e0277e6d85189a1/resolv.conf",
        "ShmPath": "/var/lib/isulad/engines/lcr/42fc2595f876b5a18f7729dfb10d0def29789fb73fe0f1327e0277e6d85189a1/mounts/shm/",
        "LogPath": "/var/lib/isulad/engines/lcr/42fc2595f876b5a18f7729dfb10d0def29789fb73fe0f1327e0277e6d85189a1/console.log",
        "BaseFs": "/var/lib/isulad/storage/overlay/42fc2595f876b5a18f7729dfb10d0def29789fb73fe0f1327e0277e6d85189a1/merged",
        "Name": "42fc2595f876b5a18f7729dfb10d0def29789fb73fe0f1327e0277e6d85189a1",
        "id": "42fc2595f876b5a18f7729dfb10d0def29789fb73fe0f1327e0277e6d85189a1"
    },
    "Image": "sha256:c7c37e472d31c1685b48f7004fd6a64361c95965587a951692c5f298c6685998",
    "State": {
        "FinishedAt": "0001-01-01T00:00:00Z",
        "Pid": 19232,
        "PPid": 19229,
        "StartTime": 2731408,
        "PStartTime": 2731402,
        "Running": true,
        "StartedAt": "2020-07-27T16:25:16.286812971+08:00"
    }
}
```

## iSulad 代码架构介绍

前面介绍了 iSulad 的主要的功能特性，现在让我们深入到 iSulad 的代码中，介绍下 iSulad 的代码框架。
