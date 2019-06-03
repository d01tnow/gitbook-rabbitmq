# Rabbitmq HA 策略

rabbitmq 队列和交换机的 HA 是通过策略(Policy)实现的. 队列的HA策略仅提供高可用, 并不能提供负载均衡功能. 负载均衡可以使用 sharding 插件.

## HA 策略

主要的参数及含义

项目 | 含义
--- | ---
ha-mode | HA 模式

### ha-mode
