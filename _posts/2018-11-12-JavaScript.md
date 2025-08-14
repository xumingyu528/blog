---
title: JavaScript
category: 
  - 编程语言
tag: 
  - 前端技术
  - 编程语言
  - JavaScript
author: xmy
---

# JavaScript

> JavaScript是一种网络脚本语言，用来改进设计、验证表单、检测浏览器、创建cookie等，可以插入到HTML页面中。


## 简介
### 概念

*   运行在客户端浏览器中，每一个浏览器都有JavaScript的解析引擎
*   脚本语言：不需要编译，直接就可以被浏览器解析执行了

### 功能

*   可以来增强用户和html页面的交互过程，可以来控制html元素，让页面有一些动态的效果，增强用户的体验

### 发展史

*   1992年，Nombase公司，开发出第一门客户端脚本语言，专门用于表单的校验。命名为 ： C--	，后来更名为：ScriptEase
*   1995年，Netscape(网景)公司，开发了一门客户端脚本语言：LiveScript。后来，请来SUN公司的专家，修改LiveScript，命名为JavaScript
*   1996年，微软抄袭JavaScript开发出JScript语言
*   1997年，ECMA(欧洲计算机制造商协会)，制定出客户端脚本语言的标准：ECMAScript，就是统一了所有客户端脚本语言的编码方式。

## 语法

ECMAScript 语法

### 引用JS文件的方式

*   **插入JavaScript代码**\
    在HTML网页中在`<script></script>`标签中来插入JavaScript代码内容
*   **引入JS外部文件**\
    JS文件可以单独写在.js结尾的文件中，在JS文件中不需要`<script>`标签，但JS文件不能直接运行，需嵌入到HTML文件中执行
    `<script src="引用的JS文件名称"></script>`

### 位置

*   JS代码可以放在html文件中任何位置，但一般放在head或者body部分。
*   浏览器解析时是按照先后顺序，前面的script会先执行，通常会把用来初始化的js放在head中

### 基本语法

*   每行语句以`;`结尾，如果一行只有一条语句则 `;`可以省略 (不建议)
*   变量的定义使用var关键字，也可以不使用
    *   用： 定义的变量是局部变量
    *   不用：定义的变量是全局变量(不建议)

#### 注释

*   单行注释在前面加`//`
*   多行注释使用`/*`开始`*/`结尾。

#### 数据类型

*   原始数据类型(基本数据类型)
    *   number：数字。 整数/小数/NaN(not a number 一个不是数字的数字类型)
    *   string：字符串。 字符串  "abc" "a" 'abc'
    *   boolean: true和false
    *   null：一个对象为空的占位符
    *   undefined：未定义。如果一个变量没有给初始化值，则会被默认赋值为undefined
*   引用数据类型：对象

#### 变量

*   概念：一小块存储数据的内存空间

*   Java语言是强类型语言，而JavaScript是弱类型语言。
    *   强类型：在开辟变量存储空间时，定义了空间将来存储的数据的数据类型。只能存储固定类型的数据
    *   弱类型：在开辟变量存储空间时，不定义空间将来的存储数据类型，可以存放任意类型的数据。

*   定义变量语法：`var 变量名 = 值`
    *   必须使用字母、下环线`_`或美元符号`$$`开始
    *   可以由任意多个英文字母、数字、下划线`_`或者美元符号`$$`组成
    *   不能使用JavaScript关键字或JavaScript保留字
    *   变量要先声明再赋值
    *   变量可以重复赋值
    *   变量名区分大小写

*   typeof运算符：获取变量的类型。
    *   注：null运算后得到的是object

#### 运算符

1.  一元运算符：只有一个运算数的运算符 `++，-- ， +(正号)`
    *   \++ --: 自增(自减)
        *   \++(--) 在前，先自增(自减)，再运算
        *   \++(--) 在后，先运算，再自增(自减)
    *   \+(-)：正负号
    *   注意：在JS中，如果运算数不是运算符所要求的类型，那么js引擎会自动的将运算数进行类型转换
        *   其他类型转number：
            *   string转number：按照字面值转换。如果字面值不是数字，则转为NaN（不是数字的数字）
            *   boolean转number：true转为1，false转为0
2.  算数运算符 `+ - * / % ...`
3.  赋值运算符`= += -+....`
4.  比较运算符`> < >= <= == ===(全等于)`
    *   比较方式
        1.  类型相同：直接比较
            *   字符串：按照字典顺序比较。按位逐一比较，直到得出大小为止。
        2.  类型不同：先进行类型转换，再比较
            *   \===：全等于。在比较之前，先判断类型，如果类型不一样，则直接返回false
5.  逻辑运算符
    `&& || !`
    *   其他类型转boolean：
        1.  number：0或NaN为假，其他为真
        2.  string：除了空字符串("")，其他都是true
        3.  null\&undefined:都是false
        4.  对象：所有对象都为true
6.  三元运算符
    `? : 表达式`
        var a = 3;
        var b = 4;

        var c = a > b ? 1:0;
    *   语法：
        *   表达式? 值1:值2;
        *   判断表达式的值，如果是true则取值1，如果是false则取值2；

#### 流程控制

1.  if判断语句
    ```js
    if(条件)
    {
        执行代码;
    }
    else{
        执行代码;
    }
    ```
1.  switch:
    *   在java中，switch语句可以接受的数据类型： byte int shor char,枚举(1.5) ,String(1.7)
        *   switch(变量):
            case 值:
    *   在JS中,switch语句可以接受任意的原始数据类型
2.  while
3.  do...while
4.  for

#### 练习
```html
练习：99乘法表
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>99乘法表</title>
    <style>
        td{
            border: 1px solid;
        }

    </style>

    <script>

        document.write("<table  align='center'>");


        //1.完成基本的for循环嵌套，展示乘法表
        for (var i = 1; i <= 9 ; i++) {
            document.write("<tr>");
            for (var j = 1; j <=i ; j++) {
                document.write("<td>");

                //输出  1 * 1 = 1
                document.write(i + " * " + j + " = " + ( i*j) +"&nbsp;&nbsp;&nbsp;");

                document.write("</td>");
            }
            /*//输出换行
            document.write("<br>");*/

            document.write("</tr>");
        }

        //2.完成表格嵌套
        document.write("</table>");

    </script>
</head>
<body>

</body>
</html>
```

### 基本对象

#### Function：函数(方法)对象

1.  创建：
    ```js
    1.  var fun = new Function(形式参数列表,方法体);  //忘掉吧
    2.  function 方法名称(形式参数列表){
        方法体
        }
    3.  var 方法名 = function(形式参数列表){
        方法体
        }
    ```
2.  方法：
    在需要执行的位置调用函数名，函数格式如下：
    ```js
    function 函数名()
    {
        函数代码;
    }
    ```
    *   匿名函数：
        ```js
        function () {
            
        }
        ```
    *   自执行函数
        ```js
        (function(){
            //代码
        })();
        ```
     ```
    
     ```
    
3.  属性
    *   `length` : 代表形参的个数
        ```js
        function fun1(a,b){
            ...
        }
        var parameter_num = fun1.length;
        输出 parameter_num 为2，因为fun1有2个参数
        ```
4.  特点：
    1.  方法定义是，形参的类型不用写,返回值类型也不写。
    2.  方法是一个对象，如果定义名称相同的方法，会覆盖
    3.  在JS中，方法的调用只与方法的名称有关，和参数列表无关
    4.  在方法声明中有一个隐藏的内置对象（数组），arguments,封装所有的实际参数

5.  调用：
    方法名称(实际参数列表);
    ```js
    function fun1(a,b){
        ...
    }

    调用：
    fun1(1,2);
    ```
#### Array：数组对象

1.  创建：
    1.  var arr = new Array(元素列表);
    2.  var arr = new Array(默认长度);
    3.  var arr = [元素列表];
2.  方法
    *   join(参数):将数组中的元素按照指定的分隔符拼接为字符串
    *   push()	向数组的末尾添加一个或更多元素，并返回新的长度。
    *   其他方法可以参考W3cSchool
3.  属性
    length:数组的长度
4.  特点：
    1.  JS中，数组元素的类型可变的。
    2.  JS中，数组长度可变的。

#### Date：日期对象

1.  创建：
    var date = new Date();

2.  方法：
    *   toLocaleString()：返回当前date对象对应的时间本地字符串格式
    *   getTime():获取毫秒值。返回当前如期对象描述的时间到1970年1月1日零点的毫秒值差
    *   其他方法可以参考W3cSchool

#### Math：数学对象

1.  创建：
    *   特点：Math对象不用创建，直接使用。  Math.方法名();

2.  方法：
    *   random():返回 0 \~ 1 之间的随机数。 含0不含1
    *   ceil(x)：对数进行上舍入。
    *   floor(x)：对数进行下舍入。
    *   round(x)：把数四舍五入为最接近的整数。
    *   其他方法可以参考W3cSchool

3.  属性：
    PI

#### RegExp：正则表达式对象

1.  正则表达式：定义字符串的组成规则。
    1.  单个字符:\[]
        *   如： \[a] \[ab] \[a-zA-Z0-9\_]
        *   特殊符号代表特殊含义的单个字符:
            *   \d:单个数字字符 \[0-9]
            *   \w:单个单词字符\[a-zA-Z0-9\_]
    2.  量词符号：
        *   ?：表示出现0次或1次
        *   \*：表示出现0次或多次
        *   \+：出现1次或多次
        *   {m,n}:表示 m<= 数量 <= n
            *   m如果缺省： {,n}:最多n次
            *   n如果缺省：{m,} 最少m次
    3.  开始结束符号
        *   ^:开始
        *   \$:结束
2.  正则对象：
    1.  创建
        1.  var reg = new RegExp("正则表达式");
        2.  var reg = /正则表达式/;
    2.  方法
        1.  test(参数):验证指定的字符串是否符合正则定义的规范
            ```js
            示例
            var username = "zhangsan";
            var reg = RegExp("^\\w{6,12}");
            var reg2 = /^\w{6,12}/;

            reg.test(username);
            reg2.test(username);
            ```

#### Global：全局对象

1.  特点：全局对象，这个Global中封装的方法不需要对象就可以直接调用。  方法名();
2.  方法：
    *   `encodeURI()`:url编码
    *   `decodeURI()`:url解码
    *   `encodeURIComponent()`:url编码,编码的字符更多
    *   `decodeURIComponent()`:url解码
    *   `parseInt()`:将字符串转为数字
        *   逐一判断每一个字符是否是数字，直到不是数字为止，将前边数字部分转为number
    *   `isNaN()`:判断一个值是否是NaN
        *   NaN六亲不认，连自己都不认。NaN参与的==比较全部问false
    *   `eval()`:将字符串转换为JavaScript代码，并把它作为脚本代码来执行。
3.  URL编码
    随便写点 =  %E9%9A%8F%E4%BE%BF%E5%86%99%E7%82%B9

```js
var str = "随便写点";
var encode = encodeURI(str);
document.write(encode + "<br>"); //%E9%9A%8F%E4%BE%BF%E5%86%99%E7%82%B9

var s = decodeURI(encode);
document.write(s + "<br>"); 

// parseInt() 方法
var str = "123abc";
var number = parseInt(str);
document.write(number + 1); // 124

// eval() 方法
var jscode = "alert(123)";
eval(jscode);

```

#### 其他对象

*   Boolean
*   Number
*   String

## BOM

### 概念

Browser Object Model 浏览器对象模型

*   将浏览器的各个组成部分封装成对象

### 组成

*   Window：窗口对象
*   Navigator：浏览器对象
*   Screen：显示器屏幕对象
*   History：历史记录对象
*   Location：地址栏对象

### Window：窗口对象

1.  创建

2.  方法
    1.  与弹出框有关的方法：
        *   alert()	显示带有一段消息和一个确认按钮的警告框。
        *   confirm()	显示带有一段消息以及确认按钮和取消按钮的对话框。
            *   如果用户点击确定按钮，则方法返回true
            *   如果用户点击取消按钮，则方法返回false
                prompt()	显示可提示用户输入的对话框。
            *   返回值：获取用户输入的值
    2.  与打开关闭有关的方法：
        *   close()	关闭浏览器窗口。
            *   谁调用我 ，我关谁
        *   open()	打开一个新的浏览器窗口
            *   返回新的Window对象
    3.  与定时器有关的方式
        *   setTimeout()	在指定的毫秒数后调用函数或计算表达式。
            *   参数：
                1.  js代码或者方法对象
                2.  毫秒值
            *   返回值：唯一标识，用于取消定时器
        ```js
        定义一个方法：
        function fun(){
            alert('boom~~');
        }
        // 一次性定时器
        setTimeout("fun();",2000);
        setTimeout(fun,2000);

        //循环定时器
        setInterval(fun,2000);

        ```
        *   clearTimeout()	取消由 setTimeout() 方法设置的 timeout。
            ```js
            function fun(){
                alert('boom~~');
            }
            var id = setTimeout(fun,2000);
            clearTimeout(id);
            ```
        *   setInterval()	按照指定的周期（以毫秒计）来调用函数或计算表达式。
        *   clearInterval()	取消由 setInterval() 设置的 timeout。

3.  属性：
    1.  获取其他BOM对象：
        *   history
        *   location
        *   Navigator
        *   Screen:
    2.  获取DOM对象
        *   document

4.  特点
    *   Window对象不需要创建可以直接使用 window使用。 window.方法名();
    *   window引用可以省略。  方法名();

### Location：地址栏对象

1.  创建(获取)：
    1.  window.location
    2.  location

2.  方法：
    *   reload()	重新加载当前文档。刷新

3.  属性
    *   href	设置或返回完整的 URL。

### History：历史记录对象

1.  创建(获取)：
    1.  window\.history
    2.  history

2.  方法：
    *   back()	加载 history 列表中的前一个 URL。
    *   forward()	加载 history 列表中的下一个 URL。
    *   go(参数)	加载 history 列表中的某个具体页面。
        *   参数：
            *   正数：前进几个历史记录
            *   负数：后退几个历史记录

3.  属性：
    *   length	返回当前窗口历史列表中的 URL 数量。

## 常用互动方法

一般互动方法，在用户点击对话框的按钮之前，不能进行其它操作

### 输出内容

`document.write()`可用于直接向HTML输出流写内容。

*   用""括起，直接输出""号中的内容
*   通过变量，输出内容
*   输出多项内容，内容之间用 `+` 号连接
*   输出HTML标签，并起作用，标签使用""括起来

示例：
```js
<script type="text/javascript">
        document.write("hello JS");
</script>
```

### 警告

`alert`：在网页中弹出消息窗口，弹出的消息对话框包含一个确定按钮\
语法：`alert(字符串或变量);`\
特性：

*   点击对话框“确定”按钮前，不能进行任何其他操作
*   消息对话框通常用于调试程序
*   alert输出内容，可以是字符串或变量，与document.write相似

示例：
```js
<script type="text/javascript">
        alert("hello JS");
</script>
```

### 确认

`confirm` 消息对话框通常用于允许用户选择的动作，弹出对话框包括一个确定按钮和取消按钮\
语法：`confirm(str);`，参数说明：

*   str是消息对话框中要显示的文本
*   返回值：Boolean值，用户点击“确定”按钮时，返回true；点击“取消”按钮时，返回false。通过返回值可以判断用户点击的按钮

示例
```js
<script type="text/javascript">
    var mymessage=confirm("你喜欢JavaScript吗？");
    if(mymessage==true)
    { document.write("很好，加油");}
    else
    { document.write("你有选择不学的权利");}
</script>
```

### 提问

`prompt`消息对话框，通常用于询问一些需要与用户交互的信息，弹出消息对话框包含一个确定按钮、取消按钮和一个文本输入框\
语法：`prompt(str1, str2);`，参数说明

*   str1：要显示在消息对话框中的文本，不可修改
*   str2：文本框中的内容，可以修改
*   返回值：点击确定按钮，文本框中的内容将作为函数返回值；点击取消按钮，将返回null

示例：
```js
<script type="text/javascript">
    prompt("hello","JS");
</script>
</script>
```

### 打开新窗口

`window.open`可以查找一个已经存在或者新建的浏览器窗口\
语法：`window.open([URL], [窗口名称], [参数字符串1,参数字符串2,...])`\
参数说明：

*   URL：可选参数，在窗口中要显示网页的网址或路径，如果省略这个参数，或者它的值是空字符串，那么窗口就不显示任何文档。
*   窗口名称：可选参数，被打开窗口的名称
    *   由字母、数字和下划线字符组成
    *   `_top`、`_blank`、`_self`具有特殊意义的名称
        *   `_blank`：在新窗口显示目标网页
        *   `_self`：在当前窗口显示目标网页
        *   `_top`：框架网页中在上部窗口中显示目标网页
    *   相同name的矿口只能创建一个，要想创建多个窗口则name不能相同
    *   name不能包含有空格
*   参数字符串：可选参数，设置窗口参数，各参数用逗号隔开
    ![image](http://img.mukewang.com/52e3677900013d6a05020261.jpg)<http://img.mukewang.com/52e3677900013d6a05020261.jpg>\
    * 示例：打开www\.xmy.com网站，大小300\*200px，无菜单，无工具栏，无状态栏，有滚动条窗口
        ```js
        <script type="text/javascript">
        window.open('http://www.xmy.com','_blank','width=300,height=200,menubar=no,toolbar=no,status=no,scrollbars=yes')
        </script>
        ```

### 关闭窗口

`close()`关闭窗口\
用法：`window.close();` 或 `<窗口对象>.close();`\
示例：关闭新建的窗口
```js
<script type="text/javascript">
var mywin=window.open('http://www.xmy.com');
mywin.close();
</script>
```

## DOM

### 概念

Document Object Model 文档对象模型

*   将标记语言文档的各个组成部分，封装为对象。可以使用这些对象，对标记语言文档进行CRUD的动态操作

### 分类

W3C DOM 标准被分为 3 个不同的部分：

*   核心 DOM - 针对任何结构化文档的标准模型
    *   Document：文档对象
    *   Element：元素对象
    *   Attribute：属性对象
    *   Text：文本对象
    *   Comment:注释对象
    *   Node：节点对象，其他5个的父对象
*   XML DOM - 针对 XML 文档的标准模型
*   HTML DOM - 针对 HTML 文档的标准模型

### 核心DOM模型：

#### Document：文档对象

1.  创建(获取)：在html dom模型中可以使用window对象来获取
    1.  window\.document
    2.  document
2.  方法：
    1.  获取Element对象：
        1.  getElementById()	： 根据id属性值获取元素对象。id属性值一般唯一
        2.  getElementsByTagName()：根据元素名称获取元素对象们。返回值是一个数组
        3.  getElementsByClassName():根据Class属性值获取元素对象们。返回值是一个数组
        4.  getElementsByName(): 根据name属性值获取元素对象们。返回值是一个数组
    2.  创建其他DOM对象：
        *   createAttribute(name)
        *   createComment()
        *   createElement()
        *   createTextNode()
3.  属性

#### Element：元素对象

1.  获取/创建：通过document来获取和创建
2.  方法：
    1.  removeAttribute()：删除属性
    2.  setAttribute()：设置属性

#### Node：节点对象，其他5个的父对象

*   特点：所有dom对象都可以被认为是一个节点
*   方法：
    *   CRUD dom树：
        *   appendChild()：向节点的子节点列表的结尾添加新的子节点。
        *   removeChild()	：删除（并返回）当前节点的指定子节点。
        *   replaceChild()：用新节点替换一个子节点。
*   属性：
    *   parentNode 返回节点的父节点。

### HTML DOM

1.  标签体的设置和获取：innerHTML
2.  使用html元素对象的属性
3.  控制元素样式
    1.  使用元素的style属性来设置
        ```js
        <div id="div1">
            div
        </div>
        //修改样式方式1
        var div1 = document.getElementById("div1");
        	div1.onclick = function(){
            div1.style.border = "1px solid red";
            div1.style.width = "200px";
            //font-size--> fontSize
            div1.style.fontSize = "20px";
        }
        ```
    2.  提前定义好类选择器的样式，通过元素的className属性来设置其class属性值。

### DOM操作

> DOM：文档对象模型 Document Object Model，定义访问和处理HTML文档的标准方法。DOM将HTML文档呈现为带有元素、属性和文本的树结构(节点树)。

常见的DOM节点：

*   元素节点：`<html>`、`<body>`、`<p>`等都是元素节点，即标签
*   文本节点：向用户展示的内容，如`<li>...</li>`中的内容
*   属性节点：元素属性，如`<a>`标签的链接属性herf="<http://www.abc.com>"

#### 通过ID获取元素

语法：`document.getElementById("id名称")`

#### innerHTML属性

innerHTML属性用于获取或替换HTML元素的内容。\
语法：`Object.innerHTML`

*   Object是获取的元素对象，如通过`document.getElementById("ID")`获取的元素
*   innerHTML区分大小写

示例：
```html
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<title>innerHTML</title>
</head>
<body>
<h2 id="con">javascript</H2>
<p> JavaScript是一种基于对象、事件驱动的简单脚本语言，嵌入在HTML文档中，由浏览器负责解释和执行，在网页上产生动态的显示效果并实现与用户交互功能。</p>
<script type="text/javascript">
    var mychar=     document.getElementById("con");     
    document.write("原标题:"+mychar.innerHTML+"<br>"); //输出原h2标签内容
    mychar.innerHTML="Hello World!"
    document.write("修改后的标题:"+mychar.innerHTML); //输出修改后h2标签内容
</script>
</body>
</html>
```

#### 改变HTML样式

语法：`Object.style.property=new style;`\
Object是获取的元素对象，property是元素属性。\
示例：改变标签`<p>`的样式为红色，背景颜色为蓝色
```html
<p id="pcon">Hello World!</p>
<script>
    var mychar = document.getElementById("pcon");
    mychar.style.color="red";
    mychar.style.fontSize="20";
    mychar.style.backgroundColor ="blue";
</script>
```

#### 显示和隐藏

语法：`Object.style.display = value `\
value取值none(元素不会被显示)或者block(元素将显示为块级元素)

示例：
```html
<!DOCTYPE HTML>
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=gb2312">
<title>display</title>
    <script type="text/javascript"> 
        function hidetext()  
        {  
        var mychar = document.getElementById("con");
            Object.style.display = "none";
        }  
        function showtext()  
        {  
        var mychar = document.getElementById("con");
            Object.style.display = "block";
        }
    </script> 
</head> 
<body>  
    <h1>JavaScript</h1>  
    <p id="con">做为一个Web开发师来说，如果你想提供漂亮的网页、令用户满意的上网体验，JavaScript是必不可少的工具。</p> 
    <form>
        <input type="button" onclick="hidetext()" value="隐藏内容" /> 
        <input type="button" onclick="showtext()" value="显示内容" /> 
    </form>
</body> 
```

#### 控制类名

`className`属性设置或返回元素的class属性\
语法：`object.className = classname`

1.  获取元素的class属性
2.  为网页内的某个元素指定一个css样式来更改该元素的外观
    慕课网示例：
    ```html
    <!DOCTYPE HTML>
    <html>
    <head>
    <meta http-equiv="Content-Type" content="text/html; charset=gb2312">
    <title>className属性</title>
    <style>
        body{ font-size:16px;}
        .one{
    		border:1px solid #eee;
    		width:230px;
    		height:50px;
    		background:#ccc;
    		color:red;
        }
    	.two{
    		border:1px solid #ccc;
    		width:230px;
    		height:50px;
    		background:#9CF;
    		color:blue;
    	}
    	</style>
    </head>
    <body>
        <p id="p1" > JavaScript使网页显示动态效果并实现与用户交互功能。</p>
        <input type="button" value="添加样式" onclick="add()"/>
    	<p id="p2" class="one">JavaScript使网页显示动态效果并实现与用户交互功能。</p>
        <input type="button" value="更改外观" onclick="modify()"/>

    	<script type="text/javascript">
    	   function add(){
    	      var p1 = document.getElementById("p1");
    	      p1.className="one";
    	   }
    	   function modify(){
    	      var p2 = document.getElementById("p2");
    	      p2.className="two";
    	   }
    	</script>
    </body>
    </html>
    ```

#### 取消所有样式

`Object.removeAttribute('style');`  
```html
<!DOCTYPE HTML>
<html>
<head>
<meta http-equiv="Content-Type" Content="text/html; charset=utf-8" />
<title>javascript</title>
<style type="text/css">
body{font-size:12px;}
#txt{
    height:400px;
    width:600px;
    border:#333 solid 1px;
    padding:5px;}
p{
    line-height:18px;
    text-indent:2em;}
</style>
</head>
<body>
    <h2 id="con">JavaScript课程</H2>
    <div id="txt"> 
        <h5>JavaScript为网页添加动态效果并实现与用户交互的功能。</h5>
        <p>1. JavaScript入门篇，让不懂JS的你，快速了解JS。</p>
        <p>2. JavaScript进阶篇，让你掌握JS的基础语法、函数、数组、事件、内置对象、BOM浏览器、DOM操作。</p>
        <p>3. 学完以上两门基础课后，在深入学习JavaScript的变量作用域、事件、对象、运动、cookie、正则表达式、ajax等课程。</p>
    </div>
    <form>
    <!--当点击相应按钮，执行相应操作，为按钮添加相应事件-->
    <input type="button" value="改变颜色" onclick="fc1()" >  
    <input type="button" value="改变宽高" onclick="fc2()" >
    <input type="button" value="隐藏内容" onclick="fc3()" >
    <input type="button" value="显示内容" onclick="fc4()" >
    <input type="button" value="取消设置" onclick="fc5()" >
    </form>
    <script type="text/javascript">
    var obj=document.getElementById("txt");
//定义"改变颜色"的函数
function fc1(){
    obj.style.color=("red");
    obj.style.backgroundColor=("yellow"); 
}

//定义"改变宽高"的函数
function fc2(){
    obj.style.width=("600px");
    obj.style.height=("500px");  
}

//定义"隐藏内容"的函数
function fc3(){
    obj.style.display="none";
}

//定义"显示内容"的函数
function fc4(){
    obj.style.display="block";
}


//定义"取消设置"的函数
function fc5(){
    if(confirm("确定取消所有样式吗？")==true)
    {
        obj.removeAttribute('style');   
    }

}


    </script>
</body>
</html>
```

### 事件监听机制

#### 概念

某些组件被执行了某些操作后，触发某些代码的执行。

*   事件：某些操作。如： 单击，双击，键盘按下了，鼠标移动了
*   事件源：组件。如： 按钮 文本输入框...
*   监听器：代码。
*   注册监听：将事件，事件源，监听器结合在一起。 当事件源上发生了某个事件，则触发执行某个监听器代码。

#### 常见的事件

1.  点击事件：
    1.  onclick：单击事件
    2.  ondblclick：双击事件
2.  焦点事件
    1.  onblur：失去焦点
    2.  onfocus:元素获得焦点。
3.  加载事件：
    1.  onload：一张页面或一幅图像完成加载。
4.  鼠标事件：
    1.  onmousedown	鼠标按钮被按下。
    2.  onmouseup	鼠标按键被松开。
    3.  onmousemove	鼠标被移动。
    4.  onmouseover	鼠标移到某元素之上。
    5.  onmouseout	鼠标从某元素移开。
5.  键盘事件：
    1.  onkeydown	某个键盘按键被按下。
    2.  onkeyup		某个键盘按键被松开。
    3.  onkeypress	某个键盘按键被按下并松开。
6.  选择和改变
    1.  onchange	域的内容被改变。
    2.  onselect	文本被选中。
7.  表单事件：
    1.  onsubmit	确认按钮被点击。
        *   可以组织表单的提交
            *   方法返回false则表单被阻止提交
    2.  onreset	重置按钮被点击。

## 数组

创建数组语法：\
`var 数组名=new Array([数组长度]);`

*   创建的新数组是空数组，没有值，如输出数组则显示undefined
*   数组长度可变，即使指定了数组长度，超过长度仍可以存储元素并使用
*   数组存储的数据可以是任何类型(数字、字符、布尔值等)
*   数组索引号从0开始
*   使用新索引可以随时向数组中增加新元素

示例：

```js
var myarray=new Array(); //创建一个新的空数组
myarray[0]=66; //存储第1个人的成绩
myarray[1]=80; //存储第2个人的成绩
myarray[2]=90; //存储第3个人的成绩
myarray[3]=77; //存储第4个人的成绩
myarray[4]=59; //存储第5个人的成绩

简化方法一：
var myarray = new Array(66,80,90,77,59);//创建数组同时赋值
简化方法二：
var myarray = [66,80,90,77,59];//直接输入一个数组（称 “字面量数组”）

```

获取数组长度`数组名.length`\
示例`document.write(数组名.length)`