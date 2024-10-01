# epoll-server

a simple game server

> 注意：需要 epoll 系统调用，并没有为 drawin 上的 kqueue 或 windows 上的 IOCP 提供实现

-   [epoll-server](#epoll-server)
    -   [游戏介绍](#游戏介绍)
    -   [关于服务器设计（软件）](#关于服务器设计软件)
        -   [根据帧同步的消息交互和协议设计](#根据帧同步的消息交互和协议设计)

## 游戏介绍

游戏名称：Squash the Creeps 通过 Godot engine 3.5.3 制作：

![godot](./resource/godot.png)

Godot Engine 是一款开源的跨平台游戏引擎，具有图形化的场景编辑器和强大的脚本语言 GDScript。由于其免费、开源、轻量级和灵活的特性，Godot 已成为游戏开发者和创作者的热门选择。它支持 2D 和 3D 游戏开发，提供丰富的功能集，包括物理引擎、动画系统、网络功能等，使得开发者能够创建各种类型的游戏和交互性应用。

**同步方式**：**状态同步**

在游戏开发中，状态同步是一种重要的技术手段，用于确保所有玩家在不同设备上看到的游戏状态保持一致。在多人在线游戏中，玩家的操作和游戏世界的变化需要在多个客户端之间实时同步，以实现流畅的游戏体验。状态同步的核心在于将游戏对象的状态（如位置、速度、生命值等）从服务器发送到所有连接的客户端。服务器通常负责管理游戏的核心逻辑，所有重要的状态更新都在服务器上处理，然后将更新的信息广播给各个客户端。这种方式确保了游戏的公平性和一致性，因为客户端不能直接操控游戏状态，只能通过服务器进行交互。为了优化性能和减少网络延迟，状态同步常采用预测和插值技术，允许客户端在接收到更新之前先进行局部预测，从而提升游戏的流畅度和响应性。通过合理设计状态同步机制，开发者可以在保证游戏公平性的同时，提高玩家的沉浸感和互动体验。

## 关于服务器设计（软件）

> 硬件为树莓派 4b： `Linux raspberrypi 6.1.0-rpi6-rpi-v8 #1 SMP PREEMPT Debian 1:6.1.58-1+rpt2 (2023-10-27) aarch64 GNU/Linux`

### 根据帧同步的消息交互和协议设计

既然服务器端只设计到转发操作，那么我们首先应该把服务器的“操作消息”以及与之搭配的“消息协议”定义出来。考虑到网络传输的效率和性能，首选的协议类型当然是二进制协议。这里的二进制协议十分简单，就是一个包含了操作消息类型和操作消息长度的结构体：

```c
// include/binary_protocol.h
// 定义消息头
typedef struct
{
    MessageType type; // 消息类型
    uint16_t length;  // 消息长度
} MessageHeader;
```

消息类型 MessageType 是一个枚举类型：

```c
typedef enum
{
    UNKWON_TYPE = 1,    // 未知类型, 用于初始化
    RESPONSE_UUID,      // 响应UUID, 用于客户端初始化自己的UUID
    GLOBAL_PLAYER_INFO, // 全局玩家信息, 用于客户端初始化其他玩家的相关信息
    SOME_ONE_JOIN,      // 有玩家加入, 用于客户端初始化其他玩家的相关信息
    SOME_ONE_QUIT,      // 有玩家退出, 用于客户端初始化其他玩家的相关信息
    GAME_UPDATE,        // 游戏更新, 用于客户端更新游戏状态
    PLAYER_INFO_CERT,   // 玩家信息认证, 用于客户端向服务器端认证玩家信息
    CLIENT_READY        // 客户端准备就绪, 用于客户端通知服务器自己已经准备就绪
} MessageType;
```

这个消息头会最后会被放在一个字节数组之中和消息体一起被发送过去，我们不妨想想这个消息体之中应该会包含哪些消息：

1. 游戏角色实例的状态信息，用于在各个客户端之中对游戏角色进行同步，包含角色所处的 x、y、z 坐标以及在 x、y、z 坐标上的旋转角度。
2. 服务器端向客户端分配全局唯一 UUID 号。
3. 客户端向服务器端响应的玩家名称。
4. 服务器端向客户端响应的所有其他玩家的相关信息，包括所有在线玩家的 UUID 和名称，用于初始化游戏。
5. 服务器端向客户端更新的新加入的玩家的 UUID 和名称。
6. 客户端向服务端发送退出游戏消息。
7. 服务器端向其他客户端广播关于此玩家退出游戏的消息。
8. 玩家信息认证, 用于客户端向服务器端认证玩家信息。
9. 客户端准备就绪, 用于客户端通知服务器自己已经准备就绪。

这里实际上不需要同步玩家的死亡消息，因为如果一个玩家在自己的视角被其它玩家踩扁了，根据帧同步的理论，其他玩家都能看到这一过程发生，所以他们都会在自己的客户端上“让这个玩家死亡”。

服务器端和客户端之间的消息交流模型图大致如下：

**新玩家加入游戏**：

![new_in_proto](./resource/new_in_proto.png)
