# vless-ban 一键安装教程

功能：节点的 ws 路径（比如 `/mm`）以外的访问一律拉黑 IP，并返回 nginx 默认页（蜜罐）。正常用户不受影响。规则细节见 [docs-guard.md](docs-guard.md)。

不需要装 Go，程序由 GitHub 自动编译好，脚本直接下载。

## 一、安装

在节点 VPS 上用 root 执行：

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/acc66/vless-ban/main/vless-ban.sh)
```

选 `1. 安装`，按提示填：

| 提示 | 填什么 |
| --- | --- |
| 面板地址 | Xboard 面板网址，如 `https://panel.example.com` |
| 面板通讯密钥 | 面板 → 系统设置 → 服务端 → 通讯密钥 |
| 节点 ID | 面板节点列表里的 ID |
| 内核 | 直接回车（auto） |
| 白名单 IP | 你自己电脑 / 手机的公网 IP，防止测试时把自己拉黑 |
| 走 CDN 吗 | 节点域名开了 Cloudflare 小黄云选 `y`，否则 `n` |

最后显示 `服务运行中` 和 `guard: enabled ... path=/mm` 就成功了。

## 二、面板设置

节点要满足：

- 协议 `vless` / `vmess` / `trojan`，传输方式 `ws`，**不开 TLS**
- 或 `shadowsocks` + 插件 `v2ray-plugin`，插件参数 `mode=websocket;path=/mm;host=你的host`（客户端插件加 `mux=0`）

不满足的节点（开了 TLS、tcp、hysteria 等）照常运行，只是不拉黑。

客户端示例（就是你现在这条）：

```
gcp1.help600.com;2086;0;ws;;path=/mm|server=gcp1.help600.com|host=xg2.apple9.org|outside_port=2086
```

## 三、自定义路径

`/mm` 可以随便改，脚本和程序里没有写死：

1. 面板编辑节点，把 ws 路径改成比如 `/a8f3k2x9`
2. 节点几十秒内自动同步，拉黑规则跟着换成新路径，不用重启
3. 客户端更新订阅

建议用长一点的随机路径。不要用 `/`，否则扫描器访问首页不会被拉黑。

## 四、测试

用一台**不在白名单里**的机器（或手机开流量热点）：

```bash
curl -i http://节点IP:2086/
```

1. 第一次返回 `Server: nginx/1.18.0` 和 `Welcome to nginx!` 页面
2. 再执行一次直接断开，说明已被拉黑

再运行脚本选 `5. 查看黑名单`，能看到这个 IP、时间和它访问的路径。

最后用客户端正常连接，确认能上网。

## 五、日常管理

再次运行安装命令就是管理菜单：

```
 1. 安装
 2. 升级（保留配置和黑名单）
 3. 修改白名单 / CDN 设置
 4. 查看状态和拉黑日志
 5. 查看黑名单
 6. 解封 IP
 7. 清空黑名单
 8. 关闭拉黑功能
 9. 卸载
```

也可以直接带参数：

```bash
S=https://raw.githubusercontent.com/acc66/vless-ban/main/vless-ban.sh
bash <(curl -fsSL $S) status   # 状态和日志
bash <(curl -fsSL $S) list     # 黑名单
bash <(curl -fsSL $S) unban    # 解封
bash <(curl -fsSL $S) upgrade  # 升级
```

黑名单文件是 `/etc/XboardNode-Plus/blacklist.txt`，每行一个 IP，手动删行也能解封，5 秒内生效。

## 六、走 Cloudflare CDN 的注意事项

开了小黄云时，节点看到的是 Cloudflare 的 IP。安装时「走 CDN 吗」一定要选 `y`，否则会把 CF 的 IP 拉黑，所有用户断线。

选 `y` 后程序按 `CF-Connecting-IP` 头里的真实 IP 拉黑。这个头别人直连时可以伪造，所以最好让端口只接受 Cloudflare 回源：

```bash
for ip in $(curl -s https://www.cloudflare.com/ips-v4); do
  iptables -A INPUT -p tcp --dport 2086 -s $ip -j ACCEPT
done
iptables -A INPUT -p tcp --dport 2086 -j DROP
```

不确定有没有走 CDN：先选 `n` 安装后马上选 `4` 看日志，`guard: probe ip=` 后面如果是 `104.`、`172.64.` 这类 Cloudflare 段，就选 `3` 改成走 CDN，再选 `7` 清空黑名单。

## 七、常见问题

**自己被拉黑了**：换个网络运行脚本选 `6` 解封，再选 `3` 把自己的 IP 加进白名单。家宽 IP 会变，变了要重新加。

**日志里没有 `guard: enabled`**：节点开了 TLS 或不是 ws，看日志里 `guard: not guarding node reason=` 的原因。

**已经装过原版 XboardNode-Plus**：直接选 `1` 安装会覆盖成本版本，或选 `2` 升级。注意之后不要再用原版的安装命令升级，会被换回原版。
