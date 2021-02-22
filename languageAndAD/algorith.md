


- 只要定义好循环变量的含义/范围 (定义的变量的范围，比如是左开右闭还是，左闭右开).
  - 含义，比如在下面的```帮一个数组【正数，负数，zero】排序```,它的```positive```范围就是```positive: [left, ...)```/含义就是代表最终正数的值都在这个```positive```范围里.
  - 循环变量：在运行过程中，里面的值可以变，但是含义固定不变.



- quickSort
  - https://play.golang.org/p/682cBXcpapw



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










