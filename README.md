# NeoHomeProxy

> HomeProxy 的增强分支 —— 在 [XiaoHaiSly/luci-app-homeproxy](https://github.com/XiaoHaiSly/luci-app-homeproxy)
> 之上加入 **可复用的节点组（Node Groups）**，并修复若干在部分设备上会导致「订阅一个都拉不到」的问题。

包名保持 `luci-app-homeproxy`，**可直接覆盖官方/上游包**（升级链完整，版本号形如 `2026.10.20-rNNN`）。

## 为什么需要节点组

上游的「分流规则」里，每条规则选择「独立 URLTest」时，节点清单是**挂在规则自己身上的**。
于是「Google 和 AI 都要走美国节点」就得把同一批美国节点在两条规则里各存一遍，
节点变动要改两处，规则一多就很容易漏。

节点组把这件事拆成两层：

- **组（group）**：一个命名节点集合（`美国组`、`日本组`…），成员节点只维护一份
- **规则**：分流规则的出口直接选组，多条规则可共用同一个组

最终生成的是一个**共享的 urltest 出站**（`group-<组名>-out`），所有引用该组的规则指向同一个 outbound。

## 新增功能：节点组

### 界面用法

1. 服务 → **NeoHomeProxy** → 客户端设置 → **Node Groups**（节点组）页签 → 添加
   - 填组名（如 `美国组`）
   - 勾选属于该组的节点
   - 可调测速间隔 / 容差
   - 保存应用
2. 同一页面 **Routing Rules**（分流规则）页签 → 编辑目标规则的「节点」下拉 →
   选 `Node group: 美国组` → 保存应用
3. 浏览器若没出现新页签，**强制刷新（Ctrl+F5）**。

### 命令行用法

```sh
G=$(uci add homeproxy group)
uci set homeproxy.$G.enabled='1'
uci set homeproxy.$G.label='美国组'
uci set homeproxy.$G.urltest_interval='120'
uci set homeproxy.$G.urltest_tolerance='40'
uci add_list homeproxy.$G.nodes='<节点 section id>'
# 规则指向该组
uci set homeproxy.@app_rule[1].node='group:美国组'
uci commit homeproxy
/etc/init.d/homeproxy restart
```

### 行为约定

- 每个组生成**一个** urltest 出站 `group-<组名>-out`；同一组的规则共享它。
- 组内成员失效（如订阅更新后节点 id 变化）会被自动剔除；组被清空时，引用它的规则回退到主节点。
- **改名必须同步改引用它的规则**，否则该规则会回退到主节点。

## 更新源（sing-box 核心）

界面里的「更新核心」默认从 **本仓库** 拉取，而不是上游的 `shtorm-7/sing-box-extended`：

| 项 | 值 |
| --- | --- |
| 默认更新源 | `xiaodandaodao/NeoHomeProxy` |
| 核心资产命名 | `sing-box-linux-<goarch>-musl.tar.gz` |
| 可覆盖 | 服务 → NeoHomeProxy → 服务状态 → 「更新源」（`owner/repo` 格式，留空即用默认） |

上游的更新器只认「`linux` + 架构 + `.tar.gz`」的资产，而本仓库的 CI 原本只发 OpenWrt 的
`apk`/`ipk` 包，所以额外做了两件事：

1. CI 在打完核心包后，把二进制抽出来打成 `sing-box-linux-<goarch>-musl.tar.gz` 一并发布；
2. 更新器改为**逐个 release 扫描**（本仓库同时存在 homeproxy 包与核心两种 release，
   只看最新一个会找不到核心资产）。

没改之前，界面上的「更新核心」对 NeoHomeProxy 用户是不工作的。

> **关于限流**：更新源走的是 GitHub REST API，未认证请求每个出口 IP 每小时只有 60 次。
> 家庭宽带 / 软路由多人共用出口时很容易触发 403（`API rate limit exceeded`），
> 此时在「服务状态」页填一个 **GitHub token** 即可（上限提升到 5000 次/小时）。

## 相对上游的改动

| 文件 | 改动 | 原因 |
| --- | --- | --- |
| `root/etc/homeproxy/scripts/generate_client.uc` | 新增 UCI 段类型 `group`：每个组生成共享 urltest 出站 `group-<组名>-out`；分流规则的出口支持 `group:<组名>` | 节点组核心 |
| `root/etc/homeproxy/scripts/homeproxy.uc` | `curlGET()` 去掉 `--compressed` | 见下 |
| `htdocs/luci-static/resources/homeproxy.js` | `reconcileUrltestNodes()` 增补 `group:` 白名单与组内失效节点清理 | 原逻辑会把不在节点表里的出口值打回 `main-out`，`group:` 会被误清 |
| `htdocs/luci-static/resources/view/homeproxy/client.js` | 新增 **Node Groups** 页签；分流规则的「节点」下拉追加 `Node group: <组名>` | 节点组界面 |
| `root/usr/share/luci/menu.d/luci-app-homeproxy.json` | 菜单标题改为 NeoHomeProxy | 品牌标识 |
| `po/zh_Hans/homeproxy.po`、`po/templates/homeproxy.pot` | 补 5 条新界面字符串的中文翻译 | i18n |

### `--compressed` 修复（重要）

`curlGET()` 原本固定拼接 `--compressed`：

```
/usr/bin/curl -fsSL --compressed --retry 3 --retry-all-errors ...
```

而 **OpenWrt 自带的 curl 7.83.1 编译时未带 zlib**，遇到不认识的选项会**直接 exit 2**：

```
curl: option --compressed: the installed libcurl version doesn't support this
```

`curlGET()` 在 exitcode ≠ 0 时返回 null，于是**所有订阅一个都进不来**
（与订阅格式、User-Agent、是否走代理都无关；换浏览器 UA 一样失败）。
本项目已移除该选项，兼容所有设备。

## 安装

从 [Releases](https://github.com/xiaodandaodao/NeoHomeProxy/releases) 下载 `luci-app-homeproxy_*_all.ipk`：

```sh
opkg install --force-reinstall luci-app-homeproxy_*.ipk
```

或走 Actions：推送到 `main` 触发 `Build luci-app-homeproxy`，产物在 Artifacts 里。

> 安装后如界面无变化，执行 `rm -f /tmp/luci-indexcache* && rm -rf /tmp/luci-modulecache/ && killall -HUP rpcd`，
> 浏览器再强制刷新。

## 与上游的差异

除上表所列外，功能与 [XiaoHaiSly/luci-app-homeproxy](https://github.com/XiaoHaiSly/luci-app-homeproxy) 一致：
DNS Fallback、XHTTP 支持、sing-box 面板、TUN 模式、应用分流预设（YouTube / TikTok / Telegram /
Twitter / Google / Cloudflare / GitHub / AI 服务）、在线更新核心等。

上游自带的增强（相对 immortalwrt/homeproxy）：

- 移除了 redirect / tproxy 等旧代理模式，统一使用 TUN（`auto_redirect`）
- 核心要求提升到 sing-box-extended ≥ 1.14.0

## 许可证与致谢

本项目以 GPL-2.0 协议发布，详见 [LICENSE](LICENSE)。

- [XiaoHaiSly/luci-app-homeproxy](https://github.com/XiaoHaiSly/luci-app-homeproxy)：本项目的直接上游
- [immortalwrt/homeproxy](https://github.com/immortalwrt/homeproxy)：HomeProxy 原始上游
- [shtorm-7/sing-box](https://github.com/shtorm-7/sing-box-extended)：代理核心
