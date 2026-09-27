# `pages-assets/` —— 主仓库这边与资源站相关的文件

> ⚠️ **这里不是资源站的完整内容** —— 只是「由主仓库维护、需要同步过去」的那一部分。
> 站点其余内容在独立仓库 [`xtun-assets`](https://github.com/jenvan/xtun-assets)。

## 谁在哪

| 东西 | 位置 |
|---|---|
| `index.html`（站点首页 / **网关未登录首页的基底**）| **本目录** → 由 CI 同步到 `xtun-assets` |
| `rdp.html`（RDP 页面，**唯一源**）| **本目录** → 由 CI 同步到 `xtun-assets` |
| 站点其余内容（noVNC 全套 / IronRDP 产物 / `CNAME`）| **`xtun-assets` 仓库**（GitHub Pages 的站点根，发布在 `assets.xtun.dev`）|
| 网关 Worker（`x-edge.js`）| 本仓库根目录 |

## 怎么同步

[`.github/workflows/sync-assets.yml`](../.github/workflows/sync-assets.yml)：

- **触发**：push main 且动了 `pages-assets/**`（日常走这条）／`release: published`（兜一次）／手动
- **清单**：`git ls-files pages-assets/` —— 往本目录加文件并提交即可同步，**不用改 workflow**
- ✅ **本目录整体入库**（`.gitignore` 里故意不写 `pages-assets/*`）：往这里放什么都会
  自动带上，不必逐个补例外
  - ❌ 旧的写法是「整目录忽略 + 逐个补 `!` 例外」，漏补一条 = **静默不同步**
    （文件在本地、线上永远没有、**没有任何报错**）—— 2026-09-26 踩过，还误以为是
    workflow 坏了
  - ⚠️ 例外现在**反过来列**：**第三方产物被显式排除**（`core/` `app/` `vendor/` `po/`
    `tests/` `utils/` `docs/` `snap/`、两个 IronRDP js，以及 `vnc.html`/`vnc_lite.html`）
    —— 别把整套 noVNC 拷回本目录，`git add -A` 会吞掉它
- ⚠️ workflow 另有一道 **2 MB 体积护栏**：单个文件超限就**明确报错**，不静默推送
- ⚠️ **只覆盖同名文件、不删除** `xtun-assets` 里的其他文件
- ⚠️ 同步前会校验 `rdp.html` 的四个标记（`<!-- xtun-rdp -->`、`__RDP_WS_PATH__`、
  `__RDP_DEVICE__`、`__ASSETS__`）—— 缺了就**拒绝同步**，宁可不同步也不让线上页面变白
- 需要 secret `XTUN_ASSETS_TOKEN`（对 `xtun-assets` 有 Contents: Read and write）

## 改了怎么验

```bash
npm run test:rdp        # 本仓库就能跑：断言 rdp.html 的标记与占位符
npm run check:deploy    # 对线上做 sha256 比对：一眼看出「同步没跑成」还是「线上被改过」
```

⚠️ 改 `rdp.html` / `index.html` **只改本目录这份**（它们就是唯一源；
直接改 `xtun-assets` 里那份会被下次同步覆盖掉）。

## 改 `index.html` 时的两条约束

它不只是资源站首页，**也是网关未登录首页的基底** —— 网关会把它取回、注入
「双击 / 长按 → `/login`」的脚本再返回（见 `x-edge.js` 的 `injectLoginGesture`）：

1. ⚠️ **不要在 `document` 上再占用 `dblclick`**（那个手势已被注入的脚本用掉）。
2. ⚠️ **链接只用绝对 URL（`https:` 开头）或 `#` 锚点** —— Worker 返回前会跑
   `rewriteAssetPaths` 把相对引用改写成资源站地址。
