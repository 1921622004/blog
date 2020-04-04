## 缘由
自从上次写了这篇[简单易懂的红黑树原理及实现(js)](https://juejin.im/post/5e3a8ebdf265da572f14054d)之后，总觉得光是图片展示还是不太够，最好是有一个动画能够展示出来，然后突然想起了之前后端小伙伴给我看的一个页面实现的数据结构的展示动画（具体地址不记得了），突发奇想，为什么不自己实现一版红黑树的呢？于是，开始搞起了这个。

### 效果
> 请参考上面提到的文章对比观看

- 插入后无需恢复
![](https://user-gold-cdn.xitu.io/2020/3/24/1710cccf15f7fd0b?w=1734&h=835&f=gif&s=23928)

- 插入后需变色，指针上移
![](https://user-gold-cdn.xitu.io/2020/3/24/1710cdc2771d0509?w=2184&h=835&f=gif&s=198531)

- 插入后旋转一次即可
![](https://user-gold-cdn.xitu.io/2020/3/24/1710ce851cd5e81b?w=2243&h=741&f=gif&s=40570)

- 插入后需旋转两次
![](https://user-gold-cdn.xitu.io/2020/3/24/1710ce8bd3bd3406?w=2412&h=893&f=gif&s=54596)

- 删除红色叶子节点
![](https://user-gold-cdn.xitu.io/2020/3/24/1710ceab56c5398c?w=2412&h=893&f=gif&s=38790)

- 删除 1-1
![](https://user-gold-cdn.xitu.io/2020/3/24/1710cecf703def03?w=2027&h=584&f=gif&s=34460)

- 删除 1-2 
![](https://user-gold-cdn.xitu.io/2020/3/24/1710cc9d605004be?w=1938&h=653&f=gif&s=31532)

- 删除 2-1
![](https://user-gold-cdn.xitu.io/2020/3/25/17111fe592b9a3c3?w=1886&h=894&f=gif&s=83036)

- 删除 2-2
![](https://user-gold-cdn.xitu.io/2020/3/24/1710d0eb5ff0e2a7?w=1823&h=499&f=gif&s=23556)

- 删除 2-3
![](https://user-gold-cdn.xitu.io/2020/3/24/1710d20c1bae38ef?w=1912&h=575&f=gif&s=43234)

### 还有些问题
- 上面应该也看到了，在树层数稍多后，整体就会变的非常不美观，暂时也没有想到什么好的解决方法
- 旋转的动画做的不够直接，现在已经有思路改进了，这周末应该会动手开始弄

### 用了什么
- [SpriteJs](https://github.com/spritejs/spritejs) 很符合我的需求

### 最后
附上[GitHub地址](https://github.com/1921622004/canvas-tree) 欢迎大家提出一些在动画上面的建议