## 写在前面
在家憋的已经快要疯了，正好好久不发文章了，索性把这些天对红黑树的总结梳理一下总结成这篇文章。图挺多的（都是一点一点画的啊...），代码如果看不下去，理解下原理就好了。顺便安利一下YouTube的一个up主（该这么叫？），[Tushar Roy](https://www.youtube.com/user/tusharroy2525)，他的视频相对更简单，更好懂。下面开始正文，不多废话。

## 红黑树的特点
- 根结点是黑色的
- 如果一个节点是红色的，那么他的两个子节点必然是黑色的。（也就是不能有R-R的父子关系， 下文中R-R皆指的是这种关系）
- 从根节点到每个叶子节点的路径上都有相同个数的黑色节点。

## 知识铺垫
### 旋转
树的旋转其实很简单，目的就是改变树的原有结构。左旋和右旋可以当作是一个相反的过程，先看下图：

#### 左旋

![](https://user-gold-cdn.xitu.io/2020/2/5/17014c59ab3295de?w=1434&h=1188&f=png&s=192369)

经过左旋后


![](https://user-gold-cdn.xitu.io/2020/2/5/17014c5f03dfe652?w=1188&h=982&f=jpeg&s=58363)

#### 右旋
![](https://user-gold-cdn.xitu.io/2020/2/5/17014c5f03dfe652?w=1188&h=982&f=jpeg&s=58363)

经过旋转后

![](https://user-gold-cdn.xitu.io/2020/2/5/17014c59ab3295de?w=1434&h=1188&f=png&s=192369)

然后看下代码：

```js
_rotateLeft(node) {
  let rightNode = node.right;
  let rightNodeLeft = rightNode.left;
  node.right = rightNodeLeft;
  if (rightNodeLeft) rightNodeLeft.parent = node;
  rightNode.left = node;
  if (node.parent) {
    if (node.parent.left === node) {
      node.parent.left = rightNode;
    } else {
      node.parent.right = rightNode;
    }
    rightNode.parent = node.parent;
  } else {
    // 如果当前进行旋转的节点是根结点，重新设置。
    this.root = rightNode;
    rightNode.parent = null;
  }
  node.parent = rightNode;
}
```


### 插入
这里暂且不谈与红黑树相关的东西。插入的操作很简单，找到合适的位置，生成一个新的节点即可。

简单点的代码写一下
```js
insert(val) {
  let node = this.root;
  while (node) {
    if (node.val > val) {
      if (!node.left) {
        node.left = new TreeNode(val);
      } else {
        node = node.left;
      }
    } else if (node.val < val) {
      if (!node.right) {
        node.right = new TreeNode(val);
      } else {
        node = node.right;
      }
    } else return;
  }
}

```

### 删除
也先不谈红黑树的特点。树的删除相对插入会复杂一点，主要分两种情况：

1. 如果目标节点缺失左节点或者右节点，直接用他的子节点（或者null）替换他当前的位置
2. 如果不缺失子节点的话，步骤如下：
  1. 找出右子树的最小值（或者左子树的最大值），将目标节点的值设置为这个值。
  2. 既然这个值已经是最大值或者最小值了，那么他肯定是缺失子节点的，重复第一种情况即可。

简单的代码：
```js
_remove(val, node) {
  if (node === null) return null;
  if (node.val > val) {
    node.left = this._remove(val, node.left);
  } else if (node.val < val) {
    node.right = this._remove(val, node.right);
  } else {
    let newVal = this._findMin(node.right); // 选择右子树的最小值
    node.val = newVal;
    node.right = this._remove(newVal, node.right);
  }
}

remove(val) {
  this.root = this._remove(val, this.root);
}
```

### 重头戏
接下来，我们来说一下红黑树在插入删除之后，如果破坏了红黑树的特点，如何去修复。

### 插入后红黑树性质的恢复
我们首先需要要找到插入节点的位置，找到了之后生成新节点，除了第一次插入，每次新插入的节点都是红色的。
```js
let t = this.root;
let p = null;
while (t) {
  p = t;
  if (t.val < val) {
    t = t.right;
  } else if (t.val > val) {
    t = t.left;
  } else return;
}
let node = new RedBlackTreeNode(val);
node.parent = p;
if (p === null) {
  this.root = node;
} else if (p.val < node.val) {
  p.right = node;
} else {
  p.left = node;
}
node.color = RED;
```

接下来开始修复的过程：
- 如果新生成的节点的父节点是黑色的，那么此次插入操作结束，因为这样肯定是不会破坏红黑树性质的。
- 如果新生成的节点的父节点是红色的，那么破坏了红黑树不能有R-R关系的性质，所以需要进行修复

现在我们需要一个指针，这个指针初始将指向当前的新节点，也就是出现问题的节点通过一个while循环不断的从底层向上排，直到到达根结点或者当前的节点的父节点是黑色的，同时在循环的过程中需要根据情况来进行操作：

**如果指针指向的节点的叔节点是红色的，我们需要将当前节点的父节点以及叔节点设置黑色，然后讲爷爷节点设置成红色，然后将问题的指针移动到爷爷节点上**


![](https://user-gold-cdn.xitu.io/2020/2/5/17014c7b9847576f?w=1398&h=1154&f=jpeg&s=75882)

在经过操作后


![](https://user-gold-cdn.xitu.io/2020/2/5/17014c8a2891be10?w=1496&h=996&f=png&s=156399)

**如果指针指向的节点的叔节点是黑色的，这种情况下如果叔叔节点和父节点在同一侧的话，即都在左侧或者右侧，旋转父节点，同时指针移动到父节点，进入下一次while循环；**

![](https://user-gold-cdn.xitu.io/2020/2/5/17014d0c2f20e879?w=1358&h=1168&f=jpeg&s=68929)

操作之后


![](https://user-gold-cdn.xitu.io/2020/2/5/17014d12ae201d0e?w=1356&h=1250&f=jpeg&s=73844)

**如果叔叔节点和父节点不在同一个侧的话，旋转爷爷节点，并且将爷爷节点设置成红色，父节点设置成黑色，指针保持不变，下一次循环条件不符合，结束循环**


![](https://user-gold-cdn.xitu.io/2020/2/5/17014d1a46d5b841?w=1226&h=1206&f=jpeg&s=67238)

然后

![](https://user-gold-cdn.xitu.io/2020/2/5/17014d1dc0709d03?w=1230&h=958&f=jpeg&s=59419)

```js
insertFix(node) {
  while (node.parent && node.parent.color !== BLACK) {
    if (node.parent === node.parent.parent.left) {
      let uncleNode = node.parent.parent.right;
      // 如果叔叔节点的颜色是红色
      if (uncleNode && uncleNode.color === RED) {
        node.parent.color = BLACK;
        uncleNode.color = BLACK;
        node.parent.parent.color = RED;
        node = node.parent.parent;
      } else if (node === node.parent.right) {
        // 如果叔叔节点和父节点不在同一侧
        node = node.parent;
        this._rotateLeft(node);
      } else {
        // 如果叔叔节点和父节点在同一侧
        node.parent.color = BLACK;
        node.parent.parent.color = RED;
        this._rotateRight(node.parent.parent);
      }
    } else {
      // 镜像操作
      let uncleNode = node.parent.parent.left;
      if (uncleNode && uncleNode.color === RED) {
        node.parent.color = BLACK;
        uncleNode.color = BLACK;
        node.parent.parent.color = RED;
        node = node.parent.parent;
      } else if (node === node.parent.left) {
        node = node.parent;
        this._rotateRight(node);
      } else {
        node.parent.color = BLACK;
        node.parent.parent.color = RED;
        this._rotateLeft(node.parent.parent);
      }
    }
  }
  // 在循环结束后，将根结点设置成黑色
  this.root.color = BLACK;
}
```

### 删除后红黑树性质的恢复
删除操作之后的恢复会更复杂，我们仍需要一个指针指向一个节点，通过循环不断将问题向上排，直到到达根结点，或者红色节点，在循环的过程中碰到的情况通过总结后大致可分为两种情况

#### 可直接恢复

1.1 指针指向的节点为黑色，父节点为红色，且兄弟节点为黑色。
> 此时我们需要做的就是将父节点设置为红色，将兄弟节点设置为黑色，然后将指针指向根结点即可。

![](https://user-gold-cdn.xitu.io/2020/2/5/17014d33016948b0?w=1910&h=1136&f=jpeg&s=117617)


![](https://user-gold-cdn.xitu.io/2020/2/5/17014d368ff5caaf?w=1880&h=1098&f=jpeg&s=131084)

1.2 当前节点为父节点的左（右）节点，兄弟节点为黑色，且兄弟节点的右（左）节点为红色：
> 此时我们把兄弟节点设置为父节点的颜色，然后父节点设置为黑色，兄弟节点的右（左）节点设置为黑色，同时对父节点进行左旋，然后指针指向根结点即可。

![](https://user-gold-cdn.xitu.io/2020/2/5/17014d3c4564252b?w=1810&h=1230&f=jpeg&s=107263)

![](https://user-gold-cdn.xitu.io/2020/2/5/17014d3fed8c27e9?w=1952&h=1156&f=jpeg&s=173116)

#### 不可直接恢复
那我们需要做的就是将这种情况变成上述可直接回复的情况，也分以下几种情况：

2.1 兄弟节点为红色（父节点必为黑色）
> 如果当前节点是父节点的左（右）节点，那么将父节点进行左（右）旋，同时兄弟节点设置为黑色，父节点设置为红色，指针保持不变，进入下一轮循环

![](https://user-gold-cdn.xitu.io/2020/2/5/17014d637a83d564?w=1450&h=1130&f=jpeg&s=80419)

![](https://user-gold-cdn.xitu.io/2020/2/5/17014d660e05097d?w=1370&h=1232&f=jpeg&s=81487)

2.2 兄弟节点为黑色，父节点为黑色，且兄弟节点的子节点都为黑色（null也视为黑色）
> 将兄弟节点设置为红色，指针指向父节点，进入下一轮循环。

![](https://user-gold-cdn.xitu.io/2020/2/5/17014d6b672e5a2d?w=1448&h=1158&f=jpeg&s=82872)

![](https://user-gold-cdn.xitu.io/2020/2/5/17014d6db6b9c166?w=1556&h=1260&f=jpeg&s=85400)

2.3 当前节点为父节点的左（右）节点，兄弟节点为黑色，且兄弟节点的右（左）节点为黑色，兄弟节点的左（右）节点为红色：
> 将兄弟节点设置为红色，兄弟节点的左（右）节点设置为黑色，同时将兄弟节点进行右（左）旋转，指针保持不变，进入下一轮循环，也就变成了1.2的情况。

![](https://user-gold-cdn.xitu.io/2020/2/5/17014d756140ddf3?w=1532&h=1148&f=jpeg&s=82198)

![](https://user-gold-cdn.xitu.io/2020/2/5/17014d798be39d82?w=1656&h=1446&f=jpeg&s=95290)


然后上代码
```js
_removeFix(node) {
  let nodeRef = node;
  while (node !== this.root && node.color === BLACK) {
    if (node.parent.left === node) {
      let brotherNode = node.parent.right;
      if (brotherNode.color === RED) { 
        // 2.1
        brotherNode.color = BLACK;
        node.color = BLACK;
        node.parent.color = RED;
        this._rotateLeft(node.parent);
      } else if (
        (!brotherNode.left || brotherNode.left.color === BLACK)
        &&
        (!brotherNode.right || brotherNode.right.color === BLACK)
      ) {
        if (node.parent.color === RED) {
          // 1.1
          brotherNode.color = RED;
          node.parent.color = BLACK;
          node = this.root;
        } else {
          // 2.2
          brotherNode.color = RED;
          node = node.parent;
        }
      } else if (brotherNode.left.color === RED && brotherNode.right.color === BLACK) {
        // 2.3
        this._rotateRight(brotherNode);
        brotherNode.color = RED;
        brotherNode.parent.color = BLACK;
      } else if (brotherNode.right.color === RED) {
        // 1.2
        brotherNode.color = node.parent.color;
        node.parent.color = BLACK;
        brotherNode.right.color = BLACK;
        this._rotateLeft(node.parent);
        node = this.root;
      }
    } else {
      // 以下镜像操作
      let brotherNode = node.parent.left;
      if (brotherNode.color === RED) {
        brotherNode.color = BLACK;
        node.color = BLACK;
        node.parent.color = RED;
        this._rotateRight(node.parent);
      } else if (
        (!brotherNode.left || brotherNode.left.color === BLACK)
        &&
        (!brotherNode.right || brotherNode.right.color === BLACK)
      ) {
        if (node.parent.color === RED) {
          brotherNode.color = RED;
          node.parent.color = BLACK;
          node = this.root;
        } else {
          brotherNode.color = RED;
          node = node.parent;
        }
      } else if (brotherNode.right.color === RED && brotherNode.left.color === BLACK) {
        this._rotateLeft(brotherNode);
        brotherNode.color = RED;
        brotherNode.parent.color = BLACK;
      } else if (brotherNode.left.color === RED) {
        brotherNode.color = node.parent.color;
        node.parent.color = BLACK;
        brotherNode.left.color = BLACK;
        this._rotateRight(node.parent);
        node = this.root;
      }
    }
  }
  node.color = BLACK;
  this._removeNode(nodeRef);
}
```

#### 各位看官辛苦
附上[代码地址](https://github.com/1921622004/leetcode-practice/blob/master/def/RedBlackTree.js)



## 武汉加油，中国加油

参考：
- [算法导论](https://book.douban.com/subject/1885170/)
- [YouTube红黑树视频](https://www.youtube.com/watch?v=CTvfzU_uNKE&list=PLrmLmBdmIlpv_jNDXtJGYTPNQ2L1gdHxu&index=26)