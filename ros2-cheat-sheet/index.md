# ROS2 cli 清单


<!--more-->
## Ros2 命令行接口

- 所有 ROS2 命令行 开始都要添加前缀 ros2，后面跟一条指令，一个动词，若干参数

- 文档参数

```bash
ros2 指令 --help 
# 或者 
ros2 指令 -h
```

## 指令

### action

允许手动发送目标并显示关于action的debug信息

#### 动词

- info 输出关于action的信息

```bash
ros2 action info /fibonacci
```
- list 输出 action的名称列表

```bash
ros2 action list
```

- send_goal 发送action目标

```bash
ros2 action send_goal /fibonacci action_tutotials/action/Fibonacci "order"
```

- show 输出action的定义

```bash
ros2 action show action_tutorials/action/Fibonacci
```

### bag

允许从rosbag 里 播放 topic 或者 录制 topic 到 rosbag

#### 动词

- info 输出bag的信息

```bash
ros2 info <bag-name>
```

- play 播放bag

```bash
ros2 play <bag-name>
```
- record 录制bag

```bash
ros2 record -a 
```


### component

各种组件动词

#### 动词

- list 输出正在运行的容器列表和组件

```bash
ros2 component list
```

- load 载入组件到 容器节点

```bash
ros2 component load /ComponentManager composition composition::Talker
```

- standalone 运行组件到它所属的独立容器节点中


- types 输出在ament索引中注册组件的列表

```bash
ros2 component types
```
- unload 从容器节点卸载组件

```bash
ros2 component unload /ComponentManager 1
```

### daemon

守护进程的动词

#### 动词

- start 如果守护进程没有运行就开启
- status 输出守护进程的状态
- stop 如果守护进程运行就停止

### doctor

检查ROS 设置和其他潜在问题(如网络，包版本，rmw中间件等)的工具

```bash
ros2 doctor
```

#### 同义词
wtf

```bash
ros2 wtf
```

#### 参数

- --report/-r 输出所有检查的报告

```bash
ros2 doctor --report
```

- --report-fail/-rf 只输出失败检查的报告


```bash
ros2 doctor --report-fail
```

- --include-warning/-iw 包含失败的检查报告

```bash
rosw doctor --include-warning
ros2 doctor --include-warning --report-fail
```

### extension_points

列出 扩展 points

### extensions

列出 扩展

### interface

不同ROS接口(action/topic/service)相关动词。接口类型可以通过以下选项过滤掉: --only-actions, --only-msgs, --only-srvs

#### 动词
- list 列举所有可用的接口类型

```bash
ros2 interface list
```

- package 输出包内可用接口类型的列表

```bash
ros2 interface package std msgs
```

- packages 输出提供接口的包的列表

```bash
ros2 interface packages --only-msgs
```

- proto 打印接口的原型(主体)

```bash
ros2 interface proto example interfaces/srv/AddTwoInts
```

- show 输出接口定义

```bash
ros2 interface show geometry msgs/msg/Pose
```

### launch

允许在任意一个包里运行一个启动文件，不用 cd 到那个包里

```bash
ros2 launch <package> <launch.file>

ros launch demo.nodes.cpp add_two_ints_launch.py
```

### lifecycle

生命周期相关动词 

#### 动词
- get 获取一个和多个节点的生命周期状态
- list 输出可用的转换的李彪
- nodes 输出具有生命周期的节点列表
- set 触发生命周期状态转换

### msg(弃用)

展示有关消息的调试信息
#### 动词
- list 输出消息类型列表

```bash
ros2 msg list
```
- package 输出给定包的消息列表

```bash
ros2 msg package std_msgs
```
- packages 输出包含该消息的包

```bash
ros2 msg packages
```
- show 输出消息定义

```bash
ros2 msg show geometry_msgs/msg/Pose
```

### multicast

多播相关的动词

#### 动词
- receive 接收单个UDP多播包
- send 发送单个UDP多播包

### node
#### 动词
- info 输出节点信息

```bash
ros2 node info /talker
```
- list 输出可用节点列表

```bash
ros2 node list
```

### param

允许操作参数

#### 动词
- delete 删除参数

```bash
ros2 param delete /talker /user_sim_time
```
- describe 展示已声明参数的描述性信息

- dump 将给定节点的参数以yaml格式转储到终端或文件中

- get 获取参数

```bash
ros2 param get /talker /user_sim_time
```

- list 输出可用参数的列表

```bash
ros2 param list
```
- set 设置参数

```bash
ros2 param set /talker /user_sim_time false
```

### pkg

创建ros2包或者输出 包相关的信息

#### 动词

- create 创建新的ros2包
- executables 输出指定包的可执行文件列表

```bash
ros2 pkg executables demo_nodes_cpp
```
- list 输出可用包的列表

```bash
ros2 pkg list
```
- prefix 输出包的前缀路径

```bash
ros2 pkg prefix std_msgs
```
- xml 输出包xml清单列表里的信息

```bash
ros2 pkg xml -t version
```

### run

在任意包中允许运行可执行文件，而不用cd

```bash
ros2 run <package> <executable>
ros2 run demo_node_cpp talker
```

### security

安全相关动词

#### 动词

- create_key 创建key

```bash
ros2 security create_key demo keys /talker 
```
- create_permission 创建keystore

```bash
ros2 security create_permission demo keys /talker policies/sample policy.xml
```

- generate_artifacts 创建权限

```bash
ros2 security_generate artifacts
```
- list_keys 分配key
- create_keystore 从身份和策略文件中生成key和权限文件

```bash
ros2 security create keystore demo keys
```
- distribute_key 从ros 图数据生成 XML策略文件
- generate_policy 列出key

### service

允许手动调用服务，并显示有关服务的调试信息

#### 动词
- call 调用服务

```bash
ros2 service call /add two ints example interfaces/AddTwoInts ”a: 1, b: 2”
```
- find 输出 给定类型的服务列表

```bash
ros2 service find rcl interfaces/srv/ListParameters
```
- list 输出 服务名称列表

```bash
ros2 service list
```
- type 输出服务类型

```bash
 ros2 service type /talker/describe parameters
```

### srv(弃用)
服务相关动词
#### 动词

- list 输出可用服务类型
- package 输出包中的可用服务类型
- packages 输出包含服务的包
- show 输出服务定义

### test

运行ros2启动测试

### topic

用于显示有关ROS主题的调试信息的工具，包括发布者、订阅者、发布速率和消息。
#### 动词

- bw 展示 topic的带宽

```bash
ros2 topic bw /chatter
```
- delay 从header的时间戳展示topic的延迟
- echo 输出给定topic的消息到屏幕

```bash
ros2 topic echo /chatter
```
- find 查找给定类型的topic类型

```bash
 ros2 topic find rcl interfaces/msg/Log
 ```
- hz 展示topic的发布率

```bash
ros2 topic hz /chatter
```
- info 输出给定topic的信息

```bash
ros2 topic info /chatter
```
- list 输出 活动的topic列表

```bash
ros2 topic list
```
- pub 发布数据到topic

```bash
ros2 topic pub /chatter std msgs/msg/String ’data: Hello ROS 2 world’
```
- type 输出topic的类型
```bash
ros2 topic type /rosout
```

---

> 作者: toxi  
> URL: https://example.com/ros2-cheat-sheet/  

