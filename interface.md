# interface内部实现
## 空接口
>`空接口类型`可以接受任意类型的数据，作用只要记录这个数据在哪儿，是什么类型的就可以了
```
//go/src/runtime/runtime2.go
//空接口数据结构
type eface struct {
    _type *_type
    data unsafe.Pointer
}
```

*  **\_type：** 指向接口的`动态`类型元数据
*  **data：** 指向接口的动态值

#### 类型元数据

**\_type类型元数据:** 类型名称、类型大小、对齐边界、是否自定义等，是每个类型元数据都要记录的信息，所以被放到了runtime.\_type结构体中，作为每个类型元素的`Heade`
![avatar](https://img-blog.csdnimg.cn/img_convert/1511cc6ee4595065841ec9a3c2a7f23f.png)

**其它的描绘信息:** 就是存储各种类型额外需要描述的信息,如上图slice的类型元数据在_type结构体后面，记录着一个\*\_type指向其存储的元素的类型元数据。

**如果是自定义类型**，后面还会有一个`uncommontype`结构体，如图
![avatar](https://img-blog.csdnimg.cn/img_convert/dd83792f0d39a33bf0e877ebf8f9fb9b.png)


| uncommontype | 意思 |
| --- | --- |
| pkgpath | 记录类型所在的包路径 |
| mcount |记录改类型关联到多少个方法  |
| moff | 记录的是这些`方法元`数据组成的数组，相对于这个uncommontype结构体偏移了多少字节 |
>通过uncommontype这里记录的moff信息，我们就可以找到给自定义结构的方法元数据在哪儿了


例如我们基于[ ]string定义一个新类型myslice，他就是一个自定义类型，可以给它定义两个方法Len和Cap。myslice类型元数据，首先是[ ]string的类型描述信息，然后在后面加上uncommontype结构体

![avatar](https://img-blog.csdnimg.cn/img_convert/b24aed3c24351d9bf4a6c73389f2f670.png)

明白了数据的类型元数据和方法元数据，空接口的数据类型就很明白了，如下图



**空接口赋值变化：**
![03c60023c777088086742647de4e766f.png](en-resource://database/824:1)

当把f赋值给e这个空接口之后，data就从nil赋值为f的值，_type的动态数据元等于\*os.file的数据元数据信息，通过uncommontype和偏移量可以很轻松的找到方法元数据数组。
>一个空接口类型变量，再被赋值以前_type和data都为nil

* * *



##  **非空接口的数据结构**

>首先一个变量要实现一个非空接口类型，就是要实现接口中定义的所有方法，下面我们看看非空接口的数据结构
```
// src/runtime/runtime2.go
type iface struct {
    tab *itab                //itab 存放类型及方法指针信息
    data unsafe.Pointer      //动态值
}
```

可以看到 iface 结构很简单，有两个指针类型字段。
* **itab**：接口自身的信息和接口动态类型信息
* **数据指针 data**：指向接口的动态值

data指针，跟空接口的相似，我们只要讲解`itab`
![avatar](https://img-blog.csdnimg.cn/img_convert/1dec2e9f0dc86cfa00b999daf9fdf95a.png)


* inter： 指向interface的类型元数据
    * mhdr： 接口要求的方法列表
* \_type： 指向接口的动态类型元数据
*  hash： 是从动态类型元数据中拷贝来的类型哈希值，用于快速判断类型是否相等时使用
* fun ：记录的是这个动态类型实现的那些接口要求的方法地址

我们通过一个例子来说明这些字段对应的意识
![avatar](https://img-blog.csdnimg.cn/img_convert/4bca4a5497933516ce1cd38b870b838d.png)

此时这个itab结构体中的接口类型inter就是io.ReadWriter
动态类型_type为\*os.File

注意：itab这里的fun，会从动态类型元数据中拷贝接口要求的那些方法的地址，以便通过rw快速定位到方法，而无需再去类型元数据哪里查找

>注：关于itab，一旦接口类型确定了，动态类型也确定了，那么itab的内容就不会改变了。

**哪接口是怎么做到被多个自定义结构体的实现的呢，其实itab是可以复用及缓存的**

* * *


#### itab结构体的复用

go语言会把用到的itab结构体缓存起来，并且以接口和动态类型的组合为key`<接口类型哈希值，动态类型哈希值>`，以itab结构体指针为value，构造一个hash表，用于存储与查询itab缓存信息

但是这里的hash表和map底层的哈希表不同，是一种更为简便的设计，需要一个itab时，会首先去这里查找，

key的哈希值是**用类型接口的类型哈希值与动态类型的类型哈希值**进行异或运算，如果有对应的itab指针，就直接拿来使用，若itab缓存中没有，就要创建一个itab结构体，然后添加到这个哈希表中

* * *
#### 类型断言

接口的类型断言一共分为了4种：
空接口.(具体类型)、非空接口.(具体类型)、空接口.(非空接口)、非空接口.(非空接口)
>注：只有接口才能有类型断言；若断言成功，就把该判断的类型接口的动态数据类型赋值为判断类型，如果失败直接赋值为判断类型的零值

* **空接口.(具体类型)**

`e.(*os.File)`是要判断e的动态类型是否为\*os.File，这只需要确定这个_type是否指向\*os.File的类型元数据就好
>注：Go语言里每种类型的元数据都是全局唯一的

![avatar](https://img-blog.csdnimg.cn/img_convert/cb2b62610ad57265eb30ada9af83de35.png)

e的动态值就是f，动态类型就是\*os.File(os.Open函数返回的类型为\*os.File)，如果成功r被复制为e的动态值，如果不成功就会被赋值为\*os.File的类型零值nil

* **非空接口.(具体类型)**
  ![avatar](https://img-blog.csdnimg.cn/img_convert/3191eee91e7048fe5eb0ba1403dbf407.png)
>补充：os.Open返回os.File这个结构体，这个结构体实现了io.ReadWriter的方法，所以实现了这个接口。

* **空接口.(非空接口)**
  ![avatar](https://img-blog.csdnimg.cn/img_convert/0144de69ee160696d79ed9f21572c222.png)

虽然\*os.File类型元数据后面可以找到类型关联的方法元数据数组，也不必每次都去检查io.ReadWriter类型元数这里是否有对应接口要求的所有方法。因为有itab缓存，可以先去itab缓存中查找一下，如果没有io.ReadWriter和\*os.File对应的itab结构体，再去检查\*os.File的方法列表。


>`注意：`就算能够从缓存中查找到对应的itab，也要判断itab.fun[0]是否等于0，这是因为断言失败的类型组合其对应的itab结构体也会被缓存起来，只是会把itab.fun[0]置为0，用以标识这里的动态类型并没有实现对应的接口。这样以后再遇到同种类型断言时就不用再去检查方法列表了，可以直接断言失败

* **非空接口.(非空接口)**

`w.(io.ReadWriter)`是要判断w存储的动态类型是否实现了io.ReadWriter接口，w时io.Writer类型，接口要求一个Writer方法，io.ReadWriter要求实现Read和Writer两个方法。要确定\*os.File是否实现了io.ReadWriter接口，同样会先去itab缓存里查找这个组合对应的itab指针，若存在，且itab.fun[0]不等于0，则断言成功，如不存在，再去检查\*os.File的方法列表，并缓存itab信息

![avatar](https://img-blog.csdnimg.cn/img_convert/fc8fdcb65a54e911f83f82d5fc8f9009.png)


>**类型断言的关键是明确接口的动态类型，以及对应的类型实现了哪些方法，而明确这些的关键还是类型元数据**

学习资源参考：https://blog.csdn.net/Lin_Bolun/article/details/116562771