# GL-MT3000 维护记录

## 当前环境

- 设备：GL.iNet GL-MT3000，已通过 SSH 核实内核为 `5.4.211`，软件包架构为 `aarch64_cortex-a53`。
- 系统：GL.iNet `4.6.2`，基于 OpenWrt `21.02-SNAPSHOT`，修订 `r15812+899-46b6ee7ffc`，目标 `mediatek/mt7981`。界面标识为 `Openwrt like iStoreOS Style by wukongdaily`；这是 GL.iNet 系统上的风格化，不是官方 iStoreOS。
- 拓扑：光猫 → 上级路由器 LAN → GL-MT3000 WAN → 日常使用的设备。这是二级路由，不是旁路由；上级路由器不承担代理功能。
- 当前管理地址：`192.168.50.10`，已由用户确认，且本机默认网关也是此地址。旧记录中的 `192.168.8.1` 不再适用。
- 日常入口是 [LuCI 首页](http://192.168.50.10/cgi-bin/luci/admin/quickstart/)。[原厂管理页](http://192.168.50.10/)也可访问，但这套 CN 固件未显示原厂 VPN 菜单，`#/wgclient` 页面实测返回 `Not Found`，切换英文也未解决；不依赖它管理隧道。旧记录中的 `8080` 端口目前不可达，不要照搬。
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

现机已安装 `kmod-wireguard`、`wireguard-tools` 和 netifd 的 WireGuard 协议脚本。LuCI 的 WireGuard 协议编辑组件未安装；「网络 → 接口」页还会要求迁移旧 `ifname` 配置并重启网络，本次未执行，不能为查看状态就直接点「继续」。当前通过 SSH/UCI 维护，不把缺少网页编辑器误判为内核不支持，也不强装不匹配的软件包。

## 手机浏览器的小范围体验

2026-11-08 已安装独立的路由器 Xray 客户端，提供 **LAN 显式 HTTP 代理 `192.168.50.10:18080`**。Android 手机连这台路由器的 Wi-Fi 后，在该 Wi-Fi 的高级设置选择手动代理；排除地址可设 `localhost,127.*,192.168.*` 以保留内网访问。浏览器访问 `https://www.cloudflare.com/cdn-cgi/trace` 应看到 `ip=104.36.65.170`。试完恢复手机原来的 Wi-Fi 代理设置（通常为无）。不需要在手机安装 App、证书或 root；部分 App 不遵循 Wi-Fi HTTP 代理，不代表整机流量接管。

- **Mac 仅作 SSH 控制终端**，不安装/运行代理客户端，不设置 Mac 代理。此前临时 macOS 测试二进制已删除。
- 服务为 `/etc/init.d/cg-vps-proxy`；脚本与当前操作入口在本机 `~/code/config-guides/vps/proxy/openwrt/README.md`。私有配置为 `/etc/cg-vps-proxy/client.json`，不提交。
- 进程运行于既有 `nobody`（UID 65534）；旧 OpenClash 恰好排除此 UID 以防代理套娃。同配置 root 对照握手失败、此 UID 对照成功，不新增出站绕过规则。规则最初引入时间未查明。
- HTTP 仅监听 LAN 地址，SOCKS 仅监听 `127.0.0.1:10808`；自有 INPUT 链只放行 LAN 接口/网段和 loopback，`/etc/firewall.user` 只注册这段保护的刷新。网络/DHCP/UCI 防火墙/OpenClash 配置未变，旧 OpenClash 进程没有重启。
- 不会自动把全家切到新节点。重复安装、服务重启、保护规则幂等、非 LAN 源拒绝、外部 HTTPS 和并发已测；没有操作真实手机，也没有整机重启。路由器路径 3×5 MB 下载中位数约 4.99 Mbps，不是手机速率保证。

停用时先恢复手机原代理设置，之后可执行：

```sh
ssh glmt3000 '/etc/init.d/cg-vps-proxy disable; /etc/init.d/cg-vps-proxy stop'
# 恢复：ssh glmt3000 '/etc/init.d/cg-vps-proxy enable; /etc/init.d/cg-vps-proxy start'
```

本轮安装备份在路由器 `/root/.local/cg-vps-proxy/backups/install.*`，均属私有；保留不删除。

## VPS 隧道与切换边界

连接层已部署，**日常设备的流量尚未切换**。不依赖订阅；后续策略路由直接使用标准 OpenWrt 接口，不另起一套原厂 VPN 管理逻辑。

- VPS：`104.36.65.170:51820/UDP`，AlmaLinux `9.7`；服务为 `wg-quick@wg0`，实际配置及私钥在 VPS 的 `/etc/wireguard/wg0.conf`（权限 `600`）。IPv4 转发通过 `/etc/sysctl.d/70-glmt3000-wireguard.conf` 启用；NAT/转发规则由隧道服务启停管理，只覆盖客户端隧道地址。
- 软路由：标准 netifd 接口 `wgvps`，实际配置及私钥在 `/etc/config/network`；VPS 隧道地址为 `10.77.0.1`，路由器为 `10.77.0.2`。隧道已配置自动启动，但未通过整机重启验证。
- 路由安全边界：`route_allowed_ips=0`、`defaultroute=0`、`peerdns=0`，只添加到 `10.77.0.1/32` 的直连路由。Peer 的 `AllowedIPs=0.0.0.0/0` 是加密流量的允许范围，不等于已经给全家添加默认路由。
- 验证证据：握手成功，接口停启后能重连；短时连续 ping 与接近 MTU 的大包均返回。通过仅限测试源地址和目标的临时路由/代理例外，外部 HTTPS 服务观察到出口为 `104.36.65.170`。这些临时例外和测试接口已全部删除；该结果不代表长期稳定性、吞吐量或 LAN 设备切换已验证。

**切换前必须解决旧代理的所有权冲突。** OpenClash 仍保持原有启用状态，DNS 转发到 `127.0.0.1#7874`，其 NAT OUTPUT 链会截获本机 TCP 请求；实测会使指定了测试源地址的 HTTPS 请求仍走旧出口。临时绕开截获仅用于诊断，不是完成修复。正式切换时需协调停用旧代理、DNS 和策略路由，确保选定流量只由新方案控制；不能只叠加一条路由，也不能直接清空防火墙。

检查与停止连接层（不会替你切换日常设备）：

```sh
ssh glmt3000 'wg show wgvps; ping -I wgvps -c 5 10.77.0.1'
ssh glmt3000 'ifdown wgvps'
# 重新启动：ssh glmt3000 'ifup wgvps'
ssh bandwagon 'systemctl status wg-quick@wg0 --no-pager'
# 停止服务端：ssh bandwagon 'systemctl disable --now wg-quick@wg0'
```

配置前备份保存在两台设备各自的 `/root/.local/glmt3000-vps/baseline.*`，含敏感信息，不能上传。私有客户端导入文件另存于软路由的 `/root/.local/glmt3000-vps/profiles/` 和本仓库被忽略的 `.local/`，不得强制提交；标准 UCI 接口才是当前客户端配置的权威来源，不再同步维护导入文件。VPS 的包安装事务记录在 `/root/.local/glmt3000-vps/package-transaction.txt`：安装 `wireguard-tools` 时还安装了 `systemd-resolved` 并更新 systemd 相关依赖；SSH、NetworkManager 和原有 DNS 配置复查正常，未重启整机。

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
