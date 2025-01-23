---
title: CSS
id: 2078eec4-e10c-40e1-8dbd-2c9e7b2de8a9
date: 2024-03-29 10:40:37
auther: xmy
cover: null
excerpt: CSS CSS 是一门样式表语言，Cascading Style Sheets 层叠样式表，多个样式可以作用在同一个html的元素上，同时生效，这也就是说人们可以用它来选৤
permalink: /archives/css
categories:
 - coding
 - qian-duan
tags: 
 - qian-duan
 - web
 - css
---



# CSS
> **CSS 是一门样式表语言**，Cascading Style Sheets 层叠样式表，多个样式可以作用在同一个html的元素上，同时生效，这也就是说人们可以用它来选择性地为 HTML 元素添加样式


## 介绍

CSS样式由选择符（选择器）和声明组成，而声明又由属性和值组成。\
示例：
```css
    p { color: blue ;font-size:12px; }

    p是选择符， 花括号内是声明 
    color是属性，  blue是值  
    多条声明之间用分号 ; 隔开  
    属性和值之间用冒号 : 隔开 
```

*   最后一条声明可以没有分号，但是为了以后修改方便，一般也加上分号
*   为了使样式更加容易阅读，可以将每条代码写在一个新行中

## 注释代码

CSS中使用`/*   */`注释代码

## 分类

从CSS插入代码的形式可以分为三类：

### 内联式

    * 直接把css代码写在现有的HTML标签中  

### 嵌入式（内部式）

*   把CSS样式代码写在`<style>` `</style>`之间
    *   必须写在在`<style>` `</style>`之间
    *   一般情况下写在`<head></head>`中

### 外部式

*   把CSS样式代码写在单独的外部文件中，在`<head>`内使用`<link>`标签将css样式链接到html文件内
    *   `<link>`语法：`<link href="base.css" rel="stylesheet" type="text/css" />`
    *   css样式文件名称以有意义的英文字母命名
    *   `rel="stylesheet" type="text/css"`是固定写法不可修改
    *   `<link>`标签位置一般写在`<head>`标签之内

### 优先级

相同权值下，CSS样式采用就近原则（离被设置元素越近优先级越高），一般情况下CSS样式的优先级是 内联式 > 嵌入式 > 外部式

## 选择器

每一条css样式声明由两部分组成，形式如下：

    选择器{
        样式;
    }

选择器指明了{}中“样式”的作用对象

### 基础选择器

#### 标签（元素）选择器

*   html中的标签，如`<html>` 、`<body>` 、`<h1>` 、`<p>`  、`<img>`等
    * 用法举例：
    ```
    h1{
    color\:red;
    }
    ```

#### 类选择器

*   语法：`.类选择器名称{css样式代码;}`
*   英文圆点开头
*   可以任意起名
*   在需要设置样式的代码处使用`<span class="类选择器名称"> </span>`
*   同一个元素可以使用类选择器设置多个样式
    * 用法举例：
    ```
    <head>
    <style>
    .cls1{
    color: green;
    }
    </style>
    </head>
    ...
    <body> <div>Hello CSS</div>
    </body>
    ```

#### ID选择器

*   语法：`#ID选择器名称{样式;}`
*   `#`开头
*   在需要设置样式的代码处使用`<span id="ID选择器名称"> </span>`
*   一个html文档中ID选择器只能使用一次
    * 用法举例：
    ```
    <head>
    <style>
    #div1{
    color: yellow;
    }
    </style>
    </head>
    ...
    <body> <div id="div1">Hello CSS</div>
    </body>
    ```
    
### 扩展选择器

#### 父选择器

*   使用大于符号`>` 选择指定标签元素的第一代子元素
*   用法举例
    .food > li{border:1px solid red;}
    使class名为food下的子元素li加入红色实线边框

#### 包含（后代 、子选择器）选择器

*   筛选选择器1元素下的选择器2元素
*   语法：`选择器1 选择器2{}`
    使用空格指定标签元素下的后辈元素，作用于所有后代元素
*   用法举例
    `.first span{color:red;}`

#### 通用选择器

*   使用 `*` 指定，用法`* {color:red;}` ，适用于html中所有标签元素

#### 伪类选择符

*   用于给html中鼠标划过的标签元素设置样式。
*   语法：`元素:状态`
*   状态
    *   `link`：初始化的状态
    *   `hover`：鼠标悬浮状态
    *   `active`：正在访问的状态
    *   `visited`：被访问过的状态
*   例
    ```
    <head>
    <style>
    a:link{color:pink}
    a:hover{color:red;font-size:20px;}
    a:active{color:green}
    a:visited{color:yellow}
    </style>
    </head>
    ...
    <body> <a>Hello CSS</a>
    </body>
    ```

#### 分组选择符(并集选择器)

*   选择符`,` 为html中多个标签元素设置同一个样式
*   例\
    `h1,span{color:red;font-size:20px;}`  `.first,#second>span{color:blue;font-size:30px;}`

#### 属性选择器

*   语法：元素名称\[属性名="属性值"]{}
*   用法举例：
    ```
    <head>
    <style>
    input[type='text']{
    border: 5px solid;
    }
    </style>
    </head>
    ...
    <body>
    <input type="text">Hello CSS</input>
    </body>
    ```

## 继承、层叠和特殊性

*   继承：CSS的某些样式具有继承性，但有些样式不具有继承性，例如`color:red`可以继承，但`border:1px solid red;`则不可继承
*   特殊性：浏览器根据权值来判断应该使用的CSS样式，权值高的优先
    *   标签的权值为1
    *   类选择符的权值为10
    *   ID选择符的权值最高为100
    *   继承的权限最低，为0.1
*   层叠：html中的元素，有多个权值相同的样式存在时，会根据这些css样式的前后顺序来选择，靠近元素的优先。
*   重要性：`!important` 具有最高权重
    *   浏览器默认的样式 < 网页制作者样式 < 用户自己设置的样式
    *   `!important` 优先级高于所有样式
    *   用法举例 `p{color:red!important;}` ，`!important` 要放在`;`号前面

## 属性、格式化排版

### 文字排版属性

*   字体：`font-family:"字体";` 如：`p{font-family:微软雅黑;}`
*   字号：`font-size:字号;` 如：`.stress{font-size:12px;}`
*   颜色：`color:颜色;` 如：`body{color:red;}`
*   粗体：`font-weight:bold;`
*   斜体：`font-style:italic;`
*   下划线：`text-decoration:underline;`
*   删除线：`text-decoration:line-through;`

### 段落排版属性

*   缩进：`text-indent:缩进大小;` 如：`p{text-indent:2em;}`表示缩进文字的2倍大小
*   行间距(行高)：`line-height:行高;` 如：`p{line-height:1.5em;}` 表示行间距为1.5倍
*   中文字间距、字母间距：
    *   `letter-spacing:字母或中文字间距;` 如：`h1{hetter-spacing:50px;}`
    *   `word-spacing:单词间距;` 如：`h1{word-spacing:50px;}`
*   对齐：`text-align:对齐方式(left|center|right);` 如：`h1{text-align:center;}`

### 背景属性

*   复合属性，可以设置对象的背景特性
*   `background`用法：
    *   `color`：背景颜色
    *   `image`：背景图像
    *   `position`：背景图像的位置，需要先指定image属性
    *   `repeat`：背景图像辅排填充，是否重复，需要先制定image属性
    *   `size`：尺寸大小
    *   其它属性参考 W3cSchool
*   示例
    ```
    <head>
    <style>
    div{
    background: url("img/logo.jpg") no-repeat center;
    }
    </style>
    </head>
    ...
    <body> <div>Hello CSS</div>
    </body>
    ```

## 元素分类

在CSS中，html中的标签元素大体被分为三种不同的类型：块状元素、内联元素（右脚行内元素）和内联块状元素，常见的各类型元素有：

### 块状元素

*   `display:block` 可以将元素设置为块级元素，例如`a{display:block;}`是将内联元素转换为块级元素，使得`a`元素具有块状元素的特点。
*   块级元素特点：
    *   每个块级元素都从新的一行开始，并且其后的元素也另起一行。
    *   元素的高度、宽度、行高及顶和底边距都可以设置。
    *   元素宽度在不设置的情况下，是他本身父容器的100%。
*   常见块状元素：
    *   `<div>`
    *   `<p>`
    *   `<h1>...<h6>`
    *   `<ol>`
    *   `<ul>`
    *   `<dl>`
    *   `<table>`
    *   `<address>`
    *   `<blockquote>`
    *   `<form>`

### 内联元素

*   `display:inline` 可以将元素设置为内联元素，如`div{display:inline;}`是将`div`转换为内联元素
*   内联元素特点：
    *   和其他元素都在一行上
    *   元素的高度、宽度及顶部和底部边距不可设置
    *   元素的宽度就是它包含的文字或图片的宽度，不可改变
*   常见内联元素：
    *   `<a>`
    *   `<span>`
    *   `<br>`
    *   `<i>`
    *   `<em>`
    *   `<strong>`
    *   `<label>`
    *   `<q>`
    *   `<var>`
    *   `<cite>`
    *   `<code>`

### 内联块状元素

*   `display:inline-block` 将元素设置为内联块状元素， 同时具备内联元素、块状元素的特点
*   内联块状元素特点：
    *   和其他元素都在一行上
    *   元素的高度、宽度、行高及顶和底边距都可以设置
*   CSS 2.1 新增了一些inline-block元素：`<img>`、`<input>`

## 盒模型

盒模型主要用于控制布局

### 边框(border)

*   盒子模型的边框就是围绕着内容及补白的线，可以设置线的粗细、样式和颜色(即边框的三个属性) ，
    * 举例：
    ```
    设置div边框为2px、样式为实心、颜色为红色
    div{
    border:2px solid red;
    }
    上面是border代码的缩写形式，分开写为：
    div{
    border-width:2px;
    border-style\:solid;
    border-color\:red;
    }
    ```
*   `border-style`常见样式有：`dashed(虚线)|dotted(点线)|solid(实线)`
*   `border-color`的颜色可以设置为十六进制颜色，如`border-color:#888;`
*   `border-width`中的宽度可以设置为`thin|medium|thick` 但最常用的还是像素 `px`
*   css样式允许只为一个方向的边框设置样式，`border-top 顶部|border-right 右侧|border-bottom 底部|border-left 左侧`， 如`div{border-bottom:1px solid red;}` 设置边框底部样式为1px像素宽度的实心红色线条
*   `box-sizing`：`border-box`，设置盒子的属性，让width和height就是最终盒子的大小

### 填充(padding)

*   内边距，元素内容与边框之间的距离称之为“填充”
    ```    
    用法举例：
    默认填充为上、右、下、左(顺时针)顺序
    div{
    padding:20px 10px 15px 30px;
    }
    分开写：
    div{
    padding-top:20px;
    padding-right:10px;
    padding-bottom:15px;
    padding-left:30px;
    }
    如果上、右、下、左填充都为10px，可以写为：
    div{padding:10px;}
    如果上下为10px，左右为20px，可以写为：
    div{padding:10px 20px;}
    ```

### 边界(margin)

*   外边距，元素与其它元素之间的距离可以使用边界来设置
    ```
    举例：
    div{margin:20px 10px 15px 30px;}
    分开写：
    div{
    margin-top:20px;
    margin-right:10px;
    margin-bottom:15px;
    margin-left:30px;
    }
    如果上、右、下、左边界都为10px，可以写为：
    div{margin:10px;}
    如果上下为10px，左右为20px，可以写为：
    div{margin:10px 20px;}
    ```

### 宽度和高度

*   盒模型的宽度和高度是指“内容+填充+边框+边界”的内容，而css内定义的宽(width)和高(height)是指填充以里的内容范围
    * 例
        * css代码：
            ```
            div{
                width:200px;
                padding:20px;
                border:1px solid red;
                margin:10px;
            }
            ```
        * html代码
            ```
            <body>
                <div>文本内容</div>
            </body>
            ```

元素实际宽度为：10px+1px+20px+200px+20px+1px+10px=262px，即左边界+左边框+左填充+元素内容宽度+右填充+右边框+右边界  ，元素的长度也是同理。

### 填充和边界区别

padding和margin的区别在于边框的内外，padding在边框里，margin在边框外

## 布局模型

布局模型与盒模型一样是CSS最基本、最核心的概念。CSS包含3中基本的布局模型：

*   流动模型 Flow
*   浮动模型 Float
*   层模型 Layer

### 流动模型

流动模型(Flow)是默认的网页布局模式，默认状态下的HTML网页元素都是根据流动模型来分布网页内容的。流动布局模型具有两个典型的特征：

1.  **块状元素**都会在所处的**包含元素内**自上而下按照顺序垂直延伸分布，默认状态下块状元素的宽度都为100%，以行的形式占据位置
2.  **内联元素**都会在所处的**包含元素内**从左到右水平分布显示

### 浮动模型

浮动元素可以实现多个块状元素并排显示，在块状元素中使用`float:位置`即可，如`div{float:left}`

### 层模型

层模型可以让每个图层精确定位操作，层模型有三种形式：

1.  **绝对定位**(position\:absolute)：将元素从文档流中拖出来，然后使用left、right、top、bottom属性相对于其==最接近的一个具有定位属性的父包含块==进行绝对定位。如果不存在这样的包含块，则相对于 body 元素，即相对于==浏览器窗口==
    * 举例
        ```
        div{
            width:200px;
            height:200px;
            border:2px red solid;
            position:absolute;
            left:100px;
            top:50px;
        }
        <div id="div1"></div>
        ```
        表示div元素相对于浏览器窗口向右移动100px，向下移动50px。
2.  **相对定位**(position\:relative)：通过left、right、top、bottom属性确定元素在**正常文档流**中的偏移位置。相对定位完成的过程是首先按static(float)方式生成一个元素(并且元素像层一样浮动了起来)，然后相对于==以前的位置移动==，移动的方向和幅度由==left、right、top、bottom==属性确定，==偏移前的位置保留不动==。
    * 举例
        ```
        div{
        width:200px;
        height:200px;
        border:2px red solid;
        position\:relative;
        left:100px;
        top:50px;
        } <div id="div1"></div>
        ```
        表示div元素相对于以前的位置向右移动100px，向下移动50px。
3.  **固定定位**(position\:fixed)：与absolute定位类型类似，但它相对移动的坐标是视图(**屏幕内的网页窗口**)本身。固定定位的元素会始终位于浏览器窗口内视图的某个位置，不会受文档流动影响。
    * 举例
        ```
        div{
        width:200px;
        height:200px;
        border:2px red solid;
        position\:fixed;
        left:100px;
        top:50px;
        } <div id="div1"></div>
        ```
        表示div元素相对于**浏览器视图**向右移动100px，向下移动50px，拖动滚动条时位置固定不变。
4.  Relative与Absolute组合使用：可以实现元素相对于其他元素进行定位，但必须遵守规范：
    *   参照定位的元素必须是相对定位元素的前辈元素 `<div id="box1"> <div id="box2">相对参照定位元素进行定位</div> </div>`
    *   参照定位的元素必须加入`position:relative;`
        ```
        #box1{
        width:200px;
        height:200px;
        position\:relative;
        }
        ```
    *   定位元素加入`position:absolute`，使用`top`、`bottom`、`left`、`right`进行偏移定位
        ```
        #box2{
        position\:absolute;
        top:20px;
        left:30px;
        }
        ```

## 代码缩写

### 盒模型代码简写

盒模型部分中有说明

### 颜色值缩写

设置颜色为16进制的色彩值时，如果两位的值相同，可以缩写一半。例如`p{color:#000000;}`可以缩写为`p{color:#000;}`，`p{color:#336699;}`可以缩写为`p{color:#369;}`

### 字体缩写

css样式代码也可以缩写，如下
    ```
    body{
        font-style:italic;
        font-variant:small-caps;
        font-weight:bold;
        font-size:12px;
        line-height:1.5em;
        font-family:"宋体",sans-serif;
    }
    ```

可以缩写为：
    ```
    body{
        font:italic small-caps bold 12px/1.5m “宋体”,sans-serif;
    }
    ```

需要注意：

*   至少需要指定 font-size 和 font-family 属性，其他属性未指定自动使用默认值
*   缩写时 font-size 与 line-height 中间要加入`/`斜杠

## 单位和值

### 颜色值

网页中的颜色有字体颜色(color)、背景颜色(background-color)、边框颜色(border)等，设置颜色的方法也很多:

1.  英文命令颜色，如`p{color:red;}`
2.  RGB颜色，可以为`0~255`的值，也可以是`0%~100%`的百分数，如`p{color:rgb(133,45,200);}`或`p{color:rgb(10%,20%,30%);}`
3.  十六进制颜色，现在比较普遍使用的方法，原理同RGB设置，值由0-255变为了十六进制00-ff，如`p{color:#00ffff;}`

### 长度单位

长度单位比较常用的有px(像素)、em、%百分比，都为相对单位

1.  px(像素)，像素是指显示器上的小点(CSS规范中假设“90像素=1英寸”)，实际上浏览器会使用显示器的实际像素值
2.  em，本元素给字体设定的font-size值，如元素的font-size为14px，那1em=14px；如果font-size为18px，那1em=18px。如`p{font-size:12px;text-indent:2em;}`表示段落首行缩进24px。
    ==当给font-size设置单位为em时，计算的标准以父元素的font-size为基础==
3.  百分比，`p{font-size:12px;line-height:130%}`设置行高为字体的130%(即12*1.3=15.6px)。

## 设置技巧

### 水平居中

*   行内元素：如果被设置元素为文本、图片等行内元素时，水平居中是通过给父元素设置`text-align:center`来实现
*   块状元素
    *   定宽块状元素：块状元素的宽度width为固定值称为定宽块状元素，可以通过设置“左右margin”值为“auto”来实现居中。
    *   不定宽块状元素有三种方法居中：
        1.  加入 table 标签
        2.  设置 display:inline 方法：与第一种类似，显示类型设为行内元素，进行不定宽元素的属性设置。优势是不用增加无语义标签，缺点是少了一些功能，比如设定长度值
        3.  设置 position\:relative 和left:50%：利用相对定位的方式，将元素想做偏移50%，即达到居中的定位目的

### 垂直居中

*   父元素高度确定的单行文本：通过设置父元素的height 和 line-height 高度一致实现。
*   父元素高度确定的多行文本
    1.  使用插入table(包括tbody、tr、td)标签，同时设置`vertical-align:middle`。注：td标签==默认情况下设置了vertical-align为middle==。
    2.  在chrome、firefox及IE8以上的浏览器下可以设置块级元素的display为table-cell(设置为表格单元显示)，激活 vertical-align 属性，但IE6、7不支持这个样式，兼容性较差。

### 隐形改变display类型

当为元素(不论之前是什么类型元素，`diaplay:none`除外)设置一下2个语句之一：

*   `position:absolute`
*   `float:left`或`float:right`
    元素的display显示类型就会自动变为以`display:inline-block(块状元素)`的方式显示，即可以设置width和height了，且默认宽度不占满父元素。