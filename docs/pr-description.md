# 双端统一重构 + 运行时栈升级 + 合并前修复

## 概要

本 PR 把桌面端与浏览器端的渲染层合并为 `packages/ui`，设备协议下沉到 `packages/core`，两端只保留平台适配器；同时把构建链和 UI 运行时栈升级到当前稳定版，并修复合并前实测发现的兼容性/回归问题。

- 基线：`main` @ `1d1dd5e`（1.1.4 之后的统一改造分支）
- 目标版本：`1.2.0`
- 合并后仍需真机确认的项目见「验证结论」与「待人工验证」

## 1. 结构调整

| 目录 | 内容 |
| --- | --- |
| `packages/core` | 串口协议编解码、`Device` 门面、`SerialTransport` / `PlatformAdapter` 接口，无 UI、无 Electron 依赖 |
| `packages/ui` | 两端共享的 React 渲染层：组件、jotai、i18n、storage、样式 |
| `apps/browser` | Vite SPA 入口 + Web Serial / 浏览器平台适配器 |
| `apps/desktop` | electron-vite 入口 + IPC 平台适配器；主进程退化为原始字节串口传输 |
| `tools/lint` | 独立 ESLint 工具链（typescript-eslint 暂不兼容 TS 7，单独使用 TS 5.9） |

桌面端与浏览器端的行为差异（端口枚举、自动重连、系统媒体权限、自动更新）全部收敛到平台适配器。

## 2. 依赖升级清单（相对 1.1.4）

| 依赖 | 1.1.4 | 1.2.0 | 备注 |
| --- | --- | --- | --- |
| Electron | 35.7.5 | 45.4.1 | 主进程 API/权限模型回归见验证结论 |
| React / React DOM | 18.3.1 | 19.3.1 | 移除旧版 `ReactDOM.render` 等用法 |
| antd | 5.29.3 | 6.7.1 | 废弃 API 已按 v6 调整；`StyleProvider layer` 与 Tailwind 层顺序修复 |
| @ant-design/icons | 5.6.1 | 6.3.4 | 随 antd 6 |
| jotai | 2.12.1 | 3.0.2 | atom 用法保持兼容 |
| i18next | 24.2.2 | 27.2.1 | 语言资源结构保持兼容 |
| react-i18next | 15.4.1 | 18.0.2 | 随 i18next 升级 |
| lucide-react | 0.476.0 | 1.75.0 | 图标 API 兼容 |
| electron-vite | 4.0.1 | 6.0.0 | main/preload/renderer 三段构建 |
| Vite | 桌面 6.1.0 / 浏览器 7.3.1 | 8.3.8 | 两端统一 |
| TypeScript | 5.7.3 / ~5.6.2 | 7.0.2 | 依赖 exact 版本；lint 工具链单独保留 TS 5.9 |
| Tailwind CSS | 桌面 4.0.6 / 浏览器 3.4.18 | 4.3.5 | 两端统一 v4，补充显式 `@source` |
| Node.js | 22.x | 24 LTS | `.nvmrc` / `engines` 同步 |
| 包管理器 | npm | pnpm 12.21.0 | 单一 lockfile + workspace |
| electron-builder | 26.0.12 | 26.18.0 | 三平台打包配置收敛 |
| electron-updater | 6.3.9 | 6.8.11 | 自动更新接口保持兼容 |
| @electron/notarize | 2.5.0 | 3.1.2 | 公证 API 调整 |
| ESLint | 9.39.2 | 10.15.0 | 全仓统一配置 |
| @types/node | 22.13.4 | 26.6.6 | 与 Node 26 开发环境对齐 |

> 说明：升级清单按 `package.json` 的 1.1.4 与 1.2.0 实际版本整理；其余运行时依赖（`serialport`、`react-responsive`、`react-simple-keyboard`、`vaul` 等）只做版本对齐或保持兼容。

## 3. 合并前修复

### 必修

| 问题 | 修复 |
| --- | --- |
| 老用户升级后设置全部丢失 | `apps/desktop/electron-builder.yml` 增加 `extraMetadata.name: nanokvm-usb`，主进程最早处 `app.setName('nanokvm-usb')`；Electron 的 `userData` 目录恢复为 1.1.4 的 `nanokvm-usb`，语言/视频设备/分辨率/串口/快捷键等设置不再回默认 |
| 网页版 Docker 镜像构建失败 | `Dockerfile` 的 `COPY` 增加根 `tsconfig.base.json`；`apps/browser/tsconfig.json` 的 `extends` 可解析 |
| 先开 App 再插设备，选不中上次保存的串口 | `serialPortAtom` 初值改为空，保存的端口只用于自动重连；下拉框打开时刷新端口列表；打开失败/切换波特率失败时清空选中值，允许再次选择同一端口 |

### 应修

| 问题 | 修复 |
| --- | --- |
| 桌面端全屏按钮没反应 | 主进程权限白名单补充 `fullscreen`、`keyboardLock` |
| 设备弹窗视频下拉框与串口行无间距 | `packages/ui/src/styles/index.css` 把层顺序声明移到 `@import 'tailwindcss'` 之前：`@layer theme, base, components, antd, utilities;`，构建产物中 antd 层位于 utilities 之前，Tailwind 边距/颜色恢复 |
| 桌面主进程串口接收缓冲只进不出 | `SerialPort` 接收缓冲增加 1 MiB 硬上限，溢出时丢弃最旧字节；正常 14 字节协议读取不受影响 |
| 网页版已连接时重选串口、取消被当成断开 | 先执行 `request()`，用户取消（`NotFoundError` / `No port selected`）时恢复原连接状态且不报错；只有真正 `open()` 失败才置为断开 |
| 发布 tag 版本与安装包版本无校验 | release workflow 校验根 `package.json`、`apps/desktop/package.json`、`apps/browser/package.json` 三处版本一致 |
| 已存在 tag 会复用并覆盖已发布资产 | 手动发布时 tag 已存在直接失败；发布前检查 GitHub Release 是否已存在，已发布则拒绝继续；`softprops/action-gh-release` 显式 `overwrite_files: false` |
| mac 缺签名证书时静默跳过 | release workflow 在 macOS job 上校验 `APPLE_CERTIFICATE` / `APPLE_CERTIFICATE_PASSWORD` / `APPLE_ID` / `APPLE_APP_SPECIFIC_PASSWORD` / `APPLE_TEAM_ID`，缺失直接失败 |
| 证书 secrets 暴露范围变大 | 移除 job 级 secrets env；macOS 证书/账号只在 macOS 的校验/导入/构建 step 注入，Windows 证书先检测再只注入 Windows 构建 step |
| 串口打开失败不显示具体原因 | 错误信息改为 `Failed to connect serial: <具体原因>`，桌面 IPC 与 Web Serial 的底层错误都会带出 |
| 相对鼠标提示每次重渲染都弹 | 提示移入 `useEffect` 并加一次性 ref 守卫，只在进入相对模式时弹一次 |

## 4. 验证结论

验证环境：Linux（WSL2）/ Node 26.13.1 / pnpm 12.21.0。硬件相关项目本机无法覆盖，见下节。

| 检查项 | 结果 |
| --- | --- |
| `pnpm install --frozen-lockfile` | ✅ lockfile 与 manifest 一致 |
| `pnpm typecheck` | ✅ core / ui / browser / desktop(node + web) 全绿 |
| `pnpm lint` | ✅ 0 error，23 warning（均为 react-hooks v7 既有规则告警，未新增） |
| `pnpm format:check` | ✅ 全部文件符合 Prettier |
| `pnpm test` | ✅ `packages/core` 协议单测 16 例通过 |
| `pnpm build:browser` | ✅ Vite 8.3.8 产物生成 |
| `pnpm build:desktop` | ✅ electron-vite 6 产物生成 |
| `electron-builder --dir --publish never` | ✅ Linux unpacked 打包成功；`app.asar` 内 `package.json.name === "nanokvm-usb"` |
| CSS 层顺序 | ✅ 浏览器/桌面构建产物均含 `@layer components, antd;` 且位于 `@layer utilities` 之前 |
| Docker 构建 | ✅ `docker build -t nanokvm-usb-web:pr-check .` 成功（前端构建 + nginx 镜像） |
| macOS 签名/公证 | ⏳ 需在 release workflow + 真实证书环境验证 |
| 硬件/GUI 冒烟 | ⏳ 摄像头、串口热插拔/自动重连、全屏 + 键盘锁、相对鼠标提示、Web Serial 取消选择、快捷键等需按手工清单在真机复测 |

## 5. 待人工验证清单

1. 老版本（1.1.4）升级安装后，语言/视频设备/分辨率/串口/快捷键设置仍保留。
2. 先启动 App 再插入设备，能选中上次保存的串口；仅一个串口时连接成功后弹窗可关闭。
3. 桌面端点全屏按钮可进入/退出全屏，快捷键捕获正常。
4. 设备弹窗视频下拉框与串口行间距正常，antd 组件上的 Tailwind 边距/颜色正常。
5. 网页版已连接时重选串口并取消，连接状态保持；取消不显示错误。
6. 串口打开失败时能看到具体原因（如权限、端口占用、路径不存在）。
7. 相对鼠标模式只弹一次操作提示，重渲染不再重复弹出。
8. 长时间串口运行（含异常上报）内存不持续增长。
9. release workflow 在版本不一致、tag 已存在、release 已存在、mac secrets 缺失时均能失败阻断。

## 6. 风险与回滚

- **Electron 45 / React 19 / antd 6** 属于大版本升级，已通过类型检查、构建与协议单测；视觉与硬件交互仍需真机回归。
- **pnpm workspace** 改变了安装/打包拓扑，`electron-builder` 已改用 `pnpm` 检测；回滚时需同时恢复 lockfile 与 workspace 配置。
- **发布流程** 改为“缺 mac 证书即失败”，首次使用前需在仓库 secrets 中配齐 5 个 Apple 凭据；这是有意的防呆变更。
- 若需回滚单点修复，`extraMetadata.name` / `app.setName`、Dockerfile COPY、CSS 层顺序、串口缓冲上限均可独立回退，不影响其他改动。
