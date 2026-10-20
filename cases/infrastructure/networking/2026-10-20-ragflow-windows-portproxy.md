---
id: ENG-20261020-001
date: 2026-10-20
updated: 2026-10-20
status: workaround_verified
domain: infrastructure
category: networking
systems:
  - RAGFlow v1.0.0-rc2
  - Docker Desktop
  - Windows IP Helper (iphlpsvc)
  - Nginx
tags:
  - windows
  - docker
  - portproxy
  - loopback
  - nginx
  - connection-reset
  - bad-gateway
---

# RAGFlow：Windows 80 端口连接重置，实际被 portproxy 规则干扰

## 问题定义

- **工程问题类型**：基础设施 → 网络 → 主机监听地址、端口映射与端口代理重叠。
- **受影响软件**：自托管 RAGFlow `v1.0.0-rc2`，使用 Docker Desktop 和内置 Nginx。
- **现象**：最初访问 `http://localhost/api/v1/system/healthz` 返回 HTTP 502；后续访问 `http://127.0.0.1/api/v1/system/healthz` 出现 `curl: (56) Recv failure: Connection was reset`。
- **期望**：经主机 80 端口进入容器 Nginx，再转发到 RAGFlow Go API（9380），返回健康检查结果。
- **结果状态**：已找到并验证绕过方式；**没有删除规则或完成原始 127.0.0.1:80 入口的根因修复**。

## 运行环境与基础状态

本次 Docker 容器：
- RAGFlow 容器：`docker-ragflow-cpu-1`，镜像 `infiniflow/ragflow:v1.0.0-rc2`，主机发布了 80、443、9380、9381 等端口。
- MySQL、Elasticsearch、Kvrocks、MinIO、NATS 依赖容器均报告为 healthy。
- RAGFlow 启动日志显示 `Database connected and migrated successfully`，版本检查通过，并有 `Server starting on port: 9380`。启动日志中的部分 warn 不能直接当作 502 原因。
- 此前误用的 `docker logs --tail 50 ragflow-cpu` 查找的是不存在的容器名，正确容器名来自 `docker ps -a`。

## 关键证据

### A. 绕过 Nginx 访问容器 Go API：成功

```cmd
curl.exe --noproxy "*" -i http://127.0.0.1:9380/api/v1/system/healthz
```

结果：HTTP 200，包含 `"status":"ok"`、`"db":"ok"`、`"doc_engine":"ok"`、`"storage":"ok"`、`"message_queue":"ok"` 等字段。

### B. 容器内部通过 Nginx 访问 API：成功

```cmd
docker exec docker-ragflow-cpu-1 curl --noproxy "*" -i --max-time 10 http://127.0.0.1/api/v1/system/healthz
```

结果：`HTTP/1.1 200 OK`，`Server: nginx/1.31.3`，`X-Api-Source: go`。说明容器内部 Nginx 到 API 这段链路实测可用。

### C. Windows 上的 80 端口有两类监听者

通过：

```cmd
netstat -ano -p TCP | findstr LISTENING | findstr ":80 "
```

当时观察到：
- `0.0.0.0:80`：`com.docker.backend.exe` 监听。
- `127.0.0.1:80`：`svchost.exe`，服务为 `iphlpsvc`（Windows IP Helper）。

PID 只是当次会话进程编号，重启后可能变化。

### D. Windows 有指向 8088 的端口代理规则

```cmd
netsh interface portproxy show all
```

返回：

```text
127.0.0.1  443  ->  127.0.0.1  8088
127.0.0.1   80  ->  127.0.0.1  8088
```

该规则的目标 `127.0.0.1:8088` 是否有服务、由什么程序创建，在本次实验中**没有验证**。443 端口虽也存在规则，但**没有单独做 HTTPS 请求测试**。

### E. 改用另一回环地址，主机 80 端口恢复正常响应

```cmd
curl.exe --noproxy "*" -i --max-time 10 http://127.0.0.2/api/v1/system/healthz
```

结果：HTTP 200，`Server: nginx/1.31.3`，`X-Api-Source: go`，所有健康检查项返回 `ok`。之后使用浏览器访问 `http://127.0.0.2`，成功进入 RAGFlow 登录/注册页面。

## 排查过程及解释

| 测试 | 结果 | 有助于区分什么 |
| --- | --- | --- |
| Docker 容器状态和依赖 | 主容器 Up、依赖 healthy | 没有出现主容器不存在或依赖全面离线 |
| API 9380 直接健康检查 | 200 | Go API 当前可正常响应 |
| Nginx 进程与 `nginx -t` | 进程运行、配置语法通过 | 排除 Nginx 不存在/明显语法错误；**不能证明转发正确** |
| 容器内 80 → 9380 | 200 | 容器内部 Nginx 代理链路可用 |
| Windows `127.0.0.1:80` | 连接重置 | 主机回环入口存在问题 |
| 主机 `netstat` 与 `portproxy` | 回环 80 由 iphlpsvc 监听，代理到 8088 | 锁定主机回环端口代理是关键候选 |
| Windows `127.0.0.2:80` | 200 | 绕过 127.0.0.1 专用代理规则后可通过 Docker 入口 |

## 原因与证据边界

- **已证实**：Go API 与容器内部 Nginx → Go API 健康检查均成功；Windows `127.0.0.1:80` 异常，而 `127.0.0.2:80` 正常；Windows 存在监听 `127.0.0.1:80` 的 `portproxy` 规则。
- **高可信推断**：`127.0.0.1:80 -> 127.0.0.1:8088` 的端口代理规则是该入口异常的关键因素，连接没有按预期进入 Docker 发布的 80 端口。
- **仍未知**：8088 目标实际运行状态；最初 502 的准确响应来源；其他网络软件是否参与；为何当时创建这些 portproxy 规则。
- **不能下的结论**：RAGFlow 本体有缺陷；Nginx 配置错误；RAGFlow 后端数据库迁移失败；原始 127.0.0.1 入口已永久修复。

## 处理与验证

**实际采用的无破坏性绕过办法：** 在这台 Windows 主机上改为访问 `http://127.0.0.2`。健康检查 HTTP 200，Web 登录页面也能打开。

未执行：关闭 `iphlpsvc`、删除 `portproxy`、修改防火墙、重新安装 RAGFlow、清理容器或数据卷。

如果未来需要恢复 `127.0.0.1:80`，应先调查是谁在使用 8088 和谁创建了端口代理，评估对其他程序的影响；有明确需要和回滚方式后再修改。

## 可迁移经验

1. **按链路分层验证**：服务进程 → API 直连 → 容器内反向代理 → Docker 主机端口 → 本机回环地址，避免把每一层的失败都归咎于应用。
2. `docker ps` 的 Up、依赖 healthy 与 `nginx -t` 成功都不是端到端健康检查；直接 HTTP 200 才能证明某条路径正常。
3. 当 Docker `0.0.0.0:80` 监听与 `iphlpsvc` 的 `127.0.0.1:80` 同时存在时，及时检查 `netsh interface portproxy show all`。
4. 用改变一个变量的方法（`127.0.0.1` → `127.0.0.2`）检验端口绑定和路由假设，但**绕过成功不等于定位全部原因**。
5. 读日志按时间顺序看成功与异常；不能仅看到 `warn` 就确定它引发 502。

## 参考

- RAGFlow v1.0.0-rc2 Nginx 配置：<https://github.com/infiniflow/ragflow/blob/v1.0.0-rc2/docker/nginx/ragflow.conf>
- RAGFlow v1.0.0-rc2 Docker 启动脚本：<https://github.com/infiniflow/ragflow/blob/v1.0.0-rc2/docker/entrypoint.sh>
- RAGFlow v1.0.0-rc2 README：<https://github.com/infiniflow/ragflow/blob/v1.0.0-rc2/README.md>

> 信息来自 2026-10-20 本地命令实际回显及相应版本的官方配置，属于排障案例而非通用保证；公开仓库中仅保留必要的脱敏信息。
