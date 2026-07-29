---
name: go-project-init
description: 用于按团队 Go 项目规范初始化新的 Go Web/API 项目骨架。当用户要求初始化 Go 新项目、创建 Go 项目脚手架、套用基础代码库格式，或输入 /go-project-init 时触发。
disable-model-invocation: true
user-invocable: true
---

# Go 项目初始化

激活本技能后，在当前项目目录创建符合团队规范的 Go Web/API 项目骨架。

## 执行原则

- 默认在当前目录执行
- 如果当前目录已有业务代码，先停止并询问用户
- 如果用户指定 module 名，使用用户指定值
- 如果用户未指定 module 名，默认使用 `github.com/tiamxu/<当前目录名>`
- 默认使用 `github.com/tiamxu/kit/log`、`github.com/tiamxu/kit/http`、`github.com/tiamxu/kit/sql`
- 禁止生成真实密码、Token、API Key、第三方密钥
- 初始化完成后必须执行 `go mod tidy` 和 `go test ./...`

## 一、创建目录

创建以下目录：

```text
api/
service/
repo/
model/
types/
middleware/
config/
routes/
pkg/e/
pkg/constants/
pkg/utils/
doc/
sql/
```

## 二、生成文件

### 1. `pkg/e/code.go`

```go
package e

const (
	SUCCESS       = 200
	InvalidParams = 400
	Unauthorized  = 401
	Forbidden     = 403
	NotFound      = 404
	ERROR         = 500
)

var msgMap = map[int]string{
	SUCCESS:       "操作成功",
	InvalidParams: "参数错误",
	Unauthorized:  "未授权",
	Forbidden:     "无权限",
	NotFound:      "资源不存在",
	ERROR:         "服务器内部错误",
}

func GetMsg(code int) string {
	if msg, ok := msgMap[code]; ok {
		return msg
	}
	return "未知错误"
}
```

### 2. `api/common.go`

```go
package api

import (
	"net/http"

	"github.com/gin-gonic/gin"
	"{{MODULE}}/pkg/e"
)

type Response struct {
	Code    int         `json:"code"`
	Message string      `json:"message"`
	Data    interface{} `json:"data,omitempty"`
}

func Success(c *gin.Context, data interface{}) {
	c.JSON(http.StatusOK, Response{
		Code:    e.SUCCESS,
		Message: e.GetMsg(e.SUCCESS),
		Data:    data,
	})
}

func Error(c *gin.Context, httpStatus int, bizCode int, message string) {
	c.JSON(httpStatus, Response{
		Code:    bizCode,
		Message: message,
	})
}
```

### 3. `config/config.go`

```go
package config

import (
	"fmt"
	"os"

	"github.com/koding/multiconfig"
	"{{MODULE}}/repo"
	kithttp "github.com/tiamxu/kit/http"
	"github.com/tiamxu/kit/log"
	"github.com/tiamxu/kit/sql"
)

var cfg *Config

type Config struct {
	ENV     string               `yaml:"env"`
	Log     log.Config           `yaml:"log"`
	HttpSrv kithttp.ServerConfig `yaml:"http_srv"`
	DB      *sql.Config          `yaml:"db"`
}

func LoadConfig() *Config {
	cfg = new(Config)

	env := os.Getenv("ENV")
	if env == "" {
		env = "dev"
	}

	configPath := "config/config.yaml"
	switch env {
	case "dev":
		configPath = "config/config-dev.yaml"
	case "test":
		configPath = "config/config-test.yaml"
	case "prod":
		configPath = "config/config-prod.yaml"
	}

	multiconfig.MustLoadWithPath(configPath, cfg)
	return cfg
}

func (c *Config) Initial() error {
	if err := log.InitLogger(&c.Log); err != nil {
		return fmt.Errorf("log init failed: %w", err)
	}

	if err := repo.Init(c.DB); err != nil {
		return fmt.Errorf("database init failed: %w", err)
	}

	log.Infof("config initialized, env: %s", c.ENV)
	return nil
}

func GetConfig() *Config {
	return cfg
}
```

### 4. `config/config-dev.yaml`

```yaml
env: dev

log:
  level: info
  type: stdout
  format: console

http_srv:
  address: ":8800"
  keep_alive: true
  read_timeout: 30s
  write_timeout: 30s
  multipart_memory: 33554432
  body_limit: 8388608
  cors:
    allow_origins:
      - "*"
    allow_methods:
      - GET
      - POST
      - PUT
      - DELETE
      - OPTIONS
    allow_headers:
      - Origin
      - Content-Type
      - Accept
      - Authorization
      - X-Request-ID
    expose_headers:
      - Content-Length
      - Content-Type
      - X-Request-ID
      - X-Response-Time
    allow_credentials: false
    max_age: 12h

db:
  driver: mysql
  host: 127.0.0.1
  port: 3306
  database: app
  username: root
  password: ""
  max_idle_conns: 5
  max_open_conns: 10
  conn_max_lifetime: 300
  conn_max_idle_time: 60
```

同时复制生成 `config/config.yaml`，内容与 `config/config-dev.yaml` 一致。

### 5. `repo/init.go`

```go
package repo

import (
	"fmt"

	"github.com/tiamxu/kit/sql"
)

var DB *sql.DB

func Init(cfg *sql.Config) error {
	if cfg == nil {
		return fmt.Errorf("database config is nil")
	}

	db, err := sql.Connect(cfg)
	if err != nil {
		return err
	}

	DB = db
	return nil
}

func Close() error {
	if DB != nil {
		return DB.Close()
	}
	return nil
}

func IsNoRows(err error) bool {
	return sql.IsNoRows(err)
}
```

### 6. `routes/routes.go`

```go
package routes

import (
	"github.com/gin-gonic/gin"
	"{{MODULE}}/api"
)

func InitRoutes(r *gin.Engine) {
	r.GET("/health", func(c *gin.Context) {
		api.Success(c, gin.H{"status": "up"})
	})

	v1 := r.Group("/api/v1")
	{
		auth := v1.Group("/auth")
		{
			_ = auth
		}
	}
}
```

### 7. `main.go`

```go
package main

import (
	"context"
	"os"
	"os/signal"
	"syscall"
	"time"

	"{{MODULE}}/config"
	"{{MODULE}}/repo"
	"{{MODULE}}/routes"
	kithttp "github.com/tiamxu/kit/http"
	"github.com/tiamxu/kit/log"
)

func main() {
	cfg := config.LoadConfig()
	if err := cfg.Initial(); err != nil {
		log.Fatalf("config init failed: %v", err)
	}
	defer log.Sync()
	defer repo.Close()

	srv := kithttp.NewServer(cfg.HttpSrv)
	routes.InitRoutes(srv.Engine)

	log.Infof("server starting on %s", cfg.HttpSrv.Address)
	if err := srv.Start(); err != nil {
		log.Fatalf("server start failed: %v", err)
	}

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	log.Infoln("server shutting down")
	if err := srv.Shutdown(ctx); err != nil {
		log.Errorf("server shutdown failed: %v", err)
	}
}
```

### 8. `AGENTS.md`

生成项目执行规范，内容使用团队通用模板，必须包含：

- 任务流程
- 目录规范
- 分层规范
- 命名规范
- 方法对应规范
- 路由规范
- 响应与错误规范
- 技术栈与基础代码库规范
- 文档同步规范
- 最小变更原则

Go 项目技术栈规则必须包含：

```markdown
- Go 项目默认使用 `kit/log`、`kit/http`、`kit/sql`
- 禁止擅自替换日志库、HTTP 框架、数据库访问库或 ORM
- 如需偏离默认技术栈，必须先说明原因、替代方案和风险，并等待确认
```

### 9. `CLAUDE.md`

```markdown
# CLAUDE.md

## 执行规范

本项目开发规范以 `AGENTS.md` 为准。

Claude Code 执行任何代码修改前，必须先阅读：

- `AGENTS.md`
- `doc/项目开发规范.md`

## 核心约束

- 默认使用中文回复
- 修改前先判断任务复杂度
- 复杂任务必须先出方案，等待确认后再改代码
- 按 `api -> service -> repo -> model` 分层
- Go 项目默认使用 `kit/log`、`kit/http`、`kit/sql`
- 禁止擅自替换日志库、HTTP 框架、数据库访问库或 ORM
- 禁止硬编码密钥、Token、数据库密码
- 每次修改后说明修改文件、影响范围、是否需要联动修改、是否已更新文档

## 文档入口

- AI 执行规范：`AGENTS.md`
- 完整开发规范：`doc/项目开发规范.md`
```

### 10. `doc/项目开发规范.md`

生成完整开发规范，内容使用团队通用模板，并补充：

```markdown
## 技术栈与基础代码库规范

新项目优先基于指定基础代码库创建，不从零搭建。

Go 项目默认技术栈：

| 能力 | 默认库 |
|---|---|
| 日志 | `github.com/tiamxu/kit/log` |
| HTTP 服务 | `github.com/tiamxu/kit/http` |
| 数据库 | `github.com/tiamxu/kit/sql` |

使用约定：

- 启动阶段调用 `log.InitLogger`
- 程序退出前调用 `log.Sync`
- HTTP 服务默认使用 `kithttp.NewServer`
- 注册路由后调用 `srv.Start()` 启动监听
- 路由统一放在 `routes/`
- 数据库默认使用 `kit/sql`
- SQL 只允许写在 `repo/`
- 事务优先使用 `TransactCallback` 或 `TransactCallbackCtx`
- 查询无数据使用 `sql.IsNoRows` 判断
```

### 11. `README.md`

````markdown
# Project Name

## 技术栈

- Go
- Gin
- `github.com/tiamxu/kit/log`
- `github.com/tiamxu/kit/http`
- `github.com/tiamxu/kit/sql`

## 项目结构

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

## 快速开始

```bash
go mod tidy
go run main.go
```

## 文档

- [项目开发规范](doc/项目开发规范.md)
- [AI 执行规范](AGENTS.md)
- [Claude Code 入口](CLAUDE.md)
- [接口文档](doc/接口文档.md)
- [设计文档](doc/设计文档.md)
````

### 12. `doc/设计文档.md`

```markdown
# 设计文档

## 一、背景

说明需求背景和目标。

## 二、范围

说明本次功能范围和边界。

## 三、数据结构

| 字段 | 类型 | 说明 |
|---|---|---|
| id | bigint | 主键 |

## 四、接口设计

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | /api/v1/resources | 列表 |

## 五、实现计划

- [ ] Repo 层
- [ ] Service 层
- [ ] API 层
- [ ] 文档同步
```

### 13. `doc/接口文档.md`

````markdown
# 接口文档

## 统一响应

```json
{
  "code": 200,
  "message": "操作成功",
  "data": {}
}
```

## 错误码

| 错误码 | 说明 |
|---|---|
| 200 | 操作成功 |
| 400 | 参数错误 |
| 401 | 未授权 |
| 500 | 服务器内部错误 |

## 接口列表

### GET /health

健康检查。
````

## 三、初始化命令

按顺序执行：

```bash
go mod init {{MODULE}}
go get github.com/gin-gonic/gin
go get github.com/koding/multiconfig
go get github.com/tiamxu/kit
go mod tidy
go test ./...
```

如果 `go mod init` 已执行或 `go.mod` 已存在，跳过 `go mod init`。

## 四、占位符替换

生成文件时必须替换：

```text
{{MODULE}} -> 实际 module 名
Project Name -> 当前项目名
```

## 五、交付报告

完成后输出：

```text
Go 项目脚手架创建完成

项目结构：
- api/common.go
- config/config.go
- config/config-dev.yaml
- config/config.yaml
- repo/init.go
- routes/routes.go
- pkg/e/code.go
- main.go
- AGENTS.md
- CLAUDE.md
- README.md
- doc/项目开发规范.md
- doc/设计文档.md
- doc/接口文档.md

默认技术栈：
- github.com/tiamxu/kit/log
- github.com/tiamxu/kit/http
- github.com/tiamxu/kit/sql

验证结果：
- go mod tidy
- go test ./...
```

## 六、禁止事项

- 禁止生成真实密钥、真实数据库密码、真实 Token
- 禁止引入 ORM
- 禁止替换 `kit/log`、`kit/http`、`kit/sql`
- 禁止把 SQL 写到 `api` 或 `service`
- 禁止使用 `/getXxx`、`/createXxx` 这类动词式路由
