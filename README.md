####  1. ruyibuild功能介绍

ruyibuild是一个辅助开发工具，不需要手动搭建复杂的环境和下载代码，只需要几条命名，就可以直接获取所需要的构建好的软件包。

#### 2. 工作原理

ruyibuild根据配置文件自动下载需要编译的源码和构建脚本，启动一个docker容器中，在docker容器中运行构建脚本来完成构建

#### 3. 使用方法

下面以ubuntu系统为例来进行说明

##### 3.1 环境准备

ruyibuild是由python编写而成，需要通过pip命令完成安装，

````
$ sudo apt install python3 python3-pip
````

由于需要用到docker容器，所以需要安装docker

````
$ sudo apt install docker docker.io -y
$ sudo groupadd docker
$ sudo usermod -a -G docker $(whoami)
$ sudo systemctl restart docker
$ sudo chmod o+rw /var/run/docker.sock
````

确保不用加sudo就可以直接执行docker命令

##### 3.2 安装ruyibuild

下载 ruyibuild.wheel , 执行pip安装

````
$ pip install <ruyibuild.wheel>
````

##### 3.3初始化工作

执行以下命令创建工作目录, 后续自动下载的源码以及构建生成的软件包都会在此目录下

````
$ ruyibuild init -d <directory> [-f <ruyicfg_directory>]
````

-d \<directory> 表示要创建的工作目录，其中 \<directory>可以是绝对路径也可以是相对路径

-f \<ruyicfg_directory> 是可选参数，当需要从git仓库获取构建配置文件 config.yaml 文件而不想通过手动修改时，需要添加这个参数，其中 \<ruyicfg_directory> 是 包含 config.yaml 文件所在git仓库的信息的yaml文件所在目录，格式如下：

````
config_file:
    repo_url: https://gitee.com/jean9823/ruyibuild_script.git
    branch: master
    path: qemu/config_ubuntu.yaml
````

repo_url: config.yaml 文件所在git仓库地址

branch: config.yaml 文件所在git仓库分支

path:  config.yaml 文件所在git仓库相对于根目录的位置，如果是存储在根目录，该栏位填写为 path: config_ubuntu.yaml

例如: 执行 

````
$ ruyibuild init -d qemu -f /home/ruyicfg.yaml 
````

 就是在当前目录下创建工作目录 qemu , 并从 /home/ruyicfg.yaml 获取 ruyicfg.yaml 文件

工作目录下会生成四个目录：

build: 构建完成的相关文件会存放在该目录下

script: 存放执行构建的脚本

src: 存放需要构建的源码

./ruyibuild: 存放构建配置文件 config.yaml，如果命令设置的 -f 参数，该目录下还会生成一个git目录，用来存放从git仓库获取的构建配置文件config.yaml

可以根据自己的需求进行修改配置文件，文件内容如下：

````
docker:
    repo_url: amd64/ubuntu
    tag: "latest"
basic_repo:
    repo_url: https://github.com/qemu/qemu.git
    branch: master
build_script:
    repo_url: https://gitee.com/jean9823/ruyibuild_script.git
    branch: master
    path: qemu/qemu_ubuntu.sh
````

配置文件分为三部分：

A) docker容器信息

docker: 需要使用的docker容器的镜像信息，包括：

repo_url: docker容器镜像地址

tag: docker容器镜像tag

B) 源码git仓库地址

basic_repo: 需要构建的源码git仓库信息，包括：

repo_url: git仓库地址

branch: 源码在仓库中的branch

C) build_script: 执行构建的脚本git仓库信息，包括：

repo_url: 脚本git仓库地址

branch:  脚本在git仓库中的branch

path: 脚本在git仓库中相对于根目录的位置，如果是存储在根目录，该栏位填写为 path: qemu_ubuntu.sh

脚本会统一存放在 https://github.com/ruyisdk/ruyici ，可以根据自己的需求选择合适的文件，或者也可以使用自己的脚本，关于脚本内容的要求后面会介绍

##### 3.4 准备构建环境和代码

在工作目录下执行以下命令下载./ruyibuild/config.yaml配置的docker镜像，需要构建源码和构建脚本

````
$ ruyibuild update
````

##### 3.5 执行构建

在工作目录下执行以下命令运行容器并执行构建

````
$ ruyibuild generate [<name>]
````

如果不输入\<name>，即执行以下命令

````
$ ruyibuild generate
````

会运行docker容器，并进入容器，容器名称是 ruyibuild-\<workspace>, \<workspace> 表示当前工作目录的文件夹的名称，例如当前工作目录是/home/tester/qemu, 运行的容器名称就是ruyibuild-qemu，执行这个命令后方便用户直接在docker容器中执行手动构建和debug

若要销毁该 docker 容器有2种方式：

一种方式是直接使用 docker 命令进行销毁

````
$ docker stop \<container name or container id>    //停止容器运行
$ docker rm \<container name or container id>     //删除容器
````

另一种方式是在当前工作目录下执行以下命令进行销毁

````
$ ruyibuild destory
````

这里需要注意的是如果容器不用了，一定要删除，否则，在相同工作目录下执行 ruyibuild generate \<name> 时会因为容器名称已经存在而报错

如果输入\<name>，即执行以下命令

````
$ ruyibuild generate <name>
````

\<name> 表示构建完成后，生成的软件压缩包的包名。

执行该命令后，先运行 docker 容器，容器名称一样是 ruyibuild-\<workspace>，然后根据 config.yaml 中的 build_script 设置去工作目录下获取相应的脚本xxx.sh，并执行 sh xxx.sh \<name> 

构建结束后，不论构建成功还是失败，docker容器会自动销毁，如果构建成功，生成的软件压缩包会存放在工作目录下的output目录下，即在output/\<name>.tar.gz就是构建出的软件压缩包，其余目录会被删除，如果构建失败，之前执行 ruyibuild init 生成的目录不会被删除，方便用户debug后再次直接执行 ruyibuild generate \<name> 进行构建即可。

#### 4. ruyibuild命令

ruyibuid目前支持的命令如下：

````
$ ruyibuild help    //帮助
$ ruyibuild version     //查询版本
$ ruyibuild init <directory> [-f <ruyicfg_directory>]    //创建工作目录
$ ruyibuild update      //下载镜像和代码
$ ruyibuild generate [<name>]    //运行docker容器，执行构建
$ ruyibuild destory   //销毁docker容器，当使用ruyibuild generate进入容器debug完成后，可以使用该命令销毁容器
````

#### 5. 执行构建脚本

容器和工作目录映射关系如下:

\<workspace>/src：/home/src

\<workspace>/build：/home/build

\<workspace>/script：/home/script

从上面的对应关系，以及3.2中介绍的各个目录的作用，可以知道，在容器中:

构建脚本路径是/home/script

所要构建的源码路径是/home/src

构建执行结果路路径是/home/build/\<name> ,  \<name>同ruyibuild generate \<name>

构建完成后，程序需要将构建结果从 \<workspace>/build 中取出并打包放在 \<workspace>/output 目录下

由于程序对于这些目录的要求，所以编写脚本时一定要注意这些目录，否则会导致构建无法正确执行，例如：

执行 cd /home/src 确保进入源码目录，再执行编译和构建

执行 ./configure 时，通过 --prefix=/home/build/\\$1 来确保构建结果存放到容器中的 /home/build/\<name>下，\$1表示接收执行脚本命令 sh xxx.sh \<name> 中的 \<name>

下面是一个在 x86 ubuntu 容器中构建 qemu 的 shell 脚本的例子，供参考

````
#!/bin/bash

set -x

apt update -y
apt install -y python3 python3-pip build-essential git libglib2.0-dev libfdt-dev libpixman-1-dev zlib1g-dev ninja-build libslirp-dev
pip install meson
cd /home/src
./configure --target-list=riscv64-softmmu,riscv64-linux-user --prefix=/home/build/$1
make -j $(nproc)
make install
````