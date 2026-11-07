# GL-MT3000 维护记录

## 当前环境

- 设备：GL.iNet GL-MT3000，ARMv8；用户提供的系统截图显示内核为 `5.4.211`。
- 界面标识：`Openwrt like iStoreOS Style by wukongdaily`。历史记录是在 GL.iNet 系统上安装 iStore 商店和主题，不代表刷入了官方 iStoreOS；实际固件版本仍需登录后确认。
- 拓扑：光猫 → 上级路由器 LAN → GL-MT3000 WAN → 日常使用的设备。这是二级路由，不是旁路由；上级路由器不承担代理功能。
- 当前管理地址：`192.168.50.10`，已由用户确认，且本机默认网关也是此地址。旧记录中的 `192.168.8.1` 不再适用。
- 已实测可访问：[原厂管理页](http://192.168.50.10/)、[LuCI](http://192.168.50.10/cgi-bin/luci/)。旧记录中的 `8080` 端口目前不可达，不要照搬。
- LuCI 登录页标示 `openwrt-21.02 branch`；这是 LuCI 的版本标识，不能代替实际固件版本。

## SSH 登录

历史记录使用 `root`，当前 SSH 服务可达；尚未完成这台电脑的主机信任和登录授权。

```sh
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

当前目标是用自有 VPS 建立隧道，不依赖订阅或 OpenClash。WireGuard 是待实测的首选，尚未部署；先验证实际 UDP 通路，再接入一台测试设备，后续由软路由的策略路由和 DNS 决定分流。WireGuard 支持情况、旧 OpenClash 的运行状态以及已有 DNS/防火墙规则均需现机检查。

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
