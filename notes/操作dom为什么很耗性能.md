### 前端页面卡顿、也许是DOM操作惹的祸？
界面上UI的更改都是通过DOM操作实现的，并不是通过传统的刷新页面实现 的。尽管DOM提供了丰富接口供外部调用，但DOM操作的代价很高，页面前端代码的性能瓶颈也大多集中在DOM操作上，所以前端性能优化的一个主要的关注 点就是DOM操作的优化。**DOM操作优化的总原则是尽量减少DOM操作**。

- 先来看看DOM操作为什么会影响性能

在浏览器中，DOM的实现和ECMAScript的实现是分离的。比如 在IE中，ECMAScrit的实现在jscript.dll中，而DOM的实现在mshtml.dll中；在Chrome中使用WebKit中的 WebCore处理DOM和渲染，但ECMAScript是在V8引擎中实现的，其他浏览器的情况类似。所以通过JavaScript代码调用DOM接 口，相当于两个独立模块的交互。相比较在同一模块中的调用，这种跨模块的调用其性能损耗是很高的。但**DOM操作对性能影响最大其实还是因为它导致了浏览器 的重绘（repaint）和回流（reflow）\***。

### 浏览器的渲染原理

从下载文档到渲染页面的过程中，浏览器会通过解析HTML文档来构建DOM树，解析CSS产生CSS规则树。JavaScript代码在解析过程中， 可能会修改生成的DOM树和CSS规则树。之后根据DOM树和CSS规则树构建渲染树，在这个过程中CSS会根据选择器匹配HTML元素。渲染树包括了每 个元素的大小、边距等样式属性，渲染树中不包含隐藏元素及head元素等不可见元素。最后浏览器根据元素的坐标和大小来计算每个元素的位置，并绘制这些元 素到页面上。重绘指的是页面的某些部分要重新绘制，比如颜色或背景色的修改，元素的位置和尺寸并没用改变；回流则是元素的位置或尺寸发生了改变，浏览器需 要重新计算渲染树，导致渲染树的一部分或全部发生变化。渲染树重新建立后，浏览器会重新绘制页面上受影响的元素。回流的代价比重绘的代价高很多，重绘会影 响部分的元素，而回流则有可能影响全部的元素。**如下的这些DOM操作会导致重绘或回流**：

 - 增加、删除和修改可见DOM元素

 - 页面初始化的渲染

 - 移动DOM元素

 - 修改CSS样式，改变DOM元素的尺寸

 - DOM元素内容改变，使得尺寸被撑大

 - 浏览器窗口尺寸改变

 - 浏览器窗口滚动

 1. 合并多次的DOM操作为单次的DOM操作

 最常见频繁进行DOM操作的是频繁修改DOM元素的样式，代码类似如下：
```javascript
element.style.borderColor = '#f00';
element.style.borderStyle = 'solid';
element.style.borderWidth = '1px';
```
这种编码方式会因为频繁更改DOM元素的样式，触发页面多次的回流或重绘，上面介绍过，现代浏览器针对这种情况有性能的优化，它会合并DOM操作，但并不是所有的浏览器都存在这样的优化。推荐的方式是把DOM操作尽量合并，如上的代码可以优化为：
```javascript
// 优化方案1
element.style.cssText += 'border: 1px solid #f00;';
// 优化方案2
element.className += 'empty';
```
示例的代码有两种优化的方案，都做到了把多次的样式设置合并为一次设置。方案2比方案1稍微有一些性能上的损耗，因为它需要查询`CSS`类。但方案2的维护性最好。很多时候，如果性能问题并不突出，选择编码方案时需要优先考虑的是代码的维护性。

类似的操作还有通过 `innerHTML` 接口修改`DOM`元素的内容。不要直接通过此接口来拼接 `HTML` 代码，而是以字符串方式拼接好代码后，一次性赋值给`DOM`元素的 `innerHTML` 接口。

2. 把DOM元素离线或隐藏后修改
把DOM元素从页面流中脱离或隐藏，这样处理后，只会在DOM元素脱离和添加时，或者是隐藏和显示时才会造成页面的重绘或回流，对脱离了页面布局流的DOM元素操作就不会导致页面的性能问题。这种方式适合那些需要大批量修改DOM元素的情况。具体的方式主要有三种：

（1）使用文档片段
文档片段是一个轻量级的document对象，并不会和特定的页面关联。通过在文档片段上进行DOM操作，可以降低DOM操作对页面性能的影响，这 种方式是创建一个文档片段，并在此片段上进行必要的DOM操作，操作完成后将它附加在页面中。对页面性能的影响只存在于最后把文档片段附加到页面的这一步 操作上。代码类似如下：

```javascript
var fragment = document.createDocumentFragment();
// 一些基于fragment的大量DOM操作
...
document.getElementById('myElement').appendChild(fragment);
```

（2）通过设置DOM元素的display样式为none来隐藏元素
这种方式是通过隐藏页面的DOM元素，达到在页面中移除元素的效果，经过大量的DOM操作后恢复元素原来的display样式。对于这类会引起页面重绘或回流的操作，就只有隐藏和显示DOM元素这两个步骤了。代码类似如下：

```javascript
var myElement = document.getElementById('myElement');
myElement.style.display = 'none';
// 一些基于myElement的大量DOM操作...
myElement.style.display = 'block';
```
（3）克隆DOM元素到内存中
这种方式是把页面上的DOM元素克隆一份到内存中，然后再在内存中操作克隆的元素，操作完成后使用此克隆元素替换页面中原来的DOM元素。这样一来，影响性能的操作就只是最后替换元素的这一步操作了，在内存中操作克隆元素不会引起页面上的性能损耗。代码类似如下：

```javascript
var old = document.getElementById('myElement');
var clone = old.cloneNode(true);
// 一些基于clone的大量DOM操作
...
old.parentNode.replaceChild(clone, old);
```

在现代的浏览器中，因为有了DOM操作的优化，所以应用如上的方式后可能并不能明显感受到性能的改善。但是在仍然占有市场的一些旧浏览器中，应用以上这三种编码方式则可以大幅提高页面渲染性能。

3. 设置具有动画效果的DOM元素的position属性为fixed或absolute
把页面中具有动画效果的元素设置为绝对定位，使得元素脱离页面布局流，从而避免了页面频繁的回流，只涉及动画元素自身的回流了。这种做法可以提高动 画效果的展示性能。如果把动画元素设置为绝对定位并不符合设计的要求，则可以在动画开始时将其设置为绝对定位，等动画结束后恢复原始的定位设置。在很多的 网站中，页面的顶部会有大幅的广告展示，一般会动画展开和折叠显示。如果不做性能的优化，这个效果的性能损耗是很明显的。使用这里提到的优化方案，则可以 提高性能。

4. 谨慎取得DOM元素的布局信息
前面讨论过，获取DOM的布局信息会有性能的损耗，所以如果存在重复调用，最佳的做法是尽量把这些值缓存在局部变量中。考虑如下的一个示例：

```javascript
for (var i=0; i < len; i++) {
    myElements[i].style.top = targetElement.offsetTop + i*5 + 'px';
}
```

如上的代码中，会在一个循环中反复取得一个元素的offsetTop值，事实上，在此代码中该元素的offsetTop值并不会变更，所以会存在不必要的性能损耗。优化的方案是在循环外部取得元素的offsetTop值，相比较之前的方案，此方案只是调用了一遍元素的offsetTop值。更改后的代码如下：

```javascript
var targetTop = targetElement.offsetTop;
for (var i=0; i < len; i++) {
    myElements[i].style.top = targetTop+ i*5 + 'px';
}
```

另外，因为取得DOM元素的布局信息会强制浏览器刷新渲染树，并且可能会导致页面的重绘或回流，所以在有大批量DOM操作时，应避免获取DOM元素 的布局信息，使得浏览器针对大批量DOM操作的优化不被破坏。如果需要这些布局信息，最好是在DOM操作之前就取得。考虑如下一个示例：

```javascript
var newWidth = div1.offsetWidth + 10;
div1.style.width = newWidth + 'px';
var newHeight = myElement.offsetHeight + 10; // 强制页面回流
myElement.style.height = newHeight + 'px'; // 又会回流一次
```

根据上面的介绍，代码在遇到取得DOM元素的信息时会触发页面重新计算渲染树，所以如上的代码会导致页面回流两次，如果把取得DOM元素的布局信息提前，因为浏览器会优化连续的DOM操作，所以实际上只会有一次的页面回流出现，优化后的代码如下：

```javascript
var newWidth = div1.offsetWidth + 10;
var newHeight = myElement.offsetHeight + 10;

div1.style.width = newWidth + 'px';
myElement.style.height = newHeight + 'px';
```

5. 使用事件托管方式绑定事件
在DOM元素上绑定事件会影响页面的性能，一方面，绑定事件本身会占用处理时间，另一方面，浏览器保存事件绑定，所以绑定事件也会占用内存。页面中 元素绑定的事件越多，占用的处理时间和内存就越大，性能也就相对越差，所以在页面中绑定的事件越少越好。一个优雅的手段是使用事件托管方式，即利用事件冒 泡机制，只在父元素上绑定事件处理，用于处理所有子元素的事件，在事件处理函数中根据传入的参数判断事件源元素，针对不同的源元素做不同的处理。这样就不 需要给每个子元素都绑定事件了，管理的事件绑定数量变少了，自然性能也就提高了。这种方式也有很大的灵活性，可以很方便地添加或删除子元素，不需要考虑因 元素移除或改动而需要修改事件绑定。示例代码如下：

```javascript
// 获取父节点，并添加一个click事件
document.getElementById('list').addEventListener("click",function(e) { // 检查事件源元素 if(e.target && e.target.nodeName.toUpperCase == "LI") { // 针对子元素的处理 ...
    }
});
```

上述代码中，只在父元素上绑定了click事件，当点击子节点时，click事件会冒泡，父节点捕获事件后通过e.target检查事件源元素并做相应地处理。
在JavaScript中，事件绑定方式存在浏览器兼容问题，所以在很多框架中也提供了相似的接口方法用于事件托管。比如在jQuery中可以使用如下方式实现事件的托管（示例代码来自jQuery官方网站）：

```javascript
$( "table" ).on( "click", "td", function() { $( this ).toggleClass( "chosen" );
});
```