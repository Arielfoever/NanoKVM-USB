# 开发指南（双端统一后的仓库结构）

本仓库包含 NanoKVM-USB 的两个应用：浏览器版与 Electron 桌面版。两者共享同一套渲染层代码，平台差异通过适配器注入。

## 环境要求

- Node.js **24 LTS**（`.nvmrc` 已固定）
- pnpm **12**（`packageManager` 字段已固定，建议 `corepack enable` 后使用）

## 目录结构

```
NanoKVM-USB/
├── apps/
│   ├── browser/                 # Vite SPA：入口 + Web Serial / 浏览器平台适配器
│   └── desktop/                 # electron-vite：main / preload + IPC 平台适配器
├── packages/
│   ├── core/                    # 无 UI：设备协议（proto）、Device 门面、Transport/PlatformAdapter 接口
│   └── ui/                      # 共享 React 渲染层：组件、jotai、i18n、storage、样式
├── pnpm-workspace.yaml          # workspace + overrides + allowBuilds
└── pnpm-lock.yaml               # 唯一锁文件
```

依赖版本统一在根 `package.json`（构建/工具链）与各包 `package.json`（运行时依赖）中管理，**全部使用精确版本**，升级必须走显式 PR。

## 常用命令

```bash
pnpm install --frozen-lockfile   # 安装（CI 同款）
pnpm dev:browser                 # 浏览器版开发服务器（:3001）
pnpm dev:desktop                 # 桌面版开发（electron-vite dev）
pnpm build:browser               # 浏览器版构建 → apps/browser/dist
pnpm build:desktop               # 桌面版构建 → apps/desktop/out
pnpm --filter @nanokvm/desktop build:unpack  # 桌面端免安装产物（验证打包）
pnpm typecheck                   # 所有包的类型检查
pnpm lint                        # ESLint
pnpm format                      # Prettier 全仓格式化
```

## 分层与边界

```
apps/*            ← 只能提供平台能力与入口，不写业务 UI
  platform/       ← PlatformAdapter / SerialTransport 实现
packages/ui       ← 全部 React UI、jotai atoms、i18n、localStorage 封装
packages/core     ← 协议编解码、Device 门面、接口定义（不依赖 DOM/Electron）
```

**规则：**

1. `packages/ui` 与 `packages/core` 中**禁止**出现 `window.electron`、`navigator.serial`、`node:*`、`electron` 等平台 API。
2. 组件需要平台能力时使用 `getPlatform()` / `getDevice()`（见 `packages/ui/src/services.ts`），平台能力通过 `PlatformAdapter` 暴露。
3. 需要区分平台能力时，判断**能力**（如 `transport.supportsEnumeration()`、`platform.updater`），不要判断"是不是 Electron"。
4. 新增平台能力：先在 `packages/core/src/platform.ts` 扩展接口，再在两个 app 的适配器里实现。

## 串口传输层

- 浏览器：`apps/browser/src/platform/web-serial.ts`（Web Serial，需用户手势选择端口）
- 桌面：`apps/desktop/src/renderer/src/platform/ipc-transport.ts` → preload `window.api.serial.*` → 主进程 `node-serialport`（只收发原始字节）
- 协议与 `Device` 门面只在 `packages/core` 维护一份。

## i18n / storage

- 语言文件：`packages/ui/src/i18n/locales/*.ts`，新增 key 时至少补 `en.ts`（其余语言缺 key 时自动回退英文）。
- localStorage key 统一以 `nanokvm-usb-` 前缀；桌面端历史 key `nanokvm-usb-keyboard-shortcuts`（复数）在读取时兼容，写入统一为单数形式。

## 代码检查与格式化

- **ESLint 10 + typescript-eslint**：配置在根 `eslint.config.mjs`，工具链独立安装在 `tools/lint`（因为 typescript-eslint 目前只支持 TypeScript < 6.1，而构建链已用 TypeScript 7 原生编译器；`tools/lint` 自带 TypeScript 5.9 供 parser 使用）。
- **Prettier**：根 `.prettierrc.yaml`（`semi: false`、单引号、import 排序、Tailwind v4 类名排序），全仓统一，不再保留各 app 的独立配置。
- 提交前：`pnpm lint && pnpm format:check && pnpm typecheck`。

## 验证清单

仓库目前没有自动化 UI 测试，重构/升级后至少运行：

1. `pnpm install --frozen-lockfile`（干净克隆）
2. `pnpm typecheck`、`pnpm lint`
3. `pnpm build:browser`、`pnpm build:desktop`；桌面端再跑一次 `build:unpack`
4. 手工冒烟：摄像头出图、串口连接/自动重连、键盘（含右 Shift/AltGr）、鼠标三种模式、快捷键、粘贴、虚拟键盘、菜单折叠持久化、分辨率/旋转/缩放、音频、拔插断连、语言切换、桌面端自动更新检查。

> 冒烟清单的完整 16 项见 [docs/refactor-plan.md 附录 C](./refactor-plan.md)。
