![](/assets/http_process.png)潜在性能优化点

* dns是否可以通过缓存减少查询时间
* 网络请求的过程走最近的网络环境
* 相同的静态资源是否可以缓存
* 能否减少请求http请求大小
* 减少http请求
* 服务端渲染

# 资源的合并和压缩

* 减少http请求数量
* 减少请求资源的大小

html压缩： 去除html中无意义的字符

css压缩

* 无效代码删除
* css语义合并

js压缩与混乱

* 无效字符的删除
* 剔除注释
* 代码语义的缩减和优化
* 代码保护

文件合并存在的问题

* 首屏渲染问题
* 缓存失效问题

做法

* 公共库合并： 公共库一般不会改动，业务代码经常改动，分别打包
* 不同页面的合并：不同页面js单独打包

# 重绘与回流

css如果频繁触发重绘和回流，会导致UI频繁渲染，最终导致JS变慢

回流

* 当render tree中的一部分或全部因为元素的规模尺寸，布局，隐藏等改变而需要重新构建。这就称为回流\(reflow\)
* 当页面布局和几何属性改变时就需要回流

重绘

* 当render tree中的一些元素需要更新属性，而这些属性只是影响元素的外观，风格，而不会影响布局的，比如Background-color。则称为重绘。

尽量选用不触发重绘和回流的方案

如何避免重绘和回流

将频繁重绘回流的DOM元素单独作为一个独立图层，那么这个DOM元素的重绘和回流的影响只会在这个图层中。

Chrome创建图层的条件

1. 3D或透视变换CSS属性\(perspective transform\)
2. 使用加速视频解码的&lt;video&gt;节点
3. 拥有3D（WebGL）上下文或加速的2D上下文的&lt;canvas&gt;节点
4. 混合插件（如Flash）
5. 对自己的opacity做CSS动画或使用一个动画webkit变化的元素
6. 拥有加速CSS过滤器的元素
7. 元素有一个包含复合层的后代节点（一个元素拥有一个子元素，该子元素在自己的层里）
8. 元素有一个z-index较低且包含一个复合层的兄弟元素（该元素在复合层上面渲染）

**创建一个图层**

法1

```css
transform: translateZ(0)
```

法2

```css
will-change: transform
```

通过Chrome layers和Performance工具来调试和测试性能

实战优化点

重绘对应performance中的paint

回流对应performance中的layout

1. 用translate替代top改变
   1. top会触发重绘和回流
   2. translate只会触发重绘
2. 用opacity替代visibility
   1. visibility只会触发重绘不会触发回流
   2. opacity不触发重绘也不触发回流\(在当前图层无兄弟节点的情况下\)
3. 不要一条一条地修改 DOM 的样式，预先定义好 class，然后修改 DOM 的 className
   1. 为了保证修改后的样式优先级大于原优先级，例子`#rect{} #rect.active{}`
4. 把 DOM 离线后修改，比如：先把 DOM 给 display:none \(有一次 Reflow\)，然后你修改100次，然后再把它显示出来
5. 不要把 DOM 结点的属性值放在一个循环里当成循环里的变量，
   1. 如offsetHeight, offsetWidth, 获取offsetHeight和offsetWidth时，为了获取真实的数据，浏览器会刷新回流的缓冲区域
6. 不要使用 table 布局，可能很小的一个小改动会造成整个 table 的重新布局
7. 动画实现的速度的选择
8. 对于动画新建图层
9. 启用 GPU 硬件加速

用translate替代top改变

```html
<!DOCTYPE html>
<html>
  <head>
    <style media="screen">
      #rect {
        top: 0;
        width:100px;
        height: 100px;
        background: blue;
      }
    </style>
  </head>
  <body>
    <div id="rect"></div>
    <script type="text/javascript">
      setTimeout (() => {
        document.getElementById('rect').style.top = '100px';
      }, 2000)
    </script>
  </body>
</html>
```

触发回流的表现为多了一段layout时间![](/assets/reflow.png)

修改为

```html
<!DOCTYPE html>
<html>
  <head>
    <style media="screen">
      #rect {
        transform: translateY(0);
        width:100px;
        height: 100px;
        background: blue;
      }
    </style>
  </head>
  <body>
    <div id="rect"></div>
    <script type="text/javascript">
      setTimeout (() => {
        document.getElementById('rect').style.transform = 'translateY(100px)';
      }, 2000)
    </script>
  </body>
</html>
```

![](/assets/no reflow result.png)

用opacity代替visibility

```html
<!DOCTYPE html>
<html>
  <head>
    <style media="screen">
      #rect {
        width:100px;
        height: 100px;
        background: blue;
      }
    </style>
  </head>
  <body>
    <div id="rect"></div>
    <script type="text/javascript">
      setTimeout (() => {
        document.getElementById('rect').style.visibility = 'hidden';
      }, 2000)
    </script>
  </body>
</html>
```

修改为

需要为rect添加单独图层，确保该图层只有rect。否则opacity会导致整个重绘并回流整个图层

```html
<!DOCTYPE html>
<html>
  <head>
    <style media="screen">
      #rect {
        width:100px;
        height: 100px;
        background: blue;
        opacity: 1;
        transform: translateZ(0);
      }
    </style>
  </head>
  <body>
    <div id="rect"></div>
    <script type="text/javascript">
      setTimeout (() => {
        document.getElementById('rect').style.opacity = '0';
      }, 2000)
    </script>
  </body>
</html>
```

# 懒加载

* 减少无效资源的加载
* 并发加载的资源过多会阻塞js的加载，影响网站的正常使用
  * 浏览器并发的线程是有限的

原理：

在浏览器进入可视区域之前，img的src指向占位图片，只有在进入可视区域之后，替换src为真正的图片位置

做法

监听scroll事件，在scroll事件的回调中，去判断我们的懒加载图片是否进入可视区域

判断图片是否进入可视区域： 图片的上边缘是否小于设备的Height

注意：在图片加载之前，需要占位，即设置好原本的高度

```html
<img src="" class="image-item" lazyload="true" data-original="http://image.png">
```

```js
var viewHeight = document.documentElement.clientHeight

function lazyload() {
    var eles = document.querySelectorAll('img[data-original][lazyload]')
    Array.prototype.forEach.call(eles, function(item, index) {
        var ret
        if (item.dataset.original === '')
            return 
        rect = item.getBoundingClientRect()

        if (rect.bottom >= 0 && rect.top < viewHeight) {
            !function() {
                var img = new Image()
                img.src = item.dataset.original
                img.onload = function () {
                    item.src = img.src
                }
                item.removeAttribute('data-original')
                item.removeAttribute('lazyload')
            }()
        }
    })
}

lazyload()


document.addEventListener('scroll', lazyload)
```

预加载（Preload）

preload.js

图片等静态资源在使用之前的提前请求

* 资源使用到时能从缓存中加载，提升用户体验
* 页面展示的依赖关系维护

![](/assets/preload.png)

原理

1. &lt;img&gt;标签
2. js中new一个Image\(\)对象
3. xmlhttprequest



