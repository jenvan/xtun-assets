# `pages-assets/` —— 主仓库这边与资源站相关的文件

> ⚠️ **这里不是资源站的完整内容** —— 只是「由主仓库维护、需要同步过去」的那一部分。
> 站点其余内容在独立仓库 [`xtun-assets`](https://github.com/jenvan/xtun-assets)。

## 谁在哪

| 东西 | 位置 |
|---|---|
| `index.html`（站点首页 / **网关未登录首页的基底**）| **本目录** → 由 CI 同步到 `xtun-assets` |
| `rdp.html`（RDP 页面，**唯一源**）| **本目录** → 由 CI 同步到 `xtun-assets` |
| `ssh.html`（Web SSH 终端页面，**唯一源**）| **本目录** → 由 CI 同步到 `xtun-assets` |
| 站点其余内容（noVNC 全套 / IronRDP 产物 / `ghostty-web.js` / `CNAME`）| **`xtun-assets` 仓库**（GitHub Pages 的站点根，发布在 `assets.xtun.dev`）—— ⚠️ **第三方产物由人手工上传，CI 不管** |
| **`pins.json` + `pins.json.sig`**（**钉死表**：7 个入口文件的 sha256，带 Ed25519 签名）| ⚠️ **两边都不入库** —— CI **现场生成并签名**后直接上传（见下） |
| 网关 Worker（`x-edge.js`）| 本仓库根目录 |

## 怎么同步

[`.github/workflows/sync-assets.yml`](../.github/workflows/sync-assets.yml)：

- **触发**：push main 且动了 `pages-assets/**`（日常走这条）／`release: published`（兜一次）／手动
- **清单**：`git ls-files pages-assets/` —— 往本目录加文件并提交即可同步，**不用改 workflow**
  - ⚠️ **外加两个 CI 现场生成的文件**：`pins.json` 与 `pins.json.sig`。
    它们**不在** `git ls-files` 清单里，workflow 里是**显式拷过去**的 ——
    改动那一段时别把这两行删了，否则会**静默漏传**（现象：两端拉不到签名表 →
    拒绝出页面，或退回旧表，看起来一切正常）
- **签名**：`node scripts/pin-assets.mjs --sign`（用 GitHub Secret `PINS_SIGNING_KEY`，
  版本号 = `github.run_number`）。私钥**只在** Secret 里，本地开发/换电脑都不需要接触。
  ⚠️ **取字节的来源不是"一律线上"**（2026-10-11 改）：
  **已入库、由本 workflow 上传的页面**（`index.html` / `rdp.html` / `ssh.html`）按
  **本目录的本地文件**签，未入库的第三方产物才按线上签。原因是 CI 的步骤顺序是
  「先签名、后上传」——一律从线上取会签出**上一版**内容（线上新页面 + 旧表 =
  哈希不符、两端拒绝出页面），新增被钉死的页面时更会因线上 404 而**卡死**。
  回归测试 `npm run test:pinsign`（见 `scripts/test-pin-sign.mjs`）
- ⚠️ **`pins.json` 是端上唯一的钉死表来源**（代码里没有表）：缺了它或者签名对不上，
  Worker 与 relay 都会**拒绝出页面**（fail-closed，唯一开关 `ASSET_PIN_ENFORCE=0`）
- ✅ **本目录整体入库**（`.gitignore` 里故意不写 `pages-assets/*`）：往这里放什么都会
  自动带上，不必逐个补例外
  - ❌ 旧的写法是「整目录忽略 + 逐个补 `!` 例外」，漏补一条 = **静默不同步**
    （文件在本地、线上永远没有、**没有任何报错**）—— 2026-09-26 踩过，还误以为是
    workflow 坏了
  - ⚠️ 例外现在**反过来列**：**第三方产物被显式排除**（`core/` `app/` `vendor/` `po/`
    `tests/` `utils/` `docs/` `snap/`、两个 IronRDP js、`ghostty-web.js`，以及
    `vnc.html`/`vnc_lite.html`）—— 别把整套 noVNC 拷回本目录，`git add -A` 会吞掉它。
    ⚠️ **被排除的不等于不用管**：它们由**人手工上传**，且大部分还在钉死清单里 ——
    见下节
- ⚠️ workflow 另有一道 **2 MB 体积护栏**：单个文件超限就**明确报错**，不静默推送
- ⚠️ **只覆盖同名文件、不删除** `xtun-assets` 里的其他文件
- ⚠️ 同步前会校验两个页面各自的标记与占位符，缺一个就**拒绝同步**
  （宁可不同步，也不让线上页面变白）：
  `rdp.html` 要 `<!-- xtun-rdp -->` / `__RDP_WS_PATH__` / `__RDP_DEVICE__` / `__ASSETS__`；
  `ssh.html` 要 `<!-- xtun-ssh -->` / `__SSH_WS_PATH__` / `__SSH_DEVICE__` / `__ASSETS__`，
  并且必须从 `"__ASSETS__/ghostty-web.js"` import 终端库（写相对路径 = 白屏）
- 需要 secret `XTUN_ASSETS_TOKEN`（对 `xtun-assets` 有 Contents: Read and write）

## 新增一个被钉死的资源：引导顺序（⚠️ 顺序错了 CI 会红）

钉死清单在 [`scripts/pin-assets.mjs`](../scripts/pin-assets.mjs) 的 `ASSETS`。
加一项之前先分清它属于哪一类：

| 类别 | 例子 | 怎么上线 |
|---|---|---|
| **已入库、CI 上传** | `index.html`、`rdp.html`、`ssh.html` | 放进本目录、`git add`、推 main —— 同步与签名都归 CI，**不用手工传** |
| **不入库的第三方产物** | `iron-remote-desktop*.js`、`ghostty-web.js`、`vnc.html` | ⚠️ **必须【先】手工上传到 `xtun-assets` 的站点根**，否则 CI 的签名步骤取不到它（404、fail-closed）→ **同步步骤永远跑不到** |

⚠️ 第二条是**硬顺序**：CI 只同步 `pages-assets/` 里**已入库**的文件，
第三方产物它**没有、也传不了**（体积/许可原因不入库）。所以新增一个第三方被钉死资源时：

```
① 下载产物 → ② 手工上传到 xtun-assets 站点根 → ③ 把路径加进 ASSETS 并推 main
```

③ 之前先确认线上真的取得到：
`curl -sI https://assets.xtun.dev/ghostty-web.js | head -1`。

> 🔎 顺序颠倒的**症状**：`sync-assets.yml` 在「生成并签名 pin 表」这一步失败，
> 报错里带一句「首次引导顺序见 pages-assets/README.md」（`fetchBytes` 的 404 提示）。
> ⚠️ 它**不会**悄悄跳过这个文件 —— 表里少一项就等于**那一项不再被校验**，
> 所以这里是硬失败，不是告警。

## 改了怎么验

```bash
npm run test:rdp        # 本仓库就能跑：断言 rdp.html 的标记与占位符
npm run test:ssh        # 同上，判据针对 ssh.html（含 ghostty-web 的 import 路径）
npm run test:pinsign    # 钉死值来源：入库页面按本地文件签（真跑子进程 + 假资源站）
npm run check:deploy    # 对线上做 sha256 比对：一眼看出「同步没跑成」还是「线上被改过」
npm run check:assetpins # 核对「线上资源」与「线上 pins.json」是否一致（需联网）
```

⚠️ 改 `rdp.html` / `ssh.html` / `index.html` **只改本目录这份**（它们就是唯一源；
直接改 `xtun-assets` 里那份会被下次同步覆盖掉）。
⚠️ 新增/改动了钉死资源后，`npm run check:assetpins` 要等 CI 跑完再核（它比的是**线上**）。

## 改 `index.html` 时的两条约束

它不只是资源站首页，**也是网关未登录首页的基底** —— 网关会把它取回、注入
「双击 / 长按 → `/login`」的脚本再返回（见 `x-edge.js` 的 `injectLoginGesture`）：

1. ⚠️ **不要在 `document` 上再占用 `dblclick`**（那个手势已被注入的脚本用掉）。
2. ⚠️ **链接只用绝对 URL（`https:` 开头）或 `#` 锚点** —— Worker 返回前会跑
   `rewriteAssetPaths` 把相对引用改写成资源站地址。
