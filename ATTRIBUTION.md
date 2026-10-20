# ATTRIBUTION

本文件是「本仓库相对上游改了什么」的唯一依据。上游更新时按此清单**按功能重放**，
而不是盯着 diff 猜。

## 上游

| 项 | 值 |
| --- | --- |
| 直接上游 | https://github.com/XiaoHaiSly/luci-app-homeproxy |
| 上游分支 | `main`（另有 `main-core`、`main-v1`） |
| 基线 commit | `ff9bb7c04560d79246b3b45f7fe76ab015bc22d0` |
| 基线 tag | `homeproxy-2026.09.28-r1`（`67538dff55b4e9908cee9006c016d982a81b4014`） |
| 原始上游 | https://github.com/immortalwrt/homeproxy |
| 代理核心 | https://github.com/shtorm-7/sing-box-extended |

基线 commit 同时记录在仓库根的 `.upstream-base`。

## 改动清单

### 1. `luci-app-homeproxy/root/etc/homeproxy/scripts/generate_client.uc`

**功能：节点组（Node Groups）**

- 新增常量 `const ucigroup = 'group';`
- 在应用分流（`app_rule`）处理之前插入「组」段遍历：每个 `group` 段生成**一个共享的
  urltest 出站** `group-<label>-out`，成员节点对应的 outbound 同步补齐；
  建立 `group_outbounds[label] = tag` 映射。
- 分流规则的出口分派中新增分支：出口值形如 `group:<label>` 时，指向该共享出站；
  组不存在时回退 `main-out`。

**为什么**：上游每条规则的「独立 URLTest」节点清单挂在规则自己身上，同一批节点要在多条
规则里各存一遍。组把「节点集合」抽成独立实体后，多条规则可共享同一个 outbound。

### 2. `luci-app-homeproxy/root/etc/homeproxy/scripts/homeproxy.uc`

**功能：修复「所有订阅都拉不到」**

- `curlGET()` 里的 `-fsSL --compressed --retry` 改为 `-fsSL --retry`。

**为什么**：OpenWrt 自带 curl 7.83.1 未编 zlib，不支持 `--compressed`，遇到该选项直接
exit 2 → `curlGET()` 返回 null → 所有订阅导入失败（与格式、UA、代理均无关）。

> 注意：这是**通用修复**，不要因为自己设备的 curl 支持 zlib 就改回去。
> 若要保守处理，可改为「检测 curl 是否支持 --compressed 再决定是否加」。

### 3. `luci-app-homeproxy/htdocs/luci-static/resources/homeproxy.js`

**功能：节点组的失效清理与白名单**

- `reconcileUrltestNodes()` 中新增 `group` 段的成员清理：剔除已不存在的节点 id；
  组非空则记入 `groupMembers`。
- 应用分流的出口值判断新增 `group:` 分支：组不存在或已空 → 回退 `main-out`。

**为什么**：原逻辑会把「不在节点表里的出口值」一律重置为 `main-out`，
新增的 `group:<label>` 会被误判为失效而被清掉。

### 4. `luci-app-homeproxy/htdocs/luci-static/resources/view/homeproxy/client.js`

**功能：节点组界面**

- 读取 `group` 段构造 `node_groups` 映射（组名 → 显示名）。
- 新增 **Node Groups** 页签（GridSection，段类型 `group`）：组名、成员节点多选
  （`hp.CBIStaticList`）、测速间隔 / 容差 / 是否中断现有连接。
- 应用分流的「节点」下拉追加 `group:<组名>` 选项。

### 5. `luci-app-homeproxy/root/usr/share/luci/menu.d/luci-app-homeproxy.json`

- 顶层菜单标题 `HomeProxy` → `NeoHomeProxy`（仅品牌标识，便于与官方包区分）。

### 6. `luci-app-homeproxy/po/zh_Hans/homeproxy.po`、`po/templates/homeproxy.pot`

- 追加 5 条新界面字符串（Node Groups / Group name / Nodes / Nodes belonging to this
  group. / Reusable node group… / Node group: %s），po 给中文译文，pot 留空。

**注意**：一律**追加**到文件末尾，不要整体重排 —— 保持 diff 最小，方便与上游合并。

### 7. `.github/build-ipk.sh`

- `PKG_URL` / `PKG_MAINTAINER` / `PKG_DESC` 指向本仓库（保留上游致谢信息）。

### 8. `luci-app-homeproxy/root/etc/homeproxy/scripts/homeproxy.uc`  ← **最关键的一处**

**功能：节点组在「订阅更新」时不被误判为失效**

- `reconcileUrltestNodes()` 中新增 `group` 段的成员清理与登记（`groupMembers`）。
- 应用分流规则的出口判定新增 `group:` 分支：组存在且非空 → 保留；组不存在或被清空 →
  回退 `main-out` 并写日志。

**为什么**（这是实测踩到的坑）：本项目里有**两份** reconcile 实现——

| 文件 | 运行环境 | 是否被调用 |
| --- | --- | --- |
| `htdocs/luci-static/resources/homeproxy.js` | 浏览器 / LuCI 界面 | 本 fork 中**无调用点**（只有定义） |
| `root/etc/homeproxy/scripts/homeproxy.uc` | 路由器 / ucode | **订阅更新时会被调用**（`update_subscriptions.uc` 末尾） |

原逻辑里「出口值不在节点表里 → 打回 `main-out`」这一条会把 `group:<组名>` 误判为失效。
只补 JS 版的话，**界面里看着一切正常，一跑订阅更新规则就悄悄回落到主节点**——
必须补 ucode 版。

> 教训：改动前先确认「这段逻辑到底在哪个文件、哪条链路上执行」。
> 浏览器端和设备端各有一份同名函数时，两边都要看。

### 9. 更新源改为本仓库

**功能：界面「更新核心」从 NeoHomeProxy 拉取 sing-box 核心**

- `root/etc/homeproxy/scripts/homeproxy.uc`：
  - 新增常量 `CORE_REPO_NEO = 'xiaodandaodao/NeoHomeProxy'`；
  - 新增 `coreRepo()`：读 UCI `config.core_repo`，为空则用默认仓库；
  - 新增 `coreFetchReleaseList()` / `coreFindAsset()`：**逐个 release 扫描**，
    找带「linux + 架构 + tar.gz」资产的第一个（优先 musl）。
- `root/etc/homeproxy/scripts/update_core.uc`：改用 `coreRepo()` + `coreFindAsset()`；
  版本号解析兼容本仓库的 `sing-box-v<version>` 标签。
- `htdocs/luci-static/resources/view/homeproxy/status.js`：
  新增「更新源」展示行与可编辑项（`config.core_repo`）。
- `.github/workflows/build-sing-box.yml`：打完核心包后把二进制抽出来，
  额外发布 `sing-box-linux-<goarch>-musl.tar.gz`（矩阵新增 `go_arch` 字段）。

**为什么**：原实现写死 `shtorm-7/sing-box-extended`，且只在**最新一个** release 里找资产。
本仓库的 release 是混装的（homeproxy 包 + sing-box 核心），两处都要改才能对上。

**注意**：资产名必须用 **Go 架构**（x86_64 → `amd64`），因为更新器按 Go 架构匹配，
不是按 OpenWrt 架构。

## 同步上游的流程

1. `git fetch upstream && git log upstream/main --oneline -1` 看上游进展。
2. `git checkout upstream/main && git rebase main`（或 `git merge upstream/main`）——
   本仓库与上游共享历史，冲突只会出现在上面 7 处；按本清单**按功能重放**。
3. 冲突裁决原则：
   - 上游修复 → **采纳上游**（例如上游哪天给 curlGET 加了压缩探测，就用上游的）
   - 本项目的节点组能力 → **保留**
   - 版本号 / 品牌标识 → **永远用本仓库的**
4. 合并后重新校验并出包：
   ```sh
   node --check luci-app-homeproxy/htdocs/luci-static/resources/view/homeproxy/client.js
   msgfmt --check -o /dev/null luci-app-homeproxy/po/zh_Hans/homeproxy.po
   ```
5. 推 `main` → Actions 自动出 ipk/apk；打 tag 发 Release。
