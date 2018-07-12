# UI相关问题

## UITableView相关

### 重用机制

```
UITableView有两个属性是 <NSMutableArray *>visiableCells 和 <NSMutableDictionary *>reusableTableCells
visiableCell 内保存的是当前显示的cell
reusableTableCells 内保存的是可重用的cell
```

1. 当tableView开始显示的时候，创建出大于屏幕范围的多个cell，保存到visiableCells中；
2. 拖动tableview，当cell完全移出屏幕外，此时cell从visiableCells中移出，被回收，放入到reusableTableCells中；
3. 继续拖动tableview，此时需要创建新的cell的时候，就会先从回收池中找有无当前identifier的cell，即从reusableTableCells取出，加到visiableCells中

```
以上就是整个创建回收cell的过程
使用id避免重用出错
```

---

**如何对UITableView的滚动加载进行优化，防止卡顿**

*主要在于以下两个方面：*

```
减少cellForRowAtIndexPath代理中的计算量（cell的内容计算）
```

1. 提前计算出cell中药显示的内容，即我们常用的viewObject
2. 在接口设计的时候就可以提前考虑到列表上图片显示的尺寸，避免过大的图片引起cell的图片渲染效率
3. 图片异步加载，可参考SDImage
4. 尽量使用手动drawing视图来提升流畅性，然后覆盖drawRect

   ```
    每一个UIView 都有一个CALayer实例的图层属性，UIView的职责就是管理这些图层，每一个UIView都有一个CALayer提供内容的绘制和显示，并且UIView的尺寸样式都是由内部的Layer提供。
    相同的是这两者都是梳妆层级结构。layer内部维持着三分layer tree，分别是presentLayer tree(动画树)、model layer(模型树)、render tree(渲染树);
    两者比较大的区别就是，view可以接受处理事件，但是layer不可以，
    至于为什么要这么做，就是要做到图层和用户交互分离吧
   ```

```
减少heightForRowAtIndexPath代理中的计算量（cell的高度计算）
```

1. tableView每次update都会重新获取每一个cell,如果高度固定，则去掉代理方法，可直接设置tableView的rowHeight属性
2. 如果高度不固定，可以参考viewObject/viewModel的高度量，将高度提前计算好缓存起来

---

- 使用ReuseIdentifier。它能帮助你提升性能。
- 尝试减少cell预加载过程中的工作，尤其是从文件IO或网络IO加载图片的时间和效率。这样可以在最短的时间内显示图片。
- 如果你的应用有很多的subviews或复杂的结构，考虑自己用代码来绘制。这样可以让GPU来加速整个过程。

`WARNING: 谨慎使用drawRect，避免过多优化(导致维护和功能膨胀(在应用中添加过多的功能))`

---

### 数据源同步

***如何在tableView解决多线程情况下，数据的处理***

1. 并发访问，数据拷贝

   ```
   比如在列表接口中返回了多个数据，每条数据中的图片都是高清图片，如果采用常规做法，就是在cell显示的时候，再去请求图片url，这样就算我们有图片缓存机制，但是在快速滑动的时候，还是会看到一片片的空白cell，那我们就考虑到当数据拿回来了以后，就那一批图片url去并发请求图片数据，然后做数据拷贝缓存下来，这样再要显示图片的时候，就可以从我们本地的缓存池里面拿出图片数据直接显示。
   ```
2. 串行访问

   ```
   操作数据都放在子线程里面，建造队列来处理，处理完毕了以后，在主线程去刷新UI
   ```

---

##事件传递&视图响应

```
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event;
//返回最终响应的事件
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event
//判断点击位置是否在当前范围内
```

1. 首先判断当前视图 !hidden &$ userInteractionEnable && alpha > 0.01 条件通过的时候，到下一步. 否则返回nil，找不到当前视图
2. 通过 pointInside 判断点击的点是否在当前范围内，为YES直接下一步. 不在则直接返回nil。
3. 倒序遍历所有子视图，同时调用 hitTest 方法，如果某一个子视图返回了对应的响应视图，这个子视图会直接作为最终的响应视图给响应方，如果为 nil 则继续遍历下一个子视图。如果全部遍历结束都返回nil，那会返回当前点击位置在当前的视图范围内的视图作为最终响应视图



## 图像显示原理

### 图形的堆栈

1. 软件堆栈：我们在显示一个组件的过程中，在每一个像素都会有红、绿和蓝三种像素点组成

2. 软件组成

   - Core Graphics

   - Core Animation

     - OpenGL（ES）

       - `GPU Driver：（1.GPU Display；2.GPU主要功能是对于图像处理显示能力，是为图形并发给指定的硬件）`

       - `是直接与GPU进行交流代码块，驱动GPU进行相关的处理工作`

   - Core Image

   - 具体的显示过程：

     - Display `<->`  GPU `<->` GPU Driver`<->`OpenGL(ES)`<->`Core Animation(Core Graphics/Core Image)`<->`APP

3. 硬件参与者

   1. 我们在显示一张图片的过程中GPU会把每张图的纹理进行合成，这个过程中纹理保存在VRAM中

   2. GPU处理合成图片的速度很快，但是单位时间处理内容是有限

   3. 把数据上传到GPU(VRAM)，我们要解决数据从CPU（RAM）上传到GPU（VRAM）

4. 合成

   1. 在iOS展示的内容都是合成最终的结果，可以看作是不同的纹理进行合成结果

   2. 每一个纹理都是一个长方形RGBA值，其中包括红、绿、蓝像素值，透明度值

   3. 在图片合成过程Core Animation的layer就相当于是一个纹理

5. 不透明和透明

   1. 不透明情况下，目标像素就是源纹理，GPU不需要进行相关计算就可以服务

   2. 我们在每一个view的CALayer之中有个属性opaque设置为true，则就是不透明的

6. 像素对齐和不重合在一起

   1. 在像素对齐的过程中GPU进行相关的计算会比较方便，在没有对齐的情况下进行相关计算就比较复杂

   2. 当纹理在放大和缩小的时候，像素很难对齐；纹理的起点不是在像素边界的时候，就没有相应对齐

7. 蒙版

8. 离屏渲染

   1. 离屏渲染就是我们从GPU到CPU再到GPU的两次切换，屏幕外的渲染会合并/渲染图层树到一个新的缓冲区，然后缓冲区被显示在屏幕上面；使用场景：当一个图层室友多个图层混合而成，在动画过程中GPU会每次计算所有的该图层进行混合

   2. 当我们采用离屏运算过程，GPU会产生一个纹理缓存，在下一次调用的过程直接使用这个缓存的纹理座位图像使用；使用场景：我们使用的图片是固定的，不会经常性的进行修改

   3. 创建的缓冲去是屏幕外缓冲区，rasterized layer的空间有限，大小应该是屏幕大小两倍空间

   `WARNING:如果我们通过layer方式进行屏幕外渲染就要小心，如果设置了shouldRasterize为YES,也要设置rasterizationScale为contentsScale`

9. Core Animation OpenGL ES

   1. 可以实现动画

   2. 可以实现图像绘制

      1. 是对于OpenGL的封装，实现我们可以直接简单调用OpenGL相关功能

      2. 在core Animation中的layer中有相关的子layer，所以后面饿到一个图层树

      3. 主要工作是判断那些图层是不是需要被重新绘制

10. Core Graphics

    1. 是一个很棒的绘图框架，在绘图过程中我们可以使用它的接口，绘制多种多样的图像内容

    2. UIKit 维护一个上下间对战，它的方法总是会吧内容绘制到上下文最顶层



### GPU加速下的图像处理

就是并行做浮点运算，图像的处理和渲染就是把将要显示在窗口的像素做浮点运算

着色器：

1.  顶点着色器

2. 片段着色器





### CPU工作原理

`中央处理器（central processing unit）主要是解释计算机中的指令，处理计算机软件中的数据。CPU、内存、I/O是计算机三大核心组件。工作过程一般可以分为：提码、解码、执行和调回四个阶段`



### GPU工作原理

`图形处理器（Graphics Processing unit）主要是进行绘图运算的微型处理器，对图像进行处理和相关显示绘制工作。后面随着OpenGL APL 和DirectX功能的出现，图形处理器可以增加可变成着色的能力。`





### App绘制像素过程分析

![GPUæ˜¾ç¤º.png](http://upload-images.jianshu.io/upload_images/1877765-de4eaec88c23be92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



        

### App 绘制像素硬件分析    ![GPUç¼“å­˜.png](http://upload-images.jianshu.io/upload_images/1877765-618b4dfde6e991a7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
