# 为 Swagger UI 添一盏“夜灯”：类 Dracula 深色主题与「返回顶部」的优雅实现

> 白天写接口，夜里查文档。
>
> Swagger UI 很好用，但默认配色在深夜的屏幕前总显得有点刺眼；而当接口列表变得很长时，滚动回到页面顶端也会变成一种“重复劳动”。
>
> 这篇文章分享一套**对 swagger-ui 原生样式“足够克制”**的深色主题方案，并顺手加上一颗“返回顶部”按钮：
>
> - 深色主题采用 Dracula 配色，**只覆盖颜色，不碰布局**，最大程度避免升级 swagger-ui 版本后的样式错位。
> - 主题切换支持 localStorage 记忆，同时尊重系统 `prefers-color-scheme`。
> - 返回顶部按钮带平滑滚动，并对滚动事件做了节流，避免性能抖动。

本文示例基于你现有的三个文件：

- `swagger-ui/SwaggerModule.cs`：在 Swagger UI 中注入 CSS 与 JS
- `swagger-ui/custom.css`：Dracula 深色主题（颜色覆盖）+ 两个按钮样式
- `swagger-ui/theme-toggle.js`：主题切换 + 返回顶部（含节流与无障碍属性）

---

## 目标：不“改造”，只“点缀”

给第三方 UI 加主题最容易踩的坑，不是颜色不好看，而是：

1. **覆盖了布局类样式**：例如 `display / position / margin / padding` 等，一不小心就把 swagger-ui 原本的布局逻辑打碎。
2. **选择器过于宽泛**：改了某个 `.btn`，却影响了其他组件；或者 swagger-ui 版本升级后 DOM 结构变了，结果主题失效或错位。

所以这套方案从一开始就定下原则：

- **浅色模式完全不动 swagger-ui 原生样式**。
- 深色模式通过 `data-theme="dark"` 做“作用域开关”，并以 CSS 变量集中管理颜色。
- JS 只做两件事：
  - 设置 `data-theme`（并持久化）
  - 注入两个按钮（主题切换 / 返回顶部）

---

## 第一步：在 Swagger UI 中注入你的 CSS 与 JS

你在 `SwaggerModule.cs` 里做的事情非常直接：在 `UseSwaggerUI` 配置中注入静态资源。

关键代码如下（保持原样即可）：

```csharp
app.UseSwaggerUI(c =>
{
    // ... SwaggerEndpoint 等配置

    c.RoutePrefix = "swagger";

    // 设置 Swagger UI 支持深色模式
    c.InjectStylesheet("/swagger-ui/custom.css");

    // 注入主题切换脚本
    c.InjectJavascript("/swagger-ui/theme-toggle.js");
});
```

### 一个很现实的小提醒：静态文件如何被访问？

上面注入的路径是：

- `/swagger-ui/custom.css`
- `/swagger-ui/theme-toggle.js`

这意味着运行时必须能通过 HTTP 访问到这两份文件。

常见做法是将它们放在 `wwwroot/swagger-ui/` 下，并确保启用了 `app.UseStaticFiles()`。

更具体一点：在 ASP.NET Core（.NET Core）里，如果你使用的是默认静态文件中间件，那么注入路径与磁盘路径的对应关系通常是：

- `c.InjectStylesheet("/swagger-ui/custom.css")` → `wwwroot/swagger-ui/custom.css`
- `c.InjectJavascript("/swagger-ui/theme-toggle.js")` → `wwwroot/swagger-ui/theme-toggle.js`

也就是说，你现在这套方案推荐把这两个文件**放入项目的** `wwwroot/swagger-ui/` 目录下。

> 如果你不是用默认的 `wwwroot`，也可以通过自定义 `StaticFileOptions` 将该目录映射到 `/swagger-ui/`。

---

## 第二步：用 `data-theme` 做“开关”，让主题像呼吸一样自然

在 `custom.css` 中，你使用了这个非常稳妥的入口：

```css
[data-theme="dark"] {
  --bg-primary: #282a36;
  --bg-secondary: #44475a;
  --border-color: #6272a4;
  --text-primary: #f8f8f2;
  --accent-primary: #bd93f9;
  /* ... 省略：Dracula 配色变量 ... */
}
```

随后所有深色样式都挂在 `[data-theme="dark"]` 作用域下，例如：

- 全局背景与文字
- topbar
- opblock（并对 GET/POST/PUT/DELETE/PATCH 做了对应的 Dracula 方法色）
- 表格与 model
- code block、markdown、modal 等

### 为什么强调“只覆盖颜色”？

你在 CSS 顶部写的注释，几乎可以当作主题工程的“宪法”——

> 宗旨：只覆盖颜色，不触碰布局、间距、尺寸，以确保与原始 swagger-ui.css 完美兼容。

这会带来两个直接收益：

1. **升级 swagger-ui 的成本更低**：DOM 微调时，只要类名还在，颜色就继续生效。
2. **减少视觉抖动与错位**：布局不动，用户就不会遇到“某些按钮突然飘走”的惊吓。

### 让 HTTP 方法色一眼可辨

你用半透明背景 + 边框色做了 GET/POST/... 的区别：

```css
[data-theme="dark"] .opblock.opblock-get {
  border-color: var(--method-get);
  background: rgba(139, 233, 253, 0.15);
}
```

这种“低饱和强调”在深色背景上非常耐看：既能一眼分辨，又不会抢走内容本身的注意力。

---

## 第三步：JS 让主题“记得你”，也让页面“照顾你”

你的 `theme-toggle.js` 做得相当完整，核心点有三个：

1. **localStorage 记忆**（键名为 `swagger-ui-theme`）
2. **系统主题兜底**（`prefers-color-scheme: dark`）
3. **按钮注入与无障碍**（`aria-label` / `title`）

### 主题判定逻辑：尊重用户，也尊重系统

```js
const THEME_KEY = "swagger-ui-theme";

function getCurrentTheme() {
  const saved = localStorage.getItem(THEME_KEY);
  if (saved) return saved;

  if (
    window.matchMedia &&
    window.matchMedia("(prefers-color-scheme: dark)").matches
  ) {
    return "dark";
  }

  return "light";
}
```

当用户明确选择过主题时，就“听用户的”；
没选择过，就“听系统的”。这会让体验显得很有礼貌。

### 主题切换的关键：给 `<html>` 上 `data-theme`

```js
function setTheme(theme) {
  const html = document.documentElement;

  if (theme === "dark") {
    html.setAttribute("data-theme", "dark");
  } else {
    html.removeAttribute("data-theme");
  }

  localStorage.setItem(THEME_KEY, theme);
  updateThemeButtonText(theme);
}
```

这里的优点是：CSS 只需要关心 `[data-theme="dark"]`，而不用把逻辑散落到各处。

---

## 额外加分项：把“返回顶部”做成一个不会打扰人的按钮

你在 JS 里注入了 `back-to-top` 按钮，默认隐藏，滚动超过 200px 才出现：

```js
const handleScroll = () => {
  if (window.scrollY > 200) {
    button.style.display = "block";
  } else {
    button.style.display = "none";
  }
};
```

点击后平滑滚动：

```js
window.scrollTo({ top: 0, behavior: "smooth" });
```

### 滚动监听为什么要节流？

滚动事件触发频率很高，如果每一帧都执行 DOM 更新，某些机器上容易出现掉帧。

你用一个简单的 `throttle` 把频率限制在每 100ms 一次：

```js
window.addEventListener("scroll", throttle(handleScroll, 100));
```

这属于“看不见的性能优化”，但用户会用脚投票。

---

## UI 细节：两个按钮的样式，和主题保持同一套语气

在 `custom.css` 中，你分别定义了：

- `.theme-toggle-btn`：固定在右上角，深色模式使用 `--bg-tertiary` 和 `--text-primary`
- `.back-to-top-btn`：固定在右下角，圆形按钮，hover 有轻微上浮

并且两者都在 `[data-theme="dark"]` 下做了配色适配，让“暗黑模式”不只是把背景涂黑，而是整体风格统一。

---

## 常见坑位清单（建议收藏）

1. **按钮注入时机**：Swagger UI 的 `.topbar` 可能不是立刻存在。
   - 你通过 `querySelector(".topbar") + setTimeout 重试` 处理了这个问题。
2. **避免重复注入**：
   - 你在注入前检查了 `document.getElementById("theme-toggle-btn")` / `back-to-top-btn`。
3. **不要把浅色主题“改坏”**：
   - 你的 CSS 做到了“浅色零改动”，这是最安全的方式。

---

## 小结：少一点侵入，多一点体贴

这一套实现的妙处在于：

- **颜色用变量集中管理**，主题一致，维护轻松。
- **通过 `data-theme` 限定作用域**，浅色模式不背锅。
- **按钮注入与滚动节流**，功能有了，体验也不打折。

你最终得到的是一个“在深夜也温柔”的 Swagger UI：

- 看起来更舒服
- 用起来更顺手
- 维护起来更安心

---

## 附：本文相关文件

- `swagger-ui/SwaggerModule.cs`
- `swagger-ui/custom.css`
- `swagger-ui/theme-toggle.js`

---

## 题外话：官方暗黑模式正在路上

值得期待的是：**新版 Swagger UI 已经在上游官方合并了暗黑模式相关的 PR**。等 `Swashbuckle.AspNetCore` 在后续版本跟进更新 Swagger UI 依赖后，很多项目可能就能“开箱即用”地获得官方暗黑模式支持。

在那之前，这篇文章的方案依然很实用：它侵入性低、可控性强，也便于你按团队审美做微调。

如果你愿意再精致一点，下一步可以考虑：

- 让主题按钮支持“文字 + 图标”两种显示（移动端只显示图标）
- 把 `Back to Top` 的显示/隐藏从 `display` 改为 `opacity + pointer-events`，过渡会更丝滑
- 为代码块引入更细腻的语法高亮（在不破坏 swagger-ui 自身样式的前提下）
