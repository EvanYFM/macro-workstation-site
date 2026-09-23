# CI 门禁补丁方案

> 配套主文档：`CODE_REVIEW.md`
> **本版已用 GitHub 最新代码复核并实测验证（2026-09-23）**，替换基于 7 天前快照的第一版。

**核心原则**：先让守护进入执行路径，再给它否决权，最后才加新检查。顺序反了就是往一条没人走的路上再装几个新路标。

---

## 0 前提修正（第一版的判断已失效）

| 第一版写道 | 实测结果 |
|---|---|
| 「本机封 `github.com:443`，`publish-pages.mjs` 不可用」 | ❌ **已失效**。实测 `https://github.com` → HTTP 200；`git ls-remote`、`git push --dry-run` 均正常 |
| 「发布守护只在 `npm run publish` 内生效，靠人记得跑」 | ⚠️ 部分对。真实原因是**脚本本身坏了**（见 P0-1），不是网络 |
| 「`macro-workstation-site` 无 deploy 门禁」 | ✅ 仍成立。站点仓仍只有 `site-check.yml` |

**教训**：`docs/data-update-guide.md` 第五节把"本机网络限制"写成了现行规则，而该限制已解除、文档未更新 → 一条无守护的 REST 手工旁路被当成了默认流程。**过期约束被当作事实，本身就是一个待审的文档缺陷。**

---

# P0-1 【已完成并验证】恢复 `publish-pages.mjs`

这是本次审计中**最高价值的一项**：发布路径断掉的真正原因。

### 实测报错

```
$ node scripts/publish-pages.mjs --no-push
[ci-check-pages] OK · data 2026-09-22 · ui=garden · history 12 份
Error: Not in public artifact allowlist: .gitignore
    at walk (publish-guards.mjs:13:29)
    at async validatePublicTree (publish-guards.mjs:18:4)
    at async main (publish-pages.mjs:75:3)
```

### 根因（两个问题互相掩盖）

| # | 问题 | 后果 |
|---|---|---|
| 1 | `publish-guards.mjs:9` 白名单不含 `.gitignore`（站点仓 09-21 为忽略 `.git.corrupt-*` 引入，commit `056eb0d`） | `validatePublicTree(publicDir)` 直接抛错，脚本在 mirror 前退出 |
| 2 | 即使放过，`publish-pages.mjs` 的 mirror 步骤 `rm` 掉除 `.git` 外所有条目，而 `.gitignore` 不在 `dist/pages/` 中 | `.gitignore` 被删 → 下次运行又"缺文件" → 两个问题交替掩盖，永不稳定 |

枚举验证：站点仓 43 个追踪文件中**恰好 1 个**不匹配白名单（`.gitignore`），未追踪文件 0 个。

### 已应用的改动（2 处）

**① `scripts/publish-guards.mjs:9`** — 白名单加入 `.gitignore`（在 `\.nojekyll|` 之后插入 `\.gitignore|`）

**② `scripts/publish-pages.mjs`** — mirror 步骤保留仓库级文件

```js
// 改前
  // 2. Mirror: remove everything except .git, then copy the fresh build.
  for (const entry of await readdir(publicDir)) {
    if (entry === ".git") continue;
    await rm(path.join(publicDir, entry), { recursive: true, force: true });
  }

// 改后
  // 2. Mirror: remove everything except repo housekeeping files, then copy the fresh build.
  //    .gitignore must be preserved: it is not part of dist/pages, so the mirror would delete
  //    it every run — and validatePublicTree would then reject the next run for a missing
  //    allowlist entry (2026-09-23: this broke `npm run publish` end to end).
  const preserve = new Set([".git", ".gitignore"]);
  for (const entry of await readdir(publicDir)) {
    if (preserve.has(entry)) continue;
    await rm(path.join(publicDir, entry), { recursive: true, force: true });
  }
```

### 验证结果（实测）

```
$ node scripts/publish-pages.mjs --no-push
[ci-check-pages] OK · data 2026-09-22 · ui=garden · history 12 份
[publish-pages] commit 9189755 not pushed (dry run)
```

通过项：`ci-check-pages`（DOM 级断言）· `validatePublicTree` 构建产物 · `validatePublicTree` 公开仓 · `assertHistory` · `assertClean` · 远端 fetch + pull --rebase · 禁入目录守卫。
产物相对线上仅 `runId` / `generatedAt` 两字段变化（每次构建必然变化，属设计行为）。
`.gitignore` 存活确认：内容 `.git.corrupt-*/`。

### 建议的后续（需你确认）

`docs/data-update-guide.md` 第五节的「本机网络限制下的发布路径」**降级为备用**，把 `npm run publish` 恢复为默认，并在该节加一行：

```markdown
> 备用路径（仅当 `github.com` 再次不可达时使用）：见本文件「REST Git Data API 手工发布」。
```

**理由**：`publish-pages.mjs` 自带全部四道守护（白名单 / 凭证扫描 / 历史保护 / tree 一致性），REST 手工路径**一道都没有**。把有守护的路径恢复为默认，是这批改动里性价比最高的一件事。

---

# P0-2 让已写好的发布门禁真正生效

**问题（实测，比"缺门禁"更严重）**：

| 仓库 | Pages 方式 | deploy.yml | 最近运行结论 |
|---|---|---|---|
| `macro-workstation-site` | `legacy` / `main/` | **不存在** | — |
| `futures-workstation` | `legacy` / `main/` | 有，`needs: build` | **completed / skipped** |
| `digital-garden` | `legacy` / `main/` | 有，`needs: build` | **completed / skipped** |

三个仓的 `PAGES_DEPLOY_ENABLED` 仓库变量**都不存在**。而两个 `deploy.yml` 的 build job 都带：

```yaml
  build:
    if: vars.PAGES_DEPLOY_ENABLED == 'true'
```

变量不存在 → 条件为假 → build 被跳过 → `deploy: needs: build` 也被跳过 → **整个 workflow 每次都是 `completed/skipped`**。

**结论：`futures-workstation` 和 `digital-garden` 的发布门禁是写了但从未生效的死代码，保护力为零。** 三个站点的 Pages 都是 `legacy` 分支直发 —— `push` 即上线。

> 这比"没有门禁"更危险：**它看起来有门禁**。任何人事后翻 `.github/workflows/` 都会以为发布已被把关。

**修法分两种情况**：

| 仓库 | 动作 |
|---|---|
| `futures-workstation`、`digital-garden` | **激活即可** —— `deploy.yml` 已正确，只需设置变量 + 切换 Pages 来源 |
| `macro-workstation-site` | **补一个 `deploy.yml`**（照抄 `digital-garden` 的，含 build / deploy 两 job）+ 设置变量 + 切换 Pages 来源 |

### 步骤 1 · 设置 `PAGES_DEPLOY_ENABLED`（三个仓都要）

```bash
for r in macro-workstation-site futures-workstation digital-garden; do
  gh variable set PAGES_DEPLOY_ENABLED --body true --repo "EvanYFM/$r"
done
```

验证：

```bash
gh api repos/EvanYFM/digital-garden/actions/variables --jq '.variables[] | .name + "=" + .value'
```

> 这个变量被设计成"总开关"，好处是出问题时可以立刻置 `false` 停掉 Actions 部署。**代价是它默认关着，而没人记得打开 —— 这就是这批门禁集体失效的原因。** 建议在设置它的同一步，把"变量已设为 true"写进 `AGENTS.md`，否则下次重建仓库会重演。

### 步骤 2 · `macro-workstation-site` 补 `deploy.yml`

照抄 `digital-garden/.github/workflows/deploy.yml` 的结构（已验证可用），把校验步骤换成 macro 站点的断言 —— 即源码仓 `garden/site-check.yml` 里那套。为此在源码仓新增模板 `garden/deploy.yml`：

```yaml
name: verified-pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: verified-pages
  cancel-in-progress: true

jobs:
  build:
    if: vars.PAGES_DEPLOY_ENABLED == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11d5960a326750d5838078e36cf38b85af677262
      - uses: actions/setup-node@49933ea5288caeca8642d1e84afbd3f7d6820020
        with:
          node-version: '22'

      # 与 site-check.yml 同源的断言，在发布前执行 —— 本补丁的核心
      - name: Validate before publishing
        run: |
          node <<'EOF'
          const { readFileSync, existsSync, readdirSync } = require('fs');
          function assert(c, m) { if (!c) { console.error('pre-publish FAIL: ' + m); process.exit(1); } }
          const html = readFileSync('index.html', 'utf8');
          const manifest = JSON.parse(readFileSync('run-manifest.json', 'utf8'));
          assert(!html.includes('fonts.googleapis'), 'render-blocking Google Fonts found');
          assert(!html.includes('fonts.gstatic'), 'fonts.gstatic reference found');
          assert(!html.includes('__SNAP__'), 'unreplaced __SNAP__ placeholder');
          assert(!html.includes('__HEALTH__'), 'unreplaced __HEALTH__ placeholder');
          assert(/property="og:image" content="https:\/\//.test(html), 'og:image is not an absolute URL');

          // 产物结构 + 渲染运行时断言（防 65415de 类白屏事故复发）
          const snap = html.match(/<script type="application\/json" id="snap">([\s\S]*?)<\/script>/);
          assert(snap, 'snap data script missing');
          JSON.parse(snap[1].replaceAll('<\\/', '</'));
          const js = html.match(/<script>([\s\S]*?)<\/script>\s*<\/body>/);
          assert(js, 'inline app script missing');
          try { new Function(js[1]); } catch (e) { assert(false, 'inline script syntax error: ' + e.message); }

          const latest = JSON.parse(readFileSync('data/latest.json', 'utf8'));
          assert(manifest.checkedAt === latest.asOf, 'manifest.checkedAt != data/latest.json asOf');
          const hist = readdirSync('data/history').filter(f => f.endsWith('.json') && f !== 'index.json').sort();
          assert(JSON.stringify(manifest.historyDates) === JSON.stringify(hist.map(f => f.replace(/\.json$/, ''))), 'history out of sync with manifest');
          for (const d of manifest.historyDates) assert(existsSync('reports/' + d + '.md'), 'missing report for ' + d);
          assert(existsSync('.nojekyll') && existsSync('404.html') && existsSync('robots.txt'), 'missing pages plumbing');
          console.log('pre-publish OK · data ' + manifest.latestDate + ' · ui=' + manifest.ui);
          EOF

      - uses: actions/upload-pages-artifact@56afc609e74202658d3ffba0e8f6dda462b719fa
        with:
          path: .

  deploy:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/deploy-pages@d6db90164ac5ed86f2b6aed7e0febac5b3c0c03e
        id: deployment
```

**两条断言已实测确认**：

- `!html.includes('__HEALTH__')` — 该占位符在构建产物中实测出现 **0 次**（`build-static.mjs` 已替换），安全
- `manifest.checkedAt === latest.asOf` — 实测两者均为 `2026-09-22T18:03:14+08:00`，相等。且**这本就是站点仓现有 `site-check.yml` 在用的断言**，非新造

### 步骤 3 · `build-static.mjs` 生成该文件

在 `scripts/build-static.mjs` 第 242-248 行那段（`// Deploy-repo guard`）之后追加：

```js
  await writeFile(
    path.join(pagesDir, ".github", "workflows", "deploy.yml"),
    await readFile(path.join(gardenDir, "deploy.yml"), "utf8"),
  );
```

### 步骤 4 · 放行白名单（`publish-guards.mjs:9`）

不带上这一步，publish 会被自己的守护拒绝。把 `\.github\/workflows\/site-check\.yml|` 改为：

```
\.github\/workflows\/(?:site-check|deploy)\.yml|
```

### 步骤 5 · `ci-check-pages.mjs` 补断言

在现有断言（第 101-104 行）之后追加：

```js
const deployYml = path.join(pagesDir, ".github", "workflows", "deploy.yml");
assert(existsSync(deployYml), "deploy-repo deploy.yml not emitted");
const dy = readFileSync(deployYml, "utf8");
assert(/deploy:\s*\n\s*needs:\s*build/.test(dy), "deploy.yml lost its needs:build gate");
```

这条断言本身就是围栏：防止后续有人删掉 `needs: build` 让门禁悄悄失效。

### 步骤 6 ·（手动）切换 Pages 来源

见文末「Pages 来源切换：具体怎么点」。

> ⚠️ **顺序不可颠倒**：先设置变量、推 `deploy.yml`、**确认 `verified-pages` 成功跑过一次**，最后才切 Pages 来源。否则切换后没有成功产物，**站点会 404**。

# P0-3 `pre-push` 钩子（仅覆盖源码仓）

**问题**：仓库本地 `.git/hooks/` 只有 `*.sample`，push 前零检查。

⚠️ **边界必须说清**：这个钩子只对**源码仓**（`macro-workstation` / `trading-system` / `digital-garden`）的普通 `git push` 有意义。**对 REST API 发布路径完全无效**（钩子不触发）。它是补充，不是替代。

### 安装（每个源码仓执行一次）

```bash
mkdir -p .githooks
git config core.hooksPath .githooks
```

### `.githooks/pre-push`（含 L1 快速通道）

```sh
#!/bin/sh
# L1 改动（文档/样式/日更数据）跳过重型检查，保持每日数据更新流程的速度
set -e

changed=$(git diff --name-only origin/main...HEAD 2>/dev/null || true)
if [ -z "$changed" ]; then exit 0; fi

heavy=$(printf '%s\n' "$changed" | grep -vE '^(docs/|data/|output/|public/data/|public/reports/|.*\.md$|.*\.css$)' || true)
if [ -z "$heavy" ]; then
  echo "[pre-push] 仅 L1 改动，跳过重型检查"
  exit 0
fi

echo "[pre-push] 检测到 L2/L3 改动，运行门禁…"

# ---- macro-workstation ----
if [ -f package.json ] && grep -q '"check:pages"' package.json 2>/dev/null; then
  npm run lint
  npx tsc --noEmit
  npm test
fi

# ---- trading-system / Python 仓 ----
if [ -d scripts ] && ls scripts/*.py >/dev/null 2>&1; then
  python3 -m py_compile scripts/*.py
  command -v ruff >/dev/null 2>&1 && ruff check scripts/ tests/ || true
  [ -d tests ] && python3 -m unittest discover -s tests -p 'test_*.py' || true
fi

# ---- 通用：JS 语法 ----
for f in app.js history-store.js journal-sync.js base.js; do
  [ -f "$f" ] && node --check "$f"
done

echo "[pre-push] 门禁通过"
```

> Windows 上 `chmod +x` 对 NTFS 无效，Git Bash 依靠 shebang 执行。若报权限错误，改用 `pre-commit` 框架。

---

# P1-1 让 CI 的测试列表与 `npm test` 同源

**问题（实测）**：

| 来源 | 测试文件数 |
|---|---|
| `package.json` 的 `npm test` | **5** |
| `.github/workflows/check.yml` | **3** |

缺失：`tests/futu-quote.test.mjs`（155 行）、`tests/rendered-html.test.mjs`（16KB）→ **这两个测试在 CI 中从不运行**。

**修法**：把 `check.yml` 那一步改为调用 npm 脚本，而不是手工列举文件名。

```yaml
      - name: 数据与发布安全回归
        run: npm test
```

以后加测试只需改 `package.json`，不会再漂移。**根因是同一份逻辑写了两处** —— 这正是审查该拦下的那类问题。

---

# P1-2 `macro-workstation` 接入 lint 与 type

**问题**：有 `eslint.config.mjs`、TS 5.9、`strict: true`、`npm run lint`，但 `check.yml` 从不调用它们。

### 先量基线（**不要直接开**）

```bash
cd macro-workstation
npx tsc --noEmit 2>&1 | tail -30
npm run lint 2>&1 | tail -30
```

记下错误数。若为 0，按下面接入；若不为 0，先决定修掉还是建基线白名单。

### `check.yml` 追加（插在 `Build static site` **之前** —— 最便宜的检查应该最先失败）

```yaml
      - name: Lint
        run: npm run lint

      - name: TypeScript 类型检查
        run: npx tsc --noEmit
```

需要先 `npm ci`。注意现有 build 步骤是 "zero deps" 轨道，加依赖安装会拉长约 30-60 秒，可接受。

> **棘轮策略**：**不要**用 `continue-on-error: true`（那是假装有门禁）。若基线有存量错误，落成基线文件，只失败于新增错误。

---

# P1-3 `trading-system` 补 Python 静态检查

```bash
cd trading-system-fresh
python3 -m pip freeze | grep -iE '^(lxml|pandas|requests|beautifulsoup4|openpyxl|numpy)' > requirements.txt
```

`ci.yml` 改为：

```yaml
      - name: 安装测试依赖
        run: python3 -m pip install -r requirements.txt ruff==0.6.9

      - name: Python lint（ruff）
        run: ruff check scripts/ tests/
```

> ruff 起步用最小规则集，避免第一天刷出几百条告警被无视。必要时加 `pyproject.toml`：
> ```toml
> [tool.ruff.lint]
> select = ["E", "F"]
> ```
> 稳定后再加 `I`（import 排序）、`B`（bugbear）。

---

# P1-4 测试命名导致被静默跳过

```bash
cd trading-system-fresh
git mv tests/verify_tdx_converters.py tests/test_verify_tdx_converters.py
```

不匹配 CI 的 `-p 'test_*.py'` 通配符 → **这个测试从未在 CI 中运行**。**有测试文件、永远不跑、给人安全错觉，比没有测试更危险。**

---

# P2 统一 `futures` 源头

**实测事实**：`futures-workstation-deploy` **不是 git 仓库**，只是 8 文件的陈旧本地副本（`README.md` `app.js` `data/` `history-store.js` `index.html` `journal-sync.js` `run-manifest.json` `styles.css`），与现役公开仓 `EvanYFM/futures-workstation`（本地 `futures-workstation-deploy2`）并存、无来源标记。

这与 `macro-workstation` 已确立的双仓架构同构：

```
trading-system（私有源，唯一真相源）  →  futures-workstation（公开仓，只含产物，禁止直接修改）
```

**建议动作（涉及删除，需你确认）**：

1. 先确认 deploy 里没有独有内容：
   `diff <(git show HEAD:app.js) ../futures-workstation-deploy/app.js | head -50`（在 `futures-workstation-deploy2` 内执行）
2. 若确认无独有内容 → **归档或删除** `futures-workstation-deploy`
3. 在公开仓 README 首行写明「本仓由 `trading-system` 发布，**禁止直接修改**」

---

# Pages 来源切换：具体怎么点

这是 P0-2 的第 5 步，也是唯一必须手动完成的一步。

### 前置条件（务必先满足，缺一个都会 404）

| # | 条件 | 怎么确认 |
|---|---|---|
| 1 | `PAGES_DEPLOY_ENABLED` 变量已设为 `true` | `gh api repos/EvanYFM/<repo>/actions/variables --jq '.variables[].name'` |
| 2 | `deploy.yml` 已推送 | 仓库 `.github/workflows/` 里能看到它 |
| 3 | **`verified-pages` 已成功跑过一次，且 build 不是 skipped** | Actions 标签页里最近一次运行是**绿色 ✓**（若显示灰色 skipped，说明变量没生效，回到条件 1） |

**顺序颠倒会让站点 404** —— 切到 Actions 后若没有成功产物，Pages 没有可服务的内容。

> 条件 3 特别容易漏：变量没设置时 workflow 会"成功"以 `skipped` 结束，看起来像跑过了。**必须点进去确认 build job 是 success 而不是 skipped。**

### 操作步骤

1. 打开 `https://github.com/EvanYFM/macro-workstation-site/settings/pages`
2. 找到 **Build and deployment** → **Source**
3. 从 **Deploy from a branch** 改为 **GitHub Actions**
4. 页面自动刷新，不再显示 "Branch" 选择器
5. 回到仓库 **Actions** 标签页，应能看到 `verified-pages` workflow
6. 等最新一次 `verified-pages` 运行显示绿色 ✓

### 验收（必须做，否则等于没装门禁）

```bash
curl -s https://evanyfm.github.io/macro-workstation-site/run-manifest.json | head -20
```

然后**故意制造一次失败**：在 `macro-workstation` 里把 `garden/index.html` 某个 `<script>` 的闭合标签改成 `</scripts>`，走一次发布。

预期：
- `verified-pages` 的 **build job 失败**
- **deploy job 被跳过**（未运行，而不是运行后失败）
- 线上 `run-manifest.json` **完全没变**

这一步是整个 P0-2 的验收标准。不做这一步，就无法证明门禁真的有否决权 —— 而"没有否决权的门禁"正是本次审计要解决的核心问题。

### 回滚

Settings → Pages → Source 改回 **Deploy from a branch** → 选 `main` / `root`。**10 秒内恢复**，因为分支内容从未被删除。

---

# 验收顺序

| # | 动作 | 验收方式 | 状态 |
|---|---|---|---|
| 1 | P0-1 修复 `publish-pages.mjs` | `node scripts/publish-pages.mjs --no-push` 全链通过 | ✅ **已完成并验证** |
| 2 | `docs/data-update-guide.md` 第五节降级 REST 路径 | 默认改回 `npm run publish` | ⬜ 待确认 |
| 3 | P0-2 步骤 1 设置 `PAGES_DEPLOY_ENABLED=true`（三仓） | `gh api` 能读到该变量，且 `verified-pages` 下次运行 build 是 success 而非 skipped | ⬜ |
| 4 | P0-2 步骤 2-5 为 `macro-workstation-site` 补 `deploy.yml` | `ci-check-pages` 出现 deploy.yml 断言 | ⬜ |
| 5 | P0-2 步骤 6 切换 Pages 来源（三仓） | 故意造失败 → deploy job 被跳过、线上未变 | ⬜ |
| 5 | P0-3 pre-push 钩子 | 故意提交语法错误 → push 被拦 | ⬜ |
| 6 | P1-1 测试列表同源 | CI 输出出现 5 个测试文件 | ⬜ |
| 7 | P1-2 lint + type | CI 出现 Lint / tsc 步骤 | ⬜ |
| 8 | P1-3 ruff | CI 出现 lint 步骤 | ⬜ |
| 9 | P1-4 测试改名 | `unittest discover` 输出含 `test_verify_tdx_converters` | ⬜ |
| 10 | P2 统一 futures 源头 | 只剩一份真相 | ⬜ |

**如果只做一件事，做第 1 项**（已完成）。如果做两件，加第 3-4 项。
