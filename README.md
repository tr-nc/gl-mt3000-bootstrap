# GL-MT3000 维护记录

## 当前环境

- 设备：GL.iNet GL-MT3000，已通过 SSH 核实内核为 `5.4.211`，软件包架构为 `aarch64_cortex-a53`。
- 系统：OpenWrt `21.02-SNAPSHOT`，修订 `r15812+899-46b6ee7ffc`，目标 `mediatek/mt7981`。界面标识为 `Openwrt like iStoreOS Style by wukongdaily`；这是 GL.iNet 系统上的风格化，不是官方 iStoreOS。
- 拓扑：光猫 → 上级路由器 LAN → GL-MT3000 WAN → 日常使用的设备。这是二级路由，不是旁路由；上级路由器不承担代理功能。
- 当前管理地址：`192.168.50.10`，已由用户确认，且本机默认网关也是此地址。旧记录中的 `192.168.8.1` 不再适用。
- 已实测可访问：[原厂管理页](http://192.168.50.10/)、[LuCI](http://192.168.50.10/cgi-bin/luci/)。旧记录中的 `8080` 端口目前不可达，不要照搬。
- 上游 WAN：DHCP，当前为 `192.168.1.13`，网关 `192.168.1.1`；租约地址可能变化。LAN 为 `192.168.50.0/24`。

## SSH 登录

已通过用户登录的后台终端读取主机公钥、独立核对并保存信任，再验证 `root` 公钥登录成功。ED25519 主机指纹：`SHA256:RlMn5onw7RSHJwja5hSlSQfTtts5EdLbm7ksJtJkB/8`。

本机的 `config-guides/autoconfs/ssh/config` 已配置 `glmt3000` 别名，使用 `~/.ssh/id_ed25519`；对应用户公钥通过 LuCI 的「系统 → 管理权 → SSH 密钥」添加。其他电脑需要各自建立主机信任与登录授权。

```sh
ssh glmt3000
# 未配置别名时：
ssh root@192.168.50.10
```

首次连接时，通过可信管理页面的终端或已有可信 SSH 会话独立核对主机指纹，再接受并保存。仅用 `ssh-keyscan` 从网络读到公钥，不能证明主机身份。不要默认删除 `known_hosts` 记录或关闭主机验证；遇到密钥变化时，先确认重装、重置或地址复用等原因。

若设备使用 Dropbear，可在路由器的可信终端中查看 ED25519 主机公钥及指纹（命令不存在或文件不存在时，先确认实际 SSH 服务和密钥位置）：

```sh
dropbearkey -y -f /etc/dropbear/dropbear_ed25519_host_key
```

不要将密码、私钥、订阅链接或含密钥的备份提交到这个公开仓库。每台电脑优先使用独立用户密钥；只将用户公钥加入路由器的授权列表，不复制私钥。

## 登录后的只读检查

先确认固件、WireGuard 组件及现有路由，不能根据脚本提供过安装选项就认定组件已经安装。

```sh
cat /etc/openwrt_release
ubus call system board
command -v wg
opkg list-installed | grep -Ei 'wireguard|pbr|vpn-policy-routing|openclash'
uci -q get network.lan.ipaddr
uci -q get network.wan.proto
ip -4 route show
```

当前目标是用自有 VPS 建立隧道，不依赖订阅或 OpenClash。现机已安装 `kmod-wireguard`、`wireguard-tools`、GL.iNet 的 WireGuard 客户端和 VPN 策略组件，且有 netifd 的 WireGuard 协议脚本；尚无 WireGuard 接口。先验证实际 UDP 通路，再接入一台测试设备，后续由软路由的策略路由和 DNS 决定分流。

注意：旧 OpenClash 目前仍启用并有核心进程，DNS 转发到 `127.0.0.1#7874`，同时有透明代理规则；GL.iNet 也已有策略路由规则。不能直接清空防火墙、覆盖 DNS 或叠加默认路由。切换方案时要协调这些现有规则，并保留回退路径。

改动前在仓库外或被忽略的本地目录保存配置备份，保留管理入口和可执行的回退方式；一次只改一个环节，不先切换全家网络。不为移除旧代理而恢复出厂，也不在未核实依赖与软件源兼容性的情况下升级固件或强装软件包。

## 历史安装资料

以下文件仅保留为历史资料，不代表当前推荐操作：

- `gl-inet.sh`：旧的一键风格化脚本，会安装组件、修改软件源和防火墙等。其中包含按固定区域索引放宽 WAN 入站、移除软件包签名检查及下载后执行代码的路径，不能当作安全的日常维护入口。内部的旧地址不代表现机地址。
- `OpenClash_0.46.014+aarch_64_core.run`：旧安装包，不在当前接入方案中使用；不执行它。
- `openwrt-mt3000-4.6.2-0628-1719566865.tar`：包含 `CONTROL`、`kernel`、`root` 的 sysupgrade 固件归档，不是现机配置备份，也不能据此认定现机固件版本；不直接刷入。

旧的安装、卸载及订阅配置步骤可从 Git 历史查看，不再作为当前操作指南。历史版本曾包含订阅凭据；即使正文已删除，仍应视为公开，如尚有效需在服务端撤销或更换。不要为了清理记录直接重写或推送公共历史。

原始参考（历史来源，不代表当前兼容性或安全背书）：

- <https://didiboy0702.gitbook.io/wukongdaily/new-shou-ye/mt3000-bu-shua-ji-shi-yong-zhi-nan>
- <https://cafe.cpolar.cn/wkdaily/gl-inet-onescript/src/branch/master>
