# golang实现嵌入式开发-openwrt上golang程序的交叉编译（使用go-mips32和docker）

背景
openwrt 是一个嵌入式的 Linux 发行版，主要用在路由器领域。目前常用的家用路由器的芯片方案，主要是 mips 架构的，也有 arm 方案的，某些性能需求高一点的地方还会用到 x86 架构的。

一、openwrt 上面能否运行 golang 程序
openwrt 上面可以运行 golang 程序，这是毫无疑问的（网上随便一搜就有一大堆）。为什么要在上面运行 golang 程序？主要是考虑到 golang 拥有比 c 更高的开发效率，更友好的开发体验，比其它脚本语言更低的运行开销，更简单的部署方法，以及更好的网络和标准库支持。

二、怎么样在 openwrt 上面运行 golang 程序
在编译 openwrt 源代码的时候，加入 golang 的支持。
openwrt-go 是个不错的选择。这是一个 github 上的项目。
地址：《GeertJohan/openwrt-go》
使用交叉编译工具。例如 gomini/go-mips32。（针对 mips64 和 arm 的也可以在 Google 上找）
这也可以在 github 上面找到。
地址：《gomini/go-mips32》
直接使用别人已经构建好的 docker 镜像。（我个人在嵌入式构建方面，比较喜欢 docker，因为厌烦了那种一次次的重新构建编译，厌倦了那些层出不穷的让人头疼的编译错误）

三、一些注意事项
golang 程序体积的问题。(嵌入式领域对这个问题其实挺在意的)
可以使用以下命令来编译
go build -ldflags "-s -w"
BashCopy
具体的请参考文章：[《让 go 编译生成的文件体积小一些的方法》]（https://www.golangnote.com/topic/66.html）

四、使用docker构建golang交叉编译环境

1、为什么用 docker
开发环境是每个开发者都会碰到的问题。尤其是做大型软件编译、系统集成、嵌入式构建的时候，各种依赖冲突，总是让人心烦。如果跨项目工作，或者要经常切换版本的时候，软件之间的依赖、目录和环境变量之间的互相污染，更加是让人心烦。
我个人做开发经历几个阶段：
1).  单一主机系统开发阶段。（也就是各种冲突、项目相互污染的阶段）
2).  vmware 虚拟机阶段。（采用一个环境一个项目就新建一个虚拟机的办法，可以很好的避免冲突和项目之间的污染，但是虚拟机相对来说会比较重。复制比较耗时、数量多了比较占空间）
3).  vagrant 虚拟机管理阶段。（vagrant 是一个虚拟机管理工具，它和上面的 vmware 并不是冲突，而是包含关系）
这玩意开发出来就是为了方便开发者的，但是国内好像用户群体不大。虽然不在本文的讨论范围，但是有兴趣的不妨了解一下。后续我可能写篇详细的文章讨论一下这个。
《vagrant 官网》
《Vagrant 搭建虚拟化开发环境》
4).  docker 阶段。（个人看法，对于开发环境来说，vagrant 更适合，而 docker 更适合部署。不过在嵌入式交叉编译环境方面，docker 也是个不错的选择）

下面一段内容引用自：《Vagrant or Docker – 开发环境虚拟化》
为什么需要虚拟化

理想的开发流程
    获取源代码
    新获取的代码可以直接运行起来
    根据需求修改代码
    运行／调试代码
    提交代码
    提交测试
现实中的开发流程
    获取源代码
    我的 IDE 项目打开项目有问题
    依赖安装有问题
    编译失败
    配置 Container
    终于运行起来了
…
    服务器上崩掉
    本机没问题
    本机无法重现

开发中的常见的痛点
    环境不一致
    OS (osx / linux / windows)
    IDE
    SDKs
   Containers
   Dependencies
   Bug 分析
   “我这边跑起来没问题啊”
   “我机器上为什么是好的啊”

解决方案 - 通过虚拟化统一运行环境

2、安装和使用 docker
docker 基本概念
镜像（Image）
>Docker 镜像是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。镜像不包含任何动态数据，其内容在构建之后也不会被改变。
容器（Container）
>镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。
容器的实质是进程，但与直接在宿主执行的进程不同，容器进程运行于属于自己的独立的 命名空间。因此容器可以拥有自己的 root 文件系统、自己的网络配置、自己的进程空间，甚至自己的用户 ID 空间。
仓库（Repository）
>镜像构建完成后，可以很容易的在当前宿主机上运行，但是，如果需要在其它服务器上使用这个镜像，我们就需要一个集中的存储、分发镜像的服务，Docker Registry 就是这样的服务。
一个 Docker Registry 中可以包含多个仓库（Repository）；每个仓库可以包含多个标签（Tag）；每个标签对应一个镜像。
安装 docker（ubuntu 16.04）
#卸载旧版本
sudo apt-get remove docker docker-engine docker.io
#更新软件包
sudo apt-get update
#安装工具
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
#增加软件源
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
#更新软件列表
sudo apt-get update
#安装
sudo apt-get install docker-ce
BashCopy
简单试用 docker
sudo docker run hello-world
BashCopy
3、制作和使用 docker 镜像
1). 下载和运行 ubuntu 16.04 的 docker 镜像
sudo docker pull ubuntu:16.04
#输出结果
16.04: Pulling from library/ubuntu
b234f539f7a1: Pull complete
55172d420b43: Pull complete
5ba5bbeb6b91: Pull complete
43ae2841ad7a: Pull complete
f6c9c6de4190: Pull complete
Digest: sha256:b050c1822d37a4463c01ceda24d0fc4c679b0dd3c43e742730e2884d3c582e3a
Status: Downloaded newer image for ubuntu:16.04
BashCopy
sudo docker run -it  ubuntu:16.04
#运行成功的话，就会切换到另外一个根文件系统的命令行了
root@75b1c54b758f:/#
BashCopy
2). 构建 go-mips32 的编译环境
刚下载下来的镜像是很干净的，啥都没有。不过可以用 apt-get 安装。
apt-get update
apt-get upgrade
apt-get install build-essential wget git
BashCopy
构建的过程可以参考我上一篇文章：
《openwrt 上 golang 程序的交叉编译(使用 go-mips32 和 docker)三》

mkdir /go-mips32
cd /go-mips32/
git clone https://github.com/gomini/go-mips32.git
export GOOS=linux
export GOARCH=mips32
#切换目录
cd go-mips32/src
#开始编译（我这里如果不禁用 cgo 会报错）
CGO_ENABLED=0 ./make.bash
BashCopy
上面介绍比较简单，有兴趣可以点击上面那个文章链接。
3). 保存制作好的镜像
#请在本地系统执行，也就是说不要在 docker 里面运行这个命令
sudo docker ps
#输出（可以看到我们刚刚在用的 docker）
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
75b1c54b758f        ubuntu:16.04        "/bin/bash"         18 minutes ago      Up 18 minutes                           silly_mcnulty
BashCopy
#请在本地系统执行，也就是说不要在 docker 里面运行这个命令
#a 是提交者的名字，m 是提交的一些备注信息，75b1c54b758f 是上面查询到的容器 id，ubuntu:16.04-go-mips32 是镜像和镜像的 tag 标签
sudo docker commit -a "MaterWong" -m "docker image for go-mips32 building"  75b1c54b758f ubuntu:16.04-go-mips32
BashCopy
#请在本地系统执行，也就是说不要在 docker 里面运行这个命令
#查询一下，可以看到我们刚刚保存的镜像
sudo docker image ls
#输出
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              16.04-go-mips32     b00bf2789e95        2 minutes ago       678MB
ubuntu              16.04               5e8b97a2a082        2 weeks ago         114MB
BashCopy
4). 查看和测试一下刚刚保存好的镜像
#请在刚刚的 go-mips32 的 docker 环境里面运行这个命令
exit
#退出 docker
BashCopy
#加载刚刚制作好的 ubuntu:16.04-go-mips32 镜像，而不是一开始那个 16.04 的镜像
sudo docker run -it  ubuntu:16.04-go-mips32
#查看一下刚刚的 go-mips32 目录是否还在，如果在，就证明制作成功了
sudo ls /go-mips32
BashCopy
5). 导出和分享 docker 镜像
#请在本地系统执行，也就是说不要在 docker 里面运行这个命令
#查看当前的镜像列表和 id
sudo docker image ls
#输出
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              16.04-go-mips32     b00bf2789e95        12 minutes ago      678MB
ubuntu              16.04               5e8b97a2a082        2 weeks ago         114MB
BashCopy
可以看出，刚刚保存的镜像 id 是 b00bf2789e95

sudo docker save b00bf2789e95  > /home/ubuntu/ubuntu16.04-go-mips32.img
BashCopy
导出的镜像，可以复制传输分享给其他人。
然后通过命令就可以把别人的镜像导入到 docker 的镜像列表里。

sudo docker load < /home/ubuntu/ubuntu16.04-go-mips32.img
