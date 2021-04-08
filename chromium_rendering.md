# 漫谈浏览器渲染
# 引子
前端有一道经典的面试题：

Q：讲解一下重排重绘

A：重绘不一定导致重排，重排肯定会导致重绘，实际

以下以Chromium为例子深入解释一下这道题
# 多进程/多线程模型
## 主要进程类型
1. 浏览器进程：浏览器主进程，仅有一个，用于进程、资源调度和控制。
2. 渲染进程：基本每个浏览器标签页都是一个渲染进程，在内存紧张的时候会合并成一个进程
3. GPU进程：用户绘制3D图形和动画绘制
4. 第三方插件进程：浏览器有很多插件，都运行在第三方插件进程里，防止插件进程影响到主进程等其他进程
## 渲染进程的主要线程类型
1. UI渲染线程（renderer thread）：渲染进程的主线程
   1. 解析HTML，CSS，构建DOM树和CSSOM树，布局和绘制等
   2. 当HTML解析script标签时，就会解析script里的Javascript脚本程序（Chromium使用的是V8，Safari用的是JavaScriptCore），阻塞html、css的解析
2. I/O线程
   1. 负责转发渲染进程与浏览器主进程之间的通信消息
   2. 负责网络资源请求加载，比如UI渲染线程解析到link标签、img标签、script标签加载外部资源的时候就会通知到I/O线程转发加载资源消息到浏览器主进程的网络线程
3. 其他线程
   1. Worker线程（web worker、service worker）
   2. 光栅化线程（raster thread）：当一个个layer tree创建完并且绘制顺序确定了之后，就开始光栅化成一个个位图
   3. 复合线程（compositor thread）：合成一张张render layer位图合成一张位图
# 渲染过程
## 首次渲染过程分析
### Parsing
UI渲染线程拿到HTML文档的时候开始解析html 标签成DOM树。UI渲染线程在执行Parsing的过程也会启动一个Preload Scanner进行扫描正在形成的DOM树上有没有img、link等加载外部资源的标签或者属性，如果发现就会通知I/O线程转发加载外部资源的请求给浏览器主进程的网络线程进行资源加载。
### JavaScript
UI渲染线程在执行Parsing的过程一旦解析遇到script等包含JavaScript代码的标签或者属性，就会立马停止Parsing，去加载JavaScript代码，调用V8去解析、执行JavaScript代码。因为JavaScript代码会有DOM API操作会改变DOM树，比如appendChild、removeChild等涉及到html元素布局的修改，或者是dom.style = "color: red"之类的操作html对应的css样式。
### Style
HTML DOM树解析完之后，会进行样式计算：在这个过程中会根据匹配选择器（例如 .header 或 .footer > .confirm-button）计算出哪些元素应用哪些 CSS 规则，建立一个CSSOM树。如果修改了一个元素的html结构、css属性值，都会触发样式的重新计算。如果css选择器写太复杂，这个过程时间就会耗时长一些。
### Layout
当DOM树、CSSOM树都已经建立好了，但是却只是这些对象的自身的描述，还不知道这些对象如果最终要渲染在页面上的真正位置信息。于是就有这个Layout的过程根据DOM树、CSSOM树的节点位置信息计算、整合、建立出一棵Layout树。
### Paint
有了Layout树，根据Layout树开始绘制位图，但是由于CSS有z-index、float等改变文档流等属性会造成屏幕的垂直方向上的元素重叠，那就需要先按照垂直方向上的顺序（Stacking Order）进行绘制元素，如果垂直方向上同一层的元素就按照html的顺序绘制元素。绘制的过程也叫光栅化（Raster），光栅化之后的结果是一张render layer位图。
### Composite
一般网页应用当然不会只有一张render layer位图，如果HTML元素使用scaleZ、will-change等属性会专门新建一个render layer去给这元素进行渲染，所以提升成一个独立的render layer之后自然Style、Layout、Paint等过程的消耗自然会小很多。但是render layer增加了之后也需要合并这些render layer的位图成一张位图最终展示在屏幕上，因此需要进行Composite（复合）render layer。
## 渲染更新过程分析
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
