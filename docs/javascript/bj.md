# 布局

## 一 传统布局

> 在开始下面的内容之前，我们先需要明确HTML 中的几种布局方式
>
> 标准流 （默认布局）
>
> 定位
>
> 浮动



## 1.1 标准流

### 1.1.1 html两大元素

| 常见块级元素              | 常见内联元素 |
| ------------------------- | ------------ |
| div                       | span         |
| h1~h6                     | img          |
| 有序，无序列表 ol、ul、li | a            |
| table                     | input        |
| p段落                     |              |

> 块级元素特点：

独占一行

> 内联元素特点：

和相临元素在同一行，一行不够时，才会被挤到下一行

> 演示

![avatar](https://picture.zhanghong110.top/docsify/16608694162625.png)

![avatar](https://picture.zhanghong110.top/docsify/16608695418129.png)

![avatar](https://picture.zhanghong110.top/docsify/16608698246887.png)

![avatar](https://picture.zhanghong110.top/docsify/16608698502740.png)

> 可以得到如下结论

1、span标签是使用来组合文档中的行内元素，以便使用样式来对它们进行格式化。

2、span标签本身并没有什么格式表现，不像块级元素（如：div标签、p标签等）哪样有换行的效果，需要对它应用样式才会有视觉上的变化。

3、span标签不能设置宽度和高度，只能设置左右padding和margin值。

4、span标签可以设置id或class属性，这样不仅能增加语义，还能更方便的对span元素应用样式。

5、span在行内定义一个区域，也就是一行内可以被<span>划分成好几个区域，从而实现某种特定效果。

6、通常对未设置任何样式的span，高宽是自适应内容，多容多少，此标签就占用多少距离空间。

**<span>与<div>的区别**

1、<span>在CSS定义中属于一个行内元素，而<div>是块级元素，我们可能通俗地理解为<div>为大容器，大容器当然可以放一个小容器了，<span>就是小容器。

2、div占用的位置是一行，span占用的是内容有多宽就占用多宽的空间距离。

3、<span>在<div>中一般都是用于显示一段文本，<span>默认是横排的，而<div>默认是竖排的，用<span>有时候是为了使页面元素看起来规整，没有什么特别的用处。

> 以上的布局就是我最常见的标准流布局

### 1.1.2 display属性

> 可以用来转换块和行元素

1、block（块元素）——常用

将元素转换为块元素（块元素，可以设置宽高了）。

2、inline（行内元素）

将元素转换为行内元素（行内元素，可以共用一行了）。

3、inline-block（行内块元素）

4、none（隐藏元素）



## 1.2 定位布局

> 定位布局关键词为position 属性

**position 属性决定了元素如何定位**
**通过 top，right，bottom，left 实现位置的改变**



`position`属性有以下参数

| position 参数 | 解释                                                         |
| ------------- | ------------------------------------------------------------ |
| static        | 默认值，元素按照标准流正常的显示                             |
| relative      | 相对定位，元素依然处于正常的文档流中，可以通过 left ， right，bottom，top 改变元素的位置 |
| absolute      | 绝对定位，元素脱离文档流，可以通过 left ， right，bottom，top 改变元素的位置，它会基于游览器的四个边角进行定位 |
| fixed         | 固定定位，使用 top，left，right，bottom 定位，会脱离正常文档流，不受标准流的约束，并拥有层级的概念 |
| inherit       | 会继承父元素的属性                                           |

> 下面我们来演示下

`static` 由于是正常的标准流，就不演示了。需要注意**static(静态)** 没有特别的设定，遵循基本的定位规定，**不能**通过z-index进行层次分级。



`relative`不脱离文档流，参考自身静态位置通过 top,bottom,left,right 定位，并且可以通过z-index进行层次分级。可以看见它相对于本身的位置进行了位移

**以下截图，left: 0 这一项都可以去掉。手抖了**

![avatar](https://picture.zhanghong110.top/docsify/16608726174833.png)



![avatar](https://picture.zhanghong110.top/docsify/16608720315883.png)

> 可以看到它相对于自己本身的定位产生了位移，left，top 属性可以理解为 div 左上角为基准移动
> right，bottom 属性可以理解为 div 右下角为基准移动，并且后写的元素会覆盖先写的元素，这样层级的概念就出来了



`absolute`脱离文档流，通过 top,bottom,left,right 定位。选取其最近一个有定位设置的父级对象进行绝对定位，如果对象的父级没有设置定位属性，absolute元素将以body坐标原点进行定位，可以通过z-index进行层次分级。

![avatar](https://picture.zhanghong110.top/docsify/16608752335855.png)

![avatar](https://picture.zhanghong110.top/docsify/16608751857240.png)

可以看到`puple`相对于`body`进行了定位，高和左边都是`100px`(如果是`right`,`bottom`就会相对于右下角定位),而`green`则是对于`puple`进行了定位，相对于`puple`的左上角高和左边偏移了`100px`。

!>值得一提的是，`absolute`脱离文档流，啥意思呢，HTML中全部元素都是盒模型，盒模型占用一定空间，将窗体自上而下分成一行一行，并且在每行中按照从左往右依次排放元素，称为文档流。元素脱离文档流之后就不再在文档流中占据空间，而是处于浮动状态，相当于漂浮在其他元素上方，那么其他元素就会忽略该元素并填补这个元素原来的空间。如果是`relative`则虽然相对于自身进行了位移，但是其他原来的位置还会占用，其它元素会给它让出位置，而`absolute`脱离后则文本流中的内容会顶替绝对定位无素的位置，一点都不会客气。



`fixed`使用 fixed 固定定位的元素不会受其它元素的约束，它也是以游览器的四个边角为基准，但是当页面发生滚动的时候，使用 fixed 定位的元素，会依然在页面中的位置固定不动，类比 一些广告。很好理解，这里就不演示了。

下面我们演示下 `固定定位` 和` 绝对定位`的区别。

![avatar](https://picture.zhanghong110.top/docsify/16608762771670.png)

![avatar](https://picture.zhanghong110.top/docsify/16608764542941.png)

> 可以看到，固定定位作为子元素还是相对于body定位了，游离在父级之外。



`inherit`最后我们看一下这个属性，子元素会继承父元素的定位属性，父元素的变化，子元素也会相对变化

![avatar](https://picture.zhanghong110.top/docsify/16608768396354.png)

![avatar](https://picture.zhanghong110.top/docsify/16608769001208.png)

可以看到它集成了父标签的绝对定位。

## 1.3 浮动布局

浮动元素是脱离默认文档流，但不脱离文本流的一种布局结构。通过让元素浮动，我们可以使元素在水平上左右移动，再通过margin属性调整位置。浮动的框可以左右移动，直至它的外边缘遇到包含框或者另一个浮动框的边缘。

![avatar](https://picture.zhanghong110.top/docsify/16608874884387.png)

![avatar](https://picture.zhanghong110.top/docsify/16608875405850.png)

我们可以看到，浮动的元素并没有占用文档流位置，红的的div还是从0，0开始

> 如果我们又想浮动又想占用文档流只要像以下凡是使用即可

```html
<!DOCTYPE html>
<html>

<head>
  <meta charset="utf-8">
  <style type="text/css">
    .box1 {
      width: 200px;
      height: 200px;
      border: 1px solid yellow;
      float: left;
    }

    .box2 {
      width: 100px;
      height: 100px;
      border: 1px solid green;
      float: left;
    }

    .box {
      width: 500px;
      height: 500px;
      background: red;
    }

    .clear::before,
    .clear::after {
      content: "";
      display: block;
      clear: both;
    }
  </style>
</head>

<body>
  <div class=clear>
    <div class="box1">
      <div class="box2"></div>
    </div>
  </div>

  <div class="box"></div>
</body>

</html>
```

效果如下

![avatar](https://picture.zhanghong110.top/docsify/16608879151229.png)

其实现原理是设置一个空的div来清除浮动。只不过是使用选择器来添加空的div而已。

这样做的好处是：如果后面还有想要清除浮动的地方，只需要添加一个**父div容器，并设置class="clear"即可**

## 二 flex布局

传统布局，基于盒模型，依赖 `display`属性 、`position`属性 、`float`属性
它对于那些特殊布局非常不方便，比如垂直居中

2009年，W3C 提出了一种新的方案—-Flex 布局，可以简便、完整、响应式地实现各种页面布局。目前，它已经得到了所有浏览器的支持，这意味着，现在就能很安全地使用这项功能。

Flex 是 Flexible Box 的缩写，意为”弹性布局”，用来为盒状模型提供最大的灵活性。
任何一个容器都可以指定为 Flex 布局。

```css
.box{
  display: flex;
}

//Webkit内核的浏览器，必须加上-webkit前缀。
.box{
  display: -webkit-flex; /* Safari */
  display: flex;
}
```

注意，设为 Flex 布局以后，子元素的float、clear和vertical-align属性将失效。

采用 Flex 布局的元素，称为 Flex 容器（flex container），简称”容器”。***它的所有子元素称为 Flex 项目（flex item***），简称”项目”

> 当使用flex进行布局的时候，首先需要了解的，就是flex的两根轴线：主轴和交叉轴。

![avatar](https://picture.zhanghong110.top/docsify/16608898176155.png)

想要更高的学习和使用flex布局，需要了解flex中包含的属性。一般来说，flex的属性可以分成两类:

- flex容器属性(flex-container)
- flex子元素属性(flex-item)

所谓的flex容器属性就是将属性设置在flex容器上，而flex子元素则是将属性设置在子元素的身上。

然后这玩意实在太长了，我们看下这个文档，写的不错。

[菜鸟教程文档](https://www.runoob.com/w3cnote/flex-grammar.html)

> 有一个简写`flex：1`，常用于内容的垂直居中。

`flex：1`   即为flex-grow：1，经常用作自适应布局，将父容器的`display：flex`，侧边栏大小固定后，将内容区`flex：1`，内容区则会自动放大占满剩余空间。

**原理**
flex属性 是 flex-grow、flex-shrink、flex-basis三个属性的缩写。

**flex等于数值的情况如下:**

- flex: 1

  - flex-grow: 1; //默认0 如果所有项目的flex-grow属性都为1，则它们将等分剩余空间（如果有的话）。如果一个项目的flex-grow属性为2，其他项目都为1，则前者占据的剩余空间将比其他项多一倍。

  - flex-shrink: 1;//默认1 如果所有项目的flex-shrink属性都为1，当空间不足时，都将等比例缩小。如果一个项目的flex-shrink属性为0，其他项目都为1，则空间不足时，前者不缩小。

    负值对该属性无效。 

  - flex-basis: 0%;//默认1 如果所有项目的flex-shrink属性都为1，当空间不足时，都将等比例缩小。如果一个项目的flex-shrink属性为0，其他项目都为1，则空间不足时，前者不缩小。

    负值对该属性无效

- flex: 2

  - flex-grow: 2;
  - flex-shrink: 1;
  - flex-basis: 0%;

- flex: n

  - flex-grow: n;
  - flex-shrink: 1;
  - flex-basis: 0%;

> 当然它也可以不止一项，[扩展](http://t.zoukankan.com/liujunhang-p-14096006.html)



比如下面这个例子，效果就是按钮里面文字水平且垂直居中。

```html
      <div
        style="
          display: flex;
          background-color: #009882;
          position: relative;
          bottom: -8.6667vw;
          align-items: center;
          height: 9.6vw;
          border-radius: 6.6667vw;
        "
        @click="next"
      >
        <div style="flex: 1; text-align: center; color: #ffffff">
          {{ button }}
        </div>
      </div>
```

