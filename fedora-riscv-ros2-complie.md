# Fedora ROS2 编译指南

## **准备**

首先是依赖：

```
sudo dnf install --skip-broken \
  cmake \
  cppcheck \
  eigen3-devel \
  gcc-c++ \
  liblsan \
  libXaw-devel \
  libyaml-devel \
  make \
  opencv-devel \
  patch \
  python3-colcon-common-extensions \
  python3-coverage \
  python3-devel \
  python3-empy \
  python3-nose \
  python3-pip \
  python3-pydocstyle \
  python3-pyparsing \
  python3-pytest \
  python3-pytest-cov \
  python3-pytest-mock \
  python3-pytest-runner \
  python3-rosdep \
  python3-setuptools \
  python3-vcstool \
  poco-devel \
  poco-foundation \
  python3-flake8 \
  python3-flake8-import-order \
  redhat-rpm-config \
  uncrustify \
  wget \ 
  console-bridge \
  spdlog \
  python3-numpy
```

有些依赖没法安装的可以用 `--skip-broken` 跳过。

依赖安装完毕后导入源码：

```
mkdir -p ~/ros2_rolling/src
cd ~/ros2_rolling
vcs import --input https://raw.githubusercontent.com/ros2/ros2/rolling/ros2.repos src
```

再安装一些ROS自行管理的依赖

```
sudo dnf update

sudo rosdep init
rosdep update
rosdep install --from-paths src --ignore-src -y --skip-keys"asio cyclonedds fastcdr fastrtps ignition-cmake2 ignition-math6 python3-babeltrace python3-mypy rti-connext-dds-6.0.1 urdfdom_headers"
```

## **编译**

ROS 使用 colcon 作为自己的编译系统，colcon 需要在合适的工作目录下运行。

```
cd ~/ros2_rolling/
colcon build --symlink-install--cmake-args -DTHIRDPARTY_Asio=ON --no-warn-unused-cli
```

另外，编译所需时间较长，如果在SSH 这种可能会导致终端关闭的环境下，请考虑使用 Tmux 或者 GNU Screen。

## **一些坑**

### **清理缓存**

ROS2 的编译系统比较粗糙，里面有些直接在源码上打 Patch 的操作，有些部分编译产品在有问题的情况下也不会重新编译。如果编译中途失败可以删除这部分成品与下载的源码，让编译系统重新下载或生成这些文件。

下面是我遇到的几个需要重新生成的包：

```
# 提示 Patch 应用失败
rm -rf ~/ros2_rolling/{build,install}/uncrustify_vendor
# 提示与 atomic 相关的连接错误
rm -rf ~/ros2_rolling/{build,install}/osrf_testing_tools_cpp
# 提示源文件不存在
rm -rf ~/ros2_rolling/{build,install}/rviz_ogre_vendor
```

### **Mimick**

Mimick 是 ROS2 的一个依赖，原生不支持 Riscv64 需要 Patch。目前上游有第三方贡献者提供了支持 Riscv64 的 PR 但是没有合并。

### **Fedora 的环境变量**

另外，出于某种未知的原因，编译 `qt_gui_cpp` 的过程需要设置一些RPM 有关的环境变量。这个问题在不同架构下都会出现。

```
export RPM_ARCH=riscv64 #取决于你的架构，x86_64 也有这个问题
export RPM_PACKAGE_RELEASE="1"
export RPM_PACKAGE_VERSION="1.0"
export RPM_PACKAGE_NAME="foobar-fedora"
```

我怀疑 Fedora 打包的某个依赖可能有点问题。

### **Atomic 内联函数**

Riscv64 架构下的原子性函数没有内联，需要手动让编译器链接。

```
export CFLAGS="-latomic" ; exportLDFLAGS="-latomic" ; exportCXXFLAGS="-latomic"
```

或者

```
export CFLAGS="$CFLAGS --as-needed -latomic";
```

### **网络代理**

Colcon编译过程需要下载第三方源码，可能会遇到网络问题。此处是设置 HTTP 代理环境变量的一个实例，请根据实际情况调整。

```
ip=127.0.0.1&&port=9080&&export http_proxy=http://$ip:$port&&export https_proxy=http://$ip:$port
```

如果使用 SSH 那么 SSH命令可以把本地端口转发到远端，比如：

```
ssh -R 9080:127.0.0.1:9080 -p 59000 tux@remote-server
```

这条目录可以把本地的 9080端口转发到远端的9080。

### 挪移编译目录

ROS2 会在环境脚本中硬编码一些路径，如果把编译后文件进行了挪移可能会导致失效。因为前文在用户的家目录内编译，而不同用户可能有不同的家目录路径。这种情况可以设置 bind mount，fuse bindfs 或者符号连接解决。

## 测试

### Demo

要运行ROS2，需要先应用一些环境设置。

```
# Replace ".bash" with your shell if you're not using bash
# Possible values are: setup.bash, setup.sh, setup.zsh
. ~/ros2_rolling/install/local_setup.bash
```

然后，打开两个中断，都应用一下这些环境设置。

在一个终端里面运行：

```
ros2 run demo_nodes_cpp talker
```

另一个终端运行：

```
ros2 run demo_nodes_py listener
```

如果 ROS2设置正常，`listener` 应该能收到 `talker` 发布的消息。

另外ROS2也内置了一些更加复杂的 Demo，其中一些需要GUI环境。在测试这些例子之前，请先配置好图形环境。

### 单元测试

Colcon 有运行单元测试的工具，可以为整个 ROS2 运行测试。

```
colcon test --ctest-args tests
```

之后在工作目录里面会留下测试的日志。
