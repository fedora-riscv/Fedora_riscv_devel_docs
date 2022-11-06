<div align='center' ><font size='20'>Koji 概述</font></div>

![img](https://www.kdocs.cn/api/v3/office/copy/bERKSW9mOGF4MXpQeS8yR2ZpSk1rN1JXUnFsUzlDQStkUksvcVpZWk5JL25LanhxM2RyaXFRUHVkQWRiZG8yVVlVb09MT2tISXh0bEhBN25DYmNodDYxS3lpaWltODAvRzRNU1BQOWVaclBRYnVIR2tKZTdTYkpqRXFGUUVmSDBxL280VDFCTE92Uk84ZkNDbmsxeXNCZ2gvaXhlVHZlM0JKUUhWMnRCYXFjZHNIa1ZVMERQNzgzbDROLzdJSGI3Z29jUmplT2t5c3FVUlNFTnIzRHZEYWprUlBkUmVtVzR3SFpvMCtQQlRYSjByOVFnbmV5clFlNXk1NjZEU2s5L29MYzVseUhwTmNnPQ==/attach/object/e3cad03d6e5f248fdc0b7e96608d734ddb694fe4)

[TOC]

# 1. 简介

Koji 是Fedora 的包编译管理工具，即一个用于构建和跟踪 RPMs 的系统。

主页位于：[https://pagure.io/koji](https://pagure.io/koji)

主要文档：[https://docs.pagure.org/koji/](https://docs.pagure.org/koji/)

它的设计考虑了以下的特性：

- 安全
  - 每次构建使用新的 buildroot （编译用根文件系统）

  - NFS 大部分情况是只读的

- 利用了其他软件
  - 使用了 Dnf/Yum和 Mock 开源软件作为组件

  - 使用 XML-RPC APIs 更好地整合了其他工具

- 灵活性
  - 丰富的数据模型

  - 活跃的代码基础

- 可用性
  - 带 Kerberos 认证的 Web 接口

  - 可移植的瘦客户端

  - 用户可以创建本地 buildroots

- 可重现性
  - Buildroot 内容可在数据库中跟踪

  - 数据带有版本控制


本文档介绍了开发者需要使用 Koji 完成的基本的任务。

## a. 术语

在 Koji 系统中，有三个重要的概念：

### 软件包(Package)

表现为Source RPM（SRPM）的__名称__，表示一般意义上的软件包，没有指定软件包的具体版本和编译出的RPM包集合。比如：kernel、glibc、gcc等等。

### 构建(Build)

特定版本的软件包。表示软件包在一次构建过程中编译出的所有RPM包，包括所有架构相关包和编译出的所有包。比如：kernel-2.6.9-34.EL、glibc-2.3.4-2.19。

### RPM包(RPM)

一个特定的 rpm 包。一次构建过程中编译出的特定架构和特定名称的包。比如：kernel-2.6.9-34.EL.x86\_64、kernel-devel-2.6.9-34.EL.s390、glibc-2.3.4-2.19.i686、glibc-common-2.3.4-2.19.ia64。

### N-V-R

即Name-Version-Release， 例如kernel-5.18.4-201.0.riscv64.fc36.riscv64.rpm，Name="kernel",Version="5.18.4-201.0.riscv64.",Release="fc36".

NVR来自SPEC文件中的对应字段，而非rpm文件名。比如gcc-12.0.1-0.8.1.fc36.riscv64.rpm和gcc-c\+\+-12.0.1-0.8.1.fc36.riscv64.rpm其实是由一个srpm即gcc-12.0.1-0.8.1.fc36.src.rpm编译出来的，所以package name都是gcc，即同属一个package。

* **NEVRA(name-epoch-version-release-arch)**

### tag

koji使用tag来引用package或build的集合。也就是说可以给多个package或build打上同一个tag。必须先给package打上tag以后才能给属于该包的build打tag

- **build tag**

只是有不同的用途的tag。__tag只是一个package的集合__，__build-tag一般从某个tag继承而来且还包含arch和build-group信息__，可用来创建不同体系的build任务，比如可以为i386和x86\_64各创建一个build tag,每次只引用其中一个来为其parent tag创建不同体系的build版本。

### build group

用于指定mock chroot中需要安装的所有rpm包，然后mock还会安装要编译包的所有依赖，一起构成基础编译系统.

### build target

标明一次build从哪个build tag确定包及编译体系，build完了后标记为哪个tag（默认是与 target-name 同名的tag，__也可指定，所以build target和tag本质是不同的。__）。

### channel

频道（channel） 是用来控制Builder可以处理的任务类型。默认情况下，所有Builder都会加入到 default 频道。但是至少需要一些频道加入到 createrepo 通道，它们可以完成 kojira 发起的创建软件包仓库的任务。

# 2. 结构简介

![img](https://www.kdocs.cn/api/v3/office/copy/bERKSW9mOGF4MXpQeS8yR2ZpSk1rN1JXUnFsUzlDQStkUksvcVpZWk5JL25LanhxM2RyaXFRUHVkQWRiZG8yVVlVb09MT2tISXh0bEhBN25DYmNodDYxS3lpaWltODAvRzRNU1BQOWVaclBRYnVIR2tKZTdTYkpqRXFGUUVmSDBxL280VDFCTE92Uk84ZkNDbmsxeXNCZ2gvaXhlVHZlM0JKUUhWMnRCYXFjZHNIa1ZVMERQNzgzbDROLzdJSGI3Z29jUmplT2t5c3FVUlNFTnIzRHZEYWprUlBkUmVtVzR3SFpvMCtQQlRYSjByOVFnbmV5clFlNXk1NjZEU2s5L29MYzVseUhwTmNnPQ==/attach/object/20a88678e1afa70457da4f30b10cb531fab09af4)

示例架构图

## a. Koji 的组成：

### koji-hub

它是所有 Koji 操作的核心，是一个运行于 Apache 的 mod\_wsgi 模块下XML-RPC 服务。koji-hub 仅__以被动方式__接受 XML-RPC 请求，依赖编译守护模块(kojid)和其他模块来发起交互。koji-hub是唯一能直接访问数据库的模块，而且是对文件系统有写权限的两个模块之一(另外一个是koji)。 

### kojid

编译守护模块，运行在每一个执行编译任务的机器上。其主要是负责对编译请求进行轮询并作相应处理。Koji除了编译外也支持其他任务，如创建安装镜像(Image)。kojid同样也可以完成这样的任务。

kojid使用mock来编译。它为每次编译任务创建一个干净的编译环境（buildroot）。kojid是python实现的，通过 XML-RPC 与 koji-hub 通信。 

### koji-web

它是一套运行在 mod\_wsgi 下的脚本，使用 Cheetah 模板引擎为 Koji 提供一个 Web 接口。koji-web 不仅可显示许多信息，还可提供一些特定操作，如取消编译任务。

### koji

它是一个用 python 实现的命令行程序，提供了许多操作Koji的方法（hooks）。它能让用户查询许多信息，也能执行许多操作，如发起编译任务。 

### kojira

它是一个守护程序，可保证编译环境（build root）使用的仓库数据的不断更新。 

# 3. 软件包组织形式

## a. 标签(Tags) 与 目标(Targets)

### Koji 使用‘tag’组织软件包。

在Koji中，tag类似一个蜂巢状集合（beehive collection）实例，但是在以下方面有所不同：

- Tags 保存在数据库中而不是磁盘文件系统中
- Tags 支持多重继承
- 每个 tag 有它自己有效的软件包列表（软件包列表可以被其他的 tag 继承）
- 可根据 tag 为软件包设置不同的所有者（所有者关系也可以被其他 tag 继承）
- Tag 继承过程是可以配置的
- __当您编译软件包时，您应该指定的是一个"target"，而非一个 tag__

### 编译目标(target)表明了软件包的编译过程应在哪个环境中进行(Buildroot)（也就是build tag），以及编译生成的软件包应被标记为的（Destination） tag 。

当 tag 的名称随着所发布系统的版本变化后， target 的名称仍然可以保持不变。您可通过以下命令获取编译目标(target)的完整列表:

```shell
koji list-targets
```

您可以通过 --name 选项查看单个 target 的信息：

```shell
koji list-targets --name  f36
    Name                           Buildroot                      Destination
    ---------------------------------------------------------------------------
    f36                                f36-build                      f36
```

这告诉您利用  __f36 这个 target __编译软件包时，编译环境由  f36-build 这个 tag中的软件包构成，编译生成的软件包将放入  __f36 这个 tag__ 中。

您可以通过下面的命令查看系统中的 tag 列表： 

```shell
koji list-tags
```

### 标签（tag）继承机制

几乎koji中的一切都是通过标签和他们的继承关系处理的。那么，它有什么好处并如何配置？

每一个标签都处理它自身的配置，而系统管理员可以仅维护这个配置。而标签结构的真正能力是隐藏在继承关系中的。你可通过继承母产品数据到新版本或另一层产品的方式来创建标签。

每个标签都包含以下数据：

- __标签数据__

  - 构架

    此处指的是一组构架，会在给定标签被用于编译环境（buildroot）时（即在定义目标\[target\]时）使用。

  - comps组

    与构架类似, 这个数据在标签被用于编译环境（buildroot）时提供软件安装包分组信息。一般来说，`srpm-build`和`build` 组是在几乎所有情况下都需要的。对于编译rpm包以外的其他的操作，还可能需要一些额外的组，如`maven-build`或`livemedia-build`。组可以通过`*-group-*`的koji命令在创建和修改。

  - maven选项 

    如果maven编译工具在koji环境中被使能（需要在koji-hub中配置），`maven_support`和`include_all`是相关的选项。 第一个选项表示maven仓库应被产生，而第二个则限制了对于所有被标签过的某些软件包是否应存在于本仓库中。

- __软件包列表__

   __每个标签都携带了一个可被打标签的软件包列表__，以及 软件包/标签 组合的属主。

   注意，__属主主要用于表示默认通知接收者__，并非操作权限，可在任何时候被修改。属主关系也不限于任何能对‘标签/软件包’的构建进行 ”去除/打”标签的人。另外，这是通过 hub 协议驱动的。简而言之，软件包列表就是可被标签的软件包（它们甚至可以没有相应的构建（build））。

- __被标签的构建（build）列表__

   这是tag的最后一个成员。显然，构建列表中的软件包都是来自于上一概念（软件包列表）中。不存在某些构建基于软件包列表以外的软件包。

__这三组数据可以通过继承链被继承和传播.__

- __继承选项__

  整个继承关系可以通过命令行中的`add-tag-inheritance`和`edit-tag-inheritance`

  指令修改。他们有相同的选项，描述和举例如下:

  简单的 `add-tag-inheritance` 命令需要两个已经存在的标签，这个指令会将他们在继承链中连接在一起。

  ```shell
  $ koji add-tag parent
  
  $ koji add-tag child
  
  $ koji add-tag-inheritance child parent
  
  $ koji list-tag-inheritance child
  
  child (168)
  
  ....  └─parent (167)
  ```

  在这个例子显示了基本的继承链： `child` 标签继承了从`parent` 标签的数据。在__标签后面的数字是标签的ID__（通常无需关注，但在koji脚本中会比较有用）。行首的四个点是不同继承标志的占为符。他们是M, F, I, N ，分别表示 `maxdepth`, `pkg_filter`, `intransitive` 以及`noconfig`。它们都可以通过命令行制定。

  - priority
  
    当你往已经有母标签的标签中添加新的继承关系时，你可能想要表示出新的母标签在继承链中所处的位置。
  
    我们在之前例子的基础上继续：
  
    ```shell
    $ koji add-tag less-significant-parent
    
    $ koji add-tag-inheritance child less-significant-parent --priority 100
    
    $ koji list-tag-inheritance child
    
          child (168)
    
     ....  ├─parent (167)
    
     ....  └─less-significant-parent (169)
    ```
  
    这里，原来的 `parent` 拥有默认的优先级 0，而新母标签的优先级被设为 100。
  
    __数字越小，优先级越高__。而你总是可以通过 `edit-tag-inheritance`来改变其继承关系中的优先级。
  
    > 注意：好的经验法则是创建继承关系的同时设定优先级，并以 10 or 100 为步进.当你要在其间插入其它继承关系时，就无需改变现有的优先级 (你可像古早的Basic时代一样使用如 15 这样的设置).
  
  - maxdepth
  
    对于更长的继承链，你可能不想处理整条链。现在舍弃我们之前的示例，看看较为现实的情况。
  
    ```shell
    $ koji list-tag-inheritance linux3-build
    
    linux3-build (4380)
    
    ....  └─linux3-override (4379)
    
    ....     └─linux3-updates (4373)
    
    ....        └─linux2 (4368)
    
    ....           └─linux1 (4350)
    
    ....              └─linux0 (4250)
    
    $ koji add-tag linux4-build
    
    $ koji add-tag-inheritance linux4-build linux3-build --maxdepth 0
    
    $ koji list-tag-inheritance linux4-build
    
    linux4-build (4444)
    
    M...  └─linux3-build (4380)
    ```
  
    这种情况不会出现在Fedora的koji系统中，因为Fedora不会复用任何先前版本的软件包，而是做大规模重编。但这种情况是存在。此时，若我们只需来自 `linux3-build`的软件包，而非任何它所继承来的软件包。__则 `maxdepth` 就派上用场了，它可以斩断剩余的继承链。__
  
  - intransitive
  
    非传递性（Intransitive）继承链， 顾名思义。如果它们被用于继承链的某处，则剩下的链条将被忽略。它可被用于避免一些继承信息的错误传播。同 `maxdepth`结合使用，可在触及 `maxdepth` 前实现继承的停止。
  
  - noconfig
  
    相较于‘通常’的继承一切（如标签的配置，软件包列表和被标记的构建）的继承方式，若使用这个选项，则只继承软件包列表和构建，其余的都会被忽略（构架，锁，权限，maven支持）。
  
  - __pkg-filter__
  
    __包过滤器被定义为一个正则表达式，用于过滤通过这个继承链传播的软件包。__
