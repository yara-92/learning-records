SVG 是现代前端开发中非常重要的一项技术。它不仅仅是一种图片格式，更是一套可以直接参与网页布局、样式控制、DOM 操作、事件交互和动画制作的图形技术。

尤其是在 Vue、React 这样的前端框架中，SVG 的应用非常广泛：从一个简单的按钮图标，到复杂的数据可视化、流程图、地图、仪表盘，背后都可能使用 SVG。

我从实际开发的角度，把 SVG 拆成几个层次讲清楚。

# **一、先理解 SVG：它究竟是什么？**

SVG 全称是 **Scalable Vector Graphics（可缩放矢量图形）**。

它与 PNG、JPG 最大的区别，在于图像的描述方式不同。

[Zoom In Forever: How Math Creates Images That Never Pixelate!](https://images.openai.com/static-rsc-4/YRhKjtdgBbqbuqQgRQ8Bx7wDQHaCj7Wgkqk5UZ4mU7kPZNdnuP3ZCoZ0QritcFXaGuHGzmi_ob0MWQWoG23J7D6jdnRLTENjzPWn-jtSK94AZUeoKdDg5x7Q8FO4aQJiI97WGg0gr2RVxjWy9AKc3fX6SezAXd3IE_JEfEnjiDo?purpose=inline)

**位图（PNG、JPG）**

由像素组成，放大后可能出现锯齿和模糊。

[Global Crypto Liquidity & Fast Settlement | Aquanow](https://images.openai.com/static-rsc-4/YuP3Pz6nTz64ndszOb6l3-tx2R4YCIcI1diVtPLr4544dSTWcjrB24XeEK5nYI7DMDGdZ-jtzFOStWdgjg0-UzcmCcN0lugzGkLcWGaU-095t8HOURcqaDEN5kGgXeK2jYqckKwH01WGRgLT_TXRjH9ZQKfl4k2vbmHxIxymDdA?purpose=inline)

**矢量图（SVG）**

由路径、曲线、形状等几何信息描述，缩放时重新计算绘制。

例如，一个圆形的 SVG：

```
<svg width="200" height="200" viewBox="0 0 200 200">
    <circle
        cx="100"
        cy="100"
        r="60"
        fill="#3B82F6"
    />
</svg>
```

这里并没有存储 200×200 个像素，而是在告诉浏览器：

- 创建一个圆形。
- 圆心坐标是 `(100, 100)`。
- 半径是 `60`。
- 填充颜色是蓝色。

浏览器根据这些信息绘制图形。

这意味着 SVG 具有几个重要特点：

| 特性     | 说明                        |
| ------ | ------------------------- |
| 无限缩放   | 不会像位图一样因为放大而产生像素模糊        |
| DOM 结构 | 内部元素可以被 JavaScript 访问     |
| CSS 控制 | 可以使用 CSS 修改颜色、描边、透明度等     |
| 事件交互   | 可以绑定点击、鼠标、触摸等事件           |
| 动画     | 可以使用 CSS、JavaScript 等制作动画 |
| 文件格式   | 本质上是 XML 文本，可以直接编辑        |
| 复杂图形   | 可以组合路径、渐变、遮罩、滤镜等          |

需要注意：SVG 虽然是矢量图，但并不意味着它永远比 PNG 小，也不意味着它适合所有图片。照片、复杂纹理通常还是位图更合适。

# **二、SVG 的基本结构**

我们先看一个完整的 SVG 文件。

```
<svg
    xmlns="http://www.w3.org/2000/svg"
    width="300"
    height="200"
    viewBox="0 0 300 200"
>
    <!-- 矩形 -->
    <rect
        x="20"
        y="20"
        width="100"
        height="60"
        fill="#3B82F6"
    />

    <!-- 圆形 -->
    <circle
        cx="180"
        cy="50"
        r="30"
        fill="#10B981"
    />

    <!-- 直线 -->
    <line
        x1="20"
        y1="120"
        x2="280"
        y2="120"
        stroke="#EF4444"
        stroke-width="3"
    />

    <!-- 文字 -->
    <text
        x="20"
        y="170"
        font-size="20"
        fill="#333"
    >
        Hello SVG
    </text>
</svg>
```

它的几个核心属性：

| 属性             | 作用                     |
| -------------- | ---------------------- |
| `xmlns`        | SVG 命名空间，独立 SVG 文件通常需要 |
| `width`        | SVG 在页面中的显示宽度          |
| `height`       | SVG 在页面中的显示高度          |
| `viewBox`      | 定义 SVG 内部的坐标范围         |
| `fill`         | 填充颜色                   |
| `stroke`       | 描边颜色                   |
| `stroke-width` | 描边宽度                   |
| `opacity`      | 透明度                    |

这里特别强调 `viewBox`。

它是理解 SVG 的关键。

## **2.1 viewBox：SVG 的内部坐标系统**

例如：

```
<svg
    width="200"
    height="200"
    viewBox="0 0 100 100"
>
    <circle
        cx="50"
        cy="50"
        r="30"
        fill="orange"
    />
</svg>
```

你会看到：

示意图：内部坐标范围是 100×100，浏览器将它映射到 200×200 的显示区域。

`viewBox` 的语法：

```
viewBox="minX minY width height"
```

它定义的是 SVG 内部的逻辑坐标区域，而不是 HTML 元素的实际尺寸。

例如：

```
viewBox="0 0 24 24"
```

表示内部坐标范围为：

- X：0 到 24
- Y：0 到 24

所以很多图标都采用 24×24 的标准坐标系。

无论这个图标最终显示为 16px、24px、32px 还是 64px，都可以利用同一份图形路径进行缩放。

### **viewBox 和 width、height 的区别**

```
<svg
    width="48"
    height="48"
    viewBox="0 0 24 24"
>
</svg>
```

这里：

- `width`、`height`：控制页面上占据的显示尺寸。
- `viewBox`：控制 SVG 内部采用什么坐标系统。

这三个属性不应该混淆。

## **2.2 preserveAspectRatio：缩放时如何保持比例**

当 SVG 的显示区域和 viewBox 比例不一致时，浏览器需要决定如何处理。

常见情况：

```
<svg
    viewBox="0 0 100 100"
    preserveAspectRatio="xMidYMid meet"
>
</svg>
```

常用取值：

| 取值               | 效果                   |
| ---------------- | -------------------- |
| `xMidYMid meet`  | 等比例缩放，完整显示，默认行为      |
| `xMinYMin meet`  | 等比例缩放，靠左上角对齐         |
| `xMaxYMax meet`  | 等比例缩放，靠右下角对齐         |
| `xMidYMid slice` | 等比例放大，填满区域，超出部分可能被裁切 |
| `none`           | 不保持比例，横纵方向独立拉伸       |

这与 CSS 的 `object-fit: contain` 和 `object-fit: cover` 有相似之处。

对于一般图标，通常使用默认的 `meet` 就可以。

# **三、SVG 常用图形元素**

SVG 提供了很多原生图形元素。实际开发中，最常见的是以下几类。

## **3.1 基础几何形状**

**矩形 rect**

```
<svg viewBox="0 0 200 120">
    <rect
        x="20"
        y="20"
        width="160"
        height="80"
        rx="12"
        fill="#3B82F6"
    />
</svg>
```

`rx` 可以设置圆角半径。

**圆形 circle**

```
<circle
    cx="100"
    cy="100"
    r="50"
    fill="red"
/>
```

**椭圆 ellipse**

```
<ellipse
    cx="100"
    cy="60"
    rx="70"
    ry="35"
    fill="orange"
/>
```

**直线 line**

```
<line
    x1="10"
    y1="10"
    x2="200"
    y2="100"
    stroke="blue"
    stroke-width="3"
/>
```

**多边形 polygon**

```
<polygon
    points="100,10 190,180 10,180"
    fill="#F59E0B"
/>
```

这些元素适合构建简单的图形、装饰、按钮背景和示意图。

## **3.2 path：SVG 最重要的元素**

在真实项目中，尤其是图标开发，`path` 是最核心的元素。

例如：

```
<svg viewBox="0 0 24 24">
    <path
        d="M12 2 L19 21 L12 16 L5 21 Z"
        fill="#3B82F6"
    />
</svg>
```

`d` 属性描述路径的绘制过程。

你可以把它理解为一组绘图指令。

常用指令：

| 指令  | 含义      | 示例                   |
| --- | ------- | -------------------- |
| `M` | 移动到某个点  | `M10 10`             |
| `L` | 绘制直线    | `L20 20`             |
| `H` | 水平线     | `H100`               |
| `V` | 垂直线     | `V100`               |
| `C` | 三次贝塞尔曲线 | `C10 10 20 20 30 30` |
| `Q` | 二次贝塞尔曲线 | `Q20 10 30 30`       |
| `A` | 绘制椭圆弧线  | `A10 10 0 0 1 30 30` |
| `Z` | 闭合路径    | `Z`                  |

大写表示绝对坐标，小写表示相对坐标。

例如：

```
<path d="M10 10 L100 10 L100 100 Z"/>
```

它的含义是：

1. 移动到 `(10, 10)`。
2. 画直线到 `(100, 10)`。
3. 再画直线到 `(100, 100)`。
4. 闭合路径。

示意：用三条线段构建一个封闭的三角形路径。

### **贝塞尔曲线**

复杂图标、曲线和图形往往需要贝塞尔曲线。

三次贝塞尔曲线：

```
<path
    d="M20 100 C50 10 150 10 180 100"
    fill="none"
    stroke="#3B82F6"
    stroke-width="4"
/>
```

`C` 后面有 6 个数字：

```
C 控制点1X 控制点1Y 控制点2X 控制点2Y 终点X 终点Y
```

两个控制点决定曲线的弯曲方向和程度。

[Draw A Bézier curve using FabricJS | by Mandev | Medium](https://images.openai.com/static-rsc-4/gbt-9Isq-WBa_deAAaD71dkF1N_IctBEeS4mYCVsAEjs-FATAHGP2ZdkJGqtfKbDE6ZSB2UZb9Wyz2Gt078SDyYCauilVTY9_HZ-6XsXbuwD6bg8HkJUETHg8nhaCiA2K_EK-AltLCULzNq8ocwjAM5d1RTUVsMMIDl97Zf-yBY?purpose=inline)

[medium.com](https://medium.com/%40aditi.6ti/draw-a-b%C3%A9zier-curve-using-fabricjs-8cdc06d0f63a?utm_source=chatgpt.com)

为什么要掌握 Path？

因为很多 SVG 图标并不是由圆形、矩形简单组合而来，而是直接由路径描述。

拿到一个设计师交付的 SVG，很多时候你真正需要读懂的就是：

```
<path d="..."/>
```

不用一开始就掌握所有路径指令，但 `M`、`L`、`C`、`Q`、`A`、`Z` 值得优先学习。

# **四、SVG 的三种重要绘制方式：fill、stroke、渐变**

## **4.1 fill：填充**

```
<path
    d="M10 10 L100 10 L100 100 Z"
    fill="#3B82F6"
/>
```

填充整个封闭区域。

常见用法：

- 实心图标。
- 按钮图形。
- 背景装饰。
- 插画。

还可以设置：

```
fill="none"
fill="currentColor"
fill-opacity="0.5"
```

其中 `currentColor` 特别实用，后面会详细讲。

## **4.2 stroke：描边**

```
<circle
    cx="50"
    cy="50"
    r="30"
    fill="none"
    stroke="#3B82F6"
    stroke-width="4"
    stroke-linecap="round"
    stroke-linejoin="round"
/>
```

三个常用属性：

| 属性                | 作用      |
| ----------------- | ------- |
| `stroke-width`    | 线条粗细    |
| `stroke-linecap`  | 线条端点形状  |
| `stroke-linejoin` | 线段连接处形状 |

`stroke-linecap`：

- `butt`：平直切断。
- `round`：圆头。
- `square`：方形端点。

`stroke-linejoin`：

- `miter`：尖角。
- `round`：圆角。
- `bevel`：斜切角。

在现代 UI 图标中，通常使用：

```
stroke-linecap="round"
stroke-linejoin="round"
```

视觉效果比较柔和。

## **4.3 渐变 gradient**

SVG 支持线性渐变和径向渐变。

线性渐变：

```
<svg viewBox="0 0 200 100">
    <defs>
        <linearGradient id="myGradient" x1="0%" y1="0%" x2="100%" y2="0%">
            <stop offset="0%" stop-color="#3B82F6"/>
            <stop offset="100%" stop-color="#8B5CF6"/>
        </linearGradient>
    </defs>

    <rect
        x="10"
        y="10"
        width="180"
        height="80"
        rx="12"
        fill="url(#myGradient)"
    />
</svg>
```

径向渐变：

```
<defs>
    <radialGradient id="radial">
        <stop offset="0%" stop-color="#FDE68A"/>
        <stop offset="100%" stop-color="#F97316"/>
    </radialGradient>
</defs>

<circle cx="100" cy="100" r="80" fill="url(#radial)"/>
```

注意：`defs` 用于定义可复用资源，本身一般不会直接显示。

除了渐变，还可以在 `defs` 中定义：

- `clipPath`：裁剪路径。
- `mask`：遮罩。
- `filter`：滤镜。
- `symbol`：可复用图形。

# **五、前端如何引入 SVG？五种方式比较**

这是实际开发中最需要掌握的部分。

前端并不是只有一种使用 SVG 的方式。不同方式的能力和适用场景差别很大。

## **5.1 方式一：使用 img 标签**

这是最简单的方式。

首先准备文件：

```
public/
└── images/
    └── logo.svg
```

HTML：

```
<img
    src="/images/logo.svg"
    alt="网站 Logo"
    width="120"
    height="40"
/>
```

Vue：

```
<template>
    <img
        src="/images/logo.svg"
        alt="网站 Logo"
        class="logo"
    />
</template>

<style scoped>
.logo {
    width: 120px;
    height: 40px;
}
</style>
```

优点：

- 使用非常简单。
- 可以直接复用 SVG 文件。
- 浏览器可以缓存资源。
- SVG 与当前页面 DOM 隔离。
- 适合网站 Logo、静态插画和装饰性图片。

缺点：

- 无法直接通过外部 CSS 选择 SVG 内部的 `path`。
- 不能方便地给内部图形绑定点击事件。
- 不容易对内部图形进行动态操作。

**适用场景：静态图片、Logo、插画、装饰元素。**

## **5.2 方式二：直接使用 inline SVG**

直接把 SVG 写进 HTML 模板。

```
<svg
    width="24"
    height="24"
    viewBox="0 0 24 24"
>
    <circle
        cx="12"
        cy="12"
        r="9"
        fill="none"
        stroke="currentColor"
        stroke-width="2"
    />

    <path
        d="M12 7 V12 L16 15"
        fill="none"
        stroke="currentColor"
        stroke-width="2"
        stroke-linecap="round"
    />
</svg>
```

这是前端 UI 图标非常重要的使用方式。

它有几个优势：

1. SVG 内部元素可以被 CSS 控制。
2. 可以直接绑定 Vue、React 事件。
3. 可以通过 JavaScript 修改图形。
4. 可以配合 CSS 动画。
5. 可以使用当前组件的颜色变量。

比如：

```
<svg class="icon" viewBox="0 0 24 24">
    <path d="M12 2 L19 21 L12 16 L5 21 Z"/>
</svg>
```

```
.icon {
    width: 32px;
    height: 32px;
    fill: #2563eb;
    transition: fill 0.2s;
}

.icon:hover {
    fill: #ef4444;
}
```

**适用场景：按钮图标、交互图标、动态图形、数据可视化。**

这是我最推荐你重点掌握的一种方式。

## **5.3 方式三：使用 CSS background-image**

例如：

```
.logo {
    width: 120px;
    height: 40px;

    background-image: url("/images/logo.svg");
    background-repeat: no-repeat;
    background-position: center;
    background-size: contain;
}
```

HTML：

```
<div class="logo"></div>
```

优点：

- 适合纯装饰性图形。
- 可以通过 CSS 控制尺寸和位置。
- 不需要额外的 DOM 内容。

缺点：

- SVG 内部元素不容易操作。
- 不方便提供图片级别的无障碍文本。
- 不能直接给内部图形绑定事件。

**适用场景：装饰背景、纹理、背景图标。**

## **5.4 方式四：使用 object 标签**

```
<object
    data="/images/chart.svg"
    type="image/svg+xml"
    width="500"
    height="300"
>
    当前浏览器无法显示 SVG
</object>
```

这种方式可以把 SVG 作为独立文档嵌入页面。

在同源条件下，可以通过 `contentDocument` 访问内部 DOM，但会受到跨域、安全策略和浏览器行为的约束。

优点：

- 可以嵌入完整的 SVG 文档。
- 比普通 img 更有机会操作内部文档。

缺点：

- 操作复杂。
- 需要考虑同源策略。
- 不适合日常 UI 图标开发。

**适用场景：独立 SVG 文档、需要嵌入的复杂矢量内容。**

## **5.5 方式五：SVG Sprite 图标复用**

这是值得单独掌握的高级用法。

假设一个项目有：

- 搜索图标。
- 用户图标。
- 设置图标。
- 删除图标。
- 编辑图标。
- 下载图标。

如果每个页面都重复写一份 SVG，会出现代码重复。

可以将多个图标集中定义为 Sprite。

```
<svg xmlns="http://www.w3.org/2000/svg" style="display:none">

    <symbol id="icon-search" viewBox="0 0 24 24">
        <circle
            cx="11"
            cy="11"
            r="7"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
        />
        <path
            d="M16 16 L21 21"
            stroke="currentColor"
            stroke-width="2"
        />
    </symbol>

    <symbol id="icon-user" viewBox="0 0 24 24">
        <circle
            cx="12"
            cy="8"
            r="4"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
        />
        <path
            d="M4 22 C4 16 20 16 20 22"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
        />
    </symbol>

</svg>
```

使用时：

```
<svg width="24" height="24">
    <use href="#icon-search"></use>
</svg>

<svg width="24" height="24">
    <use href="#icon-user"></use>
</svg>
```

如果 Sprite 文件单独存在，可以这样引用：

```
<svg width="24" height="24">
    <use href="/icons.svg#icon-search"></use>
</svg>
```

这里的关键是：

- `symbol`：定义一个可复用的图形。
- `id`：给图形起唯一名称。
- `use`：引用已经定义好的图形。

优点：

- 减少重复 SVG 代码。
- 图标集中管理。
- 多个地方可以复用同一个图标定义。
- 可以统一设置外部 SVG 容器的大小和颜色。

注意：`symbol` 的 `viewBox` 很重要；外部 Sprite 还需要关注资源路径、同源限制和部署路径。外部引用的内部 DOM 不能像 inline SVG 那样直接自由操作。

## **五种方案怎么选？**

| 方案           | 内部 CSS 控制 | 事件交互  | 复用性   | 常见用途       |
| ------------ | --------- | ----- | ----- | ---------- |
| img          | 不方便       | 不方便   | 高     | Logo、静态插画  |
| inline SVG   | 强         | 强     | 可封装组件 | UI 图标、复杂交互 |
| background   | 不方便       | 不方便   | 高     | 装饰背景       |
| object       | 有条件支持     | 有条件支持 | 高     | 独立矢量文档     |
| Sprite + use | 有一定能力     | 有限    | 很高    | 大量重复图标     |

我的实际建议：

- 静态图片：优先 `img`。
- UI 图标：优先 inline SVG 或图标组件。
- 装饰背景：优先 CSS background。
- 大型图标系统：考虑 Sprite 或成熟图标库。
- 复杂交互图形：直接使用 inline SVG。

# **六、SVG 与 CSS：打造真正的 UI 图标**

这是 SVG 在日常前端开发中最有价值的应用之一。

## **6.1 currentColor：让 SVG 自动继承文字颜色**

我们先看一个问题。

按钮文字是蓝色，图标也应该是蓝色。

传统做法可能需要同时维护两份颜色：

```
.button {
    color: blue;
}

.icon {
    stroke: blue;
}
```

这样不够灵活。

SVG 提供了一个非常实用的属性：

```
stroke="currentColor"
```

例如：

```
<button class="button">
    <svg
        width="20"
        height="20"
        viewBox="0 0 24 24"
    >
        <path
            d="M12 2 L12 22 M2 12 L22 12"
            stroke="currentColor"
            stroke-width="2"
        />
    </svg>

    添加数据
</button>
```

CSS：

```
.button {
    color: #2563eb;
}

.button:hover {
    color: #dc2626;
}
```

图标的描边颜色会自动跟随按钮文字颜色。

这对于设计系统、暗黑主题、按钮状态非常有用。

还可以在 SVG 中使用：

```
fill="currentColor"
```

它控制填充颜色。

## **6.2 CSS 控制 SVG 的状态**

```
<svg class="heart-icon" viewBox="0 0 24 24">
    <path
        d="M12 21 C3 14 2 8 6 5 C9 3 12 6 12 6 C12 6 15 3 18 5 C22 8 21 14 12 21 Z"
    />
</svg>
```

```
.heart-icon {
    width: 32px;
    height: 32px;
    fill: none;
    stroke: #ef4444;
    stroke-width: 2;
    transition: all 0.2s ease;
}

.heart-icon:hover {
    fill: #ef4444;
}
```

可以制作：

- 收藏/取消收藏。
- 展开/收起。
- 加载状态。
- 勾选状态。
- 禁用状态。
- 鼠标悬停状态。

## **6.3 CSS 控制 SVG 线条动画**

一个很经典的效果：让 SVG 线条逐渐绘制出来。

```
<svg viewBox="0 0 200 100">
    <path
        class="draw-line"
        d="M10 50 C50 0 150 100 190 50"
        fill="none"
        stroke="#2563eb"
        stroke-width="4"
    />
</svg>
```

```
.draw-line {
    stroke-dasharray: 300;
    stroke-dashoffset: 300;
    animation: draw 2s ease forwards;
}

@keyframes draw {
    to {
        stroke-dashoffset: 0;
    }
}
```

它的原理：

- `stroke-dasharray`：规定描边虚线段和间隙的长度。
- `stroke-dashoffset`：控制虚线偏移量。
- 动画逐步把偏移量变为 0。

这就是很多网页中描边逐渐出现效果的实现原理。

# **七、SVG 的高级能力：裁剪、遮罩、滤镜、变换**

SVG 不仅可以画图，还可以制作非常复杂的视觉效果。

## **7.1 clipPath：裁剪**

例如，把矩形图片裁剪成圆形。

```
<svg viewBox="0 0 200 200">

    <defs>
        <clipPath id="circleClip">
            <circle cx="100" cy="100" r="80"/>
        </clipPath>
    </defs>

    <image
        href="/avatar.jpg"
        width="200"
        height="200"
        preserveAspectRatio="xMidYMid slice"
        clip-path="url(#circleClip)"
    />

</svg>
```

裁剪区域之外的内容不会显示。

应用：

- 圆形头像。
- 特殊形状图片。
- 不规则内容裁剪。
- 动态图形展示。

## **7.2 mask：遮罩**

遮罩和裁剪不完全相同。

裁剪主要判断某个区域是否显示。

遮罩可以通过颜色和透明度控制不同区域的显示程度。

```
<svg viewBox="0 0 200 100">

    <defs>
        <linearGradient id="fade">
            <stop offset="0%" stop-color="white" stop-opacity="1"/>
            <stop offset="100%" stop-color="white" stop-opacity="0"/>
        </linearGradient>

        <mask id="fadeMask">
            <rect
                width="200"
                height="100"
                fill="url(#fade)"
            />
        </mask>
    </defs>

    <rect
        width="200"
        height="100"
        fill="#3b82f6"
        mask="url(#fadeMask)"
    />

</svg>
```

这种方式适合：

- 渐隐效果。
- 图片边缘过渡。
- 特殊透明效果。
- 图形融合。

## **7.3 filter：滤镜**

SVG 支持滤镜效果。

例如高斯模糊：

```
<svg viewBox="0 0 200 100">

    <defs>
        <filter id="blur">
            <feGaussianBlur stdDeviation="4"/>
        </filter>
    </defs>

    <circle
        cx="100"
        cy="50"
        r="30"
        fill="#3b82f6"
        filter="url(#blur)"
    />

</svg>
```

常用滤镜包括：

| 滤镜               | 功能      |
| ---------------- | ------- |
| `feGaussianBlur` | 高斯模糊    |
| `feDropShadow`   | 阴影      |
| `feColorMatrix`  | 颜色矩阵变换  |
| `feBlend`        | 图层混合    |
| `feMorphology`   | 膨胀或腐蚀形态 |

注意：复杂滤镜可能影响性能，特别是大量 SVG 元素同时应用模糊、阴影时。

## **7.4 transform：图形变换**

SVG 支持平移、旋转、缩放和倾斜。

```
<svg viewBox="0 0 200 200">

    <rect
        x="50"
        y="50"
        width="60"
        height="60"
        fill="#3b82f6"
        transform="rotate(30 80 80)"
    />

</svg>
```

常见变换：

```
transform="translate(10, 20)"
transform="rotate(45)"
transform="scale(2)"
transform="skewX(20)"
```

多个变换也可以组合：

```
transform="translate(20, 20) rotate(30) scale(1.2)"
```

实际开发时要注意变换的顺序，因为矩阵变换通常不满足交换律：交换顺序可能产生不同结果。

# **八、Vue 3 项目中如何使用 SVG？**

结合现代 Vue 3 项目，实际开发通常会有三种做法：

1. 直接在组件中写 SVG。
2. 把 SVG 封装成通用图标组件。
3. 使用 SVG 图标库。

其中第二种很值得掌握。

## **8.1 直接写在 Vue 组件中**

```
<script setup>
import { ref } from 'vue'

const active = ref(false)

function toggle() {
    active.value = !active.value
}
</script>

<template>
    <button @click="toggle">
        <svg
            width="24"
            height="24"
            viewBox="0 0 24 24"
        >
            <path
                d="M12 2 L12 22 M2 12 L22 12"
                fill="none"
                stroke="currentColor"
                stroke-width="2"
            />
        </svg>

        {{ active ? '已启用' : '未启用' }}
    </button>
</template>
```

这里 SVG 是正常的 DOM 子元素。

所以 Vue 可以直接控制它所在组件的状态。

## **8.2 封装一个通用 SvgIcon 组件**

假设你做一个管理后台，界面中有很多图标：

- Search
- User
- Settings
- Edit
- Delete
- Download
- Upload

可以建立一个统一组件。

项目结构：

```
src/
├── components/
│   └── SvgIcon.vue
├── icons/
│   └── paths.js
└── App.vue
```

`paths.js`：

```
export const iconPaths = {
    search: [
        'M11 19a8 8 0 1 1 0-16 8 8 0 0 1 0 16Z',
        'm21 21-4.35-4.35'
    ],

    edit: [
        'M12 20h9',
        'M16.5 3.5a2.12 2.12 0 0 1 3 3L8 18l-4 1 1-4Z'
    ],

    delete: [
        'M3 6h18',
        'M8 6V4h8v2',
        'M19 6l-1 14H6L5 6',
        'M10 11v5',
        'M14 11v5'
    ],

    download: [
        'M12 3v12',
        'm7 10 5 5 5-5',
        'M5 21h14'
    ]
}
```

`SvgIcon.vue`：

```
<script setup>
import { computed } from 'vue'
import { iconPaths } from '@/icons/paths'

const props = defineProps({
    name: {
        type: String,
        required: true
    },

    size: {
        type: [Number, String],
        default: 24
    },

    color: {
        type: String,
        default: 'currentColor'
    },

    strokeWidth: {
        type: Number,
        default: 2
    },

    label: {
        type: String,
        default: ''
    }
})

const paths = computed(() => {
    return iconPaths[props.name] ?? []
})
</script>

<template>
    <svg
        :width="size"
        :height="size"
        viewBox="0 0 24 24"
        fill="none"
        :stroke="color"
        :stroke-width="strokeWidth"
        stroke-linecap="round"
        stroke-linejoin="round"
        :role="label ? 'img' : undefined"
        :aria-label="label || undefined"
        :aria-hidden="label ? undefined : 'true'"
    >
        <path
            v-for="(d, index) in paths"
            :key="index"
            :d="d"
        />
    </svg>
</template>
```

页面使用：

```
<script setup>
import SvgIcon from '@/components/SvgIcon.vue'
</script>

<template>
    <div class="toolbar">

        <button>
            <SvgIcon name="search" />
            搜索
        </button>

        <button>
            <SvgIcon name="edit" :size="20" />
            编辑
        </button>

        <button>
            <SvgIcon
                name="delete"
                :size="20"
                color="#ef4444"
            />
            删除
        </button>

        <button>
            <SvgIcon name="download" />
            下载
        </button>

    </div>
</template>
```

你会发现，这样做有几个好处：

- 页面不需要反复写 SVG。
- 统一维护所有图标路径。
- 图标尺寸、颜色和线条粗细可以统一控制。
- 可以配合 Vue 的响应式系统动态变化。
- 后续替换图标，只需要修改图标数据或组件实现。

上面这个实现适合小型项目，真正的图标系统还应补上未知名称校验、图标命名规范和必要的测试。

## **8.3 使用现成 SVG 图标库**

实际商业项目中，通常没有必要手动绘制所有图标。

可以使用成熟的 SVG 图标库。

[48 High-Quality Icon Packs for Free Download and Commercial Use](https://images.openai.com/static-rsc-4/D8Qa9Mticy1eGXJvhwNxBWB0dNR8rSnnOMKt6OTCNgUSGpIdg11QM6XuWeqO7N4WMM8qYr6nBe6gubt4OMMh8S1pJlkT3SaWjzQOVmYx4RbweHx8sJkRh65Ogik-B7X71WddUT8kNxMqMfNoVDK9hfsXHNN7MjFvDGI-SibnfYI?purpose=inline)

**Lucide**

线条风格统一，图标数量丰富，适合 Vue、React 和普通前端项目。

网站：[lucide.dev](https://lucide.dev/?utm_source=chatgpt.com)

[Web 图标库选型指南：当前最受欢迎、好看又好用的 Icon 推荐及选型策略 - 阿小信的博客](https://images.openai.com/static-rsc-4/h7Vy2KzOs06a8JHJyfL-iBz-SYpmFoFqWnILCLs_DYpT8FfFAo2TRKoNtc3LHfhH7VMJAnAMzn1QJnbv9fK5bMBPB5q_IT-zWJj_uWi5KPWbjtZ_kvk33ZWG2p0aN6PjQIelQRoVf__LHm9VQPWWPKtN-cCccOQMWsQ2XP6A4iM?purpose=inline)

**Iconify**

可以统一使用大量不同来源的图标集，适合图标种类特别多的项目。

网站：[iconify.design](https://iconify.design/?utm_source=chatgpt.com)

[Bootstrap Icons - Tasarım Aracı | ToolHunters](https://images.openai.com/static-rsc-4/WbbMw1OOW0syCjV-Ek7mzCYFB3p0gkGlq9LCUNf0zEmQqyENkP6NaA07LxSDIvZOqzHqaWQdh_ZOMetvW0EOC1OyOCkOknzmNoOeXZOP6xyntiwNkqmLpuFFIWhxXZB1scyjuyYqnLwY5MBC8A2ZMKk_8rbloN_nnvdeFfY8SeU?purpose=inline)

**Bootstrap Icons**

风格简洁，适合后台系统、管理平台和常规 Web 界面。

网站：[icons.getbootstrap.com](https://icons.getbootstrap.com/?utm_source=chatgpt.com)

[How to download and install Google Fonts - WhiteUI.Store Blog](https://images.openai.com/static-rsc-4/z-rL8eaNgXOjHVcHDN4_Qk4l-n549JmqEsvI_HPf56K9AUSkWFkTG8IMxw2i3DrkWnABzh1FoUy1SZ4MjY8UU-8lI4GuCKOHn_hgTY_-5sB7RdBOFT3BXoN3TcJn6S8DD2GApP_WuJ6jupyvgXtGlRqDlZZXjxljlX4MYw8AJug?purpose=inline)

**Material Symbols**

Google Material Design 的图标系统，适合 Material 风格界面。

网站：[fonts.google.com/icons](https://fonts.google.com/icons?utm_source=chatgpt.com)

以 Vue 3 使用 Lucide 为例：

```
npm install lucide-vue-next
```

组件中：

```
<script setup>
import {
    Search,
    User,
    Settings,
    Trash2
} from 'lucide-vue-next'
</script>

<template>
    <div class="toolbar">
        <Search :size="20" />
        <User :size="20" />
        <Settings :size="20" />
        <Trash2 :size="20" color="red" />
    </div>
</template>
```

图标库本质上也是在利用 SVG，只不过帮你处理了图标路径和组件封装。

对于实际业务项目，我更推荐：

**普通项目直接用成熟图标库；有独特设计需求时，再编写自己的 SVG 组件。**

# **九、SVG 交互：它不仅是图片，更是 DOM**

这是 SVG 与 PNG 最大的开发差别之一。

我们可以为 SVG 内部元素绑定事件。

例如，制作一个可以点击的圆形按钮：

```
<svg viewBox="0 0 200 200">

    <circle
        id="myCircle"
        cx="100"
        cy="100"
        r="70"
        fill="#3b82f6"
        style="cursor:pointer"
    />

    <text
        x="100"
        y="106"
        text-anchor="middle"
        fill="white"
        font-size="20"
        pointer-events="none"
    >
        点击
    </text>

</svg>
```

原生 JavaScript：

```
const circle = document.querySelector('#myCircle')

circle.addEventListener('click', () => {
    circle.setAttribute('fill', '#ef4444')
})
```

Vue 中：

```
<template>
    <svg viewBox="0 0 200 200">
        <circle
            cx="100"
            cy="100"
            r="70"
            :fill="active ? '#ef4444' : '#3b82f6'"
            @click="active = !active"
        />

        <text
            x="100"
            y="106"
            fill="white"
            text-anchor="middle"
            pointer-events="none"
        >
            {{ active ? '已选中' : '点击' }}
        </text>
    </svg>
</template>

<script setup>
import { ref } from 'vue'

const active = ref(false)
</script>
```

SVG 可以实现：

- 点击不同图形触发不同操作。
- 鼠标悬停高亮。
- 拖拽图形。
- 图形缩放和平移。
- 节点连线交互。
- 地图区域点击。
- 流程图节点操作。
- 数据图表交互。

### **一个重要问题：屏幕坐标与 SVG 坐标**

开发拖拽、画图、节点定位时，经常会碰到一个问题：

浏览器鼠标事件提供的是屏幕坐标：

```
event.clientX
event.clientY
```

但 SVG 内部使用的是自己的 `viewBox` 坐标。

这两套坐标并不一定相同。

可以通过矩阵转换：

```
function getSvgPoint(event, svgElement) {
    const matrix = svgElement.getScreenCTM()

    if (!matrix) {
        return null
    }

    const point = new DOMPoint(
        event.clientX,
        event.clientY
    )

    return point.matrixTransform(matrix.inverse())
}
```

返回的坐标就是转换到 SVG 用户坐标系统中的位置。

这是开发 SVG 绘图工具、拖拽编辑器、流程图时非常关键的知识。

# **十、SVG 在实际业务中的应用**

现在把视角从图标扩展到完整的业务系统。

SVG 的应用领域非常广泛。

[User interface design | Graphical User Interface Development](https://images.openai.com/static-rsc-4/T0rtzMKT4eh45RmM4DEhRzyNUTtnf5BDo-NyxtMxJ3H-MZGy-Z28zVDXAfu1-tFRmo3HK4yn9TGErVDqnOLzns07h4azaD5dWlg4aE8pPXjve4Na4psXTX4n3XB8ibOZnW9EOBqTbzrfIzr5NV-siwXNXu3joyRQWAFcP8Nop18?purpose=inline)

**1. 数据可视化**

折线图、柱状图、饼图、仪表盘、趋势分析、实时数据监控。SVG 非常适合绘制需要交互的图表。

[Applications built with JointJS](https://images.openai.com/static-rsc-4/w3nGzBBGPeoAj9MMFS8OWfEKFAAby_f_Pc9amJXb8DW-S0AiB_eH4-QIJIlNkYKv3j0meatoe4yRHwkAP7i_GToTEqkQrETQV-y3ftSGnbpm_1WkRTg8ZTfk6vR9v9OOMiOoIPZj9tiNLG88lk4nHgxKhfH8shLTgfVRj9F86fo?purpose=inline)

**2. 流程图和审批系统**

画布、节点、连线、箭头、节点选中、拖拽、缩放、连接关系。SVG 很适合绘制节点和连线。

[HabitatSeven | Work | ClimateData.ca](https://images.openai.com/static-rsc-4/xLv4NSCrCsPBxv9rqEWOk1Ar9nopRyYXmm3-UuYsS6dzDFUzimWukb4aULe_phSjcjSIrKMV7bCRHbwqDb-Vaj4Yyd3Zg4UeTpyrGHUbMVb6grynLh2bnkATVTA618DCHmlVQATOaFT1KDLNlYs0RaSZF4AzkVPU2i69hPiAd6k?purpose=inline)

**3. 地图与区域交互**

地图轮廓、行政区划、地理区域高亮、点击查看区域数据、路径标注。

[essential icons for web and app](https://images.openai.com/static-rsc-4/EUfNxzrEPqE6XwkWmZu9zB_TuWnaJkwDcoPhxTEbJcjHGUs2gvSjCvmWm6Vkp06ftKtQ6MFEnfLN2DZtlTbIi8nbxfnGGo3PW6e59Q4DER8ZYpaskVqumCP4ReyhmMY_EmMK3fNOLnwQ9PLDZZa63O2gsU-cie0RVBJsDpaNgQA7m_I2euBwFnWpsSiXXpR2?purpose=inline)

**4. 后台管理系统**

菜单图标、操作按钮、状态图标、提示图形、上传下载图标、状态指示器。

[Animating stroke-dasharray and stroke-dashoffset (How To) | Animating SVG with CSS | Treehouse](https://images.openai.com/static-rsc-4/qpv3PAIscsamz_F3Sgdc7zHdySp2FUvZ-51ELcXgBP2_evYjHbCpTfAN8kQ3HV9s0o6cQcgB_4k2PpYdGMY0ploGWkm6e6dEivWoHipwT4ktXAPsfaTTFR-fAwTSNjZ2H-RsyK5gtYrkJKvtXwnic7Lz0y-_Aikq5-zGHyP6ibQ?purpose=inline)

**5. 动画与品牌展示**

Logo 动画、路径描边、动态装饰、加载指示器、交互反馈。

[Vector blueprint of a one story building floor plan](https://images.openai.com/static-rsc-4/DZYjZpui02i5SZRoduHIZJmHOdlTqwpxKBdJSgL9nYEl0QGrUzXO23M76REFuhcF1wa9N-aHMQr7M-JeO5m04WUP9JmFEt-M2mm3_Y8hksPA1vJFFjovQfwRiI8zz1VKyvd7lVMy9_08BdwLwWzkbgGHOsP0cAhMMvfrBfpgbG6acLxuFWGXkZsccvzErrcQ?purpose=inline)

**6. 平面图和设备监控**

楼层平面图、工厂设备布局、设备状态灯、机房拓扑图、区域点击和状态展示。

## **10.1 SVG 与数据可视化**

以柱状图为例：

```
<svg viewBox="0 0 400 240">

    <!-- X 轴 -->
    <line
        x1="40"
        y1="200"
        x2="380"
        y2="200"
        stroke="#64748b"
    />

    <!-- 柱状图 -->
    <rect x="70" y="100" width="40" height="100" fill="#3b82f6"/>
    <rect x="140" y="60" width="40" height="140" fill="#3b82f6"/>
    <rect x="210" y="120" width="40" height="80" fill="#3b82f6"/>
    <rect x="280" y="30" width="40" height="170" fill="#3b82f6"/>

</svg>
```

如果数据来自后端：

```
const data = [
    { name: '一月', value: 100 },
    { name: '二月', value: 140 },
    { name: '三月', value: 80 },
    { name: '四月', value: 170 }
]
```

可以用 Vue 的 `v-for` 生成 SVG 元素。

```
<template>
    <svg viewBox="0 0 400 240">

        <line
            x1="40"
            y1="200"
            x2="380"
            y2="200"
            stroke="#64748b"
        />

        <g v-for="(item, index) in data" :key="item.name">

            <rect
                :x="60 + index * 80"
                :y="200 - item.value"
                width="40"
                :height="item.value"
                fill="#3b82f6"
            />

            <text
                :x="80 + index * 80"
                y="220"
                text-anchor="middle"
                font-size="12"
                fill="currentColor"
            >
                {{ item.name }}
            </text>

        </g>

    </svg>
</template>

<script setup>
const data = [
    { name: '一月', value: 100 },
    { name: '二月', value: 140 },
    { name: '三月', value: 80 },
    { name: '四月', value: 170 }
]
</script>
```

这就是前端动态生成 SVG 图形的基本思想。

当然，真正的数据可视化还需要处理比例尺、坐标轴、刻度、边距、响应式尺寸、标签碰撞和鼠标交互等问题。

# **十一、SVG、Canvas、HTML：到底如何选择？**

这是一个非常值得掌握的技术选型问题。

虽然 SVG 和 Canvas 都能绘制图形，但它们的设计思路有很大区别。

**SVG：DOM 驱动的图形**

[Archivo:Svg exemplo.png - MediaWiki](https://images.openai.com/static-rsc-4/R7m71DKRS02nYgtgkgbMb8_cp7qu_BN7wzjP3OJj1SgWSNQ-aZPheOnFUfKmHcURWNO4UBs6FS5cRiNW3d7mW0zQYR2f7KjN2DMDbvTlpo3pxX0UTmoUhQ7K44Ofegvzxq8t1msVRhnI_Cienxj2_RPPloCLO1sKBa9DMgYFWts?purpose=inline)

[wiki.cifprodolfoucha.es](https://wiki.cifprodolfoucha.es/index.php?title=Archivo:Svg_exemplo.png\&utm_source=chatgpt.com)

每个图形都是 DOM 元素，可以单独绑定事件、设置属性和样式。

**Canvas：像素绘制**

[How to Draw Beautiful Things in the Browser | Greg Tatum](https://images.openai.com/static-rsc-4/-hFILmqC2ndvhBPRKG4fuQG_R3Y5vI87Mn14icw3TXmnk8KhT06llpU5L5oeo5Vu5xG14-BA0gGtg3cZWHmbTmgyfdhbQG0CeAT-IkOb8i8KKjya-jKkUJqjHd8tjPMXwGzBdzfoFsV8itbxbcEtj39xYbRI3_6dVNCGRP27EhY?purpose=inline)

[gregtatum.com](https://gregtatum.com/writing/2014/how-to-draw-beautiful-things-in-the-browser/?utm_source=chatgpt.com)

通过 JavaScript 绘制像素画面，绘制完成后通常不会保留每个图形对应的 DOM 节点。

| 对比维度     | SVG              | Canvas      | HTML/CSS |
| -------- | ---------------- | ----------- | -------- |
| 绘制机制     | DOM 矢量元素         | 像素画布        | 页面元素     |
| 缩放清晰度    | 很好               | 取决于分辨率和缩放处理 | 很好       |
| 单个图形事件   | 方便               | 需要自行计算命中    | 方便       |
| 动画       | CSS、JS 等         | JS 重绘       | CSS、JS   |
| DOM 可访问性 | 相对方便             | 需要另行设计      | 很好       |
| 大量图元     | 复杂时可能变慢          | 通常更合适       | 视场景而定    |
| 文本布局     | 可以绘制，但不如 HTML 灵活 | 需自行处理       | 最方便      |

具体怎么选？

| 业务场景         | 推荐方案           |
| ------------ | -------------- |
| 普通按钮、图标      | SVG            |
| 复杂 Logo、矢量插画 | SVG            |
| 流程图、审批节点     | SVG 或专用图形引擎    |
| 中等规模交互图表     | SVG            |
| 大量粒子、连续动画    | Canvas         |
| 视频游戏、实时高频绘制  | Canvas 或 WebGL |
| 文字排版、表单、布局   | HTML/CSS       |
| 3D 场景        | WebGL / WebGPU |

这里有一个需要避免的误区：

**SVG 并不是因为使用矢量就一定性能更好。**

假设一个 SVG 中只有 20 个图形，浏览器管理这些 DOM 元素通常比较轻松。

但如果有几十万个图形节点，每个节点都需要参与 DOM 更新、样式计算和事件处理，性能就可能成为问题。

对于大规模、高频绘制的场景，Canvas 或 WebGL 通常更有优势。

# **十二、SVG 开发中的性能、安全与兼容性**

这部分在真正的业务项目中也很重要。

## **12.1 不要把所有 SVG 都直接写进模板**

假设你有 100 个页面，每个页面都重复写同一套图标 SVG。

这会导致：

- 代码重复。
- 修改困难。
- 维护成本增加。
- 图标尺寸和样式不统一。

建议组件化或 Sprite 化。

## **12.2 使用 SVG 优化工具**

设计工具导出的 SVG，经常包含大量冗余信息：

```
<svg width="100px" height="100px">
    <g id="Layer_1">
        <path
            id="Path_1"
            fill="#000000"
            d="..."
        />
    </g>
</svg>
```

可能包含：

- 无用注释。
- 编辑器元数据。
- 冗余分组。
- 多余属性。
- 不必要的精度。
- 重复定义。

可以使用 SVGO 优化：

```
npm install -D svgo
```

执行：

```
npx svgo input.svg -o output.svg
```

批量优化时也可以把它接入构建流程。

不过要留意：优化时不要无意中删除正在使用的 ID、`title`、`viewBox` 或者其他语义信息。

## **12.3 避免滥用复杂 SVG 滤镜**

特别是：

```
<filter>
    <feGaussianBlur stdDeviation="20"/>
</filter>
```

模糊范围越大、应用对象越多，绘制成本可能越高。

如果只是想做普通阴影，很多时候 CSS `box-shadow` 或 `filter: drop-shadow()` 就足够了。

## **12.4 小心 SVG 安全风险**

SVG 本质上是可以包含多种内容的 XML 文档。

它可能包含：

- 脚本。
- 事件属性。
- 外部资源引用。
- 链接。
- `foreignObject` 等 HTML 嵌入内容。

因此：

**不要把用户上传的、不可信的 SVG 原始文本直接通过 Vue 的&#x20;**`v-html`**&#x20;注入页面。**

例如：

```
<div v-html="userUploadedSvg"></div>
```

如果业务确实需要展示用户上传的 SVG，应使用可靠的 SVG 清洗方案，采取允许列表，剔除脚本、危险事件处理器和不必要的外部资源。

不要因为 SVG 是图片格式就把它当作绝对安全的图片。

## **12.5 注意图标的无障碍属性**

装饰性图标：

```
<svg aria-hidden="true">
    ...
</svg>
```

有实际语义的图片：

```
<svg role="img" aria-label="下载文件">
    ...
</svg>
```

另外，按钮本身也应该有明确的可访问名称：

```
<button aria-label="删除文件">
    <svg aria-hidden="true">
        ...
    </svg>
</button>
```

这样屏幕阅读器才能正确识别按钮用途。

## **12.6 注意 SVG 文字与字体**

SVG 的 `<text>` 元素可以绘制文字：

```
<text
    x="20"
    y="40"
    font-family="Arial"
    font-size="20"
>
    Hello
</text>
```

但在不同设备上，字体可能不存在或者发生替换。

如果 SVG 需要用于严格的印刷、Logo 交付等场景，需要考虑字体兼容问题；对于普通网页，通常让浏览器使用系统字体即可。

# **十三、前端 SVG 开发的完整工作流**

把前面知识串起来，一个比较成熟的 SVG 开发流程是这样的：

1. **设计阶段**

   使用 Figma、Illustrator 等工具创建矢量图形。
2. **导出阶段**

   导出 SVG 文件，检查 `viewBox`、路径、颜色及无用分组。
3. **优化阶段**

   使用 SVGO 清理冗余数据，注意保留必要属性。
4. **开发阶段**

   根据使用场景，决定采用 img、inline SVG、Sprite 或组件。
5. **封装阶段**

   统一管理图标命名、尺寸、颜色、描边和无障碍属性。
6. **交互阶段**

   添加事件、状态控制、动画、拖拽和数据绑定。
7. **测试阶段**

   检查响应式、不同尺寸、浏览器显示、性能、安全和可访问性。

常用开发工具：

| 工具                | 主要用途          |
| ----------------- | ------------- |
| Figma             | UI 设计和 SVG 导出 |
| Inkscape          | 矢量图形编辑        |
| Adobe Illustrator | 专业矢量图形设计      |
| SVGOMG            | 在线压缩和优化 SVG   |
| SVGO              | 项目构建中的 SVG 优化 |
| SVG Path Editor   | 编辑和理解 Path 路径 |

# **十四、最后总结：前端开发者需要掌握 SVG 到什么程度？**

我建议分为三个层次。

第一层 · 必须掌握

**日常 UI 开发**

- SVG 基本结构。
- `viewBox` 和坐标系统。
- `rect`、`circle`、`line`、`path`。
- `fill`、`stroke`、`currentColor`。
- img 与 inline SVG 的区别。
- SVG 图标组件封装。
- Vue / React 中的使用方法。

第二层 · 熟练掌握

**复杂页面和交互开发**

- SVG Sprite。
- 渐变、裁剪、遮罩和滤镜。
- CSS 动画和路径动画。
- SVG 坐标转换。
- 事件交互和拖拽。
- 动态生成图表。
- SVG 性能优化。

第三层 · 专项深入

**图形应用和可视化开发**

- 贝塞尔曲线和复杂 Path。
- SVG 矩阵变换。
- 图形碰撞与命中检测。
- 流程图和节点编辑器。
- 地图和复杂平面图。
- SVG、Canvas、WebGL 的混合渲染。

**我认为最值得记住的一个观点：**

SVG 不应该仅仅被看作一种图片格式，而应该被看作前端的一种图形开发技术。

它结合了矢量绘图、DOM、CSS 和 JavaScript。掌握之后，前端能够完成的不再只是普通的页面布局，而是可以进一步开发流程设计器、交互式地图、图表、平面图、设备监控界面等更复杂的应用。
