# Codex 源码拆解 27：linux-sandbox 与 process-hardening 实现层

## 这篇看什么

回答：**Linux 下的 sandbox 真正怎么落地？bwrap、seccomp、Landlock、proxy routing、process hardening 各自站在哪一层？**

## 先给结论

现在可以把 Linux sandbox 这块拆得更清楚：

1. 高层 `sandboxing` 负责**选择策略与生成 helper argv**
2. `linux-sandbox` 负责**真正落地执行**
3. 当前主路径已经是：
   - bwrap namespace
   - 然后 seccomp / no_new_privs
4. Landlock 现在更像：
   - legacy fallback
   - 或实现参考
5. `process-hardening` 是另一条线：
   - 进程 pre-main 自身硬化
   - 不是 sandbox policy 选择器

一句话：

> Linux sandbox 当前主实现是 bwrap-first；process-hardening 只是更外层的自我加固，不是同一个系统层级。

## 关键代码路径

- `codex-rs/sandboxing/src/manager.rs`
- `codex-rs/sandboxing/src/policy_transforms.rs`
- `codex-rs/sandboxing/src/landlock.rs`
- `codex-rs/linux-sandbox/src/linux_run_main.rs`
- `codex-rs/linux-sandbox/src/bwrap.rs`
- `codex-rs/linux-sandbox/src/launcher.rs`
- `codex-rs/linux-sandbox/src/landlock.rs`
- `codex-rs/linux-sandbox/src/proxy_routing.rs`
- `codex-rs/process-hardening/src/lib.rs`

## 高层怎么决定要不要上 Linux sandbox
真正的选择逻辑在 shared `sandboxing` 层：

- `SandboxManager::select_initial()`
- `should_require_platform_sandbox()`

关键判断包括：
- managed network → 强制平台 sandbox
- restricted filesystem / restricted network → 通常也要求平台 sandbox

所以 Linux helper 不是自己决定“我要不要上”，而是上层已经拍板好了。

## helper argv 生成也在高层
`sandboxing/src/landlock.rs` 虽然名字还叫 landlock，但其实承担的是：

- 生成 linux sandbox command args
- 加 `--allow-network-for-proxy` 之类 flag
- 处理 arg0 override

这也是一个容易踩坑的地方：

> 名字里还留着 Landlock，不代表当前主实现仍是 Landlock-first。

## `linux-sandbox` 的主路径：bwrap first
`linux_run_main.rs::run_main()` 体现得很清楚：

### 默认路径
1. outer stage 先起 bwrap namespace
2. 再 re-exec helper
3. inner stage 再 apply seccomp + `PR_SET_NO_NEW_PRIVS`
4. 最后 `execvp` 用户命令

这其实是一个两阶段 pipeline：

> namespace 先落地，再在更内层收紧 syscall/process 能力。

## Landlock 现在的真实位置
`linux-sandbox/src/landlock.rs` 里已经明确写了：

- filesystem restrictions 现在由 bubblewrap 负责
- Landlock filesystem rule 安装逻辑更多是 fallback/reference

所以当前更准确的理解是：

- bwrap：文件系统/namespace 主隔离器
- seccomp/no_new_privs：进程能力约束
- legacy Landlock：回退路径

## managed proxy routing 是 helper 自己做的
这块非常值。

当上层要求 `allow-network-for-proxy` 时：
- helper host stage 会读 loopback proxy env
- 生成 route spec
- 起 host UDS bridge
- 内层 netns 再起本地 TCP bridge
- 把 proxy env 改写成 `127.0.0.1:<port>`

而且是 fail-closed：
- proxy env 不对，就不放行

所以 managed proxy 不只是“上层加个 env”，而是 Linux helper 内部有完整路由桥接逻辑。

## bwrap launcher 的策略
launcher 会：
- 优先探测系统 `bwrap`
- 看它是否支持 `--argv0`
- 不行就做 argv/path 兼容修补
- 再不行才退到 vendored bwrap

这说明 OpenAI 在这里追求的是：
- 优先复用系统能力
- 但不能依赖系统环境完美

这很符合工程实践。

## `process-hardening` 是另一层东西
这个 crate 做的主要是：
- `PR_SET_DUMPABLE = 0`
- `RLIMIT_CORE = 0`
- 清一些危险 `LD_*` env

也就是说它更像：

> 进程启动前的自我硬化

而不是：
- 文件系统访问控制
- 网络访问控制
- namespace 隔离

所以它和 Linux sandbox 不是平替关系，而是更外层的一道轻量防线。

## 一个很关键的负判断
当前没有看到 `process-hardening` 真正挂进 `codex-linux-sandbox` 主链；
至少这次看到的明确接入点更像在别的二进制里（例如 responses-api-proxy）。

所以不能把它说成 Linux sandbox 主链的一部分。

## 当前边界判断

- `sandboxing`：策略选择与 helper argv 构建
- `linux-sandbox`：Linux 平台具体执行器
- `bwrap`：当前主隔离路径
- seccomp / `no_new_privs`：进程能力收紧
- legacy Landlock：回退/参考实现
- `proxy_routing`：managed proxy 的 helper 内部桥接层
- `process-hardening`：pre-main 自我硬化层

## 设计判断

### 1. Linux sandbox 现在是明显的 bwrap-first 路线
这个判断已经足够稳。

### 2. 名字里残留 Landlock 容易误导
写 guide 时必须专门解释，不然会读歪。

### 3. proxy routing 做在 helper 里是对的
否则上层会背很多 Linux netns 细节，边界会非常脏。

### 4. process-hardening 应该被看成独立层
别把它和 sandbox policy 执行混成一件事。

## 还值得继续观察的点

1. `process-hardening` 是否后续会更系统接入更多二进制
2. bwrap/argv0 兼容逻辑还有没有更多历史包袱
3. managed proxy 未来是否会扩到非-loopback 场景
