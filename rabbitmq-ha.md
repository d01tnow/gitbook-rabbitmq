# Rabbitmq HA 策略

Rabbitmq 集群解决了一个 broker 的单点故障问题, 提高了可用性. 但是, 集群并未提高队列的可用性. Rattitmq 队列和交换机的 HA (高可用性) 通过策略(Policy)实现的. 队列的 HA 策略仅提供高可用性, 并不能提供负载均衡功能. 负载均衡可以使用 sharding 插件或者人为分散队列到不同节点实现.

## 名词

- 节点: 指 rabbitmq 的物理节点
- master: 角色为主. 除发布外, 其他动作全部需要在 master 上执行.
- mirror: 主队列或交换机的镜像

## HA 策略的参数

主要的参数和含义

| 项目                   | 含义                              |
| ---------------------- | --------------------------------- |
| ha-mode                | HA 模式                           |
| ha-params              | 特定 ha-mode 的附加参数           |
| ha-sync-mode           | HA 同步模式                       |
| ha-promote-on-failure  | 故障时 HA 提升master的策略        |
| ha-promote-on-shutdown | 关闭节点时, HA 提升 master 的策略 |

### ha-mode

| 参数    | 含义                                                         | ha-params                      |
| ------- | ------------------------------------------------------------ | ------------------------------ |
| all     | 镜像到所有节点.增加/减少节点时, 动态的增加/减少镜像数量.     | 忽略 ha-params 参数            |
| exactly | 指定队列或交换机的总个数, 队列或交换机所在位置不指定.        | 类型: number. 必须 > 0.        |
|         | 仅一个 master                                                | N=1                            |
|         | 实际的个数=min(ha-params指定个数, 节点个数).                 | N>1                            |
| nodes   | 指定队列或交换机所在的节点.                                  | 类型: list, 内容是 string 类型 |
|         | **如该指定的节点不可达, 不报错；如果指定的节点全不可达, 队列或交换机被声明到当前连接的节点**. |                                |

### ha-sync-mode

指定 HA 同步模式. 指新镜像加入前的消息的同步模式. **同步过程是阻塞的**, 阻塞消息的发布与消费. 不同步, 并不会影响消息的生产与消费. 但是, 当 ha-promote-on-failure=when-synced 时, 队列不同步状态下不会提升 mirror 为 master, 造成数据丢失或无法消费.

| 选项      | 含义           | 备注   |
| --------- | -------------- | ------ |
| automatic | 自动同步旧消息 |        |
| manual    | 手动同步旧消息 | 默认值 |

### ha-promote-on-failure

设置队列 master 发生故障时, 队列同步状态对提升新 master 动作的影响. 该参数默认值为 always. 目的是为了保证可用性, 代价是在不同步状态下发生故障时, 有可能丢失数据.

| 选项        | 含义                                  | 备注   |
| ----------- | ------------------------------------- | ------ |
| when-synced | 仅在同步状态下才提升 mirror 为 master |        |
| always      | 发生故障时忽略同步状态, 提升新 master | 默认值 |

### ha-promote-on-shutdown

设置某队列 master 所在的节点正常关闭时, 队列同步状态对提升新 master 动作的影响. shutdown 是人为可控的动作, 故该参数默认值是 when-synced. 为了避免出现 shutdown 时有未同步的队列不提升新 master 的情况, 一般地, 在 shutdown 前, 通过 ha-mode=nodes 策略, 将队列迁移到其他节点.

| 选项        | 含义                                  | 备注   |
| ----------- | ------------------------------------- | ------ |
| when-synced | 仅在同步状态下才提升 mirror 为 master | 默认值 |
| always      | 关闭节点时忽略同步状态, 提升新 master |        |

## 管理UI

### 新建/更新策略

注意事项

- 优先级是>=0的整数, 默认值为 0. 数值越大优先级越高. 高优先级覆盖低优先级策略.
- 名称相同即更新策略.
- 删除高优先级策略或者不再匹配策略(比如更改了匹配模式 pattern 或者 apply to 类型不匹配或者 priority 降低)后, 按优先级匹配现有的策略.

## 测试与验证

### 策略生效过程对生产消费的影响

- 最大的影响是队列同步. 队列镜像同步会阻塞消息的发布与消费.
- 默认的 ha-sync-mode=manual. 通过 ha-nodes 方式强制迁移队列位置(比如: 由 node1 迁移到 node2)过程是先会在 node2 创建镜像, 然后**等待同步**(默认manual, 不会同步, 会出现队列的 master 为 node1, mirrors 为 node2的情况. 需要手动同步), 同步后才提升 node2 上的镜像, 最后删除 node1 上队列. 同步过程会影响消息的发布与消费.
- 镜像提升为 master 过程不会影响消息的发布与消费.

### docker 容器操作对 HA 的影响

docker stop 会触发 rabbitmqctl stop_app, 是正常的 shutdown. 在 ha-promote-on-shutdown=when-synced时, docker stop 容器不会触发提升 master 动作.

docker rm -f 会触发 failover, 在 ha-promote-on-failure=always 时, 提升一个 mirror 为 master, 未同步的消息丢失.

## 客户端开发

- basic.consume 需要加入参数 x-cancel-on-ha-failover. 这个参数保证在队列异常不可用(比如: 队列被删除, 节点宕机等)发生 failover 时, consumer 接收到 Cancellation 通知, 以便客户端进行后续处理.
- Java 客户端 listener 模式, 在队列被删除时会退出 listen. 在提升新 master 时, 底层库做重连处理, 会重建 channel, 不需要用户干预.