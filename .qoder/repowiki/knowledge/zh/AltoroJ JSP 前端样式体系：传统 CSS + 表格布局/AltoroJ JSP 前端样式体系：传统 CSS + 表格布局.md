---
kind: frontend_style
name: AltoroJ JSP 前端样式体系：传统 CSS + 表格布局
category: frontend_style
scope:
    - '**'
source_files:
    - WebContent/style.css
    - WebContent/header.jspf
    - WebContent/footer.jspf
    - WebContent/bank/membertoc.jspf
    - WebContent/swagger/css/screen.css
    - WebContent/swagger/css/reset.css
    - WebContent/swagger/css/style.css
---

## 1. 采用的系统/方法

- **纯 CSS 文件**，无预处理器（无 SCSS/Less）、无 CSS-in-JS、无组件库。唯一的全局样式入口是 `WebContent/style.css`。
- **JSP + JSP Fragment (`.jspf`)** 作为模板机制：`header.jspf` / `footer.jspf` 被各页面通过 `<%@ include file="..." %>` 复用，形成统一的头部导航与页脚。
- **布局基于 HTML `<table>` + 内联 `style="..."` 属性**，而非现代 Flexbox/Grid；响应式能力基本不存在，仅用百分比宽度（如 `width: 99%`）做简单缩放。
- 页面声明 XHTML 1.0 Transitional DOCTYPE：`<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" ...>`。
- Swagger UI 自带一套独立的 `swagger/css/*.css` 与字体资源，属于第三方静态嵌入，与主应用样式隔离。

## 2. 关键文件

- `WebContent/style.css` — 全局样式表，定义基础排版、颜色、通用 class。
- `WebContent/header.jspf` — 公共头部片段，引入 `style.css`，包含 Logo、搜索框、四栏导航。
- `WebContent/footer.jspf` — 公共页脚片段，包含隐私链接、免责声明等。
- `WebContent/bank/membertoc.jspf` — 网银侧边栏导航片段。
- `WebContent/swagger/css/{screen,print,reset,style,typography}.css` — Swagger UI 的独立样式。
- 各 `.jsp` 页面（`login.jsp`, `index.jsp`, `bank/*.jsp`, `admin/*.jsp`, `feedback*.jsp` 等）—— 通过 `<link href=".../style.css">` 引用全局样式。

## 3. 架构与约定

- **单文件全局样式**：所有页面统一引用 `style.css`，没有按模块/页面拆分的 CSS 文件（除 Swagger）。
- **命名空间极小**：全局类名全部为短标识符且语义模糊，如 `.focus`、`.credit`、`.bt`、`.br`、`.bb`、`.cc`、`.fl`、`.flp`、`.disclaimer`、`.err`。这些类在 `style.css` 中集中定义，并在多个 JSP 中重复使用。
- **布局约定**：
  - 每个页面外层包裹 `<div id="wrapper" style="width: 99%;">`。
  - 内容区使用 `<div class="fl" style="width: 99%;">` 左浮动。
  - 顶部导航采用 4 列 `<table>`，每列 `width="25%"`，并用 `.bt .br .bb .cc` 组合实现边框与背景色。
- **颜色/字体约定**（来自 `style.css`）：
  - 正文：`Verdana, Arial, Sans-Serif`，字号 10px，行高 1.6。
  - 主标题 `h1`：`#00796C`（深青色），22px。
  - 链接：`#5811B0`（紫色），hover 时变为 `#00796C`。
  - 导航背景块：`.cc` 背景 `#BFD7DA`（浅蓝灰）。
  - 边框色统一使用 `#5811B0`（`.bt/.br/.bb`）。
  - 错误提示：`.err` 背景 `#FFD0D0`。
- **模板复用**：`header.jspf` 和 `footer.jspf` 通过 JSP include 注入到业务页面，保证站点级一致的外观与版权信息。
- **Swagger UI 隔离**：`WebContent/swagger/` 下是完整的第三方文档界面，自带 CSS/JS/fonts/images，不依赖 `style.css`。

## 4. 约定与约束

- **样式来源单一**：业务页面通过 `<link href="<%=request.getContextPath()%>/style.css" rel="stylesheet" type="text/css" />` 引入全局样式，未见其他外部 CSS 框架或 CDN 引用。
- **布局策略**：页面结构以 `<table>` 为主，配合 `float:left`（`.fl`）完成两栏布局；未发现媒体查询或响应式断点。
- **类名风格**：全局类名均为短字母缩写（`.bt` = border-top, `.br` = border-right, `.bb` = border-bottom, `.cc` = background-color），由 `style.css` 统一定义，跨页面共享。
- **内联样式**：大量布局尺寸（如 `width: 99%`、`margin-top:5px`）直接写在 JSP 的 `style="..."` 属性中，未抽离到 CSS 类。
- **XHTML 1.0 Transitional**：所有 JSP 页面继承 `header.jspf` 中的 DOCTYPE，保持 XHTML 兼容写法。
- **Swagger 资源独立**：`swagger/` 目录下的样式与脚本完全自包含，不与主应用样式耦合。

总体而言，这是一个典型的早期 Web 2.0 风格：单文件 CSS + JSP 片段模板 + 表格布局 + 少量通用 class，没有设计令牌系统、CSS 模块化或现代响应式方案。