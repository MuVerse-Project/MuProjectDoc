# MuPacket API

在 MK-ServerLauncher 中，HTTP 和 WebSocket 中传输的消息皆为由 MuPacket API & MuPacketAPI4TS 转换而来的 JSON 消息

**在 MuPacketAPI 中，其本质为 `MuPacket` 及其子类，由 JSON 为信息载体**

## 消息结构

### 示例
```JSON5
{
  "MP_ID": "x.x:x", // MuPacket ID
  "MP_DATA": {
    // 动态内容, 根据不同 MP_ID 提供不同内容
  },
  "CID": 1145141919810 // 调用号，Long；进程内单调递增，用于把应答对回请求。不保证跨实例唯一
}
```

### MuPacket ID {id="mupacket-id_1"}

MuPacket ID 由类ID与操作名组合而来，形如 `类ID:操作名`

### MuPacket Data {id="mupacket-data_1"}

MuPacket Data 是由 MuPacket 实现者自行约定的结构  
无论结构如何变化，其本质仍是一个 `JsonObject`(MuPacket API) 或 `object`(MuPacketAPI4TS)

## 方向

方向由接口标记表达，不再靠反序列化抛异常：

- `MuPacket2View`：MuCore → MuView
- `MuPacket2Core`：MuView → MuCore
- 双向的包同时实现两个，不另设第三种

发送端 `sendToView(packet: MuPacket2View)` / `sendToCore(packet: MuPacket2Core)`，发错方向在编译期就过不去。  
接收端在 `toPacket` 里先看 `MuPacketInfo` 实现了哪个方向接口，不匹配直接回错误包，不再进 `fromData`。  
`fromData` 此后只做一件事：把 `MP_DATA` 解析成对象。

MuPacketAPI4TS 里接口会被擦除，所以用静态字段承载同一语义：`MuPacket.DIRECTION` 默认 `"2view"`，
MuView 发出的包标 `"2core"`（如 `CreateMuEnvPacket`）；收到 `2core` 的包会在控制台告警。

> 注意：`ExecutableMuPacket` 表达的是能力（该包需要 MuCore 执行逻辑），不是方向，两者互不替代。

## 应答与 CID

请求-应答成对：MuView 发 `muview.*` 的请求，MuCore 执行完回一个 `mucore.internal:msg`，成功与否决定 `type` 是 `OK` 还是 `WARN` / `ERR`。

回包沿用请求的 CID，这样才能把应答对回具体某一次调用；反序列化失败、方向不符、MP_ID 未注册这些错误包同样回显请求的 CID，实在拿不到才新生成一个。

MuPacketAPI4TS 侧对应 `MuPacketError`：未注册的 MP_ID 不再用匿名基类装作解析成功。

## MP_ID 规则

- 形式：`mu(core|view).域:动作`，完整正则：
  `^mu(core|view)(\.[A-Za-z0-9\-_]{2,})+:[A-Za-z0-9\-_]{2,}(\.[A-Za-z0-9\-_]{2,})*$`
  MuCore 侧注册时强校验；MuPacketAPI4TS 目前不校验
- 演进：一个 MP_ID 固定对应一份 `MP_DATA` 结构，**改结构就新开 MP_ID**，已发布的 `MP_DATA` 冻结
- 版本：不设逐包版本位，两端靠同一份发布 + 握手比对兜底

## MuPacket ID 目录 (MuCore)

| MuPacket ID                 | Description                                                                   | Direction       |
|-----------------------------|-------------------------------------------------------------------------------|-----------------|
| mucore.internal:msg         | 带状态的消息包                                                                | MuCore → MuView |
| mucore.internal:status      | 状态包，传送 MuCore 的运行状态                                                | MuCore → MuView |
| mucore.internal.config:all  | 带有 MuCore 所有设置项的包（尚未实现，planned）                               | MuCore → MuView |
| mucore.muserver:info        | 带有指定 MS_ID 的服务器信息（MuServerInfo）和服务器状态（MuServerStatus）的包 | MuCore → MuView |
| mucore.muserver:list        | 用于返回所有服务器                                                            | MuCore → MuView |
| mucore.muserver:status      | 状态包，传送服务器的当前状态                                                  | MuCore → MuView |
| mucore.muserver:console.log | 服务器控制台日志                                                              | MuCore → MuView |
| mucore.muenv:info           | 带有环境信息的包                                                              | MuCore → MuView |
| mucore.muenv:list           | 用于返回所有的环境                                                            | MuCore → MuView |


## MuPacket ID 目录 (MuView)

| MuPacket ID            | Description                       | Direction       |
|------------------------|-----------------------------------|-----------------|
| muview.muserver:create | 用于传送创建服务器请求的包        | MuView → MuCore |
| muview.muserver:import | 用于传送引入服务器请求的包        | MuView → MuCore |
| muview.muenv:create    | 在 MuView 中创建 MuEnv 时传送的包 | MuView → MuCore |


## MuPacket Data 结构一览表

***以 [MuPacket ID 目录](#mupacket-id_1) 顺序排列***

### mucore.internal:msg
```TYPESCRIPT
{
    // 该消息的类型，这会影响在 MuView-FrontView 中 Toast 消息的表现
    type: "OK" | "TIP" | "WARN" | "ERR" | "INFO"
    
    // 消息文本
    msg: string
}
```

### mucore.internal:status
```TypeScript
{
    // 系统状态
    systemStatus: {
        // CPU 占用（百分比）
        CpuUsage: number
        // 内存占用（百分比）
        MemUsage: number
    }

    // 服务器数量统计
    serverStatus: {
        totalServer: number
        onlineServer: number
        offlineServer: number
    }

    // 应用信息
    appInfoStatus: {
        // 核心名
        core: string
        // 版本
        ver: string
    }
}
```

### mucore.internal.config:all
```TYPESCRIPT
{
    // 尚未实现
}
```

### mucore.muserver:info
```typescript
{
    // 服务器 ID，这通常是一个 8 位字符串
    // 如非特别指定，这通常是以符合该正则 [A-Z0-9]{8} 生成的随机字符串
    // 这通常是唯一值
    MS_ID: string
    MS_OP: {
        // 与 MS_ID 相同
        msid: string
        // 服务器名称（请注意这与服务器 ID 不是同一概念）
        name: string
        // 服务器的描述
        desc: string
        // 服务器的版本（是指该服务器的 Minecraft 版本）
        version: string
        // 服务器的类型（例如该服务器是 Paper 核心，那么服务器的类型即为 "paper"）
        type: string
        // 服务器所使用的环境名
        env: string
        // 服务器当前的状态，以数字标识
        // 0 -> 正在创建
        // 1 -> 已停止
        // 2 -> 服务器正在执行启动前任务（MuTask）中
        // 3 -> 正在运行
        // 4 -> 正在停止
        // 5 -> 重新启动中
        // 6 -> 遇错重启中
        // 7 -> 服务器连续报告了错误，被强行停止
        status: number
    }
}
```

### mucore.muserver:list
```TypeScript
[
    {
        // 服务器 ID，这通常是一个 8 位字符串
        // 这通常是唯一值
        msid: string,
        // 服务器名称（请注意这与服务器 ID 不是同一概念）
        name: string,
        // 服务器的描述
        desc: string,
        // 服务器的版本（是指该服务器的 Minecraft 版本）
        version: string,
        // 服务器的类型（例如该服务器是 Paper 核心，那么服务器的类型即为 "paper"）
        type: string,
        // 服务器所使用的环境名
        env: string,
        // 服务器当前的状态，以数字标识
        // 0 -> 正在创建
        // 1 -> 已停止
        // 2 -> 服务器正在执行启动前任务（MuTask）中
        // 3 -> 正在运行
        // 4 -> 正在停止
        // 5 -> 重新启动中
        // 6 -> 遇错重启中
        // 7 -> 服务器连续报告了错误，被强行停止
        status: number
    }
]
```

### mucore.muserver:status
```TypeScript
{
    // 服务器 ID，这通常是一个 8 位字符串
    // 如非特别指定，这通常是以符合该正则 [A-Z0-9]{8} 生成的随机字符串
    // 这通常是唯一值
    MS_ID: string

    // 服务器当前的状态，以数字标识
    // 0 -> 正在创建
    // 1 -> 已停止
    // 2 -> 服务器正在执行启动前任务（MuTask）中
    // 3 -> 正在运行
    // 4 -> 正在停止
    // 5 -> 重新启动中
    // 6 -> 遇错重启中
    // 7 -> 服务器连续报告了错误，被强行停止
    MS_OP: number
}
```

### mucore.muserver:console.log
```TypeScript
{
    // 服务器 ID
    MS_ID: string

    MS_OP: {
        // 日志级别
        lvl: "DEBUG" | "INFO" | "WARN" | "ERROR"
        // 日志内容
        msg: string
    }
}
```

### mucore.muenv:info
```TypeScript
{
    // 环境名
    EV_NAME: string
    EV_OP: {
        // 环境名
        EV_NAME: string
        // 环境标识版本
        // 注意这并非只有 Java 版本，它通常也会显示 OpenJDK 分支自有的版本号
        EV_VER: string
        // 字节码版本
        // 这通常决定该环境
        EV_CODE: number
        // 环境位置（绝对路径）
        EV_LOC: string
    }
}
```

### mucore.muenv:list
```TypeScript
[
    {
        // 环境名
        EV_NAME: string,
        // 环境标识版本
        // 注意这并非只有 Java 版本，它通常也会显示 OpenJDK 分支自有的版本号
        EV_VER: string,
        // 字节码版本
        // 这通常决定该环境
        EV_CODE: number,
        // 环境位置（绝对路径）
        EV_LOC: string,
    }
]
```

### muview.muserver:create
```TypeScript
{
    // 服务器信息
    MSI: {
        // 服务器 ID
        // 可以是 undefined，若如此，则其会是一个符合该正则的随机字符串（[A-Z0-9]{8}）
        msid: string | undefined
        // 服务器名称（请注意这与服务器 ID 不是同一概念）
        name: string
        // 服务器的描述
        desc: string
        // 服务器的版本（是指该服务器的 Minecraft 版本）
        version: string
        // 服务器的类型（例如该服务器是 Paper 核心，那么服务器的类型即为 "paper"）
        type: string
        // 服务器所使用的环境名
        env: string
    }
    
    // 服务器启动设置
    MSSC: {
        // 分配的最小内存数（单位：M）
        minMemory: number
        // 分配的最大内存数（单位：M）
        maxMemory: number
        // 是否需要开启原生 GUI 界面，默认是不需要开启（False）
        hasGui: boolean
        // 高级：对 JVM 的参数
        // 这通常是通过 MuView-MuServerCreator 选择的固定值，或用户自行决定的
        jvmFlags: string
    }
}
```

### muview.muserver:import
```TypeScript
{
    // 服务器核心位置
    LOC: string
    
    // 服务器启动设置
    MSSC: {
        // 分配的最小内存数（单位：M）
        minMemory: number
        // 分配的最大内存数（单位：M）
        maxMemory: number
        // 是否需要开启原生 GUI 界面，默认是不需要开启（False）
        hasGui: boolean
        // 高级：对 JVM 的参数
        // 这通常是通过 MuView-MuServerCreator 选择的固定值，或用户自行决定的
        jvmFlags: string
    } 
}
```

### muview.muenv:create
```TypeScript
{
    // 环境名称
    name: string
    // 环境可执行文件位置
    // 对于 Java 环境而言，应当对应 JAVA_HOME 的值
    path: string
}
```