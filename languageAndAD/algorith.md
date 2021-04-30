


<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

  - [数据结构](#数据结构)
    - [堆](#堆)
- [算法](#算法)
  - [算法思想](#算法思想)
  - [排序](#排序)
    - [方法论](#方法论)
    - [快排](#快排)
    - [merge排序](#merge排序)
    - [堆排序](#堆排序)
  - [实操](#实操)
- [附录备份](#附录备份)
  - [Test](#test)
  - [两个go程轮流打印一个切片](#两个go程轮流打印一个切片)

<!-- /code_chunk_output -->




## 数据结构
### 堆

- 堆
  - what:
    - 堆是什么?
      - 在最大堆中，父节点的值比每一个子节点的值都要大。
      - 在最小堆中，父节点的值比每一个子节点的值都要小。这就是所谓的“堆属性”，
      - 并且这个属性对堆中的每一个节点都成立。
  - why:
    - 与二叉搜索树区别
      - 定义不同：二叉搜索树（某个数比它的左子树大，比右子树小）
      - 内存占用。**普通树占用的内存空间比它们存储的数据要多**. 你必须为节点对象以及左/右子节点指针分配内存。堆仅仅使用一个数据来存储数组，且不使用指针。
      - 平衡:
        - 二叉搜索树必须是“平衡”的情况下，其大部分操作的复杂度才能达到O(log n)。你可以按任意顺序位置插入/删除数据，或者使用 AVL 树或者红黑树，
        - 但是在堆中实际上不需要整棵树都是有序的。我们只需要满足堆属性即可，所以在堆中平衡不是问题。因为堆中数据的组织方式可以保证O(log n) 的性能。
      - 搜索: 在二叉树中搜索会很快，但是在堆中搜索会很慢。在堆中搜索不是第一优先级，
        - 因为使用堆的目的是将最大（或者最小）的节点放在最前面，从而快速的进行相关插入、删除操作。
  - how:
    - 目的
      - 使用堆的目的是将最大（或者最小）的节点放在最前面，从而快速的进行相关插入、删除操作   <--->   实现优先队列


- 深度是从**上往下数的**，高度是**从下往上数的**，深度和高度都涉及到节点的**层数**
  - (经过学习发现，深度、高度概念在不同的教材中有不同的定义，主要看高度深度的初值为几，有的为0，有的为1).
  - **层数**都是人类的从1开始的.
    - 定义一（初值为0）：
      - 节点的深度是根节点到这个节点所经历的边的个数
      - 节点的高度是该节点到叶子节点的最长路径（边数）
      - 树的高度等于根节点的高度
      - ![tree_hight_deep](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210224165525.png)



```
i: 当前的在第几层；
k: 这个堆的最高有几层.

     0        --------- 第1层
    / \
   1   2      --------- 第2层
   当在第一层： i is 1; k is 2.
   2^( i - 1 )  *  ( k - i ) ===> 2^(1-1)*(2-1) ===> 1

   当在第二层： i is 2; k is 2.
   2^( i - 1 )  *  ( k - i ) ===> 2^(2-1)*(2-2) ===> 0

  如果是满二叉树，节点的个数:                 深度为n，最多有2ⁿ-1个结点
```


- ```s = 2^( i - 1 )  *  ( k - i )；```
  - 其中 i 表示第几层，
  - ```2^( i - 1)``` 表示该层上有多少个元素
  - ( k - i) 表示子树上要比较的次数

- 因为叶子层不用交换，所以i===>[k-1, 1]===>从k-1(倒数第二层)开始到1(第一层)；
  - ```S = 2^(k-2) * 1 + 2^(k-3)*2.....+2*(k-2)+2^(0)*(k-1)```
    - ```2^(k-1-1)*(k-(k-1))```  ====>  ```2^(1-1)*(k-1)```  
  - 等式左右乘上2，然后和原来的等式相减，就变成了：```S = 2^(k - 1) + 2^(k - 2) + 2^(k - 3) ..... + 2 - (k-1)```
    - 除最后一项外，就是一个等比数列了，直接用求和公式：![An+1/An=q](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210224185720.png)
  - ```S = (2*(1-2^k))/(1-2) ===> 2^k -k -1```
        又因为k为完全二叉树的深度，所以 (2^k) <=  n < (2^k  -1 )，总之可以认为：k = logn （实际计算得到应该是 log(n+1) < k <= logn ）;

        综上所述得到：S = n - longn -1，所以时间复杂度为：O(n)

这个从深度/高度如果得到二叉树节点的个数。
https://www.cnblogs.com/GHzcx/p/9635161.html
https://blog.csdn.net/weixin_44324174/article/details/107334890
https://www.zhihu.com/question/20729324



# 算法

## 算法思想

---

- 步骤:
  - 1. 知道原理,先别想步骤;
  - 2. 定义好每个循环变量的含义-->范围( []/(]/[)/() );
  - 3. 脑海测试(五种方法).



## 排序

### 方法论

- 只要定义好循环变量的**含义/范围** (定义的变量```的范围，比如是左开右闭还是，左闭右开).```
  - 含义: 比如下面的示例(```帮一个数组【正数，负数，zero】排序```),
    - 它的```positive```范围就是```positive: [left, ...)```
    - 含义就是代表最终正数的值都在这个```positive```范围里.
  - 循环变量：在运行过程中，里面的值可以变，但是含义固定不变.

### 快排

- ```从快排里二路快排得到的经验```
  - 判断两个区间是否还需要继续靠近，如果是整个范围都由这两个区间占据才视为结束;
    - 那么如果[___)[___]这种情况还没有占据完，左边区间是左闭右开，还有一个元素没有判断，需要继续进入到循环中进行判断.
    - [___)|[___]这种情况已占据完，左边区间是左闭右开，与右边区间左闭在一个地方，所以是整个范围都由这两个区间占据了，不需要进入循环.
  - 当怀疑是否越界的时候，比如j-1 or i+1， 直接在脑海中想象这个区间范围,增大缩小，判断它在现有条件下，最大最小是多少,就可以消除怀疑了.
    ```go
    func _quickSort(arr []int, left, right int) {
    	if left >= right {
    		return
    	}
    	partition := makePartition(arr, left, right)
    	// partition := makePartition3(arr, left, right)
    	_quickSort(arr, left, partition-1)
    	_quickSort(arr, partition+1, right) // 这里不要写成_quickSort(arr, partition-1, right)
    }
    ```
    - 三路快排的时候，循环次数，不是固定的从[left+1, right],而是[left+1, j)
    ```go
    func makePartition3(arr []int, l, r int) (int, int) {
    	var partition = arr[l]
    	// [l+1, ...)
    	// [r+1, ...)
    	i, j := l+1, r+1
    	for k := l + 1; k < j; k++ {   // -----------------------here
    		if arr[k] < partition {
    			arr[i], arr[k] = arr[k], arr[i]
    			i++
    		} else if arr[k] > partition {
    			arr[j-1], arr[k] = arr[k], arr[j-1]
    			j--
    			k--
    		}
    	}
    	arr[l], arr[i-1] = arr[i-1], arr[l]
    	return i - 2, j
    }
    ```

### merge排序

- ```从meger得到的经验```
  - 1. 不用总想着把变量的含义往区间的方面去靠，有的时候，他就是一个点，然后这个点有范围；比如下面的这个```tempFirst, tempSecond```.
    - 它就是一个点，然后有范围就行了[)，然后可以确定它的边界，它的边界是右开的，所以判断的时候要大于middle.
  - 归并的步骤：
    - 1. 新建一个同样大小的空间，这个空间的主要作用是比较[left, middle],[middle+1 right]按顺序来比较大小，看是哪一个值小；
    - 2. 外面需要一个循环来定义次数（就是旧的大小）
    - 3. 注意一下不要越界


### 堆排序

- ```从heap sort得到的经验```
  - 上面先做成一个堆的结构--->
  - 下面的是每次把root节点拿出来，然后heapDown，选出新的最大值
    - 当比较左右子树的时候，使用类似指针的东西，最后才用交换（记住指针）
    - when son's node is known:  
      - parent's node = (son's node -1)/2
    - when parent's node is known:
      - left child's node = parent's node * 2 + 1
      - right child's node = parent's node * 2 + 2
    ```
       like:
        0
       / \
      1   2
     / \ / \
    3  4 5  6
    通过这个例子来想这些公式，不需要硬记住。
    ```
    ```
     _ _ _ _ _ _
    |_ _ _ _ _ _|
     0 1 2 3 4 5
              len(array)-1 is 5
              in order to find 5's parent node:   (len(array)-1-1)/2
    ```



## 实操

- quickSort
  - 三路快排：
    - https://play.golang.org/p/682cBXcpapw
  - 两路快排:

- fibo
  - https://play.golang.org/p/0mg7NVyOf9f

- 帮一个数组【正数，负数，zero】排序.
  - https://play.golang.org/p/mZKhTnYVyQP





# 附录备份

- quickSort
  - 快排
```go
package main

import "fmt"

func quickSort(array []int){
	if len(array)<=1{
		return 
	}
	_quickSort(array, 0, len(array)-1)
	return
}
func _quickSort(array []int, left, right int){
	if left >= right{
		return
	}

	p1, p2 := __quickSort(array, left, right)
	_quickSort(array, left, p1)
	_quickSort(array, p2, right)
	return
}

func __quickSort(array []int, left, right int)(int, int){
	var partition = array[left]
	var i=left+1
	var k = right+1
	for j:=left+1; j<k;j++{
		if array[j]<partition{
			array[i], array[j] = array[j], array[i]
			i++
		}else if array[j] > partition{
			array[k-1], array[j] = array[j], array[k-1]
			k--
			j--
		}
	}
	array[left], array[i-1] = array[i-1], array[left]
	return i-2, k
}


func main(){
	var array = []int{1,2,5,7,199,3,4,6,4,4,7}
	quickSort(array)
	fmt.Println(array)
}
```


- 两路快排:

```go
package main

import "fmt"

func main(){
	var arr = []int{1, 4, 8,-1, 2999, 0, 0, 0, 8 , 10}
	quickSort(arr)
	fmt.Println(arr)
}

func quickSort(arr []int){
	if len(arr)<= 1{
		return
	}
	_quickSort(arr, 0, len(arr)-1)
}

func _quickSort(arr []int, left, right int){
	if left >= right{
		return
	}

	partition := _partitionOK(arr, left, right)
	_quickSort(arr, left, partition-1)
	_quickSort(arr, partition+1, right)
}

func _partition(arr []int, left, right int)(int){
	// i : [left+1, ...)
	// j : [...,right+1)
	i, j := left+1, right+1
	partitionValue := arr[left]
	for ;i<j; {//--------------------------------------------here
		for ;i<j&&arr[i]<=partitionValue; i++{
		}
		for ;i<j&&arr[j-1]>partitionValue; j--{
		}
		// swap
		if i != j{
			//arr[i], arr[j] = arr[j], arr[i]
			arr[i], arr[j-1] = arr[j-1], arr[i]
		}
	}
	arr[left], arr[i-1] = arr[i-1], arr[left]
	return i-1
}

//判断两个区间是否还需要继续靠近，如果是整个范围都由这两个区间占据才视为结束;
  // 那么如果[___)[___]这种情况还没有占据完，左边区间是左闭右开，还有一个元素没有判断，需要继续进入到循环中进行判断.
  //       [___)|[___]这种情况已占据完，左边区间是左闭右开，与右边区间左闭在一个地方，所以是整个范围都由这两个区间占据了，不需要进入循环。
// 当怀疑是否越界的时候，比如j-1 or i+1， 直接在脑海中想象这个区间范围,增大缩小，判断它在现有条件下，最大最小是多少,就可以消除误会了。

//-------------------------------------------------
// 在这个例子中，标准的如下面所示，不能够少了
// if i>=j{
//     break
// }

func _partitionOK(arr []int, left, right int)(int){
	// i : [left+1, ...)
	// j : [...,right+1)
	i, j := left+1, right+1
	partitionValue := arr[left]
	for ;i<j; {//--------------------------------------------here
		for ;i<j&&arr[i]<=partitionValue; i++{
		}
		for ;i<j&&arr[j-1]>partitionValue; j--{
		}
		if i>=j{//--------------------------------------------here
			break//--------------------------------------------here
		}//--------------------------------------------here
		// swap
		arr[i], arr[j-1] = arr[j-1], arr[i]//--------------------------------------------here
	}
	arr[left], arr[i-1] = arr[i-1], arr[left]
	return i-1
}
```

- mege sort

```go
package main

import "fmt"

func main(){
	var arr = []int{1, 5, 7, -1, 0, 10000, 5, 9, 5, 6, 8, 7}
	mergeSort(arr)
	fmt.Println(arr)
}

func mergeSort(arr []int){
	if len(arr)<=1{
		return
	}
	_mergeSort(arr, 0, len(arr)-1)
}

func _mergeSort(arr []int, left, right int){
	if left >= right{
		return
	}
	middle := left+(right-left)/2
	_mergeSort(arr, left, middle)
	_mergeSort(arr, middle+1, right)
	merge(arr, left, middle, right)
}

/*
- 1. 不用总想着把变量的含义往区间的方面去靠，有的时候，他就是一个点，然后这个点有范围；比如下面的这个```tempFirst, tempSecond```.
  - 它就是一个点，然后有范围就行了[)，然后可以确定它的边界，它的边界是右开的，所以判断的时候要大于middle.

归并的步骤：
1. 新建一个同样大小的空间，这个空间的主要作用是比较[left, middle],[middle+1 right]按顺序来比较大小，看是哪一个值小；
2. 外面需要一个循环来定义次数（就是旧的大小）
3. 注意一下不要越界
*/

func merge(arr []int, l, m, r int){
	var temp = make([]int,r-l+1)
	for i:=0; i<len(temp); i++{
		temp[i] = arr[l+i]
	}
	//middleError := (len(arr)-1)/2      --------------------------失误
	middle := m-l
	//fmt.Println(middle, "-----", middleError)    --------------------------失误

	tempFirst, tempSecond:=0, middle+1
	for i:=l; i<=r; i++{
		if tempFirst > middle{
			arr[i] = temp[tempSecond]
			tempSecond++
		}else if tempSecond >= len(temp){
			arr[i] = temp[tempFirst]
			tempFirst++
		}else{
			if temp[tempFirst]<temp[tempSecond]{
				arr[i] = temp[tempFirst]
				tempFirst++
			}else{
				arr[i] = temp[tempSecond]
				tempSecond++
			}
		}
	}
}
```


- 帮一个数组【正数，负数，zero】排序

```go
package main

import (
	"fmt"
)

/*
 ________________________
|_____|____________|_____| invalid space |
left                 right         right+1
positive: [left, ...)
       negative: [..., right+1)

i很好理解就是它们的需要遍历的范围[left, negative)

*/

// positive, negative, zero
func specialSort(data []int, left, right int){
	// [left, ...)
	var positive int = left
	// [..., right+1)
	var zero int = right+1
	for i:=left; i<zero; i++{
		if data[i] > 0{
			data[i], data[positive] = data[positive], data[i]
			positive++
		}else if data[i] == 0{
			data[i], data[zero-1] = data[zero-1], data[i]
			zero--
			i--
		}
	}
	return
}

func main(){
	var dataArray = [...]int{1, 10, 6, 5, 0, 0, -1, -4, -3, 0, 6, -1, 0}
	fmt.Println(len(dataArray))


	var data = []int{1, 10, 6, 5, 0, 0, -1, -4, -3, 0, 6, -1, 0}
	// TODO data array ---> len(array)
	specialSort(data, 0, len(data)-1)
	fmt.Println(data)
}
```



- heap sort

```go
package main

import "fmt"

func heapSort(array []int) {
	if len(array) <= 1 {
		return
	}
	/*
 _ _ _ _ _ _
|_ _ _ _ _ _|
 0 1 2 3 4 5
          len(array)-1 is 5
	      in order to find 5's parent node:
	      (len(array)-1-1)/2
	like:
	 0
	/ |
  1   2
 / \ / \
3  4 5  6
- when son's node is known: 	 parent's node = (son's node -1)/2
- when parent's node is known:
  - left child's node = parent's node * 2 + 1
  - right child's node = parent's node * 2 + 2
通过这个例子来想这些公式，不需要硬记住。

	当比较左右子树的时候，使用类似指针的东西，最后才用交换（记住指针）

	*/
	for i := (len(array)-1-1) / 2; i >= 0; i-- {
		heapDown(array, len(array), i)
	}
	// 上面已经做成了一个堆的结构--->堆是什么?(比它的左右子树都大)   <-------->    二叉搜索树（某个数比它的左子树大，比右子树小）
	// 下面的是每次把root节点拿出来，然后heapDown，选出新的最大值
	length := len(array)
	for ; length > 1; length-- {
		array[0], array[length-1] = array[length-1], array[0]
		heapDown(array, length-1, 0)
	}
	//for j:=0; j<len(array)/2;j++{
	//	array[j], array[len(array)-1-j] = array[len(array)-1-j], array[j]
	//}
}

func heapDown(array []int, size int, index int) { // 这个跟下面那个递归是一样的,只是把递归的那个开始判断,放到for循环条件判断里面去了.
	for index*2+1 < size {
		var tempIdx = index
		if array[index] < array[index*2+1] {
			tempIdx = index*2 + 1                 // ----------------------------------- 当比较左右子树的时候，使用类似指针的东西，最后才用交换（记住指针）
		}
		if (index*2+2 < size) && array[tempIdx] < array[index*2+2] {
			tempIdx = index*2 + 2                 // ----------------------------------- 当比较左右子树的时候，使用类似指针的东西，最后才用交换（记住指针）
		}
		if tempIdx != index {
			array[index], array[tempIdx] = array[tempIdx], array[index]
			index = tempIdx
		} else {
			return
		}
	}
}

func heapDown2(array []int, size int, index int) {
	if 2*index+1 >= size {
		// complete heap down
		return
	}
	var j = index
	if array[2*index+1] > array[j] {
		j = 2*index + 1
	}
	if 2*index+2 < size && array[2*index+2] > array[j] {
		j = 2*index + 2
	}
	if j != index {
		array[j], array[index] = array[index], array[j]
		heapDown(array, size, j)
	}


}

func main() {

	var array = []int{1, 5, 7, -1, 0, 10000, 5, 9, 5, 6, 8, 7}
	//var array = []int{1, 2, 3, 5, 1, 4, 3, 11, 4, 4, 111, 8, 3}
	heapSort(array)
	fmt.Println(array)
}
```


## Test


```go

func main(){
	var array = []int{1,2,5,7,199,3,4,6,4,4,7}
    quickSort(array)
    heapSort(array)
	fmt.Println(array)
}


// 先二路，后三路

func quickSort(arr []int){
    if len(arr) <= 1{
        return
    }
    _quickSort(arr, 0, len(arr)-1)
    //_quickSort3(arr, 0, len(arr)-1)
}

func _quickSort(arr []int, left, right int){
    if left >= right{
        return
    }
    partition := makePartition(arr, left, right)
    // partition := makePartition3(arr, left, right)
    _quickSort(arr, left, partition-1)
    _quickSort(arr, partition-1, right)
}
func makePartition(arr []int, l, r int)int{

    var partition = arr[l]
    // [l+1, ...)
    // [r+1, ...)
    i, j := l+1, r+1
    for ;i<j; {
        for ;i<j&&arr[i]<partition; {
            i++
        }
        for ; i<j && arr[j-1]>=partition; {
                j--
        }
        if i==j{
            break
        }
        arr[i], arr[j-1] = arr[j-1], arr[i]
        i++
        j--
    }
    arr[l], arr[i-1] = arr[i-1], arr[l]
    return i-1
}

// ------------------------------------------------------------------

func _quickSort3(arr []int, left, right int){
    if left >= right{
        return
    }
    partitionLeft, partitionRight := makePartition3(arr, left, right)
    _quickSort(arr, left, partitionLeft)
    _quickSort(arr, partitionRight, right)
}
func makePartition(arr []int, l, r int)(int, int){
    var partition = arr[l]
    // [l+1, ...)
    // [r+1, ...)
    i, j := l+1, r+1
    for k:= l+1; k<=r; k++{
        if arr[k] < partition{
            arr[i], arr[k] = arr[k], arr[i]
            i++
        }else if arr[k] > partition{
            arr[j-1], arr[k] = arr[k], arr[j-1]
            j--
            k--
        }
    }
    arr[l], arr[i-1] = arr[i-1], arr[l]
    return i-2, j
}

```




## 两个go程轮流打印一个切片

```go
package main

import (
	"fmt"
	"sync"
)

func main(){
	//var array = []int{1, 4, 6, 4, 2, 2, 3, 3, 1, 10, 10001, 101, 564, 2, 6}
	//var array = []int{1, 2, 3, 4, 5, 6, 7, 8}
	//var array = []int{1, 2}
	var array = []int{1, 2, 3}
	printArray(array)
}

func printArray(array []int){
/*	if len(array) == 0{
		return
	}*/

	var wg sync.WaitGroup
	wg.Add(2)

	var c1 = make(chan int, 1)
	var c2 = make(chan int, 1)

	c1 <- 0

	var index int = 1

	go func(){
		for; index< len(array);{
			v := <-c1
			fmt.Println(array[v])
			index++
			c2<-v+1
		}
		wg.Done()
	}()

	go func(){
		for; index< len(array);{
			v2 := <-c2
			fmt.Println("-------", v2, "-----", index)
			fmt.Println(array[v2])
			c1<-v2+1
			index++
		}
		wg.Done()
	}()

	wg.Wait()
}
```





