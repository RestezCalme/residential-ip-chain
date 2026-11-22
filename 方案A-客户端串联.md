# 方案 A：客户端串联（Clash Verge / mihomo）

只改你电脑上的 Clash 配置，服务器不用动。原理是 mihomo 的 `dialer-proxy`：连住宅代理时，先经过你自己的节点。

```
Clash ──▶ 🚀 节点选择（你的节点，穿墙）──▶ 🏠 住宅IP（SOCKS5）──▶ 目标网站
```

**前提**：电脑上用的是 mihomo 内核的客户端（Clash Verge、Clash Verge Rev、Mihomo Party 等），并且已经有一个叫 `🚀 节点选择` 的分组（[proxy-node-manual](https://github.com/RestezCalme/proxy-node-manual) 的模板里就有；名字不同的话，下面对应地改）。

---

## 1. 准备要用到的信息
- 住宅代理的 IP、端口、用户名、密码（见《购买与注意事项》第 2 节）
- 需要走住宅 IP 的网站域名，例如 `example.com`（一个或多个）

## 2. 加入三段配置
打开 `templates/clash-residential-snippet.yaml`，把占位符换成你的信息，然后把三段**分别**合并到你的 Clash 配置文件里：

**① 在 `proxies:` 下面加一个节点**
```yaml
  - name: 🏠 住宅IP
    type: socks5
    server: <住宅IP地址>
    port: <端口>
    username: <用户名>
    password: <密码>
    udp: false
    dialer-proxy: 🚀 节点选择
```

**② 在 `proxy-groups:` 下面加一个分组**
```yaml
  - name: 🏠 住宅出口
    type: select
    proxies:
      - 🏠 住宅IP
      - 🚀 节点选择
```

**③ 在 `rules:` 的最前面加规则**（每个网站一行）
```yaml
  - DOMAIN-SUFFIX,example.com,🏠 住宅出口
  - DOMAIN-SUFFIX,example.net,🏠 住宅出口
```

三个要点：
- **规则必须放在最前面。** Clash 从上往下匹配，排在后面的规则会被前面更宽泛的规则抢先匹配掉。
- **`dialer-proxy` 必须指向不包含「🏠 住宅IP」的节点或分组**，否则会形成循环。
- `DOMAIN-SUFFIX,example.com` 会同时匹配 `example.com` 和它的所有子域名（如 `www.example.com`、`api.example.com`）。一个服务往往用到好几个域名，可以在浏览器开发者工具的「网络」页里查看它实际访问了哪些域名。

### 如果你的配置来自订阅
订阅一更新，直接写进配置文件的内容就会被覆盖。两种办法：
- **改订阅源头**：如果订阅是你自己维护的（比如私密 Gist），就改源文件。
- **用客户端的「扩展配置」或「覆写」功能**：Clash Verge 里每个订阅都可以单独设置「节点 / 分组 / 规则」的追加。把 ① 加到「节点 → prepend」、② 加到「分组 → prepend」、③ 加到「规则 → prepend」，这样订阅更新后仍会保留。

## 3. 应用并确认成功
1. 保存配置，在 Clash Verge 里重新应用这个配置（或更新订阅）。
2. 「代理」页里应该多出「🏠 住宅出口」分组，默认选中「🏠 住宅IP」。点测速有延迟数字，说明整条链路是通的。
3. **确认出口确实是住宅 IP**：临时在 ③ 里加一行 `DOMAIN-SUFFIX,ipinfo.io,🏠 住宅出口`，重新应用后用浏览器打开 `https://ipinfo.io`：
   - ✅ 显示的是住宅 IP、运营商是宽带公司 → 成功
   - ❌ 显示的是你服务器的 IP → 规则没生效，检查是否放在了最前面
   
   确认完把这一行删掉。
4. 也可以在 Clash Verge 的「连接」页里看目标网站那条连接的链路，应该是 `🏠 住宅IP → 🚀 节点选择 → <实际节点>`。

## 4. 临时停用 / 撤销
- **临时停用**：在「🏠 住宅出口」里改选「🚀 节点选择」，这些网站就恢复走普通节点。
- **彻底撤销**：删掉加入的三段配置，重新应用即可。

## 5. 手机上能不能用
- Clash 系的安卓客户端（FlClash、Clash Meta for Android）同样支持 `dialer-proxy`，配置方法一样。
- NekoBox、v2rayNG 配置串联比较麻烦。需要手机也走住宅 IP 时，建议用[方案 B](方案B-服务端串联.md)，在服务器上统一处理。
