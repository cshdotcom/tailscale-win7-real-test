# Windows 7 真机实测报告: cshdotcom/tailscale-win7-builds

> 测试日期: 2026-10-26 · 测试方式: QEMU 全程无人值守自动化实测
> 环境: Windows 7 Enterprise SP1 x64 (6.1.7601) @ QEMU 10.0.13 TCG 软件模拟, 2 vCPU / 2.5GB RAM
> 证明材料: 本仓库 release 附件 (测试日志 / 修复补丁 / 修复版二进制)

## 一、总结论 (TL;DR)

| 测试项 | 官方仓库 v1.0.0 | 本仓库修复版 v1.0.1 |
|---|---|---|
| PE 头声明 Win7 (6.1) | ✅ OSVersion=6.1 SubsystemVersion=6.1 (go-legacy-win7 工具链有效) | ✅ 同左 |
| `tailscale.exe version` 在 Win7 运行 | ✅ **成功** (1.105.0-dev20261025, go1.27.2) | ✅ 成功 (go1.27.3) |
| `tailscaled` 守护进程在 Win7 存活 | ❌ **启动即 panic** (见下) | ✅ **运行正常** (SYSTEM 服务态常驻) |
| CLI ↔ 守护进程通信 (命名管道) | ❌ (daemon 已死) | ✅ `status` → "Logged out." |
| `tailscale up` 登录流程 | ❌ 无法测试 | ✅ **签发真实登录 URL** |
| 控制面连通 (login.tailscale.com) | — | ✅ HTTP 200, 真实认证页 |
| 完成账号授权入网 | — | ⏸ 需真实账号 (登录 URL 已验证可达, 见 §4) |

**一句话结论: `tailscale-win7-builds` v1.0.0 的二进制能被 Win7 加载、CLI 可用, 但 `tailscaled` 因第三方依赖 wingoes 的版本检测 panic 而无法运行 —— 打上本仓库的一行补丁后, tailscaled 在 Win7 上正常运行并完成登录流程。**

## 二、官方 v1.0.0 实测详情

### 2.1 通过的部分
- 二进制 x64, PE OSVersion/SubsystemVersion 均为 6.1 → Win7 加载器接受, `tailscale.exe version` 输出正常:
  ```
  1.105.0-dev20261025
  tailscale commit: 566f89e0fb2aa4b2f1ebf98761e45e62aa5ef225-dirty
  go version: go1.27.2
  ```

### 2.2 失败的部分: tailscaled 启动即 panic
在真实 Win7 SP1 x64 上 (管理员权限, userspace-networking 模式), `tailscaled-win7-legacy.exe` 立即崩溃:
```
panic: change in versioning scheme -- package must be updated to interpret
github.com/dblohm7/wingoes.(*osVersionInfo).init(...)
    osversion.go:127
github.com/dblohm7/wingoes.IsWin10BuildOrGreater(0x3ad7)
    tailscale.com/net/tshttpproxy.init.func1()
    tshttpproxy_windows.go:271
```

### 2.3 根因 (源码级)
wingoes 库的 `osVersionInfo.init()` **硬性只支持 Windows 10.0**:
```go
// We only support 10.0.x.x
if osvx.MajorVersion != 10 || osvx.MinorVersion != 0 {
    if verCheckManifest(osvx) {
        panic("change in versioning scheme -- package must be updated to interpret")
    } else {
        panic("incoherent Windows version -- missing/outdated manifest?")
    }
}
```
Win7 的 6.1.7601 命中第一个 panic 分支。调用链: `tailscale.com/net/tshttpproxy` 包初始化 → `IsWin10BuildOrGreater` → panic → 整个 tailscaled 进程死亡。

> 注: 被测仓库自带的 CI smoke test 跑在 `windows-latest` (Win11 runner) 上, 因此该 panic 从未被仓库作者的流水线发现 —— "编译通过" 与 "Win7 能跑" 之间存在真实鸿沟, 这正是本次真机测试的价值。

### 2.4 附加发现
- 官方 Go 1.27.3 交叉编译 + 仅改 PE 头为 6.1 → **不够**: 二进制在 Win7 上 `0xC0000005` 立即崩溃 (runtime 本身依赖 Win10 API)。go-legacy-win7 工具链对 runtime 的适配是必需的, 仓库 README 的说法得到验证。
- `--socket=<文件路径>` 在 Win7 不可用 (AF_UNIX 为 Win10 1803+ 特性), 需用默认命名管道。

## 三、修复版 (v1.0.1) 构建

1. `go-legacy-win7` 工具链 (main 分支, 自称 go1.27.3) 从源码构建 (bootstrap: 官方 go1.27.3)
2. tailscale 源码 + 对 wingoes 打一行补丁 (见 release 附件 `wingoes-osversion.patch`):
   ```go
   if osvx.MajorVersion != 10 || osvx.MinorVersion != 0 {
       // [TS-WIN7-PATCH] legacy Windows (e.g. 6.1) uses fallback instead of panic
       osv.initFallback()
       return
   }
   ```
   走 wingoes 自带的 fallback 路径 (`VerifyVersionInfo` 判定), `IsWin10BuildOrGreater` 在 Win7 上正确返回 false。
3. `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 GOTOOLCHAIN=local go build` → 产物 PE 头原生 6.1/6.1, 无需后处理。

## 四、修复版实测 (Win7 SP1 x64, QEMU)

```
> tasklist
tailscaled.exe   2440   Services   26,776 K        ← 常驻运行

> tailscale status
Logged out.                                     ← CLI↔daemon 命名管道通信正常

> tailscale up --timeout=90s
To authenticate, visit:
    https://login.tailscale.com/a/729c9ac012dda   ← 真实控制面签发的登录 URL!
timeout waiting for Tailscale service to enter a Running state (等待用户授权, 预期行为)
```
宿主机独立验证该 URL: **HTTP 200**, 跳转至 `login.tailscale.com/login?next_url=%2Fa%2F729c9ac012dda`, 页面为真实 Tailscale 设备授权页。完成最后一步入网需真实 Tailscale 账号在浏览器登录授权 (测试环境无账号, 故止步于此; 登录链路本身已全部打通)。

## 五、测试环境与自动化方式 (可复现)

- QEMU 10.0.13, `-M pc -m 2560 -smp 2`, IDE 磁盘 + e1000 网卡 (MAC 沿用 box), SLIRP NAT
- 客户机: datacastle/windows7 Vagrant box (Windows 7 Enterprise SP1 x64 原版, WinRM vagrant/vagrant)
- 磁盘: VMDK + qcow2 覆盖层 (backing file 只读, 写入持久化)
- 全部测试经 WinRM (pywinrm, NTLM) 远程执行; 长命令经 schtasks + 文件输出避免会话阻塞
- 完整测试日志见 release 附件

## 六、鸣谢与引用

- 被测仓库: [cshdotcom/tailscale-win7-builds](https://github.com/cshdotcom/tailscale-win7-builds) (v1.0.0)
- 工具链: [thongtech/go-legacy-win7](https://github.com/thongtech/go-legacy-win7)
- Tailscale: commit 566f89e0 (main, 2026-10-24)
