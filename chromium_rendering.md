# 漫谈浏览器渲染
# 引子
前端有一道经典的面试题：

Q：讲解一下重排重绘

A：重绘不一定导致重排，重排肯定会导致重绘

以下以Chromium为例子深入解释一下这道题
# 多进程/多线程模型
## 主要进程类型
1. 浏览器进程：浏览器主进程，仅有一个，用于进程、资源调度和控制。
2. 渲染进程：基本每个浏览器标签页都是一个渲染进程，在内存紧张的时候会合并成一个进程
3. GPU进程：用户绘制3D图形和动画绘制
4. 第三方插件进程：浏览器有很多插件，都运行在第三方插件进程里，防止插件进程影响到主进程等其他进程

打开 Chrome 的任务管理器的窗口，如下图:
![image](https://user-images.githubusercontent.com/20478828/116203970-d5caa200-a76e-11eb-98f0-a44551d9a11d.png)
![image](https://user-images.githubusercontent.com/20478828/116204003-e1b66400-a76e-11eb-813e-7acbd7af5bc1.png)

## 渲染进程的主要线程类型
1. UI渲染线程（renderer thread）：渲染进程的主线程
   1. 解析HTML，CSS，构建DOM树和CSSOM树，布局和绘制等
   2. 当HTML解析script标签时，就会解析script里的Javascript脚本程序（Chromium使用的是V8，Safari用的是JavaScriptCore），阻塞html的解析
2. I/O线程
   1. 负责转发渲染进程与浏览器主进程之间的通信消息
   2. 负责网络资源请求加载，比如UI渲染线程解析到link标签、img标签、script标签加载外部资源的时候就会通知到I/O线程转发加载资源消息到浏览器主进程的网络线程
3. 其他线程
   1. Worker线程（web worker、service worker）
   2. 光栅化线程（raster thread）：当一个个layer tree创建完并且绘制顺序确定了之后，就开始光栅化成一个个位图
   3. 复合线程（compositor thread）：合成一张张render layer位图合成一张位图

如下图所示：
![image](https://user-images.githubusercontent.com/20478828/116344499-e5062a00-a818-11eb-9b74-e4bdaacb5cb0.png)

# 渲染过程
以下面的demo为例子讲解
```
// parsing.html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta http-equiv="X-UA-Compatible" content="IE=edge" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Document</title>
  <style>
    .test1 {
      color: pink;
    }
  </style>
  <link href="https://cdn.bootcdn.net/ajax/libs/normalize/8.0.1/normalize.css" rel="stylesheet" />
  <link rel="stylesheet" href="./parsing.css" />
  <style>
    html,
    body {
      margin: 0;
      padding: 0;
    }
  </style>
  <script>
    console.log("test1");
  </script>
</head>

<body>
  <div class="test1">123</div>
  <style>
    .test2 {
      color: purple;
    }
  </style>
  <div class="test2">
    <img src="http://i1.cmail19.com/ei/j/2A/BC7/816/202259/csimport/actionrocketdarkmodelogooutlinev2_0.png"
      alt="cdn" />
  </div>
  <div class="test3">456</div>
  <style>
    .test3 {
      color: green;
    }
  </style>
  <script>
    console.log("test2");
  </script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/fastdom/1.0.10/fastdom.js"></script>
  <div class="test4">789</div>
  <style>
    .test4 {
      color: blue;
    }
  </style>
  <script async="true">
    console.log("test3");
  </script>
  <div class="test5">10</div>
  <style>
    .test5 {
      color: orange;
    }
  </style>
</body>
</html>
```
```
// parsing.css
.test1 {
  background-color: green;
}
```
## 首次渲染过程分析
### Parsing
UI渲染线程拿到HTML文档的时候开始解析html 标签成DOM树。UI渲染线程在执行Parsing的过程也会启动一个Preload Scanner进行扫描正在形成的DOM树上有没有img、link等加载外部资源的标签或者属性，如果发现就会通知I/O线程转发加载外部资源的请求给浏览器主进程的网络线程进行资源加载。
#### Parsing html文档里的0-21行文本
如果没有script标签等阻塞parsing过程的，那么这个Parsing html的过程就会覆盖整个html
![image](https://user-images.githubusercontent.com/20478828/116345173-395dd980-a81a-11eb-9564-c3ceff6c1934.png)

#### Parsing过程中查找外部资源
Preload Scanner会发现整个html文档里的外部资源并且发出预请求，不只是发现0~21行html文档里的网络资源
下图是Parsing过程中发现有link标签引入了parsing.css，于是开始请求这个外部css
![image](https://user-images.githubusercontent.com/20478828/116345330-80e46580-a81a-11eb-8693-1514f1e411e5.png)

### JavaScript
UI渲染线程在执行Parsing的过程一旦解析遇到script等包含JavaScript代码的标签或者属性，就会立马停止Parsing，去加载JavaScript代码，调用V8去解析、执行JavaScript代码。因为JavaScript代码会有DOM API操作会改变DOM树，比如appendChild、removeChild等涉及到html元素布局的修改，或者是dom.style = "color: red"之类的操作html对应的css样式。

Parsing与JavaScript过程遇到script标签不一定非要加载完script标签再执行script里的JavaScript代码的，script标签的defer和async属性就是改变Parsing过程中遇到script标签的行为。
#### 正常执行：没有async和defer属性
JavaScript下载和执行会阻塞parsing的过程

上面的html代码<script src="xxx.js"></script>就是这个过程
![image](https://user-images.githubusercontent.com/20478828/116345448-b5582180-a81a-11eb-8022-3d8d6ad9af74.png)
1. Parse html 0~21行
![image](https://user-images.githubusercontent.com/20478828/116345490-cef96900-a81a-11eb-8f4f-c59af7f98252.png)

2. Parse html 22~42行
![image](https://user-images.githubusercontent.com/20478828/116345496-d587e080-a81a-11eb-89e1-cb587f9f4e72.png)

3. Parse html 43~-1行
![image](https://user-images.githubusercontent.com/20478828/116345511-dde01b80-a81a-11eb-8b10-b71fff59e165.png)

4. 跳过的打断Parsing的过程
45行的script加载fastdom.js的过程被跳过是由于在Parsing 0~21行的时候已经去请求fastdom的资源，并且在parsing 43~-1行执行之前请求就回来并且执行完了fastdom的初始化代码，于是45行本来要打断parsing过程却直接跳过了
   - 发送请求
![image](https://user-images.githubusercontent.com/20478828/116345757-5cd55400-a81b-11eb-8fed-7e60f959ee7d.png)

   - 收到数据
![image](https://user-images.githubusercontent.com/20478828/116345786-665ebc00-a81b-11eb-8a37-e030faa38fd4.png)

   - 执行javasript代码
![image](https://user-images.githubusercontent.com/20478828/116345949-be95be00-a81b-11eb-90d7-67d942eac416.png)

#### 有async属性
JavaScript异步下载不会阻塞parsing的过程，但是下载完之后会立即执行代码阻塞Parsing的过程

修改一下示例html，将加载fastdom的script标签加上async属性: <script async src="xxx.js"></script>
重新用performance分析可以发现Parsing的过程与下载script的过程也是并行的，但是Parsing过程仍然受JavaScript执行时机影响

![image](https://user-images.githubusercontent.com/20478828/116345999-db31f600-a81b-11eb-97de-fc2480c67f55.png)

#### 有defer属性
JavaScript异步下载不会阻塞parsing的过程，下载完之后不会立马执行，代码执行放在DOMContentLoaded之后执行

修改一下示例html，将加载fastdom的script标签加上defer属性: <script defer src="xxx.js"></script>
重新用performance分析可以发现Parsing的过程与下载script的过程也是并行的，Parsing过程可以比下载script的结束和执行代码前

![image](https://user-images.githubusercontent.com/20478828/116346033-f13fb680-a81b-11eb-9f52-e089f8374a1b.png)

### Style
HTML DOM树解析完之后，会进行样式计算：在这个过程中会根据匹配选择器（例如 .header 或 .footer > .confirm-button）计算出哪些元素应用哪些 CSS 规则，建立一个CSSOM树。如果修改了一个元素的html结构、css属性值，都会触发样式的重新计算。如果css选择器写太复杂，这个过程时间就会耗时长一些。
#### 构建基本的CSSOM
![image](https://user-images.githubusercontent.com/20478828/116346091-19c7b080-a81c-11eb-8738-dd726fcd5cd5.png)

#### 重新计算CSSOM树
![image](https://user-images.githubusercontent.com/20478828/116346108-26e49f80-a81c-11eb-9109-db410a3bc4e7.png)

### Layout
当DOM树、CSSOM树都已经建立好了，但是却只是这些对象的自身的描述，还不知道这些对象如果最终要渲染在页面上的真正位置信息。于是就有这个Layout的过程根据DOM树、CSSOM树的节点位置信息计算、整合、建立出一棵Layout树。
#### 建立layout树
![image](https://user-images.githubusercontent.com/20478828/116346151-3c59c980-a81c-11eb-8523-34daf582ee52.png)

#### 更新layer树
保证绘制顺序
![image](https://user-images.githubusercontent.com/20478828/116346169-467bc800-a81c-11eb-92b7-66cfc24119bf.png)

### Paint
有了Layout树，根据Layout树开始绘制位图，但是由于CSS有z-index、float等改变文档流等属性会造成屏幕的垂直方向上的元素重叠，那就需要先按照垂直方向上的顺序（Stacking Order）进行绘制元素，如果垂直方向上同一层的元素就按照html的顺序绘制元素。绘制的过程也叫光栅化（Raster），光栅化之后的结果是一张render layer位图。
#### 查看Paint绘制时机
![image](https://user-images.githubusercontent.com/20478828/116346281-85118280-a81c-11eb-843b-a57b2c93628b.png)

#### 查看Paint绘制过程
勾选左上角的paint instrumentation选项
![image](https://user-images.githubusercontent.com/20478828/116346274-80e56500-a81c-11eb-8b2e-76b9649bceb8.png)

### Composite
一般网页应用当然不会只有一张render layer位图，如果HTML元素使用scaleZ、will-change等属性会专门新建一个render layer去给这元素进行渲染，所以提升成一个独立的render layer之后自然Style、Layout、Paint等过程的消耗自然会小很多。但是render layer增加了之后也需要合并这些render layer的位图成一张位图最终展示在屏幕上，因此需要进行Composite（复合）render layer。
![image](https://user-images.githubusercontent.com/20478828/116346386-ba1dd500-a81c-11eb-96a3-d8c4109fb89b.png)

#### 单独提升渲染层
给某一个元素加上transform: scaleZ(0)的css属性
![image](https://user-images.githubusercontent.com/20478828/116346416-cefa6880-a81c-11eb-9262-a96d28943ff2.png)
![image](https://user-images.githubusercontent.com/20478828/116346430-d588e000-a81c-11eb-96f9-b752192de630.png)


## 渲染更新过程分析
这个网站可以看到哪些css属性会触发哪一个过程：https://csstriggers.com/
### Relayout
![image](https://user-images.githubusercontent.com/20478828/113597062-be1b6480-966d-11eb-8c01-d52cd3fb2377.png)

用户触发事件，开始执行JavaScript操作DOM，改变元素的几何属性（比如width、margin-left），就会触发Style过程重新计算样式规则，再触发Layout过程生成新的Layout树，再触发Paint过程绘制位图，最后再触发Composite过程将多张位图合成一张位图。
### Repaint
![image](https://user-images.githubusercontent.com/20478828/113597085-c83d6300-966d-11eb-8aab-2ace49846f85.png)

用户触发事件，开始执行JavaScript改变DOM，改变元素的外观属性（比如color、background），除了不会触发Layout过程，其余过程与Relayout过程一样。
### Composite
![image](https://user-images.githubusercontent.com/20478828/113597211-f458e400-966d-11eb-8d71-08842b78cef1.png)

用户触发事件，开始执行JavaScript改变DOM，改变元素会提升渲染层的属性（比如scaleZ、will-change），那么此时就会在独立渲染层执行Layout、Paint，性能消耗忽略不计。而主文档流的渲染层就会跳过Layout、Paint的过程，直接与独立渲染层进行合成。
# 如何提高渲染性能
## 减少长时间的JavaScript执行
不仅上面提到的首次渲染过程中的Parsing会被JavaScript执行打断，渲染更新过程中的Style和Layout过程也是会被JavaScript的执行打断。如果长时间执行JavaScript，那么就会阻止Parsing、Style、Layout的过程，于是用户就会在这段时间内发现页面样式、布局没有改变且操作无法响应。
## 减少样式选择器的数量和复杂度
Style过程取决于css选择器复杂度，如果css选择器过多以及过于复杂，那么就会影响计算CSSOM的速度。
## 避免大型、复杂的布局
Parsing过程速度、Layout过程速度与HTML标签数量、嵌套层级、复杂布局呈正相关。如果频繁地触发Relayout消耗比较大，并且Relayout基本作用于整个HTML文档的。
## 适当使用提升元素渲染层
做动画的时候，需要适当地使用will-change提升动画元素的渲染层，脱离主文档流的渲染层，避免频繁改变主文档流的布局，造成Relayout、Repaint的性能浪费。
# 参考
[Rendering Performance](https://developers.google.com/web/fundamentals/performance/rendering)
   
[Taobao FED | 淘系前端团队](https://fed.taobao.org/blog/taofed/do71ct/performance-composite/)

[The stacking context - CSS: Cascading Style Sheets | MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Positioning/Understanding_z_index/The_stacking_context)
