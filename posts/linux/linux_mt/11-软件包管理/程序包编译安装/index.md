# 10.4 程序包编译安装


程序包编译安装

![linux-mt](/images/linux_mt/linux_mt.jpg)
<!-- more -->

前面我们讲解了使用 rpm，yum 安装程序包的方法，相比于编译安装，它们更加便捷，但是由于 rpm 包是编译好的二进制程序，我们就无法根据自己的需求去定制程序特性和功能。所有如果想定制程序，则必需编译安装，在编译时启用需要的特性。本节我们首先会说一说编译安装的过程，然后再来介绍如何在 Linux 中实现编译安装 C/++ 程序。本节内容概括如下:
1. 程序的编译安装过程
2. 如何获取源代码
3. 编译安装C源代码

## 1. 程序的编译安装过程
![c_compile](/images/linux_mt/c_compile.jpg)

如上图所示，C 程序的编译要经过 `源代码 --> 预处理 --> 编译(gcc) --> 汇编 --> 链接` 四步
1. 预处理: 处理注释，宏替换，头文件
2. 编译: 将预处理之后的源代码编译成汇编代码
3. 汇编: 将汇编代码编译成二进制机器代码
4. 链接: 如果程序用到了其他 C 库的代码，则需要将这部分代码动态链接至二进制机器代码中

上述就是一个简单 C 程序编译过程的概述，如果想了解更多细节，可查阅其他资料。

我们知道现在的大型软件工程都不太可能由一个人完成，同时为了程序的可维护性和扩展性，通常都会将代码放置在多个文件中，文件中的代码之间，很可能存在跨文件依赖关系。因此在编译 C 过程中，必需按照特定的顺序编译才能编译成功。为了辅助程序的编译，因此出现了很多项目管理工具。make,cmake 则时 C 和 C++ 中最常见的项目管理工具。

make 工具的配置文件 makefile 记录了程序编译的详细过程，make 工具根据 makefile 的配置可以自动完成程序的编译安装。通常在程序包中有两个辅助生成 makefile 的文件 -- configure 和 Makefile.ini。Makefile.ini 是 makefile 的模板，configure 是一个脚本文件，有众多选项，能根据用户提供的参数，依据 Makefile.ini 模板动态生成 makefile 文件。因此用户可以借助于 configure 提供的选项定制 makefile 文件，从而定制程序包的编译安装。autoconf 工具可以辅助生成configure脚本，而 automake 则用于辅助生成 Makefile.in

顺便说一下，rpm 包中有一种类似 `testapp-VERSION-release.src.rpm` 的以 `src.rpm` 结尾的 rpm 源码包，其内部的代码是未编译的，需要使用rpmbuild命令制作成二进制格式的rpm包，而后才能安装，在 rpmbuild 过程中，用户就可以自定义程序的特性和功能。


### 1.2 C代码编译安装三步骤
根据上述的编译过程阐述， C 代码编译安装大体需要如下三个步骤:
1. ：
    - 通过选项传递参数，指定启用特性、安装路径等；执行时会参考用户的指定以及Makefile.in文件生成makefile；
    - 检查依赖到的外部环境；
2. make：根据makefile文件，构建应用程序；
3. make install: 脚本，将构建的应用程序放置到配置的目录中
4. 附注: 安装前建议查看INSTALL，README

### 1.3 configure 可用选项
`./configure [options]` 的众多选项可以分为如下几类
1. --help: 获取其支持使用的选项
2. 安装路径设定：
    - **`--prefix=`**/PATH/TO/SOMEWHERE: 指定默认安装位置；默认为/usr/local/
    - **`--sysconfdir=``**/PATH/TO/SOMEWHERE：配置文件安装位置；
3. System types: 指定目标平台系统结构
4. Optional Features: 可选特性
    - `--disable-FEATURE`: 关闭特性
    - `--enable-FEATURE[=ARG]`: 启用特性
5. Optional Packages: 可选包
    - `--with-PACKAGE[=ARG]`
    - `--without-PACKAGE`


## 2. 开源程序源代码的获取
常见的源代码有如下几个获取途径
1. 官方自建站点：
    - apache.org (ASF)
    - mariadb.org
2. 代码托管：
    - SourceForge
    - Github.com
    - code.google.com

## 3. 编译安装C源代码
完整的编译安装过程还包括安装前的环境准备以及安装后的配置操作，C源代码完整的安装过程如下
1. 提供开发工具及开发环境，通常包含在开发工具的包组中
    - 开发工具包括：make, gcc等
    - 开发环境包括：开发库，头文件, glibc(C 标准库)
    - 开发工具安装:
        - CentOS6: `yum groupinstall "Development Tools" "Server Platform Development"`
        - Centos7: `yum groupinstall "Development Tools"`
2. 执行 configure脚本，指定安装位置、指定启用的特性
3. 执行 make
4. make install
5. 安装后的配置：
    - 导出二进制程序目录至PATH环境变量中；
        - 编辑文件`/etc/profile.d/NAME.sh`，`export PATH=/PATH/TO/BIN:$PATH`
    - 导出库文件路径：
        - 编辑`/etc/ld.so.conf.d/NAME.conf` ，添加新的库文件所在目录至此文件中；
        - 让系统重新生成库文件缓存：`ldconfig [-v]`
    - 导出头文件
        - 基于链接的方式实现：`ln -sv /path/include  /usr/include/dir`
    - 导出帮助手册
        - 编辑`/etc/man.config`文件，添加一个MANPATH

