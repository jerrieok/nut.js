# nut.js 源码编译与发布指南

> 适用场景：从官方仓库拉取 nut.js 源码 → 本地编译 → 打包成 npm 包 → 发布到 GitHub 供自己或其他项目使用。
> 上游升级后，只需按「[第九章 升级后的重编译清单](#九升级上游后的重新编译清单)」逐条执行即可。

---

## 目录

1. [环境准备](#一环境准备)
2. [获取源码](#二获取源码)
3. [首次改造（只做一次）](#三首次改造只做一次)
4. [安装依赖](#四安装依赖)
5. [编译](#五编译)
6. [打包 tarball](#六打包-tarball自用分发)
7. [发布到 GitHub](#七发布到-github三种方式)
8. [用户/其他项目如何安装](#八用户其他项目如何安装)
9. [升级上游后的重新编译清单](#九升级上游后的重新编译清单)
10. [常见问题](#十常见问题)

---

## 一、环境准备

| 项目     | 要求                                                                  | 检查命令        |
| -------- | --------------------------------------------------------------------- | --------------- |
| Node.js  | ≥ 18（`package.json` 里 `engines` 写的是 `>=16`，实测 18/20/26 均可） | `node -v`       |
| pnpm     | **8.15.2**（与根 `package.json` 的 `packageManager` 字段一致）        | `pnpm -v`       |
| Git      | 任意较新版本                                                          | `git --version` |
| 操作系统 | macOS / Linux / Windows 均可，原生依赖会自动匹配平台                  | —               |

安装/切换 pnpm 版本（二选一）：

```bash
# 方式 A：corepack（Node 自带，Node 22 及以下推荐）
corepack enable
corepack prepare pnpm@8.15.2 --activate

# 方式 B：直接全局安装（Node 25+ 已移除内置 corepack，用这个）
npm i -g pnpm@8.15.2
```

验证：

```bash
pnpm -v   # 期望输出 8.15.2
```

> ⚠️ 不建议用 pnpm 9/10 编译这个仓库，lockfile 版本和 workspace 协议行为有差异，容易出现 `ERR_PNPM_OUTDATED_LOCKFILE` 之类的额外问题。

---

## 二、获取源码

官方仓库地址：**https://github.com/nut-tree/nut.js**

### 方式 A：git clone（推荐，方便后续 `git pull` 升级）

```bash
cd ~/Codes/github
git clone https://github.com/nut-tree/nut.js.git nut.js
cd nut.js
git log -1 --oneline   # 记下当前 commit，便于回溯
```

### 方式 B：下载 zip

1. 打开 https://github.com/nut-tree/nut.js
2. 点 `Code` → `Download ZIP`
3. 解压到 `~/Codes/github/nut.js`

---

## 三、首次改造（只做一次）

官方源码是**发布到公共 npm 的原始状态**，要变成「你自己的包」，需要做下面这些改动。升级上游后这些改动可能被覆盖，需要重新检查。

### 3.1 目录结构：`core/*` + `providers/*` → `packages/*`

官方原始结构：

```
nut.js/
├── core/
│   ├── configs/
│   ├── nut.js/
│   ├── provider-interfaces/
│   └── shared/
├── providers/
│   ├── clipboardy/
│   └── libnut/
├── examples/
└── e2e/
```

改造后（更通用的 monorepo 布局）：

```
nut.js/
├── packages/
│   ├── configs/
│   ├── nut.js/                  ← 主包，最终要发布的就是它
│   ├── provider-interfaces/
│   ├── shared/
│   ├── clipboardy/
│   └── libnut/
├── examples/
└── e2e/
```

迁移命令（用 `git mv` 保留文件历史）：

```bash
cd ~/Codes/github/nut.js

mkdir -p packages

git mv core/configs             packages/configs
git mv core/nut.js              packages/nut.js
git mv core/provider-interfaces packages/provider-interfaces
git mv core/shared              packages/shared
git mv providers/clipboardy     packages/clipboardy
git mv providers/libnut         packages/libnut
```

> 迁移完成后 `core/`、`providers/` 会变成空目录，手动删掉即可（git 不跟踪空目录）。
> 这些包之间靠**包名**互相引用（如 `@nut-tree/shared`），靠 pnpm 在 `node_modules` 建符号链接解析，不依赖相对路径，所以移动目录不会影响编译。

### 3.2 同步 workspace 配置（4 个文件）

**`pnpm-workspace.yaml`**

```yaml
packages:
  - "packages/*"
  - "examples/*"
  - "e2e/*"
```

**根 `package.json` 的 `workspaces` 数组**

```json
"workspaces": [
  "packages/*",
  "examples/*",
  "e2e/*"
],
```

**`typedoc.json`（可选，只影响生成文档）**

```json
"entryPoints": [
  "packages/nut.js",
  "packages/provider-interfaces",
  "packages/shared"
],
```

**`.github/workflows/ci.yaml`（可选，只影响 CI）**

```yaml
projectBaseDir: packages/nut.js
```

### 3.3 改包名与发布元信息

编辑 `packages/nut.js/package.json`：

```json
{
  "name": "@jerrieok/nut-js",
  "version": "4.2.0",
  "repository": {
    "type": "git",
    "url": "https://github.com/jerrieok/nut.js.git"
  },
  "bugs": {
    "url": "https://github.com/jerrieok/nut.js/issues"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  }
}
```

说明：

- `name` 必须改成**你自己的 scope**（`@你的GitHub用户名/xxx`）。GitHub Packages 强制要求 scope 与 GitHub 用户名/组织名一致，否则发布会被拒绝。
- `version` 发布后不可修改，升级时必须递增。
- `publishConfig.registry` 只在用 GitHub Packages 时需要；只做本地 tarball 分发可以删掉。

**其余 4 个依赖包**（`shared`、`provider-interfaces`、`libnut`、`clipboardy`）**不用改名**，因为它们的历史版本在公共 npm 上都存在（`@nut-tree/*@4.2.0`），主包发布后消费者会自动从公共源补齐。除非你要把它们一起私有化，那时才需要逐个改成 `@jerrieok/*` 并一起发布。

### 3.4 修正示例项目的依赖引用

主包改名后，`examples/*` 和 `e2e/*` 里原来写的 `"@nut-tree/nut-js": "workspace:*"` 会找不到包。

用 **workspace 别名语法**，既能指向新包，又不用改示例源码里的 `require("@nut-tree/nut-js")`：

```json
"dependencies": {
  "@nut-tree/nut-js": "workspace:@jerrieok/nut-js@*"
}
```

需要改的文件：

- `examples/jest-test/package.json`
- `examples/keyboard-test/package.json`
- `examples/mouse-test/package.json`
- `examples/screen-test/package.json`
- `examples/typescript-test/package.json`
- `e2e/window-test/package.json`（如有该依赖项）

### 3.5 移除已下架的依赖（重要）

nut.js 4.x 起图像匹配能力已内置到 `libnut`，官方仓库里部分示例仍残留对 `@nut-tree/nl-matcher` 的引用。这个包**已被作者从 npm 撤下**，安装时会报：

```
ERR_PNPM_FETCH_404  GET https://registry.npmmirror.com/@nut-tree/nl-matcher-darwin/-/nl-matcher-darwin-2.2.0.tgz: Not Found
```

处理方式：删除示例里对它的依赖声明和 `require` 行（这些示例不参与主包编译，删掉不影响产物）。

---

## 四、安装依赖

```bash
cd ~/Codes/github/nut.js
pnpm install
```

判断成功的标志：

```
Scope: all 15 workspace projects
Done in xxx ms
```

> 如果刚刚新增了 workspace 项目（比如新加了示例目录），会报 `ERR_PNPM_OUTDATED_LOCKFILE`，此时用：
>
> ```bash
> pnpm install --no-frozen-lockfile
> ```

---

## 五、编译

```bash
cd ~/Codes/github/nut.js
pnpm run compile
```

根 `package.json` 里的 `compile` 是 `pnpm -r run compile`，会**按依赖拓扑序**依次编译：

```
packages/shared
  └── packages/provider-interfaces
        └── packages/libnut / packages/clipboardy
              └── packages/nut.js   ← 主包最后编译
```

这个顺序不能乱，因为主包的类型检查依赖上游包已经生成的 `dist/index.d.ts`。

单独编译某一个包：

```bash
pnpm --filter @jerrieok/nut-js run compile
```

验证产物：

```bash
ls packages/nut.js/dist/index.js packages/nut.js/dist/index.d.ts
```

---

## 六、打包 tarball（自用分发）

### 6.1 核心命令

```bash
cd ~/Codes/github/nut.js/packages/nut.js
pnpm pack
```

产出：`jerrieok-nut-js-4.2.0.tgz`（约 44 KB）

> ⚠️ **`pnpm pack` 不会触发 `prepublishOnly`**（那是 `pnpm publish` 专属钩子）。所以必须先手动 `pnpm run compile`，否则 tarball 里是旧的或空的 `dist/`。

### 6.2 关键验证：`workspace:*` 是否被重写

这是 tarball 能否被外部项目安装的**唯一判断标准**：

```bash
tar -xzOf jerrieok-nut-js-4.2.0.tgz package/package.json \
  | node -e 'let s="";process.stdin.on("data",d=>s+=d).on("end",()=>console.log(JSON.stringify(JSON.parse(s).dependencies,null,2)))'
```

**期望输出**（全部是具体版本号，不能出现 `workspace:`）：

```json
{
  "jimp": "0.22.10",
  "node-abort-controller": "3.1.1",
  "@nut-tree/default-clipboard-provider": "4.2.0",
  "@nut-tree/libnut": "4.2.0",
  "@nut-tree/shared": "4.2.0",
  "@nut-tree/provider-interfaces": "4.2.0"
}
```

如果仍显示 `workspace:*`，说明 `pnpm-workspace.yaml` 没有正确包含 `packages/*`，回到第三章检查配置后重新 `pnpm install`。

### 6.3 汇总到统一目录

```bash
mkdir -p ~/Codes/github/nut-packages
mv ~/Codes/github/nut.js/packages/nut.js/jerrieok-nut-js-*.tgz ~/Codes/github/nut-packages/
```

最终产物：

```
~/Codes/github/nut-packages/jerrieok-nut-js-4.2.0.tgz
```

### 6.4 一次跑完的完整命令

```bash
cd ~/Codes/github/nut.js \
  && pnpm run compile \
  && cd packages/nut.js \
  && pnpm pack \
  && mv jerrieok-nut-js-*.tgz ~/Codes/github/nut-packages/
```

---

## 七、发布到 GitHub（三种方式）

### 方式 A：GitHub Release 附件（最接近「供用户下载」）

把 tarball 作为 Release 附件上传，用户可直接下载，不依赖 npm 权限配置。

```bash
cd ~/Codes/github/nut.js

# 1) 提交并打 tag
git add -A
git commit -m "chore: build nut.js 4.2.0"
git tag v4.2.0
git push origin main --tags

# 2) 创建 Release 并上传 tarball
gh release create v4.2.0 \
  ~/Codes/github/nut-packages/jerrieok-nut-js-4.2.0.tgz \
  --title "nut.js 4.2.0" \
  --notes "基于官方 nut.js 4.2.0 源码编译"
```

用户下载后安装：

```bash
pnpm add ./jerrieok-nut-js-4.2.0.tgz
```

> 私有仓库的 Release 附件需要登录才能下载，适合内部使用。

---

### 方式 B：GitHub Packages（npm registry 方式）

适合多台机器、多个项目统一 `pnpm update` 的场景。

#### 1. 创建 `.npmrc`（放在仓库根目录，**不要提交真实 token**）

```ini
@jerrieok:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

> `always-auth=true` 在 npm 10+ 已废弃，会输出 `Unknown project config "always-auth"` 警告，不要写。
> 确认 `.npmrc` 已加入 `.gitignore`：`git check-ignore -v .npmrc`

#### 2. 准备 Token

打开 https://github.com/settings/tokens → `Generate new token (classic)`，勾选：

- `write:packages`（发布必需）
- `read:packages`
- `delete:packages`（要删除线上旧版本时才需要）

```bash
export GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxxx
```

#### 3. 发布

```bash
cd ~/Codes/github/nut.js/packages/nut.js
pnpm publish --no-git-checks
```

- `--no-git-checks` 必需：`dist/` 被 `.gitignore` 忽略，pnpm 默认会因此拒绝发布
- `prepublishOnly` 会自动触发 `pnpm run compile`

#### 4. 验证

```bash
npm view @jerrieok/nut-js versions --registry=https://npm.pkg.github.com
npm view @jerrieok/nut-js dist-tags --registry=https://npm.pkg.github.com
```

#### 5. 版本冲突的处理

发布后版本号不可复用。报错：

```
npm error You cannot publish over the previously published versions: 4.2.0.
```

两种解法：

| 情况           | 做法                                                                                                                                                            |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 只是想发新版本 | 升 `version` 到比线上 `latest` 更高的号，再发                                                                                                                   |
| 想复用原版本号 | 先去 GitHub 删除线上版本：`Your packages` → `@jerrieok/nut-js` → `Package settings` → 删除对应版本（或用 `DELETE /user/packages/npm/<包名>/versions/<id>` API） |

还有一条容易踩的规则：**新版本号低于线上最高版本时，不能隐式打 `latest` 标签**，会报：

```
Cannot implicitly apply the "latest" tag because previously published version 4.5.0 is higher than the new version 4.2.0.
```

绕过方式是显式指定标签：

```bash
pnpm publish --tag latest --no-git-checks
```

---

### 方式 C：把 tarball 提交进仓库

适合「不想配 registry，但需要跨机器」的小团队场景。

```bash
cd ~/Codes/github/nut.js
mkdir -p packages-dist
cp ~/Codes/github/nut-packages/jerrieok-nut-js-4.2.0.tgz packages-dist/
git add packages-dist
git commit -m "chore: add nut.js 4.2.0 tarball"
git push
```

用户安装：

```bash
pnpm add git+https://github.com/jerrieok/nut.js.git#main
```

> 更常用的做法是把 tarball 放到独立的 `nut-packages` 仓库，各项目用相对路径引用：
> `"@jerrieok/nut-js": "file:../nut-packages/jerrieok-nut-js-4.2.0.tgz"`

---

## 八、用户/其他项目如何安装

### 从 tarball 安装

```bash
cd ~/Codes/github/app-a
pnpm add file:../nut-packages/jerrieok-nut-js-4.2.0.tgz
```

### 从 GitHub Packages 安装

```bash
npm config set @jerrieok:registry https://npm.pkg.github.com
npm login --scope=@jerrieok --registry=https://npm.pkg.github.com
pnpm add @jerrieok/nut-js@4.2.0
```

### 使用

```js
const { mouse, keyboard, screen, Key, Button } = require("@jerrieok/nut-js");

await mouse.move(straightTo(new Point(100, 100)));
await mouse.click(Button.LEFT);
```

> 包名始终是 `@jerrieok/nut-js`（由 `package.json` 的 `name` 决定），与源码目录名无关。

---

## 九、升级上游后的重新编译清单

上游发新版本后，按下面顺序走一遍：

```bash
# ① 拉取最新源码
cd ~/Codes/github/nut.js
git fetch origin
git log HEAD..origin/main --oneline        # 先看有哪些变更
git merge origin/main                       # 或 git pull

# ② 检查第三章的改造是否被覆盖（重点看这 6 处）
cat pnpm-workspace.yaml                     # 应包含 packages/*
node -p "require('./package.json').workspaces"
node -p "require('./packages/nut.js/package.json').name"      # 应是 @jerrieok/nut-js
grep -r "workspace:@jerrieok/nut-js" examples/*/package.json  # 示例别名是否还在
grep -rn "nl-matcher" examples/ e2e/        # 是否又冒出已下架的依赖
cat packages/nut.js/.npmignore

# ③ 安装依赖
pnpm install --no-frozen-lockfile

# ④ 编译
pnpm run compile

# ⑤ 升版本号（发布前必做）
#    手动改 packages/nut.js/package.json 的 version，例如 4.2.0 → 4.3.0

# ⑥ 打包
cd packages/nut.js
pnpm pack
mv jerrieok-nut-js-*.tgz ~/Codes/github/nut-packages/

# ⑦ 验证依赖重写（见 6.2）
cd ~/Codes/github/nut-packages
tar -xzOf jerrieok-nut-js-*.tgz package/package.json | head -30

# ⑧ 发布（按需选一种）
#    A. Release 附件
cd ~/Codes/github/nut.js && git tag v4.3.0 && git push origin main --tags
gh release create v4.3.0 ~/Codes/github/nut-packages/jerrieok-nut-js-4.3.0.tgz

#    B. GitHub Packages
cd ~/Codes/github/nut.js/packages/nut.js && pnpm publish --no-git-checks
```

### 快速自检表

| 检查项                             | 期望结果                                                            |
| ---------------------------------- | ------------------------------------------------------------------- |
| `pnpm install`                     | `Scope: all 15 workspace projects`（数量随示例增减，无 404 即成功） |
| `pnpm run compile`                 | 所有含 `compile` 脚本的包均编译成功，无 TS 报错                     |
| `ls packages/nut.js/dist/index.js` | 文件存在                                                            |
| tarball 依赖                       | 全部是数字版本号，无 `workspace:`                                   |
| tarball 文件数                     | 约 111 个（`tar -tzf xxx.tgz \| wc -l`）                            |
| 外部项目试装                       | `require("@jerrieok/nut-js")` 不报错                                |

---

## 十、常见问题

### 1. `ERR_PNPM_FETCH_404` 找不到 `@nut-tree/nl-matcher-*`

该包已被作者从 npm 撤下。nut.js 4.x 的图像匹配已内置到 `libnut`，删除示例里对它的依赖和 `require` 即可（见 3.5）。

### 2. `TypeError: X is not a function`（运行时）

nut.js 的 API 是**动词优先**命名，容易凭直觉写错。常用对照：

| 错误写法           | 正确写法                                         |
| ------------------ | ------------------------------------------------ |
| `screen.size()`    | `await screen.width()` / `await screen.height()` |
| `mouse.position()` | `await mouse.getPosition()`                      |
| `Key.ENTER`        | `Key.Return`                                     |

查某个对象有哪些方法：

```bash
node --no-warnings -e '
const nut = require("@jerrieok/nut-js");
const proto = Object.getPrototypeOf(nut.mouse);   // mouse / keyboard / screen
for (const n of Object.getOwnPropertyNames(proto)) if (n !== "constructor") console.log(n);
'
```

### 3. `ERR_PNPM_OUTDATED_LOCKFILE`

lockfile 与 `package.json` 不一致（通常因为新增了 workspace 项目）：

```bash
pnpm install --no-frozen-lockfile
```

### 4. `Cannot find module '@nut-tree/node-mac-permissions'`

这是 macOS 权限检查的**可选依赖**，外部项目从 tarball 安装时不会带入。代码内部 `catch` 后会 fallback，不影响使用。需要屏幕录制权限弹窗时再单独安装：

```bash
pnpm add @nut-tree/node-mac-permissions
```

### 5. 截图全黑 / 鼠标点击无效（macOS）

需要给终端（或运行 Node 的程序）授权：

- 系统设置 → 隐私与安全性 → **屏幕录制**（截图）
- 系统设置 → 隐私与安全性 → **辅助功能**（鼠标/键盘控制）

改完需要**重启终端**生效。

### 6. `npm warn Unknown cli config "--git-checks"`

`pnpm publish --no-git-checks` 会把这个参数透传给 npm，属于无害警告，可忽略。

### 7. 关于 OCR / 屏幕文字识别

**nut.js 不自带 OCR**。它只提供：

- 图像模板匹配：`screen.find(imageResource("xxx.png"))`
- 文字搜索的**接口预留**：`singleWord()` / `textLine()` / `TextFinderInterface`

但没有提供任何 `TextFinderInterface` 的实现，直接调用会抛 `No TextFinder registered`。需要 OCR 时，得自己实现一个 provider 并注册：

```js
const { providerRegistry } = require("@jerrieok/nut-js");
providerRegistry.registerTextFinder(myTesseractBasedFinder);
```

---

## 附录：关键文件速查

| 文件                            | 作用                                                             |
| ------------------------------- | ---------------------------------------------------------------- |
| `pnpm-workspace.yaml`           | workspace 成员声明                                               |
| `packages/nut.js/package.json`  | 主包名、版本、依赖、发布配置                                     |
| `packages/nut.js/.npmignore`    | 决定 tarball 里排除哪些文件                                      |
| `packages/nut.js/tsconfig.json` | 编译配置，输出到 `dist/`                                         |
| `.npmrc`                        | GitHub Packages 的 scope → registry 映射与认证（不要提交 token） |
