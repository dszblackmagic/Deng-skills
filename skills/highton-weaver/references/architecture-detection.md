# 目标项目架构识别

## 1. 目标

在写 Weaver 封装代码之前，先判断当前项目应该生成哪条链路。

不要默认所有项目都用：

- `Provider -> Service -> Impl -> Test`

这只是兜底链路，不是强制链路。

## 2. 识别顺序

先扫描代码库里现有的类名、包名和调用方向，再决定落点。

优先看：

- 是否存在 `*Provider`
- 是否存在 `*Facade`
- 是否存在 `*Controller`
- 是否存在 `*Service`
- 是否存在 `*Client`、`*Gateway`、`*Remote`、`*Adapter`
- 是否存在 `impl` 包或 `*Impl`
- 是否存在单独的 API 模块
- 是否存在统一的 web 模块

## 3. 决策规则

### 3.1 已存在 Provider 风格

如果项目已有大量：

- `XxxProvider`
- `XxxService`
- `XxxServiceImpl`

优先生成：

- `Provider -> Service -> Impl`

如果项目已有调用样例测试，再按现有风格补 `Test`。

### 3.2 已存在 Facade 风格

如果项目更偏向：

- `Facade`
- `ApplicationService`
- `DomainService`

优先生成：

- `Facade -> Service -> Impl`

此时不要硬塞一个 `Provider`。

### 3.3 已存在 Controller 风格

如果这是典型 web 项目，外层主要是：

- `Controller`
- `RestController`

并且内部已有：

- `Service`
- `Client` 或 `Gateway`

优先生成：

- `Controller -> Service -> Client`

或者：

- `Controller -> Service -> Gateway -> Impl`

这里的 Weaver OA HTTP 代码通常应该放在：

- `Client`
- `Gateway`
- `RemoteServiceImpl`

而不是直接写进 Controller。

### 3.4 只有 Service 风格

如果项目没有明显的 `Provider`、`Facade`、`Controller` 外层，而是直接暴露：

- `Service`
- `ServiceImpl`

那就生成：

- `Service -> Impl`

不要额外发明一层。

### 3.5 已有统一远程调用层

如果项目已有：

- `OpenApiClient`
- `RemoteClient`
- `ThirdPartyGateway`
- `ExternalAdapter`

优先把 Weaver OA HTTP 调用放进这类层里，再由上层服务编排。

推荐链路：

- `Service -> Client`
- `Service -> Gateway`
- `Service -> Adapter`

## 4. DTO 落点规则

### 4.1 有 API 模块

如果项目拆成：

- `api`
- `service`
- `web`

优先把对外契约 DTO 放在 API 模块。

### 4.2 只有单模块

如果只有单模块项目：

- request/response DTO 跟随现有包结构

例如：

- `dto/request`
- `dto/response`
- `model/request`
- `model/response`

### 4.3 Web 层专用 DTO

如果 Controller 层和内部服务层已经明确分开：

- Controller 入参/出参使用 web DTO
- 内部 Weaver 调用可使用更贴近 OA 的 DTO

不要混成一个类，除非项目本来就这样做。

## 5. 测试落点规则

### 5.1 用户未明确要求

默认先识别测试承载点和测试风格。

如果仓库规则没有明确禁止，就应该把测试纳入完整交付范围。

### 5.2 用户明确要求补测试

先看现有项目主流测试形式：

- `@SpringBootTest`
- 纯单元测试
- 外部接口调用样例测试

沿用现有方式，不要擅自切换测试范式。

### 5.3 默认测试决策

优先顺序：

1. 扩展现有同类接口测试类
2. 新增同目录下的同风格测试类
3. 只有在仓库规则或项目结构明确不适合时才跳过

每次都先回答这两个问题：

- 新接口对应的测试应该落在哪个类
- 这个测试应该覆盖哪一个新增方法或哪组新增方法

## 6. 常量与枚举决策

如果项目里本来就有统一的：

- URL 常量类
- 接口枚举类

继续沿用。

如果项目没有：

- 不要为了整齐硬加这一层

## 7. 快速判断表

看到这些结构时，优先采用以下链路：

- 有 `Provider` 和 `ServiceImpl`
  - `Provider -> Service -> Impl`
- 有 `Controller` 和 `Client`
  - `Controller -> Service -> Client`
- 有 `Controller` 和 `Gateway`
  - `Controller -> Service -> Gateway`
- 有 `Facade`
  - `Facade -> Service -> Impl`
- 只有 `Service`
  - `Service -> Impl`

## 8. 最后检查

- 你生成的链路是不是项目里已经大量存在的链路
- OA HTTP 代码是不是放在了最合理的下沉层
- 外层是不是只承担编排和转发，不直接处理 OA 细节
- DTO、constants、tests 的落点是不是跟项目现状一致
- 你是不是已经为新增接口确定了明确的测试承载类，而不是把测试遗漏掉
