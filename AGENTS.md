# ShadowsocksR 项目协作说明

本文件适用于整个仓库。默认使用中文沟通；代码命名、注释和格式沿用所在文件的习惯。

## 开始工作

- 先检查 `git status --short`、`git diff` 和 `git diff --cached`，区分用户已暂存的改动与本次任务。
- 保留用户已有改动和文件删除，不擅自恢复文件、重置暂存区、提交或推送。
- 优先完成当前任务所需的最小修改，避免顺带格式化历史代码或升级运行时、密码库及协议。
- `README.md`、`GEMINI.md` 和旧脚本包含历史用法，执行前核对文件是否存在及实际行为。

## 项目结构与入口

| 文件或目录 | 用途 |
| --- | --- |
| `shadowsocks/server.py` | 代理服务端入口；当前生产服务使用此入口 |
| `shadowsocks/local.py` | 本地 SOCKS 客户端入口 |
| 根目录 `server.py` | 数据库或 JSON 驱动的多用户管理入口，与上述入口不同 |
| `server_pool.py`、`db_transfer.py` | 多用户服务池、配置同步与流量统计 |
| `shadowsocks/tcprelay.py`、`udprelay.py` | TCP/UDP 转发与连接生命周期 |
| `shadowsocks/asyncdns.py`、`eventloop.py` | 异步 DNS 与事件循环 |
| `shadowsocks/lru_cache.py` | 多处共用的有过期时间的缓存 |
| `shadowsocks/obfsplugin/` | 认证协议与混淆插件 |
| `shadowsocks/crypto/`、`encrypt.py` | 加密实现与底层库适配 |
| `tests/` | 历史集成测试及配置样例 |

## 兼容性与实现约束

- 生产环境最近核实使用 **Python 2.7.5**，修改必须保留 Python 2.7 与 Python 3 兼容性；仅在现代 Python 3 下通过不代表可部署。
- 不引入 f-string、类型注解、`async/await`、`dataclasses` 或仅新版本标准库支持的接口。保留现有 `__future__` 导入和兼容分支。
- 网络报文按字节处理，明确区分 `bytes`、文本和编码；注意 Python 2/3 字节索引与 `ord`/`chr` 的差异。
- 沿用文件现有缩进，避免全文件重新排版。新依赖必须说明必要性及生产运行时兼容性。
- 事件循环内避免阻塞操作和无上限的大批量回收；共享缓存不是线程安全容器。

## DNS、认证和内存管理

以下约束用于防止已定位的问题再次出现。变更这些行为时要给出依据和针对性验证。

- **DNS 登记与清理：**先登记请求、回调反向索引和时间戳，再发送报文。成功、超时、取消及发送失败均应释放对应状态与回调引用，待处理计数保持一致。
- **DNS 回调：**通知调用方前摘除旧请求状态；隔离单个回调异常。回调重入产生的新请求不能被旧请求的完成或超时清理误删。
- **DNS 合并与容量：**同域名等待者共享在途查询及首次请求的截止时间。当前超时为 15 秒，每域名最多 256 个待处理回调，全局最多 4,096 个；计数上限不等于内存字节上限。保留 hosts 覆盖，`.local` 快速失败应给出明确错误。
- **认证状态：**不能用通用 LRU 默认 60 秒超时清理防重放记录。`auth.py` 按 180 秒活动期限处理；`auth_chain.py` 保留 `ref > 0` 的队列，仅在没有连接引用且距最后活动至少 600 秒时按过期规则回收。容量淘汰也必须保护活跃队列。
- **认证引用：**`auth_chain` 连接更新和关闭时操作其认证成功时所属的队列实例，避免修改后来替换的队列。未认证连接关闭不能扣减其他队列，重复 `dispose()` 必须安全。
- **LRU 读取：**`cache[key]` 会更新时间及次序，检查过期或淘汰候选时使用 `peek()`。先保存候选 key，再检查并删除同一项，不能重复调用会改变结果的 `first()` 组合。
- **LRU 清理预算：**`sweep()` 默认每轮最多清理 1,024 项；返回 `False` 表示预算用尽，可能需要后续清理。验证有限积压能逐轮排空，不以无限循环代替调度。
- **泄漏归因：**先检查仍存活容器、绑定方法、回调列表及 payload 的强引用链。仅增加 `gc.collect()` 或将几个属性置空不能证明已修复泄漏。

## 验证

文档修改只需核对内容、路径和差异；行为修改运行与改动相关的检查。交付时列出实际执行的命令、结果与未覆盖部分。

现有 LRU 模块包含可独立运行的基本检查，在仓库根目录执行：

```bash
PYTHONDONTWRITEBYTECODE=1 python3 -m shadowsocks.lru_cache
git diff --check
git diff --cached --check
```

- 当前仓库没有 `tests/test_*_lifecycle.py`。需要生命周期回归时新增或使用明确指定的测试，先确认文件与实际测试数量；不能把发现 0 项测试当作通过。
- DNS/内存问题优先用假时钟、模拟 socket、弱引用及独立进程复现，覆盖失败、取消、超时、重入、容量回收和对象释放。认证修改应覆盖重复连接编号、空闲连接恢复及队列替换。
- **不要直接对整个 `tests/` 做宽泛测试发现或导入 `tests/test.py`。**该文件在模块顶层解析参数并启动服务端、客户端进程。
- `tests/test_command.sh`、`test_daemon.sh` 和 `tests/jenkins.sh` 会启停进程；集成测试还可能依赖端口、外网 DNS、HTTP 服务、OpenSSL/libsodium 及额外工具。先读脚本，在隔离环境中运行。
- 根目录及 `shadowsocks/` 下的 `run.sh`、`stop.sh` 等旧脚本可能按进程名匹配并终止进程，不作为生产环境的默认测试工具。
- 涉及运行时代码时尽量用 Python 2.7 和可用的 Python 3 分别验证；若缺少某个解释器，明确说明。使用服务器验证时放入独立临时目录，避免测试导入生产目录或占用生产端口。

## 当前生产部署约定

以下信息于 **2026-10-29** 核实，执行运维前重新检查实际状态。连接地址和密钥从用户提供的信息或本地 SSH 配置获取，不把密钥内容、业务密码或完整敏感配置写入仓库、日志及回复。

| 项目 | 约定 |
| --- | --- |
| 源码目录 | `/usr/local/shadowsocksr/shadowsocks` |
| 生产配置 | `/etc/shadowsocksr/user-config.json` |
| 工作目录 | `/usr/local/shadowsocksr/shadowsocks` |
| 服务文件 | `/etc/systemd/system/ssr.service` |
| 运行方式 | 原生 systemd，`Type=simple`，跟踪实际 Python 主进程 |
| 自动恢复 | `Restart=on-failure`，`RestartSec=5s`，已启用开机自启 |
| 应用日志 | `/usr/local/shadowsocksr/shadowsocks/ssserver.log` |

- 部署、重启及服务配置修改按用户当前已授权的范围执行，不重复请求已有授权。代码修改与已部署状态应分别报告。
- 部署前完成本地验证并备份被替换的文件；保持目录结构，例如 `auth.py` 和 `auth_chain.py` 位于 `obfsplugin/`，不是源码根目录。
- 使用 `systemctl` 管理生产服务，不并行使用旧 SysV 脚本或另起一个不同名称的 SSR 服务。不要编辑会被重新生成的 `/run/systemd/generator.late/ssr.service`。
- 上传 `.py` 不会更新运行中的进程。Python 2.7 导入时会根据源码修改时间判断 `.pyc` 是否过期，通常无需手动删除；若上传保留了旧时间戳，需要核对缓存有效性。
- 验证部署时检查源码散列、服务状态、实际 Python 进程、监听状态和日志。不要只看 `active`；应确认主 PID 与目标进程一致，且没有重复实例。未经任务授权，不通过杀死生产进程测试自动恢复。

常用只读检查：

```bash
sudo systemctl show ssr -p MainPID -p Type -p Restart -p RestartUSec -p FragmentPath
systemctl is-enabled ssr
sudo systemctl status ssr --no-pager
sudo journalctl -u ssr -n 30 --no-pager
```

内存排查同时记录采样时间、进程运行时长、RSS、文件描述符/连接数、DNS 待处理量及 OOM 记录。重启后的低内存或短时间稳定，不能单独证明长期泄漏已经消失。
