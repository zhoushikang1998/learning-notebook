## 一、数据结构

### 1.1 数组

#### 初始化

```go
arr1 := [3]int{1, 2, 3}

arr2 := [...]int{1, 2, 3}	// 语法糖
```

在不考虑逃逸分析的情况下，

- 如果数组中元素的个数小于或者等于 4 个，那么所有的变量会直接在栈上初始化
- 如果数组元素大于 4 个，变量就会在静态存储区初始化然后拷贝到栈上，这些转换后的代码才会继续进入中间代码生成和机器码生成两个阶段，最后生成可以执行的二进制文件。



#### 访问和赋值

- 在内存中：一连串的内存空间。表示数组的方法就是一个**指向数组开头的指针**、**数组中元素的数量**以及**数组中元素类型占的空间大小**
- 数组访问越界判断：
  - 在编译期间由静态类型检查完成	===> gc.typecheck1
    - 访问数组的索引是非整数时会直接报错
    - 访问数组的索引是负数时会直接报错 
    - 访问数组的索引越界时会直接报错
  - 使用变量去访问数组或者字符串时，编译器就无法发现对应的错误，由运行时检查：===> panicIndex、goPanicIndex
- 赋值：（编译期间）
  - 先确定目标数组的地址 ===> 通过 `PtrIndex` 获取目标元素的地址 ====> 最后使用 `Store` 指令将数据存入地址中



### 1.2 切片

- 切片内元素的类型是在编译期间确定的，编译器确定了类型之后，会将类型存储在 `Extra` 字段中帮助程序在运行时动态获取。
- 编译期间的切片是 `Slice` 类型的。



#### 数据结构

- 运行时切片由 SliceHeader 结构体表示：

```go
type SliceHeader struct {
	Data uintptr	// 指向数组的指针
	Len  int		// 切片的长度
	Cap  int		// 切片的容量
}

// 内部安全的结构体
type sliceHeader struct {
	Data unsafe.Pointer
	Len  int
	Cap  int
}
```

- 可以将切片理解成一片**连续的内存空间**加上长度与容量的标识。

#### 初始化

- 三种初始化的方式：	===> 指向原始数组的切片值，所以**修改新切片的数据也会修改原始切片。**

  ```go
  arr[0:3] or slice[0:3]		// 使用下标
  slice := []int{1, 2, 3}		// 使用字面量
  slice := make([]int, 10)	// 使用 make
  ```

  - 使用 make 
    - 发生逃逸或者非常大时，由 makeslice 函数在堆上初始化

#### 访问元素

- 切片的操作基本都是在编译期间完成的，除了访问切片的长度、容量或者其中的元素之外，使用 `range` 遍历切片时也会在编译期间转换成形式更简单的代码。

#### 追加和扩容

- 使用 `append` 关键字向切片追加元素。

```go
// 返回值不会覆盖原值
append(slice, 1, 2, 3)

// 返回值会覆盖原值
slice = append(slice, 1, 2, 3)
```

- 调用 runtime.growslice 函数扩容 ===> 为切片分配一块新的内存空间，并将原切片的元素全部拷贝过去。
  - 先确定新的切片容量（每次 append 一个元素）
    - 如果期望容量大于当前容量的两倍就会使用**期望容量**；
    - 如果当前切片的长度小于 1024 就会将容量翻倍；
    - 如果当前切片的长度大于 1024 就会每次增加 25% 的容量，直到新容量大于期望容量；
  - 每次 append 多个元素：
    - 若是 append 多个元素，且 double 后的容量不能容纳，直接使用预估的容量。
  - 内存对齐 ===> 使用 roundupsize 函数向上取整（根据 class_to_size 数组）

#### 拷贝切片

- memmove：负责对内存进行拷贝，将整块内存中的内容拷贝到目标的内存区域中。



#### 注意

- **大切片扩容或者复制时可能会发生大规模的内存拷贝**，在使用时避免这种情况。





### 1.3 哈希表

#### 设计原理

- 哈希函数：
  - 开放寻址法（数组）
    - 对性能影响最大的就是**装载因子**，当装载率超过 70% 之后，哈希表的性能就会急剧下降，而一旦装载率达到 100%，整个哈希表就会完全失效
  - 拉链法（数组 + 链表）===> 红黑树优化
    - 装载因子 = 元素数量 / 桶数量
    - 装载因子越大，哈希的读写性能就越差

#### 数据结构

- 使用 `hmap` 结构体来表示哈希
- 哈希表 `hmap` 的桶就是 `bmap`，每一个 `bmap` 都能存储 8 个键值对，当哈希表中存储的数据过多，单个桶无法装满时就会使用 `extra.nextOverflow` 中的桶存储溢出的数据。**两种桶在内存中是连续存储的，分别称为正常桶和溢出桶。**



#### 初始化

- 字面量
  - 元素 <= 25 个时，初始化方式同 数组/切片。
  - 元素 > 25 个时，在编译期间创建两个数组分别存储键和值的信息，键值对会通过一个 for 循环加入目标的哈希： ====> 转化为使用 `make` 关键字来创建新的哈希并通过最原始的 `[]` 语法向哈希追加元素。

```go
hash := map[string]int{
	"1": 2,
	"3": 4,
	"5": 6,
}
```



- 运行时
  - 类型检查 期间将它们转换成对 `runtime.makemap `的调用
    - 计算哈希占用的内存是否溢出或者超出能分配的最大值；
    - 调用 `fastrand` 获取一个随机的哈希种子；
    - 根据传入的 `hint` 计算出需要的最小需要的桶的数量；
    - 使用 `runtime.makeBucketArray` 创建用于保存桶的数组；

#### 读写操作

- 通过下标或者遍历两种方式访问。
- 访问：
  - 当接受参数仅为一个时，会使用 `runtime.mapaccess1`，该函数仅会返回一个指向目标值的指针；
  - 当接受两个参数时，会使用 `runtime.mapaccess2`，除了返回目标值之外，它还会返回一个用于表示当前键对应的值是否存在的布尔值。
- 写入：
  - 在编译期间转换成调用 `runtime.mapassign` 函数
    - 首先是函数会根据传入的键拿到对应的哈希和桶
    - 通过遍历比较桶中存储的 `tophash` 和键的哈希
    - 如果当前桶已经满了，哈希会调用 `newoverflow` 函数创建新桶或者使用 `hmap` 预先在 `noverflow` 中创建好的桶来保存数据，新创建的桶不仅会被追加到已有桶的末尾，还会增加哈希表的 `noverflow` 计数器。
    - 如果当前键值对在哈希中不存在，哈希为新键值对规划存储的内存地址；如果当前键值对在哈希中存在，那么就会直接返回目标区域的内存地址。
    - `mapassign` 函数只会返回内存地址，真正的赋值操作是在编译期间插入的。
- 扩容：
  - 两种情况发生时触发哈希的扩容：
    - 装载因子已经超过 6.5；	====> 翻倍扩容
    - 哈希使用了太多溢出桶；   ====> 等量扩容： `sameSizeGrow`
      - 一旦哈希中出现了过多的溢出桶，它就会创建新桶保存数据，垃圾回收会清理老的溢出桶并释放内存。
  - 扩容函数：runtime.hashGrow
    - 创建一组新桶和预创建的溢出桶，随后将原有的桶数组设置到 `oldbuckets` 上并将新的空桶设置到 `buckets` 上，溢出桶也使用了相同的逻辑进行更新
  - 数据迁移：runtime.evacuate
    - 哈希表的数据迁移的过程在是 `runtime.evacuate` 函数中完成的，它会对传入桶中的元素进行『再分配』
    - evacuate 会将一个旧桶中的数据分流到两个新桶（翻倍扩容），或者分流到一个新桶（等量扩容）
  - 总结：
    - **哈希在存储元素过多时会触发扩容操作，每次都会将桶的数量翻倍**，整个扩容过程并不是原子的，而是通过`runtime.growWork` **增量触发**的，在扩容期间访问哈希表时会使用旧桶，向哈希表写入数据时会触发旧桶元素的分流；除了这种正常的扩容之外，为了解决大量写入、删除造成的内存泄漏问题，哈希引入了 `sameSizeGrow` 这一机制，在出现较多溢出桶时会对哈希进行『内存整理』减少对空间的占用。
- 缩容
  - **伪缩容**：缩容仅仅针对溢出桶太多的情况，触发缩容时 hash 数组的大小不变，即 hash 数组所占用的空间只增不减。也就是说，如果我们把一个已经增长到很大的 map 的元素挨个全部删除掉，hash 表所占用的内存空间也不会被释放。
  - **真缩容：创建一个较小的map，将需要缩容的map的元素挨个搬迁过来。**
- 删除：
  - 使用 `delete` 关键字，这个关键字的唯一作用就是将某一个键对应的元素从哈希表中删除，无论该键对应的值是否存在，**这个内建的函数都不会返回任何的结果。**
  - `delete` 关键字在编译期间经过类型检查和中间代码生成阶段被转换成 [`runtime.mapdelete`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L685-L791) 函数簇中的一员就可以，用于处理删除逻辑的函数与哈希表的 [`runtime.mapassign`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L571-L683) 几乎完全相同

#### 总结

- hash 数组 + 桶内的 key-value 数组 + 溢出的桶链表
- 发生 panic 的情况：
  - **未初始化**
  - **go map不支持并发，检测到并发直接 panic**
- 每次扩容，hash 表增大 1 倍，hash 表只增不减。



### 1.4 字符串

- 字符组成的数组（一片连续的内存空间）
- 只读的字节数组。



#### 数据结构

```go
type StringHeader struct {
	Data uintptr
	Len  int
}
```



#### 解析过程

- 解析器在词法分析时完成。
- 两种字面量声明方式：
  - 双引号：只能用于单行字符串的初始化，出现双引号要使用 \ 转义
  - 反引号：可初始化多行，直接使用双引号。适用于手写 JSON 或其他复杂数据格式。

#### 拼接

- 运行时会调用 `copy` 将输入的多个字符串拷贝到目标字符串所在的内存空间中，新的字符串是一片新的内存空间，与原来的字符串也没有任何关联，一旦需要拼接的字符串非常大，拷贝带来的**性能损失**就是无法忽略的。

#### 类型转换

- stringtoslicebyte
- slicebytetostring
- 字符串和字符数组的转换都需要对内容进行拷贝，而内存拷贝的**性能损耗**会随着字符串和 `[]byte` 长度的增长而增长。

#### 注意

- **拼接和类型转换（内容拷贝）等操作时时一定要注意性能的损耗。**







## 二、语言基础

### 2.1 函数调用

#### 调用惯例

- **使用栈传递参数和返回值** ====> 降低实现的复杂度并支持多返回值，但是牺牲了函数调用的性能。



#### 参数传递

- **传值。**	===> **无论是传递基本类型、结构体还是指针，都会对传递的参数进行拷贝**
  - 整型和数组的参数都是值传递的，在调用函数时会对内容进行拷贝，当数组大小非常大时，传值方式会对性能造成比较大的影响。（**使用指针，对原数组操作**）
  - 结构体：拷贝全部内容
  - 指针：两个指针指向原有的内存空间

#### 注意

- 在传递数组或者内存占用非常大的结构体时，尽量使用 **指针** 作为参数类型来避免发生大量数据的拷贝而影响性能。





### 2.2 接口

#### 隐式接口

- 实现接口的所有**方法**就隐式地实现了接口。

#### 类型

- 带有一组方法的接口：iface
- 不带任何方法的 interface{}：eface
  - 空接口 ===> `interface{}` 类型**不是任意类型**



#### 指针和接口

- 接受者的类型：结构体或结构体指针

- 能否通过编译器的检查：

  | 结构体实现接口       | 结构体指针实现接口 |        |
  | -------------------- | ------------------ | ------ |
  | 结构体初始化变量     | 通过               | 不通过 |
  | 结构体指针初始化变量 | 通过               | 通过   |

- 当我们使用指针实现接口时，只有指针类型的变量才会实现该接口；当我们使用结构体实现接口时，指针类型和结构体类型都会实现该接口。



#### nil 和 non-nil

- nil 经过通过向方法传入参数或变量的赋值，发生了隐式地类型转换。nil 转换成了 interface{} 类型。



#### 数据结构

- eface：

  ```go
  type eface struct { // 16 bytes
  	_type *_type
  	data  unsafe.Pointer
  }
  ```

  - 类型结构体：_type
    - Go 语言类型的运行时表示。
    - 任何类型都可以转换成 interface{}

- iface：

  ```go
  type iface struct { // 16 bytes
  	tab  *itab
  	data unsafe.Pointer
  }
  ```

  - itab结构体：

    ```go
    type itab struct { // 32 bytes
    	inter *interfacetype
    	_type *_type
    	hash  uint32
    	_     [4]byte
    	fun   [1]uintptr
    }
    ```

    - 32 字节
    - 接口类型和具体类型的组合（inter + _type）
    - hash 字段：对 `_type.hash` 的拷贝，类型断言使用；使用该字段快速判断目标类型和具体类型 `_type` 是否一致；
    - fun 数组：动态派发

#### 类型转换

- 具体类型转换成接口类型：
  - 指针类型：
    - 结构体 `Cat` 的初始化；
    - 赋值触发的类型转换过程；从 Cat 到 Duck 接口的类型转换
    - 调用接口的方法 `Quack()`；
  - 结构体类型
    - 同指针类型。

#### 类型断言

- 非空接口：
  - 将目标类型的 _type.hash 与接口变量中的 itab.hash 进行比较，相等则会同一类型。
- 空接口：同上。

#### 动态派发

- 在运行期间选择具体多态操作（方法或者函数）执行的过程。

- 有额外的开销、

  - 在关闭编译器优化的情况下：18% 左右的额外性能开销。
  - 开启编译器优化后：动态派发的额外开销会降低至 `~5%`

  | 直接调用 | 动态派发 |         |
  | -------- | -------- | ------- |
  | 指针     | ~3.03ns  | ~3.58ns |
  | 结构体   | ~3.09ns  | ~6.98ns |

- 使用结构体来实现接口带来的开销会大于使用指针实现，而动态派发在结构体上的表现非常差，这也提醒我们**应当尽量避免使用结构体类型实现接口。**

  - **动态派发：使用指针 ===> 性能翻倍。**



### 2.3 反射

#### 反射函数和类型

- reflect.TypeOf： 获取类型信息
- reflect.ValueOf：获取数据的运行时表示
- 反射包中的所有方法基本都是围绕着 `Type` 和 `Value` 这两个类型设计的。
  - Type：接口
  - Value：结构体
- 使用 `reflect.TypeOf` 和 `reflect.ValueOf` 能够获取 Go 语言中的变量对应的反射对象。一旦获取了反射对象，我们就能得到跟当前类型相关数据和操作，并可以使用这些运行时获取的结构执行方法。
- 使用 `reflect.Value.Interface` 方法将反射对象还原成接口给类型的变量。

#### 三大法则

1. 从 interface{} 变量可以反射出反射对象；
2. 从反射对象可以获取 interface{} 变量；
3. 要修改反射对象，其值必须可设置。

> 基本类型 ===> interface{} ===> 反射对象（Type、Value）
>
> 反射对象 ===> interface{} ==显式=> 基本类型

- 修改值：

  - 调用 `reflect.ValueOf` 函数获取变量指针；
  - 调用 `reflect.Value.Elem` 方法获取指针指向的变量；
  - 调用 `reflect.Value.SetInt` 方法更新变量的值。

  

#### 类型和值

- `interface{}` 类型在语言内部通过 `emptyInterface` 结构体表示。

  ```go
  type emptyInterface struct {
  	typ  *rtype				// 变量的类型
  	word unsafe.Pointer		// 指向内部封装的数据
  }
  ```

- TypeOf 的实现原理：

  - 将一个 `interface{}` 变量转换成了内部的 `emptyInterface` 表示，然后从中获取相应的类型信息。

- ValueOf 的实现原理：

  - 先调用 `reflect.escapes` 函数保证当前值逃逸到堆上，然后通过 `reflect.unpackEface` 函数从接口中获取 `Value` 结构体。
  - `unpackEface` 函数会将传入的接口转换成 `emptyInterface` 结构体，然后将具体类型和指针包装成 `Value` 结构体返回。

#### 更新变量

- 调用 `reflect.Value.Set` 方法更新反射对象，
  - 调用 `reflect.flag.mustBeAssignable`  和 `reflect.flag.mustBeExported` 方法分别检查当前反射对象是否可以被设置的以及字段是否是对外公开的。
  - 调用 `reflect.Value.assignTo` 并返回一个新的反射对象，返回的反射对象指针会直接覆盖原始的反射变量。

#### 实现协议

- `reflect.rtypes.Implements` 方法：判断某些类型是否遵循特定的接口（是否实现接口）。

- 获得接口类型：

  ```go
  reflect.TypeOf((*<interface>)(nil)).Elem()
  ```

- 实现原理：

  - 检查传入的类型是不是接口，如果不是接口或者是空值就会直接 panic 中止当前程序。
  - 如果接口中不包含任何方法，就意味着这是一个空的接口，任意类型都自动实现该接口，这时就会直接返回 `true`。
  - 在其他情况下，由于方法都是按照字母序存储的，`reflect.implements` 会维护两个用于遍历接口和类型方法的索引 `i` 和 `j` 判断类型是否实现了接口，因为最多只会进行 `n` 次比较（类型的方法数量），所以整个过程的时间复杂度是 `O(n)`。

#### 方法调用

- 参数检查
  - 检查参数个数和类型是否合法。
- 准备参数
  - 将传入的 `reflect.Value` 参数数组设置到栈上
  - 如果当前函数有**返回值**，需要为当前函数的参数和返回值分配一片内存空间 `args`
  - 如果当前函数是**方法**，需要将方法的接受者拷贝到 `args` 内存中

- 调用函数
  - 通过 `reflect.Value.Call` 函数调用。
  - 通过函数指针和输入参数调用函数。
- 处理返回值
  - 没有任何返回值，会直接清空 `args` 中的全部内容来释放内存空间；
  - 有返回值：
    - 将 `args` 中与输入参数有关的内存空间清空；
    - 创建一个 `nout` 长度的切片用于保存**由反射对象构成**的返回值数组；
    - 从函数对象中获取返回值的类型和内存大小，将 `args` 内存中的数据转换成 `reflect.Value` 类型并存储到切片中；



## 三、常用关键字

### 3.1 for 和 range

- **for/range 在遍历切片时追加的元素不会增加循环的执行次数，所以循环最终还是停了下来。**

#### 经典循环

```go
for Ninit; Left; Right {
    NBody
}
```

#### 范围循环

- 编译器会在编译期间将所有 for/range 循环变成的经典循环。
- 所有的 for/range 循环都会被 `gc.walkrange` 函数转换成不包含复杂结构、只包含基本表达式的语句。
- 数组和切片：
  - 同时遍历索引和元素的 range 循环，会额外创建一个新的 `v2` 变量存储切片中的元素，**循环中使用的这个变量 `v2` 会在每一次迭代被重新赋值而覆盖，在赋值时也发生了拷贝。	===> 在循环中获取返回变量的地址完全相同。**
  - 如果我们想要访问数组中元素所在的地址，不应该直接获取 range 返回的变量地址 `&v2`，而应该使用 `&a[index]` 这种形式。
- 哈希表：
  - `runtime.mapinterinit` 函数：初始化遍历开始的元素
    - **引入随机数保证遍历的随机性**。
  - `runtime.mapiternext` 函数：
    - **桶的选择**：正常桶 ===> 对应的所有溢出桶 ===> 下一个正常桶 ===> 对应的所有溢出桶 .... .... ===> 直到所有的都被遍历完成。
    - 桶内元素的遍历：从桶中找到下一个遍历的元素
- 字符串：
  - 遍历字符串的过程与数组、切片和哈希表非常相似，只是在遍历时会获取字符串中索引对应的字节并**将字节转换成 `rune`。**
- 通道 channel：
  - 使用 <- ch 取值
  - 调用 runtime.chanrecv2 阻塞当前协程
  - 判断当前值是否存在



### 3.2 select

- case 中的表达式必须都是 Channel 的收发操作。
- 两个现象：
  - `select` 能在 Channel 上进行非阻塞的收发操作。
  - `select` 在遇到多个 Channel 同时响应时会随机挑选 `case` 执行。

#### 非阻塞的收发

- 如果 `select` 控制结构中包含 `default` 语句，那么这个 `select` 语句在执行时会遇到以下两种情况：
  1. 当存在可以收发的 Channel 时，直接处理该 Channel 对应的 `case`；
  2. 当不存在可以收发的 Channel 是，执行 `default` 中的语句；

#### 随机执行

- `select` 在遇到多个 `<-ch` 同时满足可读或者可写条件时会随机选择一个 `case` 执行其中的代码。



#### 数据结构

- case ===> `runtime.scase`

  ```go
  type scase struct {
  	c           *hchan			// 存储 channel
  	elem        unsafe.Pointer	// 接收或者发送数据的变量地址
  	kind        uint16			// runtime.scase 的类型，四类：空/收/发/默认
  	pc          uintptr
  	releasetime int64
  }
  ```

#### 实现原理

- `select` 语句在编译期间会被转换成 `OSELECT` 节点。
- `OSELECT` 节点都会持有一组 `OCASE` 节点，如果 `OCASE` 的执行条件是空，那就意味着这是一个 `default` 节点
- 四种情况：
  - **直接阻塞**：`select` 不存在任何的 `case`；		===> 空的 `select` 语句会直接阻塞当前的 Goroutine，导致 Goroutine 进入无法被唤醒的永久休眠状态。
  - **单一管道**：`select` 只存在一个 `case`；            ===> select 语句改写成 if 语句
  - **非阻塞操作**：`select` 存在两个 `case`，其中一个 `case` 是 `default`；      ====> 使用 if/else 语句改写
    - 发送：`runtime.selectnbsend`===> 向 channel 非阻塞地发送数据。
    - 接收：从 Channel 中接收数据可能会返回一个或者两个值，返回值数量不同导致使用函数不同。
  - **常见流程**：`select` 存在多个 `case`；
    1. 将 case ===> runtime.scase。
    2. 调用运行时函数 `runtime.selectgo` 选择一个可执行的 scase。
    3. 通过 for 循环生成一组 `if` 语句，在语句中判断自己是不是被选中的 `case`。
  - `selectgo` 函数实现过程：
    - 执行一些必要的初始化操作并确定 `case` 的处理顺序；
      - 轮询顺序（pollOrder）：通过 `runtime.fastrandn` 函数引入随机性；	===> **避免 Channel 的饥饿问题，保证公平性**
      - 加锁顺序（lockOrder）：按照 Channel 的地址排序后确定加锁顺序；       ===> **避免死锁的发生**
    - 在循环中根据 `case` 的类型做出不同的处理； ===> 分三个阶段查找或者等待某个 Channel 准备就绪
      - 查找是否已经存在准备就绪的 Channel，即可以执行收发操作；
        - 根据 `case` 的四种类型分别处理（caseNil、caseRecv、caseSend、caseDefault）
      - 将当前 Goroutine 加入 Channel 对应的收发队列上并等待其他 Goroutine 的唤醒；
        - 将当前的 Goroutine(runtime.sudog 结构体) 加入到 Channel 的 `sendq` 或者 `recvq` 队列中，`sudog` 结构体都会被串成链表附着在 Goroutine上。在入队之后会调用 `runtime.gopark` 函数挂起当前 Goroutine 等待调度器的唤醒。
      - 当前 Goroutine 被唤醒之后找到满足条件的 Channel 并进行处理；
        - 从 `runtime.sudog` 结构体中获取数据，依次对比所有 `case` 对应的 `sudog` 结构找到被唤醒的 `case`，获取该 `case` 对应的索引并返回。
- `select` 语句中的 Channel 收发操作和直接操作 Channel 没有太多出入，只是由于 `select` 多出了 `default` 关键字所以会支持非阻塞的收发。



#### 注意

- 从一个关闭 Channel 中**接收数据**会直接清除 Channel 中的相关内容；
- 向一个关闭的 Channel **发送数据**就会直接 `panic` 造成程序崩溃：



### 3.3 defer

#### 作用域

- `defer` 传入的函数不是在退出代码块的作用域时执行的，它只会在当前函数和方法返回之前被调用。

#### 预计算参数

- 调用 `defer` 关键字**会立刻对函数中引用的外部参数进行拷贝**。

  - 解决方法：向 `defer` 传入匿名函数

  - 如：

    ```go
    // 下列 recover() 函数不生效
    defer recover()
    // 要改成
    defer func() {recover()}
    ```

#### 数据结构

- runtime._defer

  ```go
  type _defer struct {
  	siz     int32		// 参数和结果的内存大小；
  	started bool
  	sp      uintptr		// 栈指针
  	pc      uintptr		// 调用方的程序计数器
  	fn      *funcval	// 传入的函数；
  	_panic  *_panic		// 触发延迟调用的结构体，可能为空；
  	link    *_defer		// 串联成链表
  }
  ```

  

#### 编译过程

- `gc.state.stmt` ===>
  -  `gc.state.call` ===> 为所有函数和方法调用生成中间代码
    - 将 `defer` 关键字转换成 `runtime.deferproc` 函数。
    - 为所有调用 `defer` 的函数末尾插入 `runtime.deferreturn` 的函数调用。
  - `runtime.deferproc`：负责创建新的延迟调用；（_defer 结构体）
    - `newdefer`：获取 _defer 结构体（三种方式）
      1. 从调度器的延迟调用缓存池 `sched.deferpool` 中取出结构体；
      2. 从 Goroutine 的延迟调用缓存池 `pp.deferpool` 中取出结构体；
      3. 通过 `runtime.mallocgc` 创建一个新的结构体。
    - **都会被追加到所在的 Goroutine `_defer` 链表的最前面。** ====> **`defer` 关键字插入时是从后向前的，而 `defer` 关键字执行是从前向后的，而这就是后调用的 `defer` 会优先执行的原因。**
  - `runtime.deferreturn`：负责在函数调用结束时执行所有的延迟调用；
    - 从 Goroutine 的 `_defer` 链表中取出最前面的 `runtime._defer` 结构体
    - `runtime.jmpdefer`：跳转 `defer` 所在的代码段并在执行结束之后跳转回 `runtime.deferreturn`
    - 会多次判断当前 Goroutine 的 `_defer` 链表中是否有未执行的剩余结构，在所有的延迟函数调用都执行完成之后，该函数才会返回。

#### 注意





### 3.4 panic 和 recover

- `panic` 能够改变程序的控制流，函数调用`panic` 时会立刻停止执行函数的其他代码，并在执行结束后在当前 Goroutine 中递归执行调用方的延迟函数调用 `defer`；
- `recover` 可以中止 `panic` 造成的程序崩溃。**它是一个只能在 `defer` 中发挥作用的函数，在其他作用域中调用不会发挥任何作用；**



#### 跨线程失效

- `panic` 只会触发当前 Goroutine 的延迟函数调用；
- `defer` 关键字对应的 `runtime.deferproc`  会将延迟调用函数与调用方所在 Goroutine 进行关联。多个 Goroutine 之间没有太多的关联，一个 Goroutine 在 `panic` 时也不应该执行其他 Goroutine 的延迟函数。

#### 失效的崩溃恢复

- `recover` 只有在 `defer` 函数中调用才会生效；

#### 嵌套崩溃（少用）

- `panic` 允许在 `defer` 中嵌套多次调用；

#### 数据结构

- `panic` ：runtime._panic

  ```go
  type _panic struct {
  	argp      unsafe.Pointer	// 指向 defer 调用时参数的指针；
  	arg       interface{}		// 调用 panic 时传入的参数；
  	link      *_panic			// 指向了更早调用的 runtime._panic 结构
  	recovered bool				// 表示当前的 runtime._panic 是否被 recover 恢复
  	aborted   bool				// 当前的 panic 是否被强行终止
  
  	pc        uintptr
  	sp        unsafe.Pointer
  	goexit    bool
  }
  ```



#### 程序崩溃

- panic ===> runtime.gopanic
  - 创建新的 runtime._panic 结构并添加到所在 Goroutine _panic 链表的最前面。
  - 在循环中不断从当前 Goroutine 的 _defer 链表中获取 defer 结构体并调用 `runtime.reflectcall` 运行延迟调用函数。
  - 调用 `runtime.fatalpanic` 终止整个程序
    - 实现了无法被恢复的程序崩溃，打印出全部的 `panic` 消息以及调用时传入的参数：
    - 通过 `runtime.exit(2)` 退出程序（程序的正常退出也是通过 runtime.exit 函数实现的）

#### 崩溃恢复

- `recover` 关键字 ===> `runtime.gorecover` 函数
  - **如果当前 Goroutine 没有调用 `panic`，那么该函数会直接返回 `nil`，这也是崩溃恢复在非 `defer` 中调用会失效的原因。**
  - panic ===> 修改 `_panic` 的 `recovered` 字段，程序的恢复由 `runtime.gopanic` 负责

#### 注意

- defer 在调用 recover 函数时，要使用匿名函数：
  - **defer 会预计算参数，会立刻将 recover 函数进行拷贝，此时还未发生 panic。**
  - **如果没有发生 panic，recover 不会做任何处理。**



### 3.5 make 和 new

- `make` 的作用是**初始化内置的数据结构**，即切片（slice）、哈希表（map）和 Channel（chan）。
- `new` 的作用是根据传入的类型**分配一片内存空间并返回指向这片内存空间的指针**

#### make

- 在编译期间的**类型检查**阶段，Go 语言就将代表 `make` 关键字的 `OMAKE` 节点根据参数类型的不同转换成了 `OMAKESLICE`、`OMAKEMAP` 和 `OMAKECHAN` 三种不同类型的节点，这些节点会**调用不同的运行时函数来初始化相应的数据结构。**

#### new 

- 编译器会在**中间代码生成**阶段通过以下两个函数处理该关键字：
  - `gc.callnew`：将 `new` 关键字转换成 `ONEWOBJ` 类型的节点
  - `gc.state.expr`：根据申请空间的大小分两种情况处理
    1. 如果申请的空间为 0，就会返回一个表示空指针的 `zerobase` 变量；
    2. 其他：将 `new` 关键字转换成 `runtime.newobject` 函数
-  `runtime.newobject` ===> `runtime.mallocgc` ====> 在堆上申请一片内存空间并返回指向这片内存空间的指针





## 四、并发编程

### 4.1上下文 Context

#### 设计原理

- 在 Goroutine 构成的树形结构中对信号进行同步以减少计算资源的浪费。
- `context.Context` 的作用就是在不同 Goroutine 之间**同步请求特定数据、取消信号以及处理请求的截止日期。**
- 每一个 `context.Context` 都会从最顶层的 Goroutine 一层一层传递到最下层。
- 多个 Goroutine 同时订阅 `ctx.Done()` 管道中的消息，一旦接收到取消信号就立刻停止当前正在执行的工作。

#### 默认上下文

- `context.Background` 是上下文的默认值，所有其他的上下文都应该从它衍生（Derived）出来；
- `context.TODO` 应该只在不确定使用哪种上下文时使用；

#### 取消信号

- `context.WithCancel` 函数能够从 `context.Context` 中衍生出一个新的子上下文并返回用于取消该上下文的函数（CancelFunc）。一旦执行返回的取消函数，当前上下文以及它的子上下文都会被取消，所有的 Goroutine 都会同步收到这一取消信号。

- `context.WithDeadline` 和 `context.WithTimeout` 都能创建可以被取消的计时器上下文 `context.Timerctx`。

  - `context.WithDeadline` 通过判断父上下文截止时间与当前日期，并通过 `time.AfterFunc` 创建定时器，当时间超过了截止日期后会调用 `context.timerCtx.cancel` 方法同步取消信号。

  - `context.timeCtx` 结构体：

    ```go
    type timerCtx struct {
    	cancelCtx			// 继承了 context.cancleCtx 的相关变量和方法
        // 实现了定时取消功能
    	timer *time.Timer 	// 定时器
    	deadline time.Time	// 截止时间
    }
    ```



#### 传值方法（少用）

- `context.WithValue` 函数能够从父上下文中创建一个子上下文，传值的子上下文使用 `context.valueCtx` 类型。`context.valueCtx` 结构体会将除了 Value 之外的 Err、Deadline 等方法代理到父上下文中，它只会响应 `context.valueCtx.Value` 方法。
- 常见的使用场景是**传递请求对应用户的认证令牌**以及**用于进行分布式追踪的请求 ID。**





### 4.2 同步原语与锁

- 锁是一种并发编程中的同步原语（Synchronization Primitives）。

#### 基本原语

##### Mutex

- 数据结构： `sync.Mutex`

  ```go
  type Mutex struct {
  	state int32		// 互斥锁的状态
  	sema  uint32	// 控制锁状态的信号量
  } 
  ```

  - 在默认情况下，互斥锁的所有状态位都是 `0`，`int32` 中的不同位分别表示了不同的状态：
    - `mutexLocked` — 表示互斥锁的锁定状态；
    - `mutexWoken` — 表示从正常模式被从唤醒；
    - `mutexStarving` — 当前的互斥锁进入饥饿状态；
    - `waitersCount` — 当前互斥锁上等待的 Goroutine 个数；
  - 正常模式和饥饿模式：
    - 在正常模式下，锁的等待者会按照先进先出的顺序获取锁。**正常模式下的互斥锁能够提供更好地性能**。
    - 饥饿模式：**引入的目的是保证互斥锁的公平性。**互斥锁会直接交给等待队列最前面的 Goroutine。新的 Goroutine 在该状态下不能获取锁、也不会进入自旋状态，它们只会在队列的末尾等待。如果一个 Goroutine 获得了互斥锁并且它在队列的末尾或者它等待的时间少于 1ms，那么当前的互斥锁就会被切换回正常模式。**饥饿模式的能避免 Goroutine 由于陷入等待无法获取锁而造成的高尾延时。**

- 加锁：Mutex.Lock()

  - 如果互斥锁处于初始化状态，就会直接通过置位 `mutexLocked` 加锁；
  - 如果互斥锁处于 `mutexLocked` 并且在普通模式下工作，就会进入自旋，执行 30 次 `PAUSE` 指令消耗 CPU 时间等待锁的释放；
  - 如果当前 Goroutine 等待锁的时间超过了 1ms，互斥锁就会切换到饥饿模式；
  - 互斥锁在正常情况下会通过 `sync.runtime_SemacquireMutex` 函数将尝试获取锁的 Goroutine 切换至休眠状态，等待锁的持有者唤醒当前 Goroutine；
  - 如果当前 Goroutine 是互斥锁上的最后一个等待的协程或者等待的时间小于 1ms，当前 Goroutine 会将互斥锁切换回正常模式；

- 解锁：Mutex.UnLock()

  - 当互斥锁已经被解锁时，那么调用 `sync.Mutex.Unlock` 会直接抛出异常；
  - 当互斥锁处于饥饿模式时，会直接将锁的所有权交给队列中的下一个等待者，等待者会负责设置 `mutexLocked` 标志位；
  - 当互斥锁处于普通模式时，如果没有 Goroutine 等待锁的释放或者已经有被唤醒的 Goroutine 获得了锁，就会直接返回；在其他情况下会通过 `sync.runtime_Semrelease` 唤醒对应的 Goroutine；

##### RWMutex

- 读写互斥锁 `sync.RWMutex` 是细粒度的互斥锁，它不限制资源的并发读，但是读写、写写操作无法并行执行。	===> 读写分离

- 数据结构：

  ```go
  type RWMutex struct {
  	w           Mutex		// 复用互斥锁提供的能力；
  	writerSem   uint32		// 写等待读
  	readerSem   uint32		// 读等待写
  	readerCount int32		// 存储了当前正在执行的读操作的数量
  	readerWait  int32		// 当写操作被阻塞时等待的读操作个数
  }
  ```

- 写操作：

  - 加锁：`sync.RWMutex.Lock`
    - 调用结构体持有的 `sync.Mutex` 的 `sync.Mutex.Lock` 方法阻塞后续的写操作；
      - 因为互斥锁已经被获取，其他 Goroutine 在获取写锁时就会进入自旋或者休眠；
    - 调用 `atomic.AddInt32` 方法阻塞后续的读操作：
    - 如果仍然有其他 Goroutine 持有互斥锁的读锁（r != 0），该 Goroutine 会调用 `sync.runtime_SemacquireMutex` 进入休眠状态等待所有读锁所有者执行结束后释放 `writerSem` 信号量将当前协程唤醒。
  - 解锁：`sync.RWMutex.UnLock`
    - 调用 `atomic.AddInt32`函数将 `readerCount` 变回正数，释放读锁；
    - 通过 for 循环触发所有由于获取读锁而陷入等待的 Goroutine：
    - 调用 `sync.Mutex.Unlock` 方法释放写锁；

- 读操作：

  - 加锁：`sync.RWMutex.RLock`
    - 通过 `atomic.AddInt32` 将 `readerCount` 加一；
      1. 如果该方法返回负数 — 其他 Goroutine 获得了写锁，当前 Goroutine 就会调用 `sync.runtime_SemacquireMutex` 陷入休眠等待锁的释放；
      2. 如果该方法的结果为非负数 — 没有 Goroutine 获得写锁，当前方法就会成功返回；
  - 解锁：`sync.RWMutex.RUnLock`
    - 先减少正在读资源的 `readerCount` 整数，根据 `atomic.AddInt32` 的返回值不同分别处理：
      1. 如果返回值大于等于零 — 读锁直接解锁成功；
      2. 如果返回值小于零 — 有一个正在执行的写操作，在这时会调用 `sync.RWMutex.rUnlockSlow`  方法——减少获取锁的写操作等待的读操作数 `readerWait` 并在所有读操作都被释放之后触发写操作的信号量 `writerSem`，该信号量被触发时，调度器就会唤醒尝试获取写锁的 Goroutine。

##### WaitGroup

- `WaitGroup` 可以等待一组 Goroutine 的返回，比较常见的使用场景是**批量发出 RPC 或者 HTTP 请求**：

- 数据结构：

  ```go
  type WaitGroup struct {
  	noCopy noCopy		// 保证 sync.WaitGroup 不会被开发者通过再赋值的方式拷贝；
  	state1 [3]uint32	// 存储着状态和信号量； sema、waiter、counter
  }
  ```

- 接口：`WaitGroup` 对外暴露了三个方法

  - `sync.WaitGroup.Add`：可以更新 `sync.WaitGroup` 中的计数器 counter。该方法可以传入负数，但是计数器只能是非负数，一旦出现负数就会发生程序崩溃。当调用计数器归零，也就是所有任务都执行完成时，就会通过 `sync.runtime_Semrelease` 唤醒处于等待状态的所有 Goroutine。
  - `sync.WaitGroup.Done`：只是向 `sync.WaitGroup.Add` 传入了 -1
  - `sync.WaitGroup.Wait`：会在计数器大于 0 并且不存在等待的 Goroutine 时，调用 `sync.runtime_Semacquire` 陷入睡眠状态。当 `sync.WaitGroup` 的计数器归零时，当陷入睡眠状态的 Goroutine 就被唤醒，该方法会立刻返回。

##### Once

- `sync.Once` 可以保证在 Go 程序运行期间的某段代码只会执行一次。

- 数据结构：

  ```go
  type Once struct {
  	done uint32		// 标识代码块是否执行过
  	m    Mutex		// 互斥锁
  }
  ```

- 接口：唯一方法

  - `sync.Once.Do` ：该方法会接收一个入参为空的函数：
    - 如果传入的函数已经执行过，就会直接返回；
    - 如果传入的函数没有执行过，就会调用 `sync.Once.doSlow` 执行传入的函数：
      - 为当前 Goroutine 获取互斥锁；
      - 执行传入的无入参函数；
      - 运行延迟函数调用，将成员变量 `done` 更新成 1

- 注意：

  - `sync.Once.Do` 方法中传入的函数只会被执行一次，哪怕函数中发生了 `panic`；
  - 两次调用 `sync.Once.Do` 方法传入不同的函数也**只会执行第一次调用的函数**；

##### Cond

- 可以让一系列的 Goroutine 都在满足特定条件时被唤醒。	

- 数据结构

  ```go
  type Cond struct {
  	noCopy  noCopy			// 用于保证结构体不会在编译期间拷贝；
  	L       Locker			// 用于保护内部的 notify 字段
  	notify  notifyList		// 一个 Goroutine 的链表，它是实现同步机制的核心结构
  	checker copyChecker		// 用于禁止运行期间发生的拷贝
  }
  
  type notifyList struct {
  	wait uint32
  	notify uint32
  
  	lock mutex
  	head *sudog
  	tail *sudog
  }
  ```

- 接口：

  - `sync.Cond.Wait` 方法会将当前 Goroutine 陷入休眠状态。**在调用之前一定要使用获取互斥锁，否则会触发程序崩溃；**
  - `sync.Cond.Signal` 方法会唤醒队列最前面、等待最久的 Goroutine；
  - `sync.Cond.Broadcast` 方法会按照一定顺序唤醒队列中全部的 Goroutine；

#### 扩展原语

##### ErrGroup

- 在一组 Goroutine 中提供了同步、错误传播以及上下文取消的功能。

##### Semaphore

- 数据结构：`semaphore.Weighted`
- 接口：
  - `x/sync/semaphore.NewWeighted` 用于创建新的信号量；
  - `x/sync/semaphore.Weighted.Acquire` 阻塞地获取指定权重的资源，如果当前没有空闲资源，就会陷入休眠等待；
  - `x/sync/semaphore.Weighted.TryAcquire` 非阻塞地获取指定权重的资源，如果当前没有空闲资源，就会直接返回 `false`；
  - `x/sync/semaphore.Weighted.Release` 用于释放指定权重的资源；

##### SingleFlight

- 能够在一个服务中抑制对下游的多次重复请求。
- 一个比较常见的使用场景是 — 我们在使用 Redis 对数据库中的数据进行缓存，发生缓存击穿时，大量的流量都会打到数据库上进而影响服务的尾延时。`singleflight.Group` 能够限制对同一个 `Key` 的多次重复请求，减少对下游的瞬时流量。





### 4.3 定时器

#### 设计原理

- 全局最小四叉堆

  - 全局四叉堆共用的互斥锁对计时器的影响非常大，计时器的各种操作都需要获取全局唯一的互斥锁，这会**严重影响计时器的性能**。

- 分片四叉堆

  - 将全局的四叉堆分割成了 64 个更小的四叉堆。以牺牲内存占用的代价换取性能的提升。
  - 虽然能够降低锁的粒度，提高计时器的性能，但是造成的**处理器和线程之间频繁的上下文切换**却成为了影响计时器性能的首要因素。

- 网络轮询器

  - 所有的计时器都以最小四叉堆的形式存储在处理器 `runtime.p`  中。

  - 处理器 `runtime.p` 中与计时器相关的有以下字段：

    ```go
    type p struct {
    	...
    	timersLock mutex		// 用于保护计时器的互斥锁；
    	timers []*timer			// 存储计时器的最小四叉堆；
    
    	numTimers     uint32	// 处理器中的计时器数量；
    	adjustTimers  uint32	// 处理器中处于 `timerModifiedEarlier` 状态的计时器数量；
    	deletedTimers uint32	// 处理器中处于 `timerDeleted` 状态的计时器数量；
    	...
    }
    ```

  - 目前计时器都**交由处理器的网络轮询器和调度器触发**，这种方式能够充分利用本地性、减少线上上下文的切换开销，也是目前性能最好的实现方式。



#### 数据结构

- `runtime.timer`：私有的计时器运行时表示

  ```go
  type timer struct {
  	pp puintptr
  
  	when     int64		// 当前计时器被唤醒的时间；
  	period   int64		// 两次被唤醒的间隔；
  	f        func(interface{}, uintptr)		// 每当计时器被唤醒时都会调用的函数；
  	arg      interface{}	// 计时器被唤醒时调用 f 传入的参数；
  	seq      uintptr
  	nextwhen int64		// 计时器处于 timerModifiedXX 状态时，用于设置 when 字段；
  	status   uint32		// 计时器的状态；
  }
  ```

- `time.Timer` ：对外暴露的计时器

  ```go
  type Timer struct {
  	C <-chan Time
  	r runtimeTimer
  }
  ```

  - `time.Timer` 计时器必须通过 `time.NewTimer`、`time.AfterFunc` 或者 `time.After` 函数创建。当计时器失效时，失效的时间就会被发送给计时器持有的 Channel，订阅 Channel 的 Goroutine 会收到计时器失效的时间。

#### 状态机

- 运行时使用状态机的方式处理全部的计时器，其中包括 10 种状态和 7 种操作。

- 状态：

  | 状态                 | 解释                   |
  | -------------------- | ---------------------- |
  | timerNoStatus        | 还没有设置状态         |
  | timerWaiting         | 等待触发               |
  | timerRunning         | 运行计时器函数         |
  | timerDeleted         | 被删除                 |
  | timerRemoving        | 正在被删除             |
  | timerRemoved         | 已经被停止并从堆中删除 |
  | timerModifying       | 正在被修改             |
  | timerModifiedEarlier | 被修改到了更早的时间   |
  | timerModifiedLater   | 被修改到了更晚的时间   |
  | timerMoving          | 已经被修改正在被移动   |

  - `timerRunning`、`timerRemoving`、`timerModifying` 和 `timerMoving` — 停留的时间都比较短；
  - `timerWaiting`、`timerRunning`、`timerDeleted`、`timerRemoving`、`timerModifying`、`timerModifiedEarlier`、`timerModifiedLater` 和 `timerMoving` — 计时器在处理器的堆上；
  - `timerNoStatus` 和 `timerRemoved` — 计时器不在堆上；
  - `timerModifiedEarlier` 和 `timerModifiedLater` — 计时器虽然在堆上，但是可能位于错误的位置上，需要重新排序；

- 操作：分别由不同的提交引入运行时负责不同的工作。验证状态是否合理以及引发状态的改变。

  - `runtime.addtimer` — 向当前处理器**增加**新的计时器；
    - `timerNoStatus` -> `timerWaiting`
    - 其他状态 -> 崩溃：不合法的状态
  - `runtime.deltimer` — 将计时器标记成 `timerDeleted` **删除**处理器中的计时器；
  - `runtime.modtimer` — 网络轮询器会调用该函数**修改**计时器；
  - `runtime.resettimer` — **重置**计时器。修改已经失效的计时器的到期时间，将其变成活跃的计时器；
  - `runtime.cleantimers`  — **清除**队列头中的计时器，能够提升程序创建和删除计时器的性能；
  - `runtime.adjusttimers` — **调整**处理器持有的计时器堆，包括移动会稍后触发的计时器、删除标记为 `timerDeleted` 的计时器；
  - `runtime.runtimer` — 检查队列头中的计时器，在其准备就绪时**运行**该计时器；



#### 触发计时器

- Go 语言会在两个模块触发计时器，运行计时器中保存的函数：
  - **调度器调度时会检查处理器中的计时器是否准备就绪**；
    - `runtime.checkTimers` 是调度器用来运行处理器中计时器的函数，会在发生以下情况时被调用：
      - 调度器调用 `runtime.schedule` 执行调度时；
      - 调度器调用 `runtime.findrunnable` 获取可执行的 Goroutine 时；
      - 调度器调用 `runtime.findrunnable` 从其他处理器窃取计时器时；
  - **系统监控会检查是否有未执行的到期计时器**
    - 系统监控函数 `runtime.sysmon` 也可能会触发函数的计时器





### 4.4 Channel

- **不要通过共享内存的方式进行通信，而是应该通过通信的方式共享内存。**

#### 设计原理

- 通信顺序进程（Communicating sequential processes，CSP）：Goroutine ===> Channel ===> Goroutine
- 先入先出（FIFO）
  - 先从 Channel **读取数据**的 Goroutine 会先接收到数据；
  - 先向 Channel **发送数据**的 Goroutine 会得到先发送数据的权利；
- 无锁管道：乐观并发控制（乐观锁）==> CAS/非锁
- 从某种程度上说，Channel 是一个用于同步和通信的有锁队列。

#### 数据结构

- `runtime.hchan`

  ```go
  type hchan struct {
  	qcount   uint			// Channel 中的元素个数；
  	dataqsiz uint			// Channel 中的循环队列的长度；
  	buf      unsafe.Pointer	// Channel 的缓冲区数据指针；
  	elemsize uint16			// 能够收发的元素大小
  	closed   uint32
  	elemtype *_type			// 能够收发的元素类型
  	sendx    uint  			//  Channel 的发送操作处理到的位置；
  	recvx    uint			// Channel 的接收操作处理到的位置；
  	recvq    waitq			
  	sendq    waitq			// 存储了当前 Channel 由于缓冲区空间不足而阻塞的 Goroutine 列表
  
  	lock mutex
  }
  
  // 双向链表，链表中所有的元素都是 runtime.sudog 结构
  type waitq struct {
  	first *sudog
  	last  *sudog
  }
  ```

#### 创建管道

- `make(chan int, 10)` 	===>	`OMAKE`	===>	`OMAKECHAN`
- 对传入 `make` 关键字的缓冲区大小进行检查，如果我们不向 `make` 传递表示缓冲区大小的参数，那么就会设置一个默认值 0，也就是当前的 Channel 不存在缓冲区。
- ===>	`runtime.makechan`：根据传入的参数类型和缓冲区大小创建一个新的 Channel 结构
  - 如果当前 Channel 中不存在缓冲区，那么就只会为 `runtime.hchan` 分配一段内存空间；
  - 如果当前 Channel 中存储的类型不是指针类型，就会为当前的 Channel 和底层的数组分配一块连续的内存空间；
  - 在默认情况下会单独为 `runtime.hchan` 和缓冲区分配内存；
  - 在函数的最后会统一更新 `runtime.hchan` 的 `elemsize`、`elemtype` 和 `dataqsiz` 几个字段。

#### 发送数据

- `ch <- i`	===>	`OSEND`	===>	`runtime.chansend1`	===>	`runtime.chansend`
- `runtime.chansend` 的执行过程：
  - **直接发送：**当存在等待的接收者时，通过 `runtime.send` 直接将数据发送给阻塞的接收者；
    - 从接收队列 `recvq` 中取出最先陷入等待的 Goroutine 并直接向它发送数据
    - 调用 `runtime.send` 方法发送数据，执行过程：
      -  调用 `runtime.sendDirect` 函数将发送的数据直接拷贝到 `x = <-c` 表达式中变量 `x` 所在的内存地址上；
      - 调用 `runtime.goready` 将等待接收数据的 Goroutine 标记成**可运行状态 `Grunnable`** 并把该 Goroutine 放到发送方所在的处理器的 `runnext` 上等待执行，该处理器在下一次调度时就会立刻唤醒数据的接收方；
    - **注意：**发送数据的过程只是将接收方的 Goroutine 放到了处理器的 `runnext` 中，程序没有立刻执行该 Goroutine。
  - **缓冲区：**当缓冲区存在空余空间时，将发送的数据写入 Channel 的缓冲区；
    - 当前 Channel 的缓冲区未满，向 Channel 发送的数据会存储在 Channel 中 `sendx` 索引所在的位置并将 `sendx` 索引加一，由于这里的 `buf` 是一个循环数组，所以当 `sendx` 等于 `dataqsiz` 时就会重新回到数组开始的位置。
  - **阻塞发送：**当不存在缓冲区或者缓冲区已满时，等待其他 Goroutine 从 Channel 接收数据；
    - 当 Channel 没有接收者能够处理数据时，向 Channel 发送数据就会被下游阻塞，使用 `select` 关键字可以向 Channel 非阻塞地发送消息。
    - 创建一个 `runtime.sudog` 结构并将其加入 Channel 的 `sendq` 队列中，当前 Goroutine 也会陷入阻塞等待其他的 Goroutine 从 Channel 接收数据；
- 发送数据的过程中包含几个会**触发 Goroutine 调度的时机**：
  1. 发送数据时发现 Channel 上存在等待接收数据的 Goroutine，立刻设置处理器的 `runnext` 属性，但是并不会立刻触发调度；
  2. 发送数据时并没有找到接收方并且缓冲区已经满了，这时就会将自己加入 Channel 的 `sendq` 队列并调用 `runtime.goparkunlock` 触发 Goroutine 的调度让出处理器的使用权；



#### 接收数据

- `i <- ch`	===>	 `ORECV` 	===>	`OAS2RECV`	===>	`chanrecv1`	===>	`chanrecv`
- `i, ok <- ch`	===>	 `ORECV` 	===>	`OAS2RECV`    ===>	`chanrecv2`	===>	`chanrecv`
- 特殊情况：
  - 从一个空 Channel 接收数据时会直接调用 `runtime.gopark` 直接让出处理器的使用权。
  - 如果当前 Channel 已经被关闭并且缓冲区中不存在任何的数据，那么就会清除 `ep` 指针中的数据并立刻返回。
- 常规的三种情况：
  - **直接接收：**当存在等待的发送者时，通过 `runtime.recv` 直接从阻塞的发送者或者缓冲区中获取数据；
    - 取出队列头等待的 Goroutine，处理的逻辑和发送时相差无几。
    - 调用 `runtime.recv` 函数接收数据（根据缓冲区的大小分别处理不同的情况）：
      - 如果 Channel 不存在缓冲区；
        1. 调用 `runtime.recvDirect` 函数会将 Channel 发送队列中 Goroutine 存储的 `elem` 数据拷贝到目标内存地址中；
      - 如果 Channel 存在缓冲区；
        1. 将队列中的数据拷贝到接收方的内存地址；
        2. 将发送队列头的数据拷贝到缓冲区中，释放一个阻塞的发送方；
    - 无论发生哪种情况，运行时都会调用 `runtime.goready` 函数将当前处理器的 `runnext` 设置成发送数据的 Goroutine，在调度器下一次调度时将阻塞的发送方唤醒。
  - **缓冲区：**当缓冲区存在数据时，从 Channel 的缓冲区中接收数据；
    - 当 Channel 的缓冲区中已经包含数据时，从 Channel 中接收数据会直接从缓冲区中 `recvx` 的索引位置中取出数据进行处理：
  - **阻塞接收：**当缓冲区中不存在数据时，等待其他 Goroutine 向 Channel 发送数据；
    - 当 Channel 的发送队列中不存在等待的 Goroutine 并且缓冲区中也不存在任何数据时，从管道中接收数据的操作会变成阻塞操作；**与 `select` 语句结合使用时就可能会使用到非阻塞的接收操作。**
- 从 Channel 接收数据时，会触发 Goroutine 调度的两个时机：
  1. 当 Channel 为空时；
  2. 当缓冲区中不存在数据并且也不存在数据的发送者时；



#### 关闭管道

- `close`	===>	`OCLOSE`	===>	`runtime.closechan`
- 当 Channel 是一个空指针或者已经被关闭时，Go 语言运行时都会直接 `panic` 并抛出异常
- 将 `recvq` 和 `sendq` 两个队列中的数据加入到 Goroutine 列表 `gList` 中，与此同时该函数会清除所有 `sudog` 上未被处理的元素
- 在最后会为所有被阻塞的 Goroutine 调用 `runtime.goready` 触发调度。



### 4.5 调度器（Goroutine 原理）

#### 设计原理

- Go 语言的调度器通过**使用与 CPU 数量相等的线程**减少线程频繁切换的内存开销，同时在每一个线程上执行额外开销更低的 Goroutine 来降低操作系统和硬件的负载。

- 单线程调度器

  - G、M 数据结构，建立了 Go 语言调度器的基本框架

- 多线程调度器

  - 引入了 **`GOMAXPROCS` 变量**帮助我们灵活控制程序中的最大处理器数，即活跃线程数。
  - 主要问题是调度时的锁竞争会严重浪费资源。

- **任务窃取调度器**

  - 在当前的 G-M 模型中引入了处理器 P，增加中间层；

  - 在处理器 P 的基础上实现基于工作窃取的调度器

    - 当前处理器本地的运行队列中不包含 Goroutine 时，调用 `findrunnable` 函数会触发工作窃取，从其它的处理器的队列中随机获取一些 Goroutine。

  - GMP 模型图：

    <img src="images/golang-gmp.png" style="zoom:80%;" />

  - 基于工作窃取的多线程调度器将每一个线程绑定到了独立的 CPU 上，这些线程会被不同处理器管理，不同的处理器通过工作窃取**对任务进行再分配实现任务的平衡**，也能提升调度器和 Go 语言程序的整体性能，今天所有的 Go 语言服务都受益于这一改动。

- 抢占式调度器

  - 解决问题：
    - 某些 Goroutine 可以长时间占用线程，造成其它 Goroutine 的饥饿；
    - 垃圾回收需要暂停整个程序（Stop-the-world，STW），最长可能需要几分钟的时间，导致整个程序无法工作；
  - 基于协作的抢占式调度	===> 通过编译器插入函数实现
    - 这里的抢占是通过编译器插入函数实现的，还是需要函数调用作为入口才能触发抢占，所以这是一种**协作式的抢占式调度**。
    - 主要问题：Go 语言的调度器都有一些无法被抢占的边缘情况，直到 1.14 才被**基于信号的抢占式调度**解决。
  - 基于信号的抢占式调度（非协作）
    - 只解决了垃圾回收和栈扫描时存在的问题，未解决全部问题，但真抢占式调度时调度器正在走向完备。

- 非均匀内存访问调度器

#### 数据结构

- G（Goroutine）===> 它是一个待执行的任务。

  - Goroutine 只存在于 Go 语言的运行时，它是 Go 语言在用户态提供的线程。比线程占用了更小的内存空间，也降低了上下文切换的开销。

  - `runtime.g`

    ```go
    type g struct {
        // 栈相关
    	stack       stack
    	stackguard0 uintptr
        // 抢占相关
        preempt       bool		// 抢占信号
    	preemptStop   bool		// 抢占时将状态修改成 `_Gpreempted`
    	preemptShrink bool		// 在同步安全点收缩栈
        
        _panic       *_panic	// 最内侧的 panic 结构体
    	_defer       *_defer	// 最内侧的延迟函数结构体
        
        m              *m		// 当前 Goroutine 占用的线程，可能为空
    	sched          gobuf	// 存储 Goroutine 的调度相关的数据
    	atomicstatus   uint32	// Goroutine 的状态
    	goid           int64	// Goroutine 的 ID，该字段对开发者不可见
    }
    ```

  - 9 个状态：

    | 状态                 | 描述                                                         |
    | -------------------- | ------------------------------------------------------------ |
    | `_Gidle`             | 刚刚被分配并且还没有被初始化                                 |
    | <b>`_Grunnable`</b>  | 没有执行代码，没有栈的所有权，存储在运行队列中               |
    | <b>`_Grunning`</b>   | 可以执行代码，拥有栈的所有权，被赋予了内核线程 M 和处理器 P  |
    | <b>`_Gsyscall`</b>   | 正在执行系统调用，拥有栈的所有权，没有执行用户代码，被赋予了内核线程 M 但是不在运行队列上 |
    | <b>`_Gwaiting`</b>   | 由于运行时而被阻塞，没有执行用户代码并且不在运行队列上，但是可能存在于 Channel 的等待队列上 |
    | `_Gdead`             | 没有被使用，没有执行代码，可能有分配的栈                     |
    | `_Gcopystack`        | 栈正在被拷贝，没有执行代码，不在运行队列上                   |
    | <b>`_Gpreempted`</b> | 由于抢占而被阻塞，没有执行用户代码并且不在运行队列上，等待唤醒 |
    | `_Gscan`             | GC 正在扫描栈空间，没有执行代码，可以与其他状态同时存在      |

  - 在运行期间我们会在**等待中、可运行、运行中**这三种不同的状态来回切换：

    - **等待中：**Goroutine 正在等待某些条件满足，例如：系统调用结束等，包括 `_Gwaiting`、`_Gsyscall` 和 `_Gpreempted` 几个状态；
    - **可运行：**Goroutine 已经准备就绪，可以在线程运行，如果当前程序中有非常多的 Goroutine，每个 Goroutine 就可能会等待更多的时间，即 `_Grunnable`；
    - **运行中：**Goroutine 正在某个线程上运行，即 `_Grunning`；

- M（Machine）  ===> 表示操作系统的线程，由**操作系统的调度器**调度和管理。

  - 调度器最多可以创建 10000 个线程，但是其中大多数的线程都不会执行用户代码（可能陷入系统调用），**最多只会有 `GOMAXPROCS` 个活跃线程能够正常运行。**在默认情况下，运行时会将 `GOMAXPROCS` 设置成当前机器的核数，我们也可以使用 `runtime.GOMAXPROCS` 来改变程序中最大的线程数。

  - `runtime.m`

    ```go
    type m struct {
        // Goroutine 相关
    	g0   *g 
    	curg *g
    	...
        // 处理器字段
        p 		puintptr	// 正在运行代码的处理器 
    	nextp	puintptr	// 暂存的处理器 
    	oldp 	puintptr	// 执行系统调用之前的使用线程的处理器
    }
    ```

    

- P（Processor） ===> 处理器，可以被看作**运行在线程上的本地调度器**。

  - 线程和 Goroutine 的中间层，它能提供线程需要的上下文环境，也会负责调度线程上的等待队列，通过处理器 P 的调度，每一个内核线程都能够执行多个 Goroutine，它能在 Goroutine 进行一些 I/O 操作时及时切换，提高线程的利用率。

  - Go 语言程序的处理器数量一定会等于 `GOMAXPROCS`，这些处理器会绑定到不同的内核线程上并利用线程的计算资源运行 Goroutine。

  - `runtime.p`

    ```go
    type p struct {
    	m        muintptr	// 线程
    	
        // 运行队列
    	runqhead uint32
    	runqtail uint32
    	runq     [256]guintptr
    	runnext  guintptr
    	...
    }
    ```

  - `runtime.p` 结构体中的状态 `status` 字段会是以下五种中的一种：

    | 状态        | 描述                                                         |
    | ----------- | ------------------------------------------------------------ |
    | `_Pidle`    | 处理器没有运行用户代码或者调度器，被空闲队列或者改变其状态的结构持有，运行队列为空 |
    | `_Prunning` | 被线程 M 持有，并且正在执行用户代码或者调度器                |
    | `_Psyscall` | 没有执行用户代码，当前线程陷入系统调用                       |
    | `_Pgcstop`  | 被线程 M 持有，当前处理器由于垃圾回收被停止                  |
    | `_Pdead`    | 当前处理器已经不被使用                                       |

#### 调度器启动

- `runtime.schedinit`	===> 初始化调度器	===> maxmcount = 10000（设置最大线程数）
- Go 语言程序能够创建的最大线程数，虽然最多可以创建 10000 个线程，但是可以同时运行的线程还是由 `GOMAXPROCS` 变量控制。



#### 创建 Goroutine

- `go` 	===> 	`runtime.newproc`	===>	接收大小和表示函数的指针 `funcval`
  - 获取或者创建新的 Goroutine 结构体(`runtime.gfget`)
    - 从 Goroutine 所在处理器的 `gFree` 列表或者调度器的 `sched.gFree` 列表中查找空闲的 Goroutine，如果不存在空闲的 Goroutine，就会通过 `runtime.malg` 函数创建一个栈大小足够的新结构体。
  - 将传入的参数移到 Goroutine 的栈上；
    - 调用 `runtime.memmove`  函数将 `fn` 函数的全部参数拷贝到栈上
  - 更新 Goroutine 调度相关的属性；
    - 设置新的 Goroutine 结构体的参数，包括栈指针、程序计数器并更新其状态到 `_Grunnable`
  - 将 Goroutine 加入处理器的运行队列；
    - `runtime.runqput` 函数会将新创建的 Goroutine 运行队列上，这既可能是全局的运行队列，也可能是处理器本地的运行队列
    - Go 语言中有两个运行队列，其中一个是处理器本地的运行队列，另一个是调度器持有的全局运行队列，**只有在本地运行队列没有剩余空间时才会使用全局队列。**



#### 调度循环

- `runtime.schedule`	===>	`ececute`	===>	`gogo`	===>	`runtime.schedule`
  - `runtime.schedule`：该函数一定会返回一个可执行的 Goroutine，如果当前不存在就会阻塞等待。
  - `runtime.execute`：执行获取的 Goroutine，做好准备工作后，它会通过 `runtime.gogo` 将 Goroutine 调度到当前线程上。
  - `runtime.gogo` 在不同处理器架构上的实现都不同，但是不同的实现也都大同小异
    - 利用了 Go 语言的调用惯例成功模拟它的调用过程
    - 最后在 `runtime.goexit0` 函数会重新调用 `runtime.schedule` 触发新的 Goroutine 调度，形成调度循环。

#### 触发调度

- 主动挂起 — `runtime.gopark` -> `runtime.park_m` 
  - 该函数会将当前 Goroutine 暂停，被暂停的任务不会放回运行队列
- 系统调用 — `runtime.exitsyscall` -> `runtime.exitsyscall0` 
- 协作式调度 — `runtime.Gosched`  -> `runtime.gosched_m`  -> `runtime.goschedImpl` 
  - `runtime.Gosched`  就是主动让出处理器，允许其他 Goroutine 运行。
- 系统监控 — `runtime.sysmon` -> `runtime.retake`  -> `runtime.preemptone` 



#### 线程管理

- 提供了 `runtime.LockOSThread` 和 `runtime.UnlockOSThread` 让我们有能力绑定 Goroutine 和线程完成一些比较特殊的操作。=====> CGO 使用。
- 线程生命周期
  - `runtime.startm`：启动线程来执行处理器 P，如果我们在该函数中没能从闲置列表中获取到线程 M 就会调用 `runtime.newm` 创建新的线程。
  - `runtime.newm`：创建线程。=====> `runtime.newosproc` ====> Linux: clone(最底端)
  - `exit/mstart`：销毁线程。



### 4.6 网络轮询器

- 处理 I/O 操作的关键组件，它使用了操作系统提供的 I/O 多路复用机制增强程序的并发处理能力。



#### 设计原理

- I/O 多路复用模型
- 