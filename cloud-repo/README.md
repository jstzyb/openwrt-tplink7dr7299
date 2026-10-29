# 云端编译（GitHub Actions）操作步骤 —— 本机不用装任何东西

只要有一个 GitHub 账号即可。全程在网页上操作，代码在 GitHub 的 Runner 上编译，编完下载固件。

## 一、上传工作流（只需要 1 个文件）

1. 打开 https://github.com/new
   - Repository name：随便取，例如 `7dr7299-openwrt`
   - 选 **Public**（公开仓库 Actions 分钟数不限量；私有仓库每月只有约 2000 分钟免费）
   - 勾上 **Add a README file** → Create repository
2. 在仓库页面点 **Add file → Create new file**
3. 在文件名输入框里**先输入路径**：`.github/workflows/build.yml`（输入第一个 `/` 时会自动建目录），
   然后把本目录旁 `cloud-repo/.github/workflows/build.yml` 的**全部内容粘进去**
4. Commit message 随意 → **Commit changes**

> 也可以直接把整个 `cloud-repo` 文件夹拖拽到仓库页面上传，GitHub 会保留 `.github/...` 的目录结构。
> 上传后在 **Actions** 标签页能看到 "build-tl-7dr7299-openwrt" 出现（说明工作流被识别）。

## 二、先做一次「体检」（约 8~12 分钟，强烈建议）

1. **Actions** → 左侧选 `build-tl-7dr7299-openwrt` → 右侧 **Run workflow** → **Run workflow**
2. 参数：
   - `repository` = `WoChen5770/imoutowrt`（默认，已核实存在 7dr7299 机型定义）
   - `branch` = `master-mediatek-7dr7299`
   - `variant` = `full`（体检只编一个就行）
   - `diagnose_only` = **yes**
   - `overlay_mb` = `30`（不知道真实分区就先按 30）
   - 网络相关参数用默认值即可：`lan_ip` = `192.168.6.1`、`wan_device` = `sfp`（就是光口）、`wan_proto` = `dhcp`、`lan_extra_ports` = `eth1`
3. 跑完打开这次 run，重点看四处日志：
   - **核对机型定义与分区信息**：会打印 `7dr7299` 的机型定义、分区参数，以及**默认网口分配**那一行（确认光口在源码里到底叫什么名字）
   - **写入开机网络设置**：会把生成的开机脚本全文打印出来，核对 IP / WAN 口是不是你要的
   - **插件可用性体检 / 生成 .config**：`FOUND/MISS`、`OK/MISS` 两张表
     - `MISS adblock` → 去广告包当前源里没有，把 seed 里 `# CONFIG_PACKAGE_luci-app-simple-ac=y` 的 `#` 去掉再编
     - `MISS luci-app-openclash` → `feeds.conf` 里 openclash 源失效，换个源（见下面「三、换源」）
     - `MISS luci-app-passwall` → passwall 源失效
   把这四处日志截图/复制发给我，我一次改好。

### 网络参数（Run workflow 里可以直接改，不用改代码）

| 参数 | 默认值 | 作用 |
|---|---|---|
| `lan_ip` | `192.168.6.1` | LAN 管理地址（填 `none` 表示不改，用源码默认的 192.168.123.1） |
| `wan_device` | `sfp` | 当 WAN 的网口。**已核实源码里 7DR7299 的光口就叫 `sfp`**，默认挂在 LAN 上；留空表示沿用默认 WAN `eth1` |
| `wan_proto` | `dhcp` | 光口拨号方式：`dhcp`（光猫已注册好、直接拿 IP）/ `pppoe`（要在路由器拨号）/ `none`（只挪口不配协议） |
| `wan_vlan` | 空 | 运营商给了 VLAN ID（常见 41/45/10 之类）时填数字，会自动建 802.1q 子接口 |
| `wan_user` / `wan_pass` | 空 | 仅 `wan_proto=pppoe` 时填 |
| `lan_extra_ports` | `eth1` | 光口挪去 WAN 后腾出来的电口，加回 LAN 继续当普通口用 |

> ⚠️ `wan_user` / `wan_pass` 会打印在编译日志里。**Public 仓库的日志所有人都能看到**，要填 PPPoE 账号密码就把仓库设成 Private（Private 每月约 2000 分钟免费额度，够用），或者编完删掉那次 run。

## 三、换源（体检报错时用）

编辑 `.github/workflows/build.yml` → 铅笔图标 → 修改 `写入插件源 feeds.conf` 那一段 → Commit changes。

| 插件 | 当前用的源 |
|---|---|
| PassWall | `src-git passwall_packages https://github.com/fw9229907/luci-app-passwall.git` |
| OpenClash | `src-git openclash https://github.com/vernesx/DevService.git` |
| SSR-Plus | `src-git helloworld https://github.com/fw876/helloworld.git` |
| iStore（默认注释掉） | `src-git istore https://github.com/linkease/istore-ui.git` |

这三行失效也不会中断编译，只会让对应插件在体检里显示 MISS。

## 四、正式编译（体检通过后）

1. **Run workflow**，参数：
   - `variant` = `both`（同时编「全功能 full」和「精简 slim」，两个 job 并行，各约 60~120 分钟）
   - `diagnose_only` = `no`
2. 跑完在每个 job 页面底部 **Artifacts** 下载：
   - `openwrt-7dr7299-full` / `openwrt-7dr7299-slim`（zip）
   - 解压后取 `immortalwrt-*-tplink_tl-7dr7299-v1-squashfs-sysupgrade.itb`
   - `report-full` 里是最终 `.config`，可以核对装了哪些包
3. 免费额度：Public 仓库不限分钟；Private 约 2000 分钟/月（一次 both ≈ 3~4 小时 Runner 时间）。

## 五、拿到固件之后（先别急着刷）

- 刷机之后（第一次开机）访问：
  - 管理页面 `http://192.168.6.1`（工作流已把 LAN 地址改好了；LuCI 会要求你设 root 密码，之后 SSH 用同一个）
  - **网页终端 TTYD** `http://192.168.6.1:7681`，也可以在 LuCI 里 **服务 → TTYD 终端** 打开，用户名 `root` + 你设的密码
  - 想改回来：LuCI → 网络 → 接口 → LAN，或者 SSH 里 `uci set network.lan.ipaddr=...; uci commit network`
- WAN 已经自动改成**光口 `sfp`**（默认 DHCP）。第一次开机后如果 WAN 拿不到 IP：
  1. LuCI → 状态 → 系统日志，或 SSH 里 `ifstatus wan`、`logread | grep -i wan`、`ip link` 看 `sfp` 有没有 link 起来；
  2. 最常见的失败原因是**缺少 VLAN tag**，把 `wan_vlan` 填上运营商给的 ID 重新编一版；
  3. 运营商要在路由器上拨号的，把 `wan_proto` 改成 `pppoe` 并填账号密码；
  4. 也可以不重编，直接在 LuCI 里改（网络 → 接口 → WAN → 编辑 → 设备 选 `sfp`），改错了记得用 `uci`/界面改回来。
- 你已经刷好第三方 U-Boot，所以固件走 U-Boot 的 tftp/串口刷机路线，不用再折腾原厂「本地升级」那套 debug 固件。刷之前先确认要刷的是哪种镜像（本工作流出的是 `*-squashfs-sysupgrade.itb`，U-Boot 刷机时对应写入固件分区），**具体命令按你装的 U-Boot 教程来**，别照搬别的机型；刷错分区会变砖（U-Boot 通常还能救，但麻烦）。
- 有第三方 U-Boot 之后，如果原分区太小放不下全功能版，可以考虑放大 ubi 分区或用 extroot（见 `../FLASH_SIZE.md`）。
- 刷之前把 `cat /proc/mtd`（或 U-Boot 里的分区表）+ `df -h /overlay` 发我，我按实测值确定 `overlay_mb` 和哪个版本能刷。
- OpenClash 的核心（OpenClash/Premium、sing-box）不在固件里，需要在设备上上传或在线下载；首次在线下载可能被墙，建议提前用 iStore 或手动上传核心。

## 六、常见失败与对策

| 现象 | 原因 / 处理 |
|---|---|
| 工作流列表里看不到 | 文件必须正好在 `.github/workflows/*.yml`，注意大小写和 `/` 层级 |
| 立刻红叉，日志只有几行 | `variant` 参数解析失败 → 手动重跑一次，把 `variant` 选成 `full` 或 `slim` |
| `No space left on device` | Runner 磁盘不够；把 `variant` 改成单个，或把 `diagnose_only` 先设 yes 看看 |
| PassWall 编译报错（firewall4/nftables） | 该基线是 nftables，PassWall 兼容性不如老版本；日志发我，改走 SSR-Plus 或换分支 |
| `MISS` 一堆插件 | 第三方 feed 失效，见「三、换源」 |
