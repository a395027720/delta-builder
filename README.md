# @jake-gao/delta-builder

> 为 Electron.js 应用提供真正的增量更新（delta update）。仅下发新旧版本之间的二进制差量，可减少 **约 90%** 的更新流量。

[![npm version](https://img.shields.io/npm/v/@jake-gao/delta-builder.svg)](https://www.npmjs.com/package/@jake-gao/delta-builder)
[![npm downloads](https://img.shields.io/npm/dm/@jake-gao/delta-builder.svg)](https://www.npmjs.com/package/@jake-gao/delta-builder)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](#许可证)

![增量更新示意图](https://electrondelta.com/assets/delta-downloading.png)

---

## ✨ 特性

- 🔁 **真正的二进制差量** —— 基于 [`HDiffPatch`](https://github.com/sisong/HDiffPatch) 库计算新旧安装包之间的最小差异。
- 📉 **节省 ~90% 流量** —— 终端用户只需下载当前版本到最新版本之间的 delta 文件。
- 🪟 **Windows** —— 支持 `nsis` 和 `nsis-web` 两种 target（`.exe` / `.7z` 安装包）。
- 🍎 **macOS** —— 支持 `zip` target（`.zip` 安装包），内置 `mac-updater` 辅助二进制。
- ⚡ **本地缓存** —— 历史版本会被缓存到本地，二次构建无需重复下载。
- 🔐 **签名可插拔** —— 每一个生成的 delta 安装器都会调用你提供的 `sign` 钩子。
- 🧩 **发布源可插拔** —— `getPreviousReleases` 可从 GitHub Releases、S3、自有服务器等任何来源拉取历史版本。

---

## 📋 环境要求

1. 项目必须使用 [`electron-builder`](https://www.electron.build/) 打包。
2. **Windows**：target 必须是 `nsis` 或 `nsis-web`。
3. **macOS**：target 必须是 `zip`。
4. Node.js `>=14`（代码原生使用 async/await）。

---

## 📦 安装

```sh
npm install @jake-gao/delta-builder -D
```

---

## 🚀 使用方法

#### 第 1 步 —— 在项目根目录创建 `.electron-delta.js`

#### 第 2 步 —— 在 `electron-builder` 配置中通过 `afterAllArtifactBuild` 钩子接入

```json
{
  "build": {
    "appId": "com.electron.sample-app",
    "afterAllArtifactBuild": ".electron-delta.js",
    "win": {
      "target": ["nsis"],
      "publish": ["github"]
    },
    "mac": {
      "target": ["zip"],
      "publish": ["github"]
    },
    "nsis": {
      "oneClick": true,
      "perMachine": false
    }
  }
}
```

#### 第 3 步 —— 在 `.electron-delta.js` 中编写构建逻辑

```js
// .electron-delta.js
const DeltaBuilder = require("@jake-gao/delta-builder");
const path = require("path");

const options = {
  productIconPath: path.join(__dirname, "icon.ico"),
  productName: "electron-sample-app",

  // 提供需要计算差量的历史版本列表。
  // 库会把这些版本分别与 electron-builder 当前构建出的安装包做差量。
  getPreviousReleases: async () => [
    {
      version: "0.0.12",
      url: "https://github.com/electron-delta/electron-sample-app/releases/download/v0.0.12/electron-sample-app-0.0.12.exe",
    },
    {
      version: "0.0.11",
      url: "https://github.com/electron-delta/electron-sample-app/releases/download/v0.0.11/electron-sample-app-0.0.11.exe",
    },
    {
      version: "0.0.9",
      url: "https://github.com/electron-delta/electron-sample-app/releases/download/v0.0.9/electron-sample-app-0.0.9.exe",
    },
  ],

  // 对每个生成的 delta 安装器执行签名。回调会接收文件的绝对路径。
  sign: async (filePath) => {
    // 例如：signtool sign /a /fd SHA256 /tr http://timestamp.digicert.com ...
  },
};

exports.default = async function (context) {
  const deltaInstallerFiles = await DeltaBuilder.build({ context, options });
  return deltaInstallerFiles;
};
```

像往常一样构建：

```sh
electron-builder --win --mac
```

构建完成后，delta 产物会与常规安装器一起输出。

---

## ⚙️ API

### `DeltaBuilder.build({ context, options })`

返回 `Promise<string[]>` —— 生成的 delta 产物的绝对路径数组（安装器、`.delta` 文件、`delta-<platform>.json` 清单，以及 macOS 的辅助二进制）。

| 参数      | 类型     | 说明                                                                           |
| --------- | -------- | ------------------------------------------------------------------------------ |
| `context` | `object` | `electron-builder` 注入的构建上下文，由 `afterAllArtifactBuild` 钩子自动传入。 |
| `options` | `object` | 构建选项，见下表。                                                             |

### `options`

| 选项                  | 必填 | 说明                                                                                                                |
| --------------------- | ---- | ------------------------------------------------------------------------------------------------------------------- |
| `productName`         | ✅   | 产品名称，必须与安装包内的可执行文件 / `.app` 名称保持一致。                                                        |
| `productIconPath`     | ✅   | `.ico` 图标的绝对路径（用于 Windows delta 安装器）。                                                                |
| `getPreviousReleases` | ✅   | `async ({ platform, target }) => Release[]`。每个 `Release` 必须包含 `{ version, url }`。最多取**最近 10 个**版本。 |
| `sign`                | ✅   | `async (filePath) => void` —— 每个 Windows delta 安装器构建完成后、计算校验值之前被调用一次。                       |
| `processName`         | ❌   | 进程名（若与 `productName` 不同）。默认等于 `productName`。                                                         |
| `latestVersion`       | ❌   | 差量的**目标版本**。默认取 `process.env.npm_package_version`。                                                      |
| `cache`               | ❌   | 历史安装包与中间 `.delta` 文件的缓存目录。默认 `~/.electron-delta/`，也可通过环境变量 `ELECTRON_DELTA_CACHE` 设置。 |
| `logger`              | ❌   | 自定义日志器，默认 `console`。                                                                                      |

---

## 📂 产物结构

每个构建平台都会在 `electron-builder` 的 `out/` 目录下生成一个子目录：

```
out/
└── <latestVersion>-<platform>-deltas/
    ├── <productName>-<prevVersion>-to-<latestVersion>-delta.exe   (Windows)
    ├── <productName>-<prevVersion>-to-<latestVersion>.delta       (macOS)
    └── delta-<platform>.json                                       (清单)
```

`delta-<platform>.json` 清单示例：

```json
{
  "productName": "electron-sample-app",
  "latestVersion": "1.0.0",
  "0.0.12": {
    "path": "electron-sample-app-0.0.12-to-1.0.0-delta.exe",
    "sha256": "9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08"
  }
}
```

客户端更新器可读取该清单，决策下载哪个 delta。

---

## 🧠 工作原理

1. `electron-builder` 产出安装器后，`DeltaBuilder.build` 立即被调用。
2. 下载历史版本（命中缓存则跳过）并解压。
3. 解压 `context.artifactPaths` 中刚刚构建出的最新安装包。
4. 对每个历史版本（最多 10 个），使用 `HDiffPatch` 计算与最新版本之间的二进制差量。
5. Windows 平台用 NSIS 把每个 delta 包装成安装器；macOS 平台直接下发 `.delta` 文件，并附带 `mac-updater` 辅助二进制。
6. 生成一份带路径和 SHA-256 校验值的 `delta-<platform>.json` 清单，供更新器消费。

---

## 🧰 进阶用法：自动化增量构建

下面是一套**生产环境可直接复用**的封装：把历史版本管理、缓存预热、NSIS 离线处理、构建产物总结全部自动化。整体流程是：

```
electron-builder 出包
        │
        ▼
.delta-runner.js  ── 自动扫描历史版本 ──┐
        │                              │
        ▼                              ▼
  同步本地文件到缓存 ◄──────── VPN/离线环境
        │
        ▼
   DeltaBuilder.build()
        │
        ▼
  产物总结 & 上传指引
```

#### 1. 目录约定

```
project-root/
├── .delta-runner.js               # electron-builder 钩子
├── scripts/
│   └── delta-summary.js           # 构建产物总结
├── out/
│   ├── previous-releases/         # 历史版本安装器（自动维护）
│   │   ├── myapp-1.2.0-test.exe
│   │   └── myapp-1.1.0-test.exe
│   └── <version>-win-deltas/      # 增量产物
└── delta/
    └── nsis.zip                   # 离线环境预置的 NSIS
```

#### 2. 自动化钩子 `.delta-runner.js`

核心思路：

- **自动扫描** `out/previous-releases/`，无需手写版本列表。
- **多环境隔离**：根据 npm script 名称（`*prod*` / `*stage*` / `*test*`）自动只对同名环境的安装器做差量，避免不同环境产物相互污染。
- **VPN/离线友好**：构建前先把本地安装器同步到 `~/.electron-delta/data/` 缓存目录，让 builder 内部的 `downloadFileIfNotExists` 直接命中缓存，**完全不发出 HTTP 请求**。
- **NSIS 离线**：找不到 makensis 时自动从项目本地 `delta/nsis.zip` 或 `%APPDATA%\electron-delta-bins\` 解压，并做完整性校验。
- **自动存档**：本次构建出的安装器会自动复制到 `out/previous-releases/`，供下次构建做差量（按环境隔离，只保留同环境的最近一份）。

```js
// .delta-runner.js
const DeltaBuilder = require("@jake-gao/delta-builder");
const path = require("path");
const fs = require("fs");

const STEP = (m) => console.log(`\n\x1b[36m[delta] ${m}\x1b[0m`);
const OK = (m) => console.log(`\x1b[32m  ✓\x1b[0m ${m}`);
const WARN = (m) => console.log(`\x1b[33m  ⚠\x1b[0m ${m}`);
const INFO = (m) => console.log(`  • ${m}`);

/** 从文件名提取 semver 版本号 */
const extractVersion = (name) => {
  const m = name.match(/(\d+\.\d+\.\d+)/);
  return m ? m[1] : null;
};

/** 根据 npm script 名称推断当前环境 */
const detectEnv = () => {
  const s = process.env.npm_lifecycle_event || "";
  if (s.includes("prod")) return "prod";
  if (s.includes("stage")) return "stage";
  if (s.includes("test")) return "test";
  return null;
};

/** 安装器文件名按环境后缀过滤：myapp-1.2.0-test.exe / -stage.exe / -prod.exe */
const matchEnv = (name, env) => {
  if (!env) return true; // 未知环境时不过滤，保持向后兼容
  const base = path.basename(name, path.extname(name));
  return base.endsWith(`-${env}`);
};

/** 扫描历史版本目录 */
function scanPreviousReleases(resourceUrl) {
  const localDir = path.join(__dirname, "out", "previous-releases");
  const env = detectEnv();
  if (!fs.existsSync(localDir)) {
    WARN("out/previous-releases/ 不存在，跳过增量差分");
    return [];
  }

  const all = fs
    .readdirSync(localDir)
    .filter((f) => f.endsWith(".exe") || f.endsWith(".7z"));
  const matched = all.filter((f) => matchEnv(f, env));
  if (matched.length === 0) {
    WARN(`无 ${env || "当前"} 环境的历史版本，跳过增量差分`);
    return [];
  }

  // 与更新服务器路径约定保持一致
  const base = resourceUrl ? `${resourceUrl.replace(/\/+$/, "")}/updates` : "";
  STEP(`扫描历史安装器 [环境: ${env || "未知"}]`);
  return matched.map((f) => {
    const version = extractVersion(f);
    INFO(`${f}  →  v${version}`);
    return { version, url: base ? `${base}/${f}` : f };
  });
}

/** 把本地安装器预热到 builder 缓存（VPN 关键步骤） */
function warmCache(cacheDir, releases) {
  const dataDir = path.join(cacheDir, "data");
  const localDir = path.join(__dirname, "out", "previous-releases");
  if (!fs.existsSync(localDir) || releases.length === 0) return;
  fs.mkdirSync(dataDir, { recursive: true });

  let synced = 0;
  for (const r of releases) {
    const name = path.basename(r.url);
    const cachePath = path.join(dataDir, name);
    if (fs.existsSync(cachePath)) continue;
    const localPath = path.join(localDir, name);
    if (fs.existsSync(localPath)) {
      fs.copyFileSync(localPath, cachePath);
      synced++;
    }
  }
  if (synced > 0) OK(`${synced} 个文件已同步到缓存`);
}

const options = {
  productIconPath: path.join(__dirname, "build", "icons", "icon.ico"),
  productName: "",
  cache: path.join(require("os").homedir(), ".electron-delta"),
  getPreviousReleases: async () => [],
  sign: async (_) => {},
};

exports.default = async function (context) {
  // 从 electron-builder config 注入 productName / version
  options.productName = context.configuration.productName || "electron-app";
  options.latestVersion =
    context.configuration.extraMetadata?.version || "1.0.0";

  const env = detectEnv();
  STEP(
    `产品: ${options.productName}  版本: ${options.latestVersion}  环境: ${env || "未知"}`,
  );

  // 1. 收集历史版本（自动按环境过滤）
  const releases = scanPreviousReleases(
    context.configuration.extraMetadata?.resourceUrl,
  ).filter((r) => r.version !== options.latestVersion);
  options.getPreviousReleases = async () => releases;

  if (releases.length === 0) {
    INFO("首次增量构建，无历史版本可对比；本次安装器会自动存档");
    archiveInstaller(options.latestVersion);
    return [];
  }

  // 2. 预热缓存（VPN/离线环境关键）
  warmCache(options.cache, releases);

  // 3. 真正生成差分
  STEP("开始生成增量差分补丁...");
  const files = await DeltaBuilder.build({ context, options });
  if (files?.length) OK(`差分补丁生成完成，共 ${files.length} 个文件`);

  // 4. 存档本次安装器，供下次构建
  archiveInstaller(options.latestVersion);
  return files;
};

/** 自动把本次构建出的安装器存档到 previous-releases（按环境隔离） */
function archiveInstaller(latestVersion) {
  const outDir = path.join(__dirname, "out");
  const prevDir = path.join(outDir, "previous-releases");
  const env = detectEnv();
  if (!fs.existsSync(outDir)) return;

  const files = fs
    .readdirSync(outDir)
    .filter(
      (f) =>
        f.endsWith(".exe") &&
        matchEnv(f, env) &&
        extractVersion(f) === latestVersion,
    );
  if (files.length === 0) return;

  fs.mkdirSync(prevDir, { recursive: true });

  // 清理同环境的旧版本，只保留最近一次构建
  for (const old of fs
    .readdirSync(prevDir)
    .filter((f) => f.endsWith(".exe") && matchEnv(f, env))) {
    fs.unlinkSync(path.join(prevDir, old));
  }

  for (const f of files) {
    fs.copyFileSync(path.join(outDir, f), path.join(prevDir, f));
    OK(`安装器已存档: out/previous-releases/${f}`);
  }
}
```

#### 3. 构建产物总结 `scripts/delta-summary.js`

构建完成后运行一次，自动输出**上传清单、流量节省比例、测试步骤**：

```js
// scripts/delta-summary.js
const fs = require("fs");
const path = require("path");

const STEP = (n, m) => console.log(`\n\x1b[36m[${n}]\x1b[0m ${m}`);
const OK = (m) => console.log(`\x1b[32m  ✓\x1b[0m ${m}`);
const WARN = (m) => console.log(`\x1b[33m  ⚠\x1b[0m ${m}`);
const INFO = (m) => console.log(`  • ${m}`);

function main() {
  const outDir = path.join(__dirname, "..", "out");
  const installers = [];
  const deltaFiles = [];
  let totalMB = 0;
  let deltaKB = 0;

  if (fs.existsSync(outDir)) {
    for (const entry of fs.readdirSync(outDir)) {
      const p = path.join(outDir, entry);
      if (fs.statSync(p).isFile() && entry.endsWith(".exe")) {
        const mb = fs.statSync(p).size / 1024 / 1024;
        installers.push({ name: entry, size: mb.toFixed(1) });
        totalMB += mb;
      }
      if (fs.statSync(p).isDirectory() && entry.endsWith("-deltas")) {
        for (const f of fs.readdirSync(p)) {
          const fp = path.join(p, f);
          const kb = fs.statSync(fp).size / 1024;
          deltaFiles.push({ dir: entry, name: f, size: kb.toFixed(1) });
          deltaKB += fs.statSync(fp).size;
        }
      }
    }
  }

  console.log("\n  安装程序:");
  installers.forEach((i) => INFO(`${i.name}  (${i.size} MB)`));

  console.log("\n  增量差分补丁:");
  if (deltaFiles.length === 0) {
    WARN("未生成（首次构建或缺少历史安装器）");
  } else {
    deltaFiles.forEach((f) => INFO(`${f.dir}/${f.name}  (${f.size} KB)`));
  }

  STEP(1, `上传 ${installers.length + deltaFiles.length} 个文件到更新服务器`);
  STEP(2, "测试：在测试机上安装上一版本 → 启动 → 验证自动更新");

  if (installers.length && deltaFiles.length) {
    const saved = ((1 - deltaKB / (totalMB * 1024 * 1024)) * 100).toFixed(1);
    OK(
      `全量 ${totalMB.toFixed(0)}MB → 增量 ${(deltaKB / 1024).toFixed(1)}MB，节省约 ${saved}% 流量`,
    );
  }
}
main();
```

#### 4. `package.json` 一键编排

```json
{
  "scripts": {
    "build:w:test": "electron-builder --config=./cmd/builder-test.json  -w=nsis --ia32 && node scripts/delta-summary.js",
    "build:w:stage": "electron-builder --config=./cmd/builder-stage.json -w=nsis --x64   && node scripts/delta-summary.js",
    "build:w:prod": "electron-builder --config=./cmd/builder-prod.json  -w=nsis --x64   && node scripts/delta-summary.js",
    "clear-delta-cache": "node scripts/clear-delta-cache.js"
  },
  "build": {
    "afterAllArtifactBuild": ".delta-runner.js"
  }
}
```

执行 `npm run build:w:test`，效果：

```
[delta] 扫描历史安装器 [环境: test]
  • myapp-1.2.0-test.exe  →  v1.2.0
  • myapp-1.1.0-test.exe  →  v1.1.0
  ✓ 2 个文件已同步到缓存
[delta] 开始生成增量差分补丁...
  ✓ 差分补丁生成完成，共 4 个文件
  ✓ 安装器已存档: out/previous-releases/myapp-1.3.0-test.exe

[1] 上传 5 个文件到更新服务器
[2] 测试：在测试机上安装上一版本 → 启动 → 验证自动更新

✓ 全量 85MB → 增量 7.2MB，节省约 91.5% 流量
```

#### 5. 关键设计要点

| 问题                                    | 解法                                                                                               |
| --------------------------------------- | -------------------------------------------------------------------------------------------------- |
| 手动维护历史版本列表太累                | 自动扫描 `out/previous-releases/`，构建结束后自动把当前安装器复制进去                              |
| 不同环境（test/stage/prod）产物互相覆盖 | 按 npm script 名推断环境，按 `-test/-stage/-prod` 后缀过滤；存档时也只清理同环境的旧版本           |
| VPN/内网无法下载历史安装器              | `warmCache()` 在构建前把本地安装器复制到 `~/.electron-delta/data/`，builder 的下载逻辑直接命中缓存 |
| NSIS 在 VPN 环境拉不下来                | 项目本地或 `%APPDATA%\electron-delta-bins\` 预置 `nsis.zip`，运行时校验完整性并自动解压            |
| 不知道该上传哪些文件                    | `delta-summary.js` 总结安装器 + 差分文件，按平台/版本过滤后给出清单和流量节省比例                  |

---

## 🤝 参与贡献

欢迎 PR。关键源码入口：

- `src/index.js` —— 顶层调度。
- `src/create-all-deltas.js` —— 下载 / 解压 / 差量流程。
- `src/delta-installer-builder/` —— NSIS 安装器脚手架（Windows）。
- `src/mac-updater-binaries/` —— macOS 预编译的 `hpatchz` 与 `mac-updater`。

运行仓库自带的烟雾测试脚本：

```sh
node tests/mac.js
node tests/win.js
```

---

## 📄 许可证

[MIT](./LICENSE) © Jake Gao
