# 防探测拉黑部署教程

把 XboardNode-Plus 换成带 guard 的版本：访问错误的 ws 路径（比如不是 `/mm`）就拉黑 IP，返回 nginx 默认页。规则和原理见 [docs-guard.md](docs-guard.md)。

以下命令在节点 VPS（Linux，root）上执行。

## 1. 编译

需要 Go 1.26 或更高版本。

```bash
# 安装 Go（已安装可跳过；arm64 机器把 amd64 换成 arm64）
cd /tmp
curl -LO https://go.dev/dl/go1.26.0.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.26.0.linux-amd64.tar.gz
export PATH=/usr/local/go/bin:$PATH

# 拉代码并编译
git clone https://github.com/acc66/vless-ban.git
cd vless-ban
make build-linux          # arm64 机器用 make build-linux-arm64
ls -lh xboard-node-linux-amd64 xbctl-linux-amd64
```

内存小于 1G 的机器编译可能被 OOM，可以在别的机器上编译好，再把 `xboard-node-linux-amd64` 和 `xbctl-linux-amd64` 传上来。

## 2. 安装

### 全新安装

```bash
bash install.sh \
  --panel https://你的面板地址 \
  --token 面板通讯密钥 \
  --node-id 节点ID \
  --binary ./xboard-node-linux-amd64 \
  --xbctl-binary ./xbctl-linux-amd64
```

### 已经装过原版

只替换程序，配置不动：

```bash
systemctl stop xboard-node
install -m 755 xboard-node-linux-amd64 /usr/local/bin/xboard-node
install -m 755 xbctl-linux-amd64 /usr/local/bin/xbctl
```

注意：以后不要直接运行 `install.sh upgrade`，否则会下载原版程序覆盖掉。

## 3. 开启 guard

编辑 `/etc/XboardNode-Plus/config.yml`，在最外层（和 `panel:`、`kernel:` 同级）加上：

```yaml
guard:
  enabled: true
  blacklist: "/etc/XboardNode-Plus/blacklist.txt"
  allow: "你自己的IP"        # 多个用逗号分隔，防止测试时把自己拉黑
```

重启：

```bash
systemctl restart xboard-node
journalctl -u xboard-node -n 50 | grep guard
```

看到类似下面这行就说明生效了：

```
guard: enabled tag=vless:2086 listen=:2086 path=/mm ... banned=0
```

如果看到 `guard: not guarding node reason=...`，表示这个节点不适用（比如开了 TLS），会按原方式运行。

## 4. 面板与客户端设置

面板节点：传输协议选 `ws`，路径填 `/mm`（或你自定义的路径），**不要开 TLS**。

客户端节点示例（就是你现在用的这条）：

```
gcp1.help600.com;2086;0;ws;;path=/mm|server=gcp1.help600.com|host=xg2.apple9.org|outside_port=2086
```

路径必须和面板完全一致，`/mm`、`/mm/`、`/MM` 是三个不同的路径。

ss 节点：插件用 `v2ray-plugin`，参数类似 `mode=websocket;path=/mm;host=xg2.apple9.org`，客户端插件要加 `mux=0`。

## 5. 自定义路径

`/mm` 不是写死的。guard 直接用面板里节点的 ws 路径，改面板就行：

1. 面板编辑节点，把路径改成比如 `/a8f3k2x9`。
2. 节点下次同步配置时（几十秒内）guard 自动切换到新路径，不用重启。
3. 客户端更新订阅，换成新路径。

建议：

- 用长一点的随机路径，比如 `/a8f3k2x9`，扫描器很难猜中。
- 不要用 `/`。路径是 `/` 时，探测 `/` 只会拿到蜜罐页，不会被拉黑。
- `?ed=2048` 这类参数可以加，guard 匹配时会忽略问号后面的部分。

## 6. 验证

用一台**不在 allow 列表里**的机器（或手机流量）测试：

```bash
curl -i http://节点IP:2086/
```

返回 `Server: nginx/1.18.0` 和 `Welcome to nginx!` 页面；再执行一次会直接断开（已被拉黑）。

在节点上查看：

```bash
cat /etc/XboardNode-Plus/blacklist.txt
# 1.2.3.4 # 2026-11-01 13:00:00 vless:2086 GET /

journalctl -u xboard-node -f | grep guard     # 实时看拉黑日志
```

然后用客户端正常连接，确认能上网、面板在线 IP 显示的是真实 IP。

## 7. 解封

```bash
# 解封一个 IP：删掉对应行，5 秒内生效，不用重启
sed -i '/^1\.2\.3\.4 /d' /etc/XboardNode-Plus/blacklist.txt

# 全部解封
rm /etc/XboardNode-Plus/blacklist.txt
```

## 8. 走 Cloudflare CDN 时

如果 `xg2.apple9.org` 开了小黄云，节点看到的来源 IP 是 Cloudflare 的。不改配置会把 CF 的 IP 拉黑，所有用户断线。

```yaml
guard:
  enabled: true
  blacklist: "/etc/XboardNode-Plus/blacklist.txt"
  real_ip_header: "CF-Connecting-IP"
```

同时要用防火墙让这个端口只接受 Cloudflare 回源，否则别人可以直连并伪造这个头：

```bash
for ip in $(curl -s https://www.cloudflare.com/ips-v4); do
  iptables -A INPUT -p tcp --dport 2086 -s $ip -j ACCEPT
done
iptables -A INPUT -p tcp --dport 2086 -j DROP
```

不确定有没有走 CDN，可以先用 `no_ban: true` 试运行：只返回蜜罐页不拉黑，观察日志里 `guard: probe ip=...` 是不是 CF 的 IP。

## 9. 关闭

把 `enabled: true` 改成 `false`（或删掉整个 `guard:` 段），`systemctl restart xboard-node`，恢复成内核直接监听。
