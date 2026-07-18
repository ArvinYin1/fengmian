# 封面工坊 (Cover Studio)

> 一个基于 Canvas 2D 的单文件静态网页应用，用于快速生成中文社交媒体封面图。
> 部署地址：https://fengmian.yin.wiki

---

## 项目概述

本项目是一个零依赖的纯前端工具，所有代码（HTML + CSS + JavaScript）集中在一个 `index.html` 文件中。用户可以通过左侧控制面板调整背景图、标题、副标题、字体、颜色、遮罩及装饰线等参数，右侧实时预览 Canvas 渲染结果，并一键导出 PNG。

核心功能包括：
- 背景图上传（支持点击、拖拽、剪贴板粘贴，亦支持视频选帧）
- 三套一键预设模板：杂志编辑 / 大字冲击 / 极简文艺
- 多行主标题与副标题编辑
- 7 款开源中文字体切换（得意黑、优设标题黑、抖音美好体等）
- 文字效果二选一：平面（默认，柔和投影）/ 3D 挤压（离屏 Canvas 超采样渲染）
- 标题大小、垂直位置（顶/中/底）、对齐（左/居中）可调
- 双向暗色遮罩：左侧水平渐变 + 底部垂直压暗，均可调节
- 副标题（kicker）四种样式：标签色块 / 侧线 / 横线 / 纯文字，留空可隐藏
- 16:9 / 4:3 / 3:4（小红书竖版）三种输出比例
- 高清 PNG 导出（宽度 1500–1600px，随比例变化）

---

## 技术栈

- **纯静态页面**：无任何构建工具、框架或包管理器
- **Canvas 2D API**：所有视觉内容通过 `canvas.getContext('2d')` 实时绘制
- **中文字体**：通过 unpkg CDN 加载 `@chinese-fonts` 系列字体包
- **部署**：GitHub Pages + 自定义域名（`CNAME` 文件指向 `fengmian.yin.wiki`）

---

## 项目结构

```
.
├── index.html          # 唯一源文件，包含全部 HTML / CSS / JS
├── CNAME               # GitHub Pages 自定义域名配置
├── .claude/            # Claude 本地权限配置（可忽略）
└── .git/               # Git 版本控制
```

> 注意：本项目没有 `package.json`、`pyproject.toml`、`Cargo.toml` 或其他构建配置文件。

---

## 本地开发

由于项目是完全静态的，使用任意 HTTP 服务器即可本地预览：

```bash
# 方式一：Python 内置服务器
python3 -m http.server 7654

# 方式二：Node.js npx
npx serve .

# 方式三：VS Code Live Server 插件
```

然后打开 http://localhost:7654 即可。

**修改代码后直接刷新浏览器即可生效，无需构建步骤。**

---

## 代码组织

`index.html` 内部按顺序分为三大块：

1. **`<head>` + `<style>`**（第 1–828 行）
   - CSS 变量定义暗色主题色彩系统
   - 布局：Header + 左右分栏（左侧控制面板 `380px`，右侧预览区自适应）
   - 控件样式：输入框、滑块、颜色选择器、字体芯片、预设芯片、切换按钮、下载按钮等
   - `shot-mode`：隐藏界面、只留画布的截图模式（配合 URL hash 使用）
   - 响应式：屏幕宽度小于 `900px` 时切换为上下堆叠布局

2. **`<body>` 结构**（第 830–1073 行）
   - `<header>`：品牌标识与实时渲染状态指示器
   - `<aside class="controls">`：预设模板 + 15 组参数控制区
   - `<section class="preview">`：Canvas 预览区与尺寸信息浮层

3. **`<script>`**（第 1075–1811 行）
   - `TITLE_FONTS`：字体配置表（字体栈、字重、尺寸修正系数）
   - `state`：全局状态对象，保存所有可编辑参数
   - `PRESETS`：三套预设模板（各自是一组打包好的 `state` 参数）
   - `render()`：主渲染循环，按顺序绘制背景 → 双向遮罩 → 副标题（kicker）→ 主标题
   - `drawFlatText()`：平面文字渲染（默认）；`draw3DText()`：可选的 3D 挤压效果
   - `applyPreset()` / `syncControls()`：预设应用与控件状态同步
   - 事件监听：文件上传、输入框、滑块、颜色、字体芯片、分段开关、比例切换、下载按钮
   - URL hash 深链：`#preset=…&ratio=…&bg=…&shot=1`，供 headless 截图验证使用
   - 拖拽与剪贴板粘贴支持

---

## 关键实现细节

### 文字渲染（`drawFlatText` / `draw3DText`）

默认使用 `drawFlatText` 平面渲染：纵向微渐变填充 + 柔和投影，干净耐看。

3D 挤压为可选效果（`state.textEffect = '3d'`）。为实现平滑的立体阴影边缘，采用 **2× 超采样**策略：
1. 在高分辨率离屏 Canvas 上绘制文字形状；
2. 在同一分辨率下逐像素偏移叠加，生成实心挤压体；
3. 通过 `drawImage` 缩小回主 Canvas，利用双线性过滤将阶梯边缘平滑为斜面；
4. 最后绘制带有上下渐变的前层面与细轮廓线。

### 遮罩

两层叠加：左侧水平线性渐变（浓度、范围可调）+ 底部垂直压暗渐变（可调），保证文字区在任意背景上都有足够对比度。

### 字体加载

页面顶部通过 `<link>` 预加载所有中文字体 CSS。首次渲染在 `document.fonts.ready` 完成后触发，避免字体未就绪导致的文字测量偏差。

### 背景图适配

使用 `drawImageCover` 实现类似 CSS `object-fit: cover` 的居中裁剪逻辑，确保不同比例上传图都能填满画布。

---

## 修改注意事项

- **单文件约束**：所有新增样式和逻辑必须写在 `index.html` 的对应区块内，不要引入外部文件（除非部署环境允许）。
- **字体来源**：中文字体均来自 `unpkg.com/@chinese-fonts`。如需新增字体，需在 `<head>` 引入对应 CSS，并在 `TITLE_FONTS` 中添加配置项。
- **预设模板**：新增或调整风格时修改 `PRESETS` 表（每组是一 bundle 的 `state` 参数），应用后由 `syncControls()` 同步到所有控件。
- **Canvas 尺寸**：16:9 为 `1600×900`，4:3 为 `1600×1200`，3:4 为 `1500×2000`。修改 `state.ratios` 即可调整输出分辨率。
- **颜色系统**：主题色变量集中在 `:root`，主强调色为 `#ffcb05`（经典金黄）。
- **浏览器兼容**：依赖 `document.fonts.ready`、`<input type="color">`、ES6 语法。目标环境为现代浏览器（Chrome / Edge / Safari / Firefox 最新两版）。

---

## 测试与部署

- **无自动化测试**：本项目为视觉工具，目前依赖手动测试。修改后应在不同浏览器验证排版与导出结果。
- **截图自查**：启动本地服务器后，可用 headless Chrome 配合 URL hash 深链核对渲染效果（`shot=1` 只留画布）：

  ```bash
  "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
    --headless=new --screenshot=/tmp/cover.png --window-size=1600,900 \
    --user-data-dir=/tmp/chrome-shot --virtual-time-budget=15000 \
    "http://localhost:7654/index.html#preset=impact&bg=demo-bg.jpg&shot=1"
  ```
- **部署流程**：
  1. 将修改提交到 `main` 分支；
  2. GitHub Pages 会自动从 `main` 分支部署；
  3. 自定义域名由 `CNAME` 文件自动配置，无需额外操作。

---

## 安全与隐私

- 所有图片处理均在浏览器本地完成（`FileReader` + Canvas），**不会上传到任何服务器**。
- 无用户追踪脚本、无 Cookie、无后端接口。
