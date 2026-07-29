# AGENTS.md

## 一、基本规则

- 默认使用中文回复
- 修改代码前先阅读项目内 `AGENTS.md`
- 不输出无关内容，说明保持简洁明确
- 未经确认，不擅自扩展需求或增加功能
- 不主动编写兼容性代码，除非用户明确要求

## 二、任务流程

简单任务可直接执行：

- 只涉及 1 个文件
- 需求明确
- 不修改接口协议、数据结构、核心流程

执行流程：

1. 修改代码
2. 执行必要测试
3. 输出修改说明
4. 如用户明确要求提交或推送，则测试通过后执行 `git commit` 和 `git push`

复杂任务必须先出方案，等待用户确认后再自动开发：

- 涉及多个模块
- 涉及 3 个及以上文件
- 修改接口协议、数据结构、核心流程
- 新增依赖
- 调整架构或分层

确认后的执行流程：

1. 修改代码
2. 更新必要文档
3. 执行自动化测试
4. 测试通过后执行 `git commit`
5. 执行 `git push`
6. 输出提交信息、推送结果、测试结果和影响范围

需求不明确时必须先提问，禁止猜测实现。

## 三、技术栈与基础代码库规范

- 新项目优先基于指定基础代码库创建
- Go 项目默认使用 `kit/log`、`kit/http`、`kit/sql`
- HTTP 服务默认使用 `kithttp.NewServer`
- 注册路由后调用 `srv.Start()` 启动监听
- 数据库默认使用 `kit/sql`
- 禁止擅自替换日志库、HTTP 框架、数据库访问库或 ORM
- 如需偏离默认技术栈，必须先说明原因、替代方案和风险，并等待确认

## 四、目录规范

```text
api/            接口层
service/        业务逻辑层
repo/           数据访问层
routes/         路由定义
model/          数据库结构体
types/          请求 / 响应结构体
middleware/     中间件
config/         配置加载
pkg/            公共包
doc/            项目文档
sql/            数据库脚本
```

## 五、分层规范

依赖方向：

```text
api -> service -> repo -> model
```

- `api`：只处理请求、参数校验、调用 service、返回响应
- `service`：只处理业务逻辑和业务规则
- `repo`：只处理 SQL 和数据访问
- `model`：只表示数据库结构
- `types`：只表示请求和响应结构

禁止跨层反向依赖。禁止在 `api` 写 SQL，禁止在 `repo` 写业务逻辑。

## 六、命名规范

- Handler：`XxxHandler`
- Service：`XxxService`
- 构造函数：`NewXxxHandler()`、`NewXxxService()`
- 请求结构体：`XxxReq`
- 响应结构体：`XxxResp`
- 列表响应：`XxxListResp`
- 详情响应：`XxxDetailResp`
- Repo 函数：`CreateXxx`、`GetXxxByID`、`GetXxxList`、`UpdateXxx`、`DeleteXxx`
- 文件命名：按模块小写命名，如 `api/product.go`、`service/product.go`、`repo/product.go`

## 七、方法对应规范

- 新增：`api.Create` -> `service.Create` -> `repo.CreateXxx`
- 详情：`api.Detail` -> `service.GetByID` -> `repo.GetXxxByID`
- 列表：`api.List` -> `service.GetList` -> `repo.GetXxxList`
- 更新：`api.Update` -> `service.Update` -> `repo.UpdateXxx`
- 删除：`api.Delete` -> `service.Delete` -> `repo.DeleteXxx`
- 状态变更：`api.UpdateStatus` -> `service.UpdateStatus` -> `repo.UpdateXxxStatus`
- 批量操作：统一使用 `BatchXxx`

## 八、路由规范

- 路由统一放在 `routes/` 目录
- 对外 API 推荐使用版本前缀，如 `/api/v1`
- 内部接口可按项目复杂度决定是否使用版本前缀
- 资源使用复数名词，如 `/api/v1/users`
- 单个资源使用 `:id`，如 `/api/v1/users/:id`
- CRUD 使用标准 HTTP Method
- 非 CRUD 动作放在资源 ID 后，如 `/api/v1/users/:id/disable`
- 认证接口放在 `/api/v1/auth`
- 管理端或内部接口按项目需要分组，如 `/api/v1/admin`、`/api/v1/internal`
- 禁止使用 `/getXxx`、`/createXxx` 这类动词式路径

## 九、响应与错误规范

- 只有 `api` 层允许处理 HTTP 响应
- 所有接口使用统一响应结构
- 错误码统一放在 `pkg/e`
- 禁止直接返回原始 `error`
- 禁止暴露 SQL 错误、数据库错误、堆栈信息

## 十、安全与依赖规范

- 配置放在 `config/` 或环境变量中
- 禁止硬编码数据库密码、Token、API Key、第三方密钥
- 禁止随意新增依赖
- 新增依赖必须说明使用目的、替代方案、风险点
- 禁止为了少量代码引入大型框架

## 十一、文档同步规范

以下变更必须同步 `doc/`：

- 新增或修改接口
- 修改请求 / 响应结构
- 修改数据库结构
- 修改核心业务流程
- 修改错误码
- 修改配置项

## 十二、最小变更原则

- 只改必要代码
- 不重构无关代码
- 不调整无关格式
- 不修改无关模块
- 不扩大影响范围
- 不引入不必要抽象

## 十三、修改后输出要求

每次修改后必须说明：

- 修改文件列表
- 修改内容摘要
- 本次修改影响范围
- 是否兼容已有功能
- 是否需要联动修改
- 是否已更新文档
## 十四、自动化测试规范

修改 Go 代码后默认执行：

```bash
go test ./...
```

涉及依赖、生成代码或 `go.mod` 变化时执行：

```bash
go mod tidy
go test ./...
```

测试失败时：

- 必须优先定位并修复
- 不允许提交失败状态
- 如失败与本次修改无关，必须说明原因和影响范围

## 十五、Git 提交与推送规范

默认允许在用户明确要求提交或推送代码，或复杂任务方案确认后进入自动开发流程时执行 `git commit` 和 `git push`。

提交和推送前必须执行：

- `git status`
- `git diff`
- 项目测试命令，如 `go test ./...`
- 检查无密钥、Token、数据库密码等敏感信息
- 确认只包含本次任务相关修改

禁止：

- 测试失败时提交或推送
- 提交无关文件
- 未经单独确认执行部署
- 使用 `git reset --hard` 等破坏性 Git 命令
