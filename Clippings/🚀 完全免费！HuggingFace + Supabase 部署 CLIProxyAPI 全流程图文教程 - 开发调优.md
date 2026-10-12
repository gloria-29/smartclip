---
Title: "🚀 完全免费！HuggingFace + Supabase 部署 CLIProxyAPI 全流程图文教程 - 开发调优"
Url: "https://linux.do/t/topic/1677712"
Origin: "LINUX DO"
Description: "本文基于实际踩坑经历整理，从零开始讲清楚如何在 HuggingFace Space 上部署 CLIProxyAPI，接入免费的 Supabase PostgreSQL 作为持久化后端，并配置保活机制，全程免费。 \n感谢社区原帖：https…"
Created: "2026-10-12 23:56:32"
Cover: "https://cdn3.ldstatic.com/optimized/4X/1/9/9/19911d108a5d17566adf15ae76a3d8f11a76dca1_2_1024x500.png"
---

Info

**真诚**、 **友善**、 **团结**、 **专业**，共建你我引以为荣之社区。 [《社区准则》](https://linux.do/guidelines)

[开发调优](https://linux.do/c/develop/4)

[ozer_23](https://linux.do/u/ozer_23)

楼主

> 本文基于实际踩坑经历整理，从零开始讲清楚如何在 HuggingFace Space 上部署 CLIProxyAPI，接入免费的 Supabase PostgreSQL 作为持久化后端，并配置保活机制， **全程免费**。
> 
> 感谢社区原帖： [https://linux.do/t/topic/1675528](https://linux.do/t/topic/1675528) 的思路，本文在此基础上修复了若干关键问题并补充了完整配置。

试试看会不会有免费的赞助哈哈哈，有没有好心人，5LDC

---

## 目录

## 整体架构

整个方案由三层组成：

- **客户端** （Cherry Studio / 任意 OpenAI 兼容工具）发出请求
- **HuggingFace Space** 运行 CLIProxyAPI，统一代理各 AI 厂商 API
- **Supabase PostgreSQL** 持久化存储 Token、配置、统计数据

> **为什么要接 Supabase？** HuggingFace Space 容器重启后数据全部清空，不挂外部数据库的话所有 API Key、用户配置都会丢失。Supabase 免费版提供 500MB PostgreSQL，完全够用。

![图 1：整体架构 — 客户端 → HuggingFace Space（CLIProxyAPI）→ 各 AI API，数据库使用 Supabase 持久化](https://res.cloudinary.com/dcxvrc0uv/image/upload/v1791820605/smartclip/smartclip/1791820604712-5qn2p6.png)

图 1：整体架构 — 客户端 → HuggingFace Space（CLIProxyAPI）→ 各 AI API，数据库使用 Supabase 持久化

---

## 前置准备

| 所需账号 | 地址 | 费用 |
| --- | --- | --- |
| HuggingFace | [https://huggingface.co](https://huggingface.co/) | 免费 |
| Supabase | [https://supabase.com](https://supabase.com/) | 免费（500MB） |
| HetrixTools | [https://hetrixtools.com](https://hetrixtools.com/) | 免费（20 个监控） |
| AI API Key | 各 AI 厂商官网 | 按量计费 |

---

## 第一步：创建 Supabase 数据库

### 1.1 注册并新建项目

登录 [supabase.com](http://supabase.com/)，点击 **New Project**，填写项目名称和数据库密码。

> **重要：密码只用字母和数字！** 不要包含 `@` `#` `!` `$` `%` 等特殊字符，否则 DSN 解析时极易出错。 推荐格式： `MyPass2026abc`

### 1.2 打开 Connect 面板

项目创建完成后，点击顶部导航栏右侧的 **Connect** 按钮：

![图 2：点击 Supabase 顶部导航栏的 Connect 按钮](https://res.cloudinary.com/dcxvrc0uv/image/upload/v1791820605/smartclip/smartclip/1791820604722-306fsg.png)

图 2：点击 Supabase 顶部导航栏的 Connect 按钮

### 1.3 复制连接字符串

在弹出面板中， **Method** 选择 **Transaction pooler**，复制连接字符串：

![图 3：将 Method 切换为 Transaction pooler，复制连接字符串。如果出错，尝试一下Session pooler](https://res.cloudinary.com/dcxvrc0uv/image/upload/v1791820605/smartclip/smartclip/1791820604727-3wtnt9.png)

图 3：将 Method 切换为 Transaction pooler，复制连接字符串。如果出错，尝试一下Session pooler

### 1.4 拼接最终 DSN

复制 Transaction pooler 连接字符串，在末尾追加 `?sslmode=require`：

```perl
postgresql://postgres.你的project-ref:你的密码@aws-0-us-west-2.pooler.supabase.com:6543/postgres?sslmode=require
```

> 把这串字符串保存好，第三步配置环境变量时会用到。

---

## 第二步：搭建 HuggingFace Space

### 2.1 新建 Docker Space

访问 [https://huggingface.co/new-space，配置如下：](https://huggingface.co/new-space%EF%BC%8C%E9%85%8D%E7%BD%AE%E5%A6%82%E4%B8%8B%EF%BC%9A)

- **Space SDK**： `Docker`
- **Visibility**： `Public`
- **Hardware**：CPU Basic（免费）
![图 4：创建自己的huggingface space](https://res.cloudinary.com/dcxvrc0uv/image/upload/v1791820606/smartclip/smartclip/1791820605472-il19nd.png)

图 4：创建自己的huggingface space

### 2.2 创建 Dockerfile

在 Space 的 **Files** 标签页中，新建 `Dockerfile`：

```bash
FROM eceasy/cli-proxy-api:latest

USER root

# 安装必要依赖
RUN apk add --no-cache bash libc6-compat gcompat

WORKDIR /app

# 复制二进制文件
RUN cp /CLIProxyAPI/CLIProxyAPI ./cli-proxy-api && chmod +x ./cli-proxy-api

# 创建必要目录
RUN mkdir -p /tmp/.cli-proxy-api /tmp/logs /tmp/pg_cache/pgstore \
    && chmod -R 777 /tmp

# 复制配置文件
COPY config.yaml /app/config.yaml

# 关键：程序启动时需要此文件，否则报错
RUN cp /app/config.yaml /app/config.example.yaml

ENV TZ=Asia/Shanghai
EXPOSE 7860

# 直接运行，不使用 Nginx
CMD ["./cli-proxy-api", "--config", "/app/config.yaml"]
```

> **不需要 nginx.conf，请勿创建！** Nginx 与程序同时监听 7860 端口会导致进程冲突退出，表现为 `Runtime error: Exit code 0`。

### 2.3 创建 config.yaml

```yaml
# 服务监听配置
host: "0.0.0.0"
port: 7860

# Supabase 连接（通过环境变量注入）
pgstore-dsn: "${PGSTORE_DSN}"

# 目录配置（使用 /tmp 避免权限问题）
auth-dir: "/tmp/.cli-proxy-api"
logging-to-file: true
logs-dir: "/tmp/logs"

# 管理后台
remote-management:
  enabled: true
  secret-key: "${MANAGEMENT_PASSWORD}"

# 性能配置
commercial-mode: true
debug: false
```

Files 目录最终只需要这两个文件：

```lua
your-space/
├── Dockerfile
└── config.yaml
```
![图5：dockerfile和配置文件进行上传](https://res.cloudinary.com/dcxvrc0uv/image/upload/v1791820606/smartclip/smartclip/1791820605502-dme47v.png)

图5：dockerfile和配置文件进行上传

---

## 第三步：配置环境变量

进入 Space → **Settings** → **Variables and secrets**，按下表配置所有变量（ `PGSTORE_DSN` 和 `MANAGEMENT_PASSWORD` 建议设为 **Secret** 加密存储，其余可设为普通 Variable）：

![图 6：在 HuggingFace Space Settings 中配置环境变量](https://res.cloudinary.com/dcxvrc0uv/image/upload/v1791820606/smartclip/smartclip/1791820605533-vh95pm.jpg)

图 6：在 HuggingFace Space Settings 中配置环境变量

| 变量名 | 占位值 / 示例 | 说明 |
| --- | --- | --- |
| `PGSTORE_DSN` | `postgresql://postgres.xxx:密码@aws-0-us-west-2.pooler.supabase.com:6543/postgres?sslmode=require` | PostgreSQL 连接字符串，包含用户名、密码、地址、端口和数据库名 |
| `MANAGEMENT_PASSWORD` | `（你的强密码）` | 管理后台登录密码，用于访问 `/management.html` |
| `MANAGEMENT_STATIC_PATH` | `/tmp` | 管理后台静态资源的存储 / 读取路径 |
| `PGSTORE_LOCAL_PATH` | `/tmp/pg_cache` | PostgreSQL 相关本地缓存或存储路径 |
| `TZ` | `Asia/Shanghai` | 系统时区设定（中国标准时间） |
| `PORT` | `7860` | 服务运行的端口号 |

保存后 Space 会自动重建，等待约 1-2 分钟。查看 **Logs** 标签，看到以下内容说明启动成功：

```yaml
API server started successfully on: 0.0.0.0:7860
server clients and configuration updated: 0 clients (...)
```

> `0 clients` 是正常的，说明数据库连接成功，只是还没添加 API Key。

---

## 第四步：通过网页后台导入 API Key

CLIProxyAPI 内置网页管理后台，无需安装任何客户端，直接浏览器访问：

```bash
https://[your_username]-[your_spacename].hf.space/management.html
```
![图7：登录你的CLIProxyAPI客户端](https://res.cloudinary.com/dcxvrc0uv/image/upload/v1791820607/smartclip/smartclip/1791820606188-x05jf9.png)

图7：登录你的CLIProxyAPI客户端

### 4.1 导入认证文件（AI 原始 API Key）

点击左侧 **认证文件** → 右上角 **上传文件**：

![图 8：管理后台「认证文件」页面 — 点击右上角「上传文件」导入 API Key](https://res.cloudinary.com/dcxvrc0uv/image/upload/v1791820607/smartclip/smartclip/1791820606276-mpqbx3.png)

图 8：管理后台「认证文件」页面 — 点击右上角「上传文件」导入 API Key

认证文件为 JSON 格式，以 Claude 为例：

```json
{
  "type": "claude",
  "keys": [
    "sk-ant-api03-xxxxxxxxxxxxxxxxxxxxxxxx"
  ]
}
```

### 4.2 配置访问密钥（分发给用户用的 Key）

点击 **配置面板** → **API 密钥列表** → **添加 API 密钥**：

![图 9：配置面板 — 添加你自己分发给用户使用的 API 密钥](https://res.cloudinary.com/dcxvrc0uv/image/upload/v1791820607/smartclip/smartclip/1791820606501-4jab5c.png)

图 9：配置面板 — 添加你自己分发给用户使用的 API 密钥

> 注意区分两种 Key：
> 
> - **认证文件里的 Key**：Claude / Gemini 等厂商的原始 API Key（不对外暴露）
> - **API 密钥列表里的 Key**：你自定义的访问密钥，分发给用户使用

---

## 第五步：Cherry Studio 验证

打开 Cherry Studio，进入 **设置 → 模型服务 → 添加自定义服务商**：

| 配置项 | 填写内容 |
| --- | --- |
| 服务商名称 | Oh My API（自定义） |
| 接口类型 | OpenAI Compatible |
| **API 地址** | `https://[your_username]-[your_spacename].hf.space` |
| **API 密钥** | 第四步在配置面板设置的自定义密钥 |

![图 10：Cherry Studio验证](https://res.cloudinary.com/dcxvrc0uv/image/upload/v1791820607/smartclip/smartclip/1791820606822-sde018.png)

图 10：Cherry Studio验证

点击 **获取模型列表** 或手动填写模型名（如 `gpt-5.3-codex`），发送测试消息，收到正常回复即部署成功

---

## 第六步：HetrixTools 保活

### 为什么需要保活？

HuggingFace 免费 Space **长时间无请求会进入休眠**，下次访问需要 30-60 秒冷启动。通过定期 ping 可以保持活跃，同时监控服务可用性。

### 使用 HetrixTools（推荐）

HetrixTools 免费版支持 20 个监控节点，最短 1 分钟检查间隔。

![图 11：HetrixTools 定期 ping HuggingFace Space，防止休眠并监控可用性](https://res.cloudinary.com/dcxvrc0uv/image/upload/v1791820608/smartclip/smartclip/1791820607013-gnz65j.jpg)

图 11：HetrixTools 定期 ping HuggingFace Space，防止休眠并监控可用性

**配置步骤：**

1. 注册 [https://hetrixtools.com](https://hetrixtools.com/)，免费计划无需信用卡
2. 进入 **Uptime Monitors** → 点击 **Add Monitor**
3. 类型选择 **Website**，HTTP Method 选择 **GET**
4. URL 填入： `https://[your_username]-[your_spacename].hf.space`
5. 检查间隔设为 **1 分钟**
6. 配置邮件告警（服务宕机时自动通知）
7. 保存即可

### 其他免费保活方案

| 工具 | 免费额度 | 最短间隔 | 特点 |
| --- | --- | --- | --- |
| **HetrixTools** | 20 个监控 | 1 分钟 | 有告警，推荐 |
| UptimeRobot | 50 个监控 | 5 分钟 | 使用最广泛 |
| Freshping | 50 个监控 | 1 分钟 | 界面友好 |
| [cron-job.org](http://cron-job.org/) | 无限制 | 1 分钟 | 纯定时任务，无告警 |

---

## 踩坑合集

### 坑一：network is unreachable（IPv6 连不上）

```markdown
[2600:1f13:838:...]:5432: connect: network is unreachable
```

**原因：** HuggingFace 不支持 IPv6，直连 `db.xxx.supabase.co` 返回 IPv6 地址。

**解决：** 改用 Pooler 域名 `aws-0-us-west-2.pooler.supabase.com`，自动走 IPv4。

---

### 坑二：config.example.yaml: no such file or directory

```lua
open /app/config.example.yaml: no such file or directory
```

**原因：** 程序启动时需要此文件作为配置模板，但镜像中没有。

**解决：** 在 Dockerfile 中添加：

```bash
RUN cp /app/config.yaml /app/config.example.yaml
```

---

### 坑三：Runtime error: Exit code 0

**原因：** Nginx 和程序同时监听 7860 端口互相冲突，进程正常退出（Exit code 0 不是崩溃，是主动退出）。

**解决：** 删除 `nginx.conf`， `CMD` 中去掉 nginx 启动命令。

---

### 坑四：prepared statement already exists（SQLSTATE 42P05）

```sql
ERROR: prepared statement "stmtcache_xxx" already exists (SQLSTATE 42P05)
```

**原因：** 早期配置中 DSN 末尾缺少 `sslmode=require` 等参数，或连接字符串格式有误。

**解决：** 确保 DSN 末尾追加 `?sslmode=require`，使用 Transaction pooler（端口 6543）正常可用。

---

### 坑五：Tenant or user not found（SQLSTATE XX000）

```sql
FATAL: Tenant or user not found (SQLSTATE XX000)
```

**原因：** 数据库密码含特殊字符，DSN 解析错误。

**解决：** 在 Supabase 重置密码，改为纯字母 + 数字组合。

---

### 坑六：碰到 space 的状态一直 building or starting

![image](https://res.cloudinary.com/dcxvrc0uv/image/upload/v1791820608/smartclip/smartclip/1791820607239-45wj7e.jpg)

**原因：** 可以直接访问看看，显示 bug，不用管！！！！！！！

**解决：** 只要 log 日志中出现了 API server started successfully on: 0.0.0.0:7860 时候，直接访问就行，访问链接为 {username}-{huggingface_space_name}.hf.space/management.html#

---

## 总结

| 步骤 | 关键点 |
| --- | --- |
| Supabase 连接 | Transaction Pooler（端口 6543），末尾加 `?sslmode=require` |
| 数据库密码 | 纯字母 + 数字，禁止特殊字符 |
| Dockerfile | 不用 Nginx，补充 `config.example.yaml` |
| 端口 | 统一用 7860 |
| 管理后台 | 直接访问 `/management.html` |
| 保活 | HetrixTools 1 分钟 ping 一次 |

最终效果： **完全免费**，HuggingFace 提供运行环境，Supabase 持久化存储，数据不会因 Space 重启丢失。

---

> 参考原帖： [https://linux.do/t/topic/1675528](https://linux.do/t/topic/1675528)
> 
> 如有问题欢迎评论区留言

[infinite](https://linux.do/u/dai_steven) 不二之选

感谢佬友总结，喂饭

[九天爱工作](https://linux.do/u/lishan1314258)

虽然我没有用这个，但是这种热心肠的佬还是值得尊敬

[NIMOBU](https://linux.do/u/nimobu) 不二之选

感谢大佬