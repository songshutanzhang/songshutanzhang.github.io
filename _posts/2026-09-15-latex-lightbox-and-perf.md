---
layout: post
title: '给博客加上 LaTeX 公式与灯箱：一次踩坑记录'
subtitle: 'MathJax · Fancybox · 静态站性能优化'
date: 2026-09-15 22:00:00 +0800
author: 松鼠探长
tags:
- LaTeX
- MathJax
- Fancybox
- 性能优化
- 前端
---

前些时间做 MNIST 手写数字识别，用到了很多数学公式，而这个网站基于 H2O-ac 主题搭建，对于 LATEX 的支持并不理想，结果就是要去专门的转义网站把 LATEX 源代码转化成 SVG 公式图片。

# 一、先说结论：效果

做完之后大概是这样：

- 正文公式支持 `$...$`、`$$...$$`、`\[...\]`，行内字号 1.65 倍、独立公式 2 倍，亮暗色自动适配
- 点击任意公式（或图片）打开灯箱，公式与图片共用同一个画廊，可以左右翻页，带缩略图条、计数器、全屏
- 灯箱里可以**一键复制 LaTeX 源码**，可以**保存为 SVG / PNG**，也可以**保存原图**
- 文章首屏图片传输从 **11.2 MB 降到 78 KB**

# 二、公式渲染：为什么选了 SVG 输出

&emsp;&emsp;MathJax 3 有两种输出：`tex-mml-chtml`（HTML + 字体）和 `tex-svg`（纯 SVG 路径）。主题原来用的是 CHTML，但它有两个问题：

1. CHTML 的公式由大量带样式的 `<mjx-*>` 元素拼出来，**颜色依赖一堆 CSS 变量**，暗色模式要改几十个变量；
2. 想做一个"点击公式看大图"的功能时，CHTML 的节点树很难直接拿出来当图片用。

&emsp;&emsp;换成 SVG 输出之后，每个公式就是一个 `<svg>`，笔画的颜色是 `currentColor`，暗色模式只要改一处：

```css
.markdown-body {
    --mathjax-color: #24292f;   /* 明亮模式 */
}
.night-mode .markdown-body {
    --mathjax-color: #e6e8eb;   /* 暗色模式 */
}
.markdown-body mjx-container {
    color: var(--mathjax-color);
}
```

&emsp;&emsp;另外主题的 `markdown.scss` 里有一条历史遗留样式：

```css
.markdown-body mjx-math {
    overflow-x: auto;
    width: 100% !important;
}
```

&emsp;&emsp;它会把**行内公式**也撑满整段宽度，导致公式后面的文字被迫折行。这个必须覆盖掉：

```css
.markdown-body mjx-container mjx-math,
.markdown-body mjx-math {
    width: auto !important;
    overflow: visible;
}
```

# 三、公式源码从哪来：一个绕不开的坑

&emsp;&emsp;要做"复制 LaTeX 源码"，就得拿到公式的**原始 LaTeX 字符串**。这里踩了两个坑。

**坑一：原始节点会被 MathJax 吃掉。**

&emsp;&emsp;本以为是先有 `<script type="math/tex">` 再由 MathJax 渲染成公式。实际用 Kramdown 时**根本不会生成这种节点**——Markdown 里的 `$E=mc^2$` 原样输出成文本，MathJax 直接从文本节点里解析并替换掉。所以「在 MathJax 之前抢下源码」这条路的窗口期极短。

**坑二：MathJax 的内部公式列表会被清空。**

&emsp;&emsp;试过读 `MathJax.startup.document.math`，它在不同时机返回的结构还不一样（数组 / `{list:[]}` / `{math:[]}`），而且排版结束后会被 `clearMathItems()` 清掉。更麻烦的是，**如果自己接管 `renderActions.find` 而忘记把结果登记回 `doc.math`，公式会直接渲染失败**，报 `s.item.renderMath is not a function`。

&emsp;&emsp;最后的方案是**自己扫文本**，不依赖 MathJax 任何内部结构：

```js
// 收集带公式的文本节点（跳过代码块，与 MathJax 的 skipHtmlTags 一致）
function collectMathTextNodes(root) {
    var out = [];
    var skip = { SCRIPT: 1, NOSCRIPT: 1, STYLE: 1, TEXTAREA: 1, PRE: 1, CODE: 1 };
    (function walk(node) {
        if (node.nodeType === 3) {
            var v = node.nodeValue;
            if (v && (v.indexOf("$") >= 0 || v.indexOf("\\(") >= 0 || v.indexOf("\\[") >= 0)) out.push(node);
            return;
        }
        if (node.nodeType !== 1 || skip[node.tagName]) return;
        for (var i = 0; i < node.childNodes.length; i++) walk(node.childNodes[i]);
    })(root);
    return out;
}
```

&emsp;&emsp;然后按文档顺序把解析出的源码，依次绑定到 MathJax 渲染出的 `mjx-container` 上（数量一致才绑，避免错位）：

```js
function bindSources() {
    var list = currentSources();
    var containers = document.querySelectorAll(".markdown-body mjx-container");
    if (containers.length !== list.length) return 0;   // 还没排完版，下次再试
    for (var i = 0; i < containers.length; i++) {
        texStore["mjx-src-" + (i + 1)] = list[i];
        containers[i].setAttribute("data-tex-id", "mjx-src-" + (i + 1));
    }
}
```

&emsp;&emsp;这里有个容易忽略的细节：扫描时判断文本节点**必须同时检查 `$` 和 `\[`**。我第一版只查了 `$`，结果所有 `\[...\]` 形式的独立公式全部漏掉，绑定数量对不上，功能静默失效。

# 四、公式图：viewBox 不能重算

&emsp;&emsp;为了把公式交给 Fancybox 展示，需要把它转成一张"图片"。做法是用官方的 `MathJax.tex2svgPromise()` 渲染一份独立 SVG，包成 data URI。

&emsp;&emsp;这里踩了个大坑：**想按公式的 ex 宽度重算 viewBox，结果公式整个跑出画布。**

&emsp;&emsp;MathJax 的 SVG 用的是它内部的坐标系统——**1000 units/em，y 轴向上，基线在 0**。它的 `viewBox` 长这样：

```
viewBox="0 -1008.2 3975.9 1090.2"
```

&emsp;&emsp;如果我按"公式宽度 8.995ex ≈ 4.5em"去重新设 `viewBox="-24 -24 257 98"`，就只覆盖了左下角一小块，而公式内容横跨到 x=6915、纵深到 y=-1061 —— **画布上只剩右下角一个小三角**，其余全是空白。

&emsp;&emsp;正确做法是**保留原始 viewBox，只用 width/height 做缩放**，需要留白时把 viewBox 向外扩：

```js
var box = svg.getAttribute("viewBox").split(/[\s,]+/).map(parseFloat);
var k = EXPORT_FONT / 1000;            // 1000 units/em → 目标字号
var pad = 28;                          // 四周留白（像素）
clone.setAttribute("width", Math.ceil(box[2] * k) + pad * 2);
clone.setAttribute("height", Math.ceil(box[3] * k) + pad * 2);
// 公式本体坐标不动，只把视野向外扩
clone.setAttribute("viewBox",
    (box[0] - pad / k) + " " + (box[1] - pad / k) + " " +
    (box[2] + (pad * 2) / k) + " " + (box[3] + (pad * 2) / k));
```

&emsp;&emsp;验证方式是拿 `getBBox()` 量出内容的真实边界，确认它完全落在 viewBox 内：

```
viewBox [-933,-1767,5712,2783]   bbox [-933,-1767,5712,2783]   ✅ 吻合
```

&emsp;&emsp;导出时**不加背景矩形**，得到的就是透明底的 SVG，可以随处粘贴；灯箱里则由 CSS 补一层白底：

```css
.fancybox__container.is-formula-slide .f-panzoom__content {
    background-color: #ffffff !important;
}
```

# 五、线稿图：让 SVG 也学会跟着主题换色

&emsp;&emsp;文章里除了公式，还有一批从 CodeCogs 导出的 SVG 线稿（矩阵、损失函数这类示意图）。它们的共同点是**黑线 + 透明底**，在暗色模式下直接糊在深色背景里。

&emsp;&emsp;最直接的办法是给它们垫一层白底。但那样页面上就会多出一堆突兀的白色方块，和正文完全不搭。

&emsp;&emsp;公式处理同样问题的方式是「**不换底，换色**」—— 公式线条走 `currentColor`，暗色模式下自动变浅，所以从来不需要衬底。那 SVG 能不能照做？

&emsp;&emsp;关键区别在节点形态：

```html
<!-- 作为图片引用：封闭文档，外部 CSS 碰不到里面的颜色 -->
<img src="/assets/img/bp.svg">

<!-- 内联进 HTML：就是普通节点，颜色可以被 CSS 接管 -->
<svg viewBox="..."><path d="..."/></svg>
```

&emsp;&emsp;所以思路很明确：**把 SVG 抓下来内联进页面**。查了一下这批文件，路径上确实没有写死颜色（没有 `fill` 属性，用默认的黑色），正好可以被 `currentColor` 接管：

```js
function inlineSvg(svgEl, source, onReady) {
    fetch(source)
        .then(function (r) { return r.ok ? r.text() : null; })
        .then(function (text) {
            var doc = new DOMParser().parseFromString(text, "image/svg+xml");
            var svg = doc.documentElement;
            // 换成随主题变化的颜色
            svg.setAttribute("fill", "currentColor");
            svg.setAttribute("stroke", "currentColor");
            svg.removeAttribute("width");
            svg.removeAttribute("height");
            svgEl.parentNode.replaceChild(svg, svgEl);
            if (onReady) onReady(svg, text);
        })
        .catch(function (err) {
            console.warn("[svg] 内联失败，保留原图：", source, err);
        });
}
```

&emsp;&emsp;配套的 CSS 只需要一行变色，不需要任何底色。但这里有个**差点翻车的细节** —— 一开始我图省事按 `role="img"` 选：

```css
/* 有问题：MathJax 公式的 SVG 同样带 role="img" */
.markdown-body svg[role="img"] {
    display: block;              /* ← 把行内公式也变成了块级 */
    color: var(--mathjax-color, #24292f);
    fill: currentColor;
    stroke: currentColor;
}
```

&emsp;&emsp;公式的 SVG 也是 `role="img"`，于是**行内公式被一起改成块级**，整行文字被公式硬生生断开。正确的做法是给内联线稿图一个专属类名，把两者区分开。

&emsp;&emsp;顺带发现另一个更隐蔽的问题：**笔画看起来变粗了**。原因是上面那句 `stroke: currentColor` —— 这些字形本来就是「用路径轮廓画出来的实心字」，原文件里根本没有 `stroke` 属性；额外加一层描边（`stroke-width` 默认 1px）叠在填充上，自然就胖了一圈。把 `stroke` 去掉即可：

```js
svg.setAttribute("fill", "currentColor");
svg.removeAttribute("stroke");      // 填充路径不需要描边
```

```html
<svg class="md-inline-svg" role="img"> … </svg>
```

```css
/* 块级居中只认这个类 */
.markdown-body svg.md-inline-svg {
    display: block;
    margin: 16px auto 10px;
    color: var(--mathjax-color, #24292f);
    fill: currentColor;      /* 只设 fill，不要设 stroke */
}

/* 行内公式必须保持行内 */
.markdown-body mjx-container svg {
    display: inline;
    margin: 0;
}
```

&emsp;&emsp;验证方式是比较公式与其后文字的垂直坐标差：正常同行时差 8px，强制 `display: block` 后变成 25px（断行）。实测效果：

| | 常规模式 | 黑夜模式 |
|---|---|---|
| 线条颜色 | `rgb(36, 41, 47)` | `rgb(230, 232, 235)` |
| 背景 | 透明 | 透明 |
| 行内公式与后文垂直差 | 8px（同一行） | 8px（同一行） |

&emsp;&emsp;还有一个尺寸问题。这些图导出时 `viewBox` 用的是 pt 量级的设计尺寸，如果一律 `width: 100%` 铺满正文列宽（695px），像 223 units 宽的小图就会被放大约三倍，看上去又大又空：

```js
// 设计宽度本身就比列宽还大的，才铺满；其余按原始比例来
if (box[2] > 695) {
    svg.style.width = "100%";
} else {
    svg.style.width = Math.round(box[2]) + "px";
}
```

&emsp;&emsp;调整后的实际宽度分布（列宽 695px）：

| 图片 | 调整前 | 调整后 |
|---|---|---|
| z.svg（734 units，宽扁矩阵） | 695px（100%） | 695px（100%） |
| softmax.svg（223 units） | 695px（100%） | **223px（32%）** |
| bp.svg（329 units） | 695px（100%） | **329px（47%）** |
| ReLU.svg（227 units） | 695px（100%） | **227px（33%）** |

&emsp;&emsp;灯箱里要另说。灯箱的图片压在**暗色遮罩**上，白底衬底是必需的，所以那里反过来：**白底 + 深色线条**。做成内联 SVG 的 HTML 幻灯片即可，衬底留白比正文宽松一些（18px），顺便要注意 Fancybox 会给 `.f-html` 幻灯片默认加一层白底，自己也要显式声明，免得依赖它的默认值：

```css
.fancybox__container .mlb-svg-slide,
.fancybox__container .mlb-svg-slide.f-html {
    background-color: #ffffff !important;
    color: #1f2328;              /* 白底上必须是深色线条 */
    padding: 18px;
    border-radius: 10px;
}
```

&emsp;&emsp;顺带解决了两个衍生问题：

**其一，缩略图退化成占位图。** SVG 变成 HTML 幻灯片后 Fancybox 取不到缩略图。解决办法是另外做一份**固定深色线条**的 data URI 专门当缩略图 —— 缩略图是白底，不能用跟随灯箱的浅色：

```js
svg.setAttribute("fill", "#222222");        // 缩略图固定深色
link.setAttribute("data-svg-thumb",
    "data:image/svg+xml;charset=utf-8," + encodeURIComponent(new XMLSerializer().serializeToString(svg)));
```

**其二，缩略图只剩中间一小截。** Fancybox 的缩略图默认是 `object-fit: cover`，而 SVG 线稿往往又宽又扁（比如 733×112），`cover` 会把两侧裁掉：

```css
.fancybox__container .fancybox__thumbs img[src^="data:image/svg+xml"] {
    object-fit: contain !important;    /* 完整显示 */
}
```

&emsp;&emsp;把这两处放在一起看，结论其实很清晰：**正文里的线稿靠换色，灯箱里的图靠衬底。** 前者不需要底，后者没有底就看不见。

# 六、灯箱：交给 Fancybox，别自己写

&emsp;&emsp;我一开始自己写了个公式灯箱，做了淡入、工具栏、缩略图……结果被一眼看穿："这跟主题里的图片灯箱不一致"。确实，主题的 `fancybox_js.html` 只有二十几行——**裸用 Fancybox 默认配置**，那套平移渐入、全屏、缩略图、前后翻页、计数器全都是 Fancybox 自带的。

&emsp;&emsp;于是思路改成一句话：**公式就是一张能复制代码的图片。** 把公式的 SVG 包成 `data:image/svg+xml` 交给同一个画廊，剩下的全白送。

&emsp;&emsp;途中还有一个 API 细节：`Fancybox.show()` **不会继承** `Fancybox.bind()` 的配置，得写进全局默认值：

```js
var OPTIONS = { Carousel: { Toolbar: { items: { /* 自定义按钮 */ } } } };
Object.assign(Fancybox.getDefaults(), OPTIONS);   // show() 与 bind() 共用
Fancybox.bind("[data-fancybox]", OPTIONS);
```

&emsp;&emsp;自定义按钮就是"复制 LaTeX"和"保存原图"，只在公式幻灯片上显示：

```css
.fancybox__container .f-button[data-mlb-copy] { display: none; }
.fancybox__container.is-formula-slide .f-button[data-mlb-copy] { display: inline-flex; }
```

&emsp;&emsp;公式的说明区单独一行居中放源码，复制按钮读的是独立的 `slide.tex` 字段，不会把说明文字带进去：

```js
caption: '<span class="mlb-formula-tex">' + escapeHtml(tex) + '</span>',
tex: tex,   // 复制用这个
```

# 七、一个 padding 引发的血案

&emsp;&emsp;灯箱修好之后遇到一个诡异现象：**少数 SVG 图片在灯箱里被压得极小**，而且放大缩小时锚点会飘。

&emsp;&emsp;原因是我想给透明 SVG 加个"白底卡片"，顺手写了：

```css
.fancybox__container.is-svg-preview .f-panzoom__content {
    padding: 18px;
    box-sizing: border-box;
}
```

&emsp;&emsp;而 Fancybox 是**按图片固有尺寸 + border-box 反推容器可用空间**的。以一张 439×37 的公式图为例：

```
wrapper: max-width: min(439px, 100%);  max-height: min(37px, 100%)
img:     439×37, box-sizing: border-box, padding: 18px
→ 高度方向 37 - 18×2 = 1px
```

&emsp;&emsp;所以图片被压成 48×37，又扁又小的图受害最重；同时 padding 改变了元素的盒模型，panzoom 计算缩放锚点时用的中心点也跟着偏了。

**结论：灯箱内的图片只允许改 `background-color`，不要碰 padding / border / border-radius。**

# 八、性能优化：图片分档

&emsp;&emsp;顺手审计了一下加载性能，问题不小：**站点此前没有任何图片压缩**，文章首屏要拉 11.2 MB 图片，其中两张 GIF 占 9.9 MB；页脚有个 Vercel 图标是 8875×2000 却只显示 91×30。

&emsp;&emsp;处理思路是**按用途分档**，而不是简单压一遍：

| 用途 | 文件 | 尺寸 |
|---|---|---|
| 正文浏览 | `<name>-sm.webp` | 695px（正文列宽） |
| 灯箱查看 | `<name>-lg.webp` | 1390px（2 倍） |
| 保存原图 | `<name>.png` | 原始文件 |

&emsp;&emsp;正文 `<img>` 只加载小图，`<a href>` 指向大图：

```js
img.setAttribute("data-src", src.replace(/\.[^.]+$/, "-sm.webp"));   // 正文
link.setAttribute("href", src.replace(/\.[^.]+$/, "-lg.webp"));      // 灯箱
link.setAttribute("data-original", src);                             // 保存原图
```

&emsp;&emsp;效果：

- 正文图片总重 **11.2 MB → 156 KB**
- 文章首屏图片传输 **11.2 MB → 78 KB**
- 页脚图标 210 KB → 16 KB
- 字体 CSS：官方 `style.css` 会 `@import` 四套霞鹜文楷变体（426 KB），只引简体那套降到 **106 KB**
- GIF：6.7 MB + 3.3 MB 的动图。这里踩了个弯路 —— 最初给正文也生成了静态封面（3.8 KB），但正文里的动图就**只剩首帧、不会动了**。最后还是让正文直接加载原 GIF：动图的价值就在动画本身，为省几 MB 把它变成一张静止图并不划算（好在 `loading="lazy"` 让它只在滚到视口时才加载）

&emsp;&emsp;还有两个小坑：

1. **GIF 不能用 canvas 保存**——canvas 只画得出首帧，存下来就是张静态图。动图必须直连下载原文件。
2. **懒加载不用引 lazysizes**，原生 `loading="lazy"` + `decoding="async"` 就够了，能少一个 8 KB 的依赖。

# 九、小结

&emsp;&emsp;回头看，这次真正花时间的不是写功能，而是几个"看起来应该没问题"的地方：

- 线稿图**内联才能换色**（`<img>` 是封闭文档，CSS 改不了里面的颜色）
- 公式图**不能重算 viewBox**（MathJax 内部坐标系不是像素）
- 灯箱图片**不能加 padding**（Fancybox 用 border-box 算尺寸）
- 公式源码**不能依赖 MathJax 内部列表**（会被清空、结构不稳定）
- 动图**不能过 canvas**（会丢帧）

&emsp;&emsp;如果只记一条：**正文里的线稿图优先让它变色，灯箱里的图一律给衬底。** 前者不加底才能和正文融为一体，后者加了底才看得见。

&emsp;&emsp;而最省事的一步反而是想通那句话：**公式就是一张能复制代码的图片**——想通之后，自建灯箱的三百多行代码全删了，换成主题原生方案，交互反而更统一。

&emsp;&emsp;最后照例贴个行内公式验证一下渲染效果：质能方程 $E = mc^2$，以及一个独立公式：

$$
\frac{\partial J}{\partial w_{jk}^{(l)}} = \delta_j^{(l)} \, a_k^{(l-1)}
$$
