


- 只要定义好循环变量的含义/范围 (定义的变量的范围，比如是左开右闭还是，左闭右开).
  - 含义，比如在下面的```帮一个数组【正数，负数，zero】排序```,它的```positive```范围就是```positive: [left, ...)```/含义就是代表最终正数的值都在这个```positive```范围里.
  - 循环变量：在运行过程中，里面的值可以变，但是含义固定不变.

- ```从快排里二路快排得到的经验```
  - 判断两个区间是否还需要继续靠近，如果是整个范围都由这两个区间占据才视为结束;
    - 那么如果[___)[___]这种情况还没有占据完，左边区间是左闭右开，还有一个元素没有判断，需要继续进入到循环中进行判断.
    - [___)|[___]这种情况已占据完，左边区间是左闭右开，与右边区间左闭在一个地方，所以是整个范围都由这两个区间占据了，不需要进入循环.
  - 当怀疑是否越界的时候，比如j-1 or i+1， 直接在脑海中想象这个区间范围,增大缩小，判断它在现有条件下，最大最小是多少,就可以消除误会了.


- ```从meger得到的经验```
  - 1. 不用总想着把变量的含义往区间的方面去靠，有的时候，他就是一个点，然后这个点有范围；比如下面的这个```tempFirst, tempSecond```.
    - 它就是一个点，然后有范围就行了[)，然后可以确定它的边界，它的边界是右开的，所以判断的时候要大于middle.
  - 归并的步骤：
    - 1. 新建一个同样大小的空间，这个空间的主要作用是比较[left, middle],[middle+1 right]按顺序来比较大小，看是哪一个值小；
    - 2. 外面需要一个循环来定义次数（就是旧的大小）
    - 3. 注意一下不要越界


---

- 步骤:
  - 1. 知道原理,先不要想步骤;
  - 2. 定义好每个循环变量的含义-->范围( []/(]/[)/() );
  - 3. 脑海测试(五种方法).



---


- quickSort
  - 三路快排：
    - https://play.golang.org/p/682cBXcpapw
  - 两路快排:


- fibo
  - https://play.golang.org/p/0mg7NVyOf9f



- 帮一个数组【正数，负数，zero】排序.
  - https://play.golang.org/p/mZKhTnYVyQP





## 附录备份

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










