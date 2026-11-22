# 方案 B：服务端串联（服务器按域名分流）

在服务器上配置：指定网站的流量从住宅代理出去，其他流量照常直连。**所有连这台服务器的设备和用户都会生效**，客户端不用改。

```
任何客户端 ──▶ 你的服务器 ──┬─(指定网站)──▶ 住宅 IP（SOCKS5）──▶ 目标网站
                          └─(其他网站)──▶ 直连
```

**Xray（Reality）和 Hysteria2 两个服务都要配。** 客户端可能走其中任意一条通道，只配一个的话，走另一条通道的流量会绕过住宅 IP。

**前提**：服务器按 [proxy-node-manual](https://github.com/RestezCalme/proxy-node-manual) 部署（配置文件在 `/etc/xray/config.json`、`/etc/hysteria/config.yaml`），并且已经用《购买与注意事项》第 3 节的 curl 命令在服务器上测通了住宅代理。

> 以下命令都在服务器上运行。先把本仓库的 `templates/` 文件夹传到服务器的家目录（`scp -r templates ubuntu@<服务器IP>:~/`）。

---

## 1. 设置变量
把 `<…>` 换成你的信息。只有一个网站时，先填一个，多个网站的写法见第 5 节：

```bash
RES_IP=<住宅IP地址>
RES_PORT=<端口>
RES_USER=<用户名>
RES_PASS=<密码>
SITE=<网站域名，例如 example.com>
```

## 2. 先备份
```bash
sudo cp -p /etc/xray/config.json /etc/xray/config.json.bak-residential
sudo cp -p /etc/hysteria/config.yaml /etc/hysteria/config.yaml.bak-residential
```

## 3. 配置 Xray
① 用模板生成「住宅出站」和「分流规则」：
```bash
sed -e "s|__RES_IP__|$RES_IP|" -e "s|__RES_PORT__|$RES_PORT|" \
    -e "s|__RES_USER__|$RES_USER|" -e "s|__RES_PASS__|$RES_PASS|" \
    -e "s|__SITE__|$SITE|" ~/templates/xray-residential-outbound.json > /tmp/res.json
```

② 合并进配置：出站加在最后（第一个出站 `direct` 仍然是默认），分流规则插在统计接口规则之后、其他规则之前：
```bash
sudo jq --slurpfile r /tmp/res.json '
  .outbounds += [$r[0].outbound]
  | .routing.rules |= ([.[] | select(.outboundTag == "api")] + [$r[0].rule] + [.[] | select(.outboundTag != "api")])
' /etc/xray/config.json.bak-residential > /tmp/xray.new.json
```

③ 自检，通过后再替换并重启：
```bash
sudo docker run --rm -v /tmp/xray.new.json:/etc/xray/config.json:ro teddysun/xray:26.3.27 \
  xray run -test -c /etc/xray/config.json
```
✅ 最后一行是 `Configuration OK.` 再继续：
```bash
sudo install -m 644 /tmp/xray.new.json /etc/xray/config.json
sudo systemctl restart xray && systemctl is-active xray
rm -f /tmp/res.json /tmp/xray.new.json
```

> Xray 能按域名分流，靠的是配置里已经开启的 `sniffing`（从流量里识别出访问的域名）。proxy-node-manual 的模板默认开启，不要关掉。

## 4. 配置 Hysteria2
① 检查配置里是否已经有 `outbounds:` 或 `acl:`（有的话要手动合并，不能重复添加）：
```bash
sudo grep -nE '^(outbounds|acl):' /etc/hysteria/config.yaml || echo "没有，可以直接追加"
```

② 用模板生成并追加到配置末尾（`tee -a` 会保留原文件的属主和权限）：
```bash
sed -e "s|__RES_IP__|$RES_IP|" -e "s|__RES_PORT__|$RES_PORT|" \
    -e "s|__RES_USER__|$RES_USER|" -e "s|__RES_PASS__|$RES_PASS|" \
    -e "s|__SITE__|$SITE|" ~/templates/hysteria-residential.yaml | sudo tee -a /etc/hysteria/config.yaml >/dev/null
```

③ 重启并检查：
```bash
sudo systemctl restart hysteria-server
systemctl is-active hysteria-server
journalctl -u hysteria-server -n 5 --no-pager
```
✅ `active`，日志里有 `server up and running`。

规则说明：
- `outbounds` 里第一个（`direct`）是默认出口。
- `acl` 从上往下匹配：`residential(suffix:example.com)` 让 `example.com` 及其子域名走住宅代理；最后的 `direct(all)` 让其他所有流量直连，**必须保留，且放在最后**。

## 5. 多个网站怎么写
- **Xray**：在 `/etc/xray/config.json` 里找到 `"outboundTag": "residential"` 那条规则，往 `domain` 数组里加：
  ```json
  "domain": ["domain:example.com", "domain:example.net"]
  ```
- **Hysteria2**：在 `acl.inline` 里每个网站加一行，都写在 `direct(all)` 前面：
  ```yaml
  acl:
    inline:
      - residential(suffix:example.com)
      - residential(suffix:example.net)
      - direct(all)
  ```
改完同样要自检（Xray）并重启两个服务。

## 6. 确认成功
1. 临时把 `ipinfo.io` 也加进分流（按第 5 节加 `domain:ipinfo.io` 和 `residential(suffix:ipinfo.io)`），重启两个服务。
2. 在客户端上**分别**用 Reality 节点和 HY2 节点打开 `https://ipinfo.io`：
   - ✅ 两次显示的都是住宅 IP → 两个服务都配好了
   - ❌ 某一次显示服务器 IP → 那个服务的配置没生效，检查对应章节
3. 确认完把 `ipinfo.io` 删掉，再重启。

## 7. 回滚
```bash
sudo cp -p /etc/xray/config.json.bak-residential /etc/xray/config.json
sudo cp -p /etc/hysteria/config.yaml.bak-residential /etc/hysteria/config.yaml
sudo systemctl restart xray hysteria-server
```

## 8. 排错

| 现象 | 处理 |
|---|---|
| Xray 自检失败 | 多半是端口填错（必须是纯数字）或 JSON 被改坏；从备份重新做第 3 步 |
| Hysteria2 启动失败 | `journalctl -u hysteria-server -n 30` 看报错；常见原因是重复的 `outbounds:`/`acl:`，或缩进不对 |
| 指定网站打不开 | 住宅代理那一跳不通：先用《购买与注意事项》第 3 节的 curl 命令测；不通就回滚，等代理修好再配 |
| 只有一种节点显示住宅 IP | 另一个服务没配好，按第 6 节分别测试 |

> 两台服务器（主、备）都要做同样的配置，否则客户端切到另一台时，这些网站会变成用那台服务器的 IP 出去。
