# 第5章 go 和 Web

## 5.2 router 请求路由

### 5.2.1 httprouter

较流行的开源go Web框架大多使用httprouter，或是基于httprouter的变种对路由进行支持。前面提到的github的参数式路由在httprouter中都是可以支持的。

### 5.2.2 原理

httprouter和众多衍生router使用的数据结构被称为压缩字典树（Radix Tree）。

*图 5-1*是一个典型的字典树结构：

<img src="https://chai2010.cn/advanced-go-programming-book/images/ch6-02-trie.png" alt="trie tree" style="zoom:50%;" />

(oni，online, ik, out)

*图 5-1 字典树*

字典树常用来进行字符串检索，例如用给定的字符串序列建立字典树。对于目标字符串，只要从根节点开始深度优先搜索，即可判断出该字符串是否曾经出现过，时间复杂度为`O(n)`，n可以认为是目标字符串的长度。为什么要这样做？字符串本身不像数值类型可以进行数值比较，两个字符串对比的时间复杂度取决于字符串长度。如果不用字典树来完成上述功能，要对历史字符串进行排序，再利用二分查找之类的算法去搜索，时间复杂度只高不低。可认为字典树是一种空间换时间的典型做法。

普通的字典树有一个比较明显的缺点，就是每个字母都需要建立一个孩子节点，这样会导致字典树的层数比较深，压缩字典树相对好地平衡了字典树的优点和缺点。是典型的压缩字典树结构：

<img src="https://chai2010.cn/advanced-go-programming-book/images/ch6-02-radix.png" alt="radix tree" style="zoom:50%;" />

*图 5-2 压缩字典树*

每个节点上不只存储一个字母了，这也是压缩字典树中“压缩”的主要含义。使用压缩字典树可以减少树的层数，同时因为每个节点上数据存储也比通常的字典树要多，所以程序的局部性较好（一个节点的path加载到cache即可进行多个字符的对比），从而对CPU缓存友好。

### 5.2.3.1 root 节点创建

httprouter的Router结构体中存储压缩字典树使用的是下述数据结构：

```go
// 略去了其它部分的 Router struct
type Router struct {
    // ...
    trees map[string]*node
    // ...
}
```

`trees`中的`key`即为HTTP 1.1的RFC中定义的各种方法，具体有：

```shell
GET
HEAD
OPTIONS
POST
PUT
PATCH
DELETE
```

每一种方法对应的都是一棵独立的压缩字典树，这些树彼此之间不共享数据。具体到我们上面用到的路由，`PUT`和`GET`是两棵树而非一棵。

简单来讲，某个方法第一次插入的路由就会导致对应字典树的根节点被创建，我们按顺序，先是一个`PUT`：

```go
r := httprouter.New()
r.PUT("/user/installations/:installation_id/repositories/:reposit", Hello)
```

这样`PUT`对应的根节点就会被创建出来。把这棵`PUT`的树画出来：

![put radix tree](https://chai2010.cn/advanced-go-programming-book/images/ch6-02-radix-put.png)

*图 5-3 插入路由之后的压缩字典树*

radix的节点类型为`*httprouter.node`，为了说明方便，我们留下了目前关心的几个字段：

```
path: 当前节点对应的路径中的字符串

wildChild: 子节点是否为参数节点，即 wildcard node，或者说 :id 这种类型的节点

nType: 当前节点类型，有四个枚举值: 分别为 static/root/param/catchAll。
    static                   // 非根节点的普通字符串节点
    root                     // 根节点
    param                    // 参数节点，例如 :id
    catchAll                 // 通配符节点，例如 *anyway

indices：子节点索引，当子节点为非参数类型，即本节点的wildChild为false时，会将每个子节点的首字母放在该索引数组。说是数组，实际上是个 string。
```

### 5.2.3.2 子节点插入

当插入`GET /marketplace_listing/plans`时，类似前面PUT的过程，GET树的结构如*图 5-4*：

![get radix step 1](https://chai2010.cn/advanced-go-programming-book/images/ch6-02-radix-get-1.png)

*图 5-4 插入第一个节点的压缩字典树*

然后插入`GET /marketplace_listing/plans/:id/accounts`，新的路径与之前的路径有共同的前缀，且可以直接在之前叶子节点后进行插入，那么结果也很简单，插入后的树结构见*图 5-5*:

<img src="https://chai2010.cn/advanced-go-programming-book/images/ch6-02-radix-get-2.png" alt="get radix step 2" style="zoom:67%;" />

接下来我们插入`GET /search`，这时会导致树的边分裂，见*图 5-6*。

![get radix step 3](https://chai2010.cn/advanced-go-programming-book/images/ch6-02-radix-get-3.png)

*图 5-6 插入第三个节点，导致边分裂*

这时 root 节点的 path 变成 "/"，并且 indices 是 "ms"，"ms"代表子节点的首字母分别为m（marketplace）和s（search）。

把`GET /status`和`GET /support`也插入到树中。这时候会导致在`search`节点上再次发生分裂，最终结果见*图 5-7*：

![get radix step 4](https://chai2010.cn/advanced-go-programming-book/images/ch6-02-radix-get-4.png)

*图 5-7 插入所有路由后的压缩字典树*

### 5.2.3.4 子节点冲突处理

在路由本身只有字符串的情况下，不会发生任何冲突。只有当路由中含有wildcard（类似 :id）或者catchAll的情况下才可能冲突。

如何判断子节点是否冲突，有几种情况：

1. 在插入wildcard节点时，父节点的children数组非空且wildChild被设置为false。例如：`GET /user/getAll`和`GET /user/:id/getAddr`，或者`GET /user/*aaa`和`GET /user/:id`。
2. 在插入wildcard节点时，父节点的children数组非空且wildChild被设置为true，但该父节点的wildcard子节点要插入的wildcard名字不一样。例如：`GET /user/:id/info`和`GET /user/:name/info`。
3. 在插入catchAll节点时，父节点的children非空。例如：`GET /src/abc`和`GET /src/*filename`，或者`GET /src/:id`和`GET /src/*filename`。
4. 在插入static节点时，父节点的wildChild字段被设置为true。
5. 在插入static节点时，父节点的children非空，且子节点nType为catchAll。

只要发生冲突，都会在初始化的时候panic。例如，在插入我们臆想的路由`GET /marketplace_listing/plans/ohyes`时，出现第4种冲突情况：它的父节点`marketplace_listing/plans/`的wildChild字段为true。