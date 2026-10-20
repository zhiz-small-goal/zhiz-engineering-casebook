# cases：工程问题分类与索引

`cases/` 里的文件**按工程问题的主要发生层级**归档，不是按发现它时打开的软件简单堆放。每个案例在 YAML 元数据中还注明受影响的产品、症状、版本和标签。

## 分类

| 分类路径 | 归档依据 | 例子 |
| --- | --- | --- |
| `infrastructure/networking/` | IP、DNS、代理、端口、TLS、网络转发 | Windows portproxy 与 Docker 端口入口重叠 |
| `infrastructure/containers/` | 容器生命周期、编排、镜像、卷、运行权限 | Compose 服务未启动 |
| `infrastructure/services/` | 基础依赖服务运行状态 | 数据库、缓存、消息队列异常 |
| `development/tooling/` | 开发工具、编辑器、调试工具、编译和构建 | VS Code 插件触发不必要的工具链配置 |
| `development/dependencies/` | 语言包、依赖冲突、版本兼容 | SDK 与运行时版本不兼容 |
| `applications/` | 应用逻辑、API、应用内部集成 | API 请求/业务逻辑异常 |
| `data/` | 数据发现、采集、转换、存储、评测 | 解析器丢失证据、源数据结构变化 |
| `ai/` | 模型、检索、Agent、推理和服务调用 | 模型工具调用行为异常 |

**分类不是必须预先创建的空目录。** 随着实际案例出现再增加层级；若某分类只有一个实例，先保持简单。

## 怎么决定把案例放在哪里？

优先选择**实际发生问题或最值得复用排障方法的层级**。例如今天的 RAGFlow 可通过 9380 访问、容器内 Nginx 也正常，而 Windows 回环地址 80 端口出现端口代理规则，因此放在 `infrastructure/networking/`；RAGFlow、Docker、Windows 在文件元数据里交叉标记。

如果归因还没确认，目录是当前暂定分类，并在正文说明可信度；新增证据后可以移动文件。**不要因为某工具报错，就断言它本身有缺陷。**

## 索引

| ID | 类型 | 案例 | 状态 |
| --- | --- | --- | --- |
| ENG-20261020-001 | 基础设施 / 网络与端口 | [RAGFlow 本机 80 端口连接重置与 portproxy](infrastructure/networking/2026-10-20-ragflow-windows-portproxy.md) | `workaround_verified` |

## 文件和字段

- 路径：`cases/{domain}/{category}/{YYYY-MM-DD}-{system}-{short-slug}.md`
- 必填：`id`、`date`、`status`、`domain`、`category`、`systems`、`tags`。
- 正文建议依次记录“现象 → 环境 → 证据 → 验证过程 → 原因边界 → 处理结果 → 可迁移经验”。
- 案例模板：[templates/case.md](../templates/case.md)。
- 新增案例时同步在本索引增加一行，避免靠目录猜测案例内容。

## 状态语义

- `investigating`：仍在定位。
- `workaround_verified`：临时或替代路径有效，但未确认原始缺陷被修复。
- `resolved_verified`：修改了根本问题，并完成修复后验证。
- `closed_unconfirmed`：停止排查，但根因或效果未证实。

**证据等级与状态分开**：即使找到可用绕过方法，也可能只对根因有高可信度推断而没有直接证据。
