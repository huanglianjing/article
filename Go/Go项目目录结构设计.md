# 1. 概述

Go 官方没有规定项目目录结构，`go mod init` 之后怎么组织代码全由团队决定。灵活性带来了问题：同样是 Go 项目，有的把所有代码堆在 `main` 包里，有的照搬 Java 写出十几层目录。

好处是团队可以自由选择，坏处是很容易出现两类问题：

- **过度扁平**：所有代码都放在 `main` 包或少数几个包里，业务一复杂就难以维护；
- **过度设计**：照搬 Java 的分层方式，堆出十几层目录和大量接口，写一个 CRUD 要改七八个文件。

一个好的目录结构应该做到以下几点：

- **职责清晰**：看目录名就知道代码放在哪里。
- **依赖单向**：上层依赖下层，核心业务不依赖具体技术实现。
- **易于测试**：业务逻辑可以脱离数据库、HTTP 框架进行单元测试。
- **便于演进**：替换 ORM、增加 gRPC 接口时，改动范围可控。

本文介绍两种在 Go 社区常用的架构风格：**简洁架构（Clean Architecture）** 和 **领域驱动设计（DDD）**。

参考 `golang-standards/project-layout` 和 Go 官方文档，有几个 Go 社区普遍认可的目录约定，两种架构共享同一套外层骨架，区别只在 `internal/` 内部。

```go
project/
├── cmd/                  # 程序入口，每个子目录对应一个可执行文件
│   └── server/
│       └── main.go
├── config/               # 配置文件模板
├── internal/             # 私有业务代码，外部模块无法 import
├── pkg/                  # 可被外部复用的公共库（谨慎使用）
├── proto/                # protobuf 定义（如有 gRPC）
├── script/               # 构建、部署、数据库迁移脚本
├── deployment/           # Dockerfile、K8s yaml
├── doc/                  # 文档、Swagger
├── go.mod
└── Makefile
```

# 2. 简介架构

简洁架构由 Robert C. Martin（Uncle Bob）提出，核心是**依赖规则（The Dependency Rule）**：

> 源码依赖只能从外层指向内层，内层不能知道外层的任何信息。

**分层与依赖**

| 层      | 职责                                       | 不应该做的事                  |
| ------- | ------------------------------------------ | ----------------------------- |
| api     | 接收请求、参数校验、调用 service、组装响应 | 写业务逻辑、直接操作数据库    |
| service | 实现业务逻辑、编排流程、控制事务           | 感知 HTTP、依赖具体数据库实现 |
| repo    | 数据持久化，对接 MySQL、Redis、第三方 API  | 包含业务判断                  |
| model   | 定义业务实体、业务错误                     | 依赖任何其他层                |

依赖方向如下：

```
        ┌─────────┐
        │   api   │
        └────┬────┘
             │ 调用
        ┌────▼────┐        ┌─────────┐
        │ service │◄───────│  repo   │  repo 实现 service 定义的接口
        └────┬────┘ 实现    └────┬────┘
             │                   │
        ┌────▼───────────────────▼────┐
        │            model            │  被所有层引用，自身不依赖任何层
        └─────────────────────────────┘
```

这里的核心是**依赖倒置**：service 不直接依赖 repo 的具体实现，而是在 service 包里**定义接口**，由 repo 去实现。这也符合 Go 的惯用法：**接口由使用方定义**。

**目录结构**

```
clean-arch-demo/
├── cmd/
│   └── server/
│       └── main.go              # 依赖组装 + 启动
├── config/
│   └── config.yaml              # 配置文件
├── internal/
│   ├── api/                     # 接口层
│   │   ├── handler/
│   │   │   └── user.go          # HTTP handler
│   │   ├── dto/
│   │   │   └── user.go          # 请求/响应结构体
│   │   ├── middleware/
│   │   │   ├── auth.go
│   │   │   └── logger.go
│   │   └── router.go            # 路由注册
│   ├── service/                 # 业务层
│   │   ├── user.go              # 业务逻辑 + repo 接口定义
│   │   └── user_test.go
│   ├── repo/                    # 数据层
│   │   ├── mysql/
│   │   │   ├── user.go          # UserRepo 的 MySQL 实现
│   │   │   └── po.go            # 数据库表映射结构体
│   │   └── redis/
│   │       └── user_cache.go
│   ├── model/                   # 模型层
│   │   ├── user.go
│   │   └── errors.go
│   └── config/                  # 配置层
│       └── config.go
├── go.mod
└── Makefile
```

**可测试性**

service 只依赖接口，单元测试可以用内存实现代替数据库。

# 3. 领域驱动设计

领域驱动设计（DDD）由 Eric Evans 提出，目标是**让软件结构贴合业务领域**。它分为两部分：

战略设计：决定系统怎么拆分。

- **限界上下文（Bounded Context）**：一个模型有效的边界。比如“订单”里的商品和“库存”里的商品，含义不同，应该分开建模；
- **上下文映射（Context Mapping）**：上下文之间如何协作，例如防腐层（ACL）、共享内核等。

战术设计：决定一个上下文内部怎么建模。

| 概念                            | 说明                                               |
| ------------------------------- | -------------------------------------------------- |
| 实体（Entity）                  | 有唯一标识，生命周期内状态可变                     |
| 值对象（Value Object）          | 没有标识，靠属性判断相等，不可变，例如金额、地址   |
| 聚合（Aggregate）               | 一组强一致的对象集合，外部只能通过聚合根访问       |
| 领域服务（Domain Service）      | 不适合放在某个实体上的领域逻辑                     |
| 领域事件（Domain Event）        | 领域中已经发生、值得关注的事情                     |
| 仓储（Repository）              | 以聚合为单位做持久化的抽象                         |
| 应用服务（Application Service） | 编排领域对象，处理事务、权限等，本身不包含业务规则 |

**四层架构**

| 层                           | 职责                                   |
| ---------------------------- | -------------------------------------- |
| Interfaces（用户接口层）     | HTTP / gRPC / MQ Consumer / CLI        |
| Application（应用层）        | 用例编排、事务、DTO 转换               |
| Domain（领域层）             | 聚合、实体、值对象、领域服务、仓储接口 |
| Infrastructure（基础设施层） | 仓储实现、MQ、缓存、外部服务           |

依赖方向如下：

```
   ┌──────────────────┐
   │  interfaces      │  HTTP / gRPC / MQ 消费者
   └────────┬─────────┘
            ▼
   ┌──────────────────┐
   │  application     │  用例编排、事务、DTO 转换
   └────────┬─────────┘
            ▼
   ┌──────────────────┐        ┌──────────────────┐
   │     domain       │◄───────│  infrastructure  │
   │ (核心，零外部依赖) │  实现   │ DB / MQ / 外部API │
   └──────────────────┘        └──────────────────┘
```

domain 层位于最内层，不依赖任何其他层，也不依赖任何框架。 仓储接口定义在 domain 中，由 infrastructure 实现。

**目录结构**

DDD 目录结构和简洁架构最大的区别在于：先按业务（限界上下文）拆分，再按技术分层。

```
ddd-demo/
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── interfaces/                  # 接口层（用户界面/适配器层）
│   │   ├── http/
│   │   │   ├── order_handler.go
│   │   │   └── router.go
│   │   ├── grpc/
│   │   │   └── order_server.go
│   │   └── consumer/
│   │       └── payment_consumer.go  # 消费支付回调消息
│   │
│   ├── application/                 # 应用层
│   │   ├── order_app.go             # 应用服务：用例编排
│   │   ├── command.go               # 写操作入参
│   │   ├── query.go                 # 读操作入参/出参
│   │   └── port.go                  # 事务管理器、事件发布器等端口接口
│   │
│   ├── domain/                      # 领域层
│   │   ├── order/                   # 订单聚合
│   │   │   ├── order.go             # 聚合根
│   │   │   ├── order_item.go        # 实体
│   │   │   ├── status.go            # 状态枚举
│   │   │   ├── event.go             # 领域事件
│   │   │   ├── errors.go            # 领域错误
│   │   │   ├── repository.go        # 仓储接口
│   │   │   └── order_test.go
│   │   ├── product/                 # 商品聚合
│   │   │   ├── product.go
│   │   │   └── repository.go
│   │   ├── shared/                  # 共享内核：通用值对象
│   │   │   └── money.go
│   │   └── service/                 # 领域服务
│   │       └── pricing.go
│   │
│   └── infrastructure/              # 基础设施层
│       ├── persistence/
│       │   └── mysql/
│       │       ├── order_repo.go
│       │       ├── order_po.go
│       │       └── tx.go            # 事务实现
│       ├── mq/
│       │   └── kafka_publisher.go
│       └── config/
│           └── config.go
├── go.mod
└── Makefile
```

