# ROS安装

## ROS版本选择

ROS1只支持在Linux上安装部署，ROS2支持win、ubuntu、macOS。[参考链接](https://www.ros.org/)

<img src="pic/37.png" style="zoom:50%;" />

而且为了稳定的开发环境。ROS的每个版本都有一个推荐运行的Ubuntu版本：

| ROS 版本 | 首选Ubuntu版本 |
| :------: | :------------: |
|  Noetic  |  Ubuntu 20.04  |
| Melodic  |  Ubuntu 18.04  |
|  Lunar   |  Ubuntu 17.04  |
| Kinetic  |  Ubuntu 16.04  |

[参考链接](http://wiki.ros.org/cn/ROS/Installation)

*建议在本地安装，不推荐用虚拟机，这样兼容性更好。*

## 安装ROS

根据不同的Linux版本安装不同的ROS，此处选用[Kinetic](http://wiki.ros.org/cn/kinetic/Installation)

并选择支持的平台：

<img src="pic/38.png" style="zoom:50%;" />

1. 将镜像添加到Ubuntu系统源列表中，建议使用国内或镜像源，这样能够保证下载速度

   ```sh
   sudo sh -c '. /etc/lsb-release && echo "deb http://mirrors.ustc.edu.cn/ros/ubuntu/ $DISTRIB_CODENAME main" > /etc/apt/sources.list.d/ros-latest.list'
   ```

2. 添加Keys

   ```sh
   sudo apt-key adv --keyserver 'hkp://keyserver.ubuntu.com:80' --recv-key C1CF6E31E6BADE8868B172B4F42ED6FBAB17C654
   ```

   公钥是Ubuntu系统的一种安全机制。

3. 更新软件包和索引

   ```sh
   sudo apt-get update && sudo apt-get upgrade
   ```

4. 安装ROS

   ROS中有很多函数库和工具，官网提供了四种默认的安装方式。推荐安装**桌面版**。

   ```sh
   sudo apt-get install ros-kinetic-desktop-full
   ```

   也可以单独安装某个软件包，如找不到

   ```sh
   xxx: Cannot locate rosdep definition for 【yocs_cmd_vel_mux】
   ```

   就可以按需安装该软件包：注意需要将'_'变为'-'，还有版本号需要与ROS版本号匹配

   ```sh
   apt-get install ros-kinetic-yocs-cmd-vel-mux
   ```

   *这些软件包的依赖关系存在于它们的package.xml*

## 初始化&配置ROS

```sh
sudo rosdep init && rosdep update
```

rosdep可以方便在你需要编译某些源码的时候为其安装一些系统依赖，同时也是某些ROS核心功能组件所必需用到的工具。

将配置写入.bashrc：

```sh
echo "source /opt/ros/kinetic/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

命令`source /opt/ros/kinetic/setup.bash` 只在当前终端有作用，即具有单一时效性，要想每次新开一个终端都不用重新配置环境，就用echo语句将命令添加到bash会话中。

安装构建ROS依赖项的工具：

```sh
sudo apt-get install python-rosinstall python-rosinstall-generator python-wstool build-essential
```

## 验证ROS是否安装成功

能启动roscore（相当于一个master节点）就说明ROS正常启动了：

```sh
roscore
```

![](pic/39.png)

保持这个终端，在另一个终端输入：

```sh
# 运行一个节点
rosrun turtlesim turtlesim_node
# wasd控制
 rosrun turtlesim  turtle_teleop_key
```

会出现：

![](pic/40.png)

至此ROS就算是安装成功了。

# ROS-dev开发环境

## 安装

ROS-dev是公司的开发环境，算是普通ROS的加强版有着需要公司开发的库。[安装方法](https://confluence.momenta.works/pages/viewpage.action?pageId=147514657)，并按照[仓库](https://devops.momenta.works/Momenta/msd/_git/ros-dev?version=GBdev&_a=contents)中步骤安装（不需要多实例初始化配置）。

其中配置文件（config.json）改为：

```json
{
  // 如果在车上，which_car 必填
  // "which_car": "thinkpad-t14",
  "ntp_server_ip": "",
  "ros_master_ip": "",
  "ros_master_port": 11311,
  "registry_host": "artifactory.momenta.works",
  //ssh server port
  "sshd_port": 2222,

  // 可配置Docker内/home/ros/Downloads目录的对应的宿主机文件夹
  "shared_downloads": "/home/mm/docker_share",

  // 实车环境需要打开下一行注释
  // "ros_ip":"192.168.4.10", 

  //------------------
  //以下为自定义，作为append配置, 自机酌情增删
  //------------------
  "volume_list": [
	["/dev/shm","/dev/shm"]
	],
  "container_volume_list":[
    // python3.7镜像，可根据需求选择增加或注释掉
    [[],"python3.7","artifactory.momenta.works/docker-msd/ros-dev/python3.7:v0.0.1",true]
  ],
  "copy_list": [],
  "env_list": {
    "WHICH_HOST":"1a",
    "SUMO_HOME": "/usr/local/share/sumo"
  },
  "port_list":[
//windows docker需export端口
    //ssh端口
    [2222, 2222],
    [4010, 4010],
    [5010, 5010],
    [5011, 5011],
    [4014, 4014],
    [4015, 4015],
    [4016, 4016],
    [4017, 4017],
    [4002, 4002],
    [4003, 4003],
    [4001, 4001],
    [4053, 4053],
    [4021, 4021],
    [4031, 4031],
    [8016, 8016],
    [4054, 4054],
    [4056, 4056],
    [4057, 4057],
    [4058, 4058],
    [4004, 4004],
    [4007, 4007],
    [4009, 4009],
    [4062, 4062],
    [4066, 4066],
    [4097, 4097],
    [4064, 4064],
    [4063, 4063],
    [4008, 4008],
    [4097, 4097],
    [4098, 4098],
    [4088, 4088],
    [4059, 4059],
    [5001, 5001],
    [15001, 15001],
    [11311, 11311]
  ],
  // "enable_hook": true,
  //匹配tmpfs自检, 应当128M以上 
  "docker_other_argv": [ "--shm-size", "32G" ]
}
```

还有ROS-dev版本需要弄对。（可以问@jingshikuo、@Chenyuan Duan  ）

## 开发

使用./script/run_dev.py脚本启动后，以fmp的功能为例

```sh
cd catkin_ws
# 初始化ROS工作空间 会出现build devel src目录，其中开发的package都在src中
catkin_make
```

下载对应的代码，如fmp就是下载的:

<img src="pic/44.png" style="zoom:50%;" />

* maf_common

  一些ros的代码，service、topic等

* msd_robo_adapter

* websocket_adapter

下载好后需要切到对应的分支（tag号）拉取submoudule：

```sh
git submodule update --init --recursive
```

如果遇到拉不下的模块，需要：

```sh
vi ~/.git-credentials
# 在账号名：密码加上MOMENTA%5c
# https://MOMENTA%5cUsername:Password@devops.momenta.works
```

***

可以使用VSCode（建议）进行**远程开发**：

<img src="pic/41.png" style="zoom:50%;" />

* 打开扩展商店
* 搜索ssh
* 下载remote-SSH

**连接ROS-dev**：

<img src="pic/42.png" style="zoom:50%;" />

* 在侧边栏打开Remote SSH

* 修改SSH配置

  Ros-dev暴露的是2222端口

* 打开工程目录（此处是catkin_ws）

**增加对C++库的支持**（改代码时不会爆红，并能找到对应的头文件，不影响编译）：

<img src="pic/43.png" style="zoom:50%;" />



在`~/catkin_ws/`下增加`.vscode`文件夹，并添加`c_cpp_properties.json`文件：

```json
{
    "configurations": [
        {
            "name": "Linux",
          // 为vscode添加include(头文件)的搜索路径
          // 仅针对本人的路径，不过只要是linux应该大差不差
            "includePath": [
                "${workspaceFolder}/**",
                "/usr/include",
                "/usr/include/c++/5",
                "/usr/include/x86_64-linux-gnu/c++/5",
                "/usr/include/c++/5/backward",
                "/usr/lib/gcc/x86_64-linux-gnu/5/include",
                "/usr/local/include",
                "/usr/lib/gcc/x86_64-linux-gnu/5/include-fixed",
                "/usr/include/x86_64-linux-gnu",
               // kinetic版本的ros
                "/opt/ros/kinetic/include/*",
                "/usr/local/include/**",
                "/usr/local/lib/**",
                "~/home/ros/catkin_ws/devel/include/**",
                "~/home/ros/poj/robo_node/devel/include/**"
            ],
            "defines": [],
            "compilerPath": "/usr/bin/gcc",
          // c/c++标准，避免写C++11标准代码时爆红
            "cStandard": "gnu11",
            "cppStandard": "c++11",
            "intelliSenseMode": "linux-gcc-x64"
        }
    ],
    "version": 4
}
```

一些**个人编辑的配置**`settings.json`：

```json

{
    "files.associations": {
        "vector": "cpp",
        // 会随着打开的头文件而变多
    },
    // 字体大小
    "editor.fontSize": 14,
    // 行高
    "editor.lineHeight": 23,
    // 文件修改后自动保存，不建议打开
    // "files.autoSave": "onFocusChange",
    "files.autoSave": "off",
    "emmet.excludeLanguages": [
        "markdown"
    ],
    // vscode默认启用了根据文件类型自动设置tabsize的选项
    "editor.detectIndentation": true,
    // 重新设定tabsize
    "editor.tabSize": 4,
    // 每次保存的时候自动格式化 
    // "editor.formatOnSave": true,
    "editor.formatOnSaveMode": "modifications",
  	// LLVM 风格的代码格式化 即左括号与右括号在同一行
    // "editor.defaultFormatter": "xaver.clang-format",
    //  "{ BasedOnStyle: LLVM, UseTab: Never, IndentWidth: 4, TabWidth: 4, }"
    // google 风格的代码格式化 即左括号与右括号不在同一行
    "C_Cpp.clang_format_fallbackStyle": "{ BasedOnStyle: Google, IndentWidth: 2, ColumnLimit: 0}",
    "C_Cpp.clang_format_style":"{ BasedOnStyle: Google, IndentWidth: 2, ColumnLimit: 0}",
    "[cpp]": {
        "editor.defaultFormatter": "ms-vscode.cpptools"
    },
    "C_Cpp.formatting": "Default",
    "python.autoComplete.extraPaths": [
        "/opt/ros/kinetic/lib/python2.7/dist-packages"
    ],
    "python.analysis.extraPaths": [
        "/opt/ros/kinetic/lib/python2.7/dist-packages"
    ],
}
```

如有错误，还请指正。