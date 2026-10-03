# sideload-keeper —— 有道词典笔「侧载应用保活」完整方案

给有道词典笔（YDPX6/X7 系列，PenOS 4.x，root 后可写 `/userdisk`）装自己编译的 miniapp 之后，
**应用会在开机约 35 秒后被系统悄悄删掉**。本仓库是逆向出这套删除机制之后写的保活方案，
包含完整脚本、离线自测、部署工具与全部原理说明。

> 实测机型：YDPX7-1（X7 Pro，RK3562，PenOS 4.8.7）。X6/X6 Pro 机制相同。

---

## 一、谁在删你的应用（逆向结论）

**不是** `APP_UPD`（那是 `JQuick::Updater::AppUpdateManager`，在 `/usr/lib/libfalcon.so` 里，
只做版本检查/下载/安装，**没有任何 remove 接口**，`rollingBack:407` 是正常返回码，
stock 应用也会是 407）。

真正的凶手是**桌面启动器里的 `AppWhitelistCleaner`**：

```
/userdata/miniapp/data/mini_app/pkg/8080222437664451/b/indexPage-93295cbd.js.bin   ← 启动器主包
```

它在**每次开机**（页面 show 之后约 35 秒，一次）执行：

1. `GET https://api-overmind.youdao.com/openapi/get/luna/hardware/dictpen-client/prod/appWhitelist`（超时 5s）
2. 拿到 `{"value":{"enabled":true,"allowedApps":[116 个 appid]}}`
3. 遍历已安装应用，凡**不在 `allowedApps` 里**的：
   ```
   removing: <appid> <name> flag: 16384
   → pm.removePackage
   → done, removed: N
   ```

**保护逻辑**（两处，都对我们有利）：

```
appWhitelist is disabled or empty, skip.                 ← enabled=false 或列表空
allowedApps list is empty, skip to avoid accidental removal.  ← 空列表保护
```

**注意两点**：

- `flag: 16384` 只是"侧载标记"（商店应用是 0 或 3），**它只是日志信息，不是删除判据**
  （已用实验证伪：把包伪装成白名单 appid 仍带 flag 16384 也能活）。
- 应用数据放在**包目录内部**（`<pkg>/<appid>/data/`），所以被删 = 数据一起没了。

---

## 二、保活思路（三层，逐层加固）

```
        ┌──────────────────────────────────────────────────────────┐
        │ ① DNS 劫持：让 cleaner 根本拉不到白名单（首选、几乎零开销）│
        ├──────────────────────────────────────────────────────────┤
        │ ② keeper：装机钩子，判断本次开机是否被清理 → 被清则修复+常驻 │
        ├──────────────────────────────────────────────────────────┤
        │ ③ cron 镜像：keeper 确认安全退出后，由 cron 每 5 分钟刷镜像   │
        └──────────────────────────────────────────────────────────┘
```

### ① DNS 劫持（`dns-blackhole.sh`）

`/etc/hosts` 在只读 erofs 上改不了，但**可以用 `mount --bind` 盖住**：

```sh
# 自定义 hosts 放 /userdisk（可写），bind mount 到 /etc/hosts
sh /userdisk/skip_re/dns-blackhole.sh apply     # api-overmind.youdao.com → 127.0.0.1
sh /userdisk/skip_re/dns-blackhole.sh status    # 看解析结果 + 挂载状态
sh /userdisk/skip_re/dns-blackhole.sh remove    # 还原（立即生效）
```

效果：cleaner 的 `httpGetOvermindConfig` 超时/失败 → 走 `skip` 分支 → **一个应用都不删**。
日志证据：`grep -a 'fetch overmind appWhitelist failed' <log>` 有记录、
`grep -a -c 'removing:' <log>` = **0**。

代价：`api-overmind.youdao.com` 上其它远端配置（目前已知只有这个 cleaner）一起失效；
域名列表在 `/userdisk/skip_re/dns-blackhole.conf`，可随时增删。

### ② keeper（`sideload-keeper.sh`，核心）

由固件自带的**开机钩子**拉起：

```sh
# /etc/init.d/S99_run_test_scripts 里唯一一行：
[ -f /userdisk/skip_re/skip_login.sh ] || exit 0
/userdisk/skip_re/skip_login.sh &
```

脚本三种模式（`KEEPER_MODE`）：

| 模式 | 触发 | 行为 |
|---|---|---|
| 默认（空） | 开机 | 完整守护流程（见下） |
| `mirror` | cron 每 5 分钟 | 只刷新镜像（硬链接），秒级退出 |
| `oneshot` | 手动 | 自检用 |

**开机流程**：

```
阶段 0  若 dns-blackhole.conf 有内容 → 先 apply（DNS 劫持）
阶段 0b 若 usb-mode.conf 有内容 → 恢复 USB 模式（MTP/ADB，见 usb-mode.sh）
阶段 1 等待 /tmp/SecondaryStartupReady（次级根挂好了）
阶段 2 从"本次开机的框架日志"里读 cleaner 的输出（按字节偏移记录已读位置）：
         grep 'removing:'                        → 被删了 → 修复 + 常驻守护
         grep 'allowedApps list is empty, skip'
          或 'appWhitelist is disabled or empty, skip'
          或 'fetch overmind appWhitelist failed' → 本次安全
               → 做最后一次镜像
               → 装 cron（*/5 * * * * KEEPER_MODE=mirror）
               → 删锁文件、写日志、exit 0   ← **不再常驻**（省电、少占用）
         300 秒内还没结论 → 保守起见常驻守护
```

**修复动作**（`sweep`）：

1. 先做镜像：`cp -al`（硬链接）把 `<pkg>` 整目录备份到 `/userdisk/skip_re/data-backup/<appid>/`
   —— 硬链接意味着**即使原文件被 unlink，数据块也不会消失**（这是能恢复数据的关键）
2. 若包已不在：`rm -rf <pkg>` → `miniapp_cli install <amr>` → 把镜像里的 `data/` 拷回去
3. 重新注册到 `packages.json`

**唯一正确的停止方式**：

```sh
kill $(cat /tmp/sideload-keeper.pid)     # 用 pid 文件
# 千万别用 pkill -f skip_login.sh —— 脚本名匹配到调用者自己的 adb shell，会把自己杀掉
```

### ③ cron 镜像

keeper 确认本次开机安全后会自行退出，把"保持镜像新鲜"的活儿交给 cron：

```
*/5 * * * * KEEPER_MODE=mirror /bin/sh /userdisk/skip_re/skip_login.sh >/dev/null 2>&1
```

**为什么镜像有用**：应用数据在包目录里，被删时一起没；镜像用硬链接，数据块会保留 ✔
**为什么 keeper 还要留着**：有道随时可能换校验域名/URL，那时 DNS 劫持失效，
下次开机 cleaner 就会真的删 —— keeper 的 `removing:` 分支会兜住 ✔

---

## 三、部署

```powershell
# 一条命令：连接 → 认证(ydpen2026) → 推送 keeper/amr → 重启 keeper → 打印状态
python tools\deploy_keeper.py
```

笔上文件布局：

```
/userdisk/skip_re/
    skip_login.sh            ← keeper 主脚本（= 本仓库 sideload-keeper.sh）
    dns-blackhole.sh         ← DNS 劫持
    dns-blackhole.conf       ← 要屏蔽的域名（默认 api-overmind.youdao.com）
    hosts-custom             ← apply 时生成的 hosts
    usb-mode.sh / usb-mode.conf  ← USB 模式持久化（可选）
    amr/                     ← 保活用的 .amr（PenTerm.amr / FileManager.amr / …）
    data-backup/<appid>/     ← 硬链接镜像
    sideload-keeper.log      ← 运行日志
```

---

## 四、离线自测（不需要真机）

```sh
python scripts\keeper_selftest.py        # 镜像/硬链接存活/重装回填/重新注册，全绿
python scripts\keeper_decision_test.py   # 决策分支：
                                         #   kt3a = 日志出现 skip → 最终镜像 + 装 cron + 退出
                                         #   kt3b = 日志出现 removing: → 修复 + 数据回填 + 常驻
```

两个测试用 busybox-w32 在 Windows 上直接跑 POSIX sh：造假 amr（真 zip）、假 pkg、假
`miniapp_cli`、假 `crontab`，断言文件内容与 inode 级别行为。

真机验证常用命令：

```sh
grep -a 'removing:' /data/applog/DictPen_*.log | wc -l     # 本次开机被删了几个（应为 0）
grep -a 'allowedApps list is empty, skip' /data/applog/DictPen_*.log
ps | grep '[s]kip_login'                                    # keeper 是否常驻
crontab -l | grep mirror                                    # cron 是否装上
grep -q ' /etc/hosts ' /proc/mounts && echo "DNS 劫持在"     # DNS 是否生效
```

---

## 五、踩过的坑（都写进代码了）

| 坑 | 说明 |
|---|---|
| `pkill -f skip_login.sh` 自杀 | 脚本名匹配到调用者 adb shell 自己的 cmdline → 只用 pid 文件 kill |
| `/tmp` 是 tmpfs | 重启即丢：pid/seen/cron 临时文件全放**日志旁边或 /userdisk** |
| 失败的重定向会杀掉整个脚本 | busybox 里 `:` 这种特殊内建的 redirect 失败直接退出 → 必须确保目录存在 |
| 硬链接镜像 ≠ 拷贝 | 必须 `cp -al`（同分区），否则数据块在被删时真的没了 |
| `flag: 16384` 不是判据 | 只是日志信息，改它没用（已实验证伪） |
| 全屏/长驻脚本与 SIGPIPE | reader 循环要容忍 EPIPE，否则 shell 一退出 keeper 就崩 |
| 顶栏注入点击无效 | 调试相关：`/dev/input` 注入在屏幕顶部 y≤25 会被框架手势层吃掉 |

---

## 六、文件说明

| 文件 | 作用 |
|---|---|
| `sideload-keeper.sh` | 主脚本：开机决策、修复、镜像、cron 安装、常驻守护 |
| `dns-blackhole.sh` | `mount --bind` 覆盖 `/etc/hosts` 的域名黑洞（apply/status/remove） |
| `usb-mode.sh` | 切 USB 模式（MTP/ADB/两者）并持久化 —— 顺带解决"插电脑没有 MTP" |
| `deploy_keeper.py` | 一键部署（PC → 笔） |
| `keeper_selftest.py` | 离线自测：镜像、硬链接存活、重装回填、重新注册 |
| `keeper_decision_test.py` | 离线测试两个决策分支（安全跳过 / 被删修复） |
| `docs/` | 逆向报告：谁在删应用、APP_UPD 真相、USB/MTP 机制 |

## 七、许可

自己写着玩的工具集合，随便用（MIT）。**改动机器有风险**：设备是你的，责任也是你的。
