

### new And make


- what: allocates AND initializes
  - new 
    - (allocates memory).
  - make
    - (allocates and initializes an object of type slice, map, or chan (only)).
- why:

- how:
  - 区别从函数定义就可以看出来:```func new(Type) *Type``` AND ```func make(t Type, size ...IntegerType) Type```
    - 入参值
      - make 只能用来分配及初始化类型为 slice、map、chan 的数据(slice, map, or chan (only))。new 可以分配任意类型的数据；
    - 输出值
      - new 分配返回的是指针，即类型 *Type。make 返回引用，即 Type；
      - new 分配的空间被清零。make 分配空间后，会进行**初始化**；

[外部参考](http://c.biancheng.net/view/5722.html)
#### example

```go
package main
import (
 "fmt"
)
func main() {
 var i *int
 *i=10
 fmt.Println(*i)
}
```

这个例子会打印出什么？0还是10?。以上全错，运行的时候会painc，原因如下：
```
panic: runtime error: invalid memory address or nil pointer dereference
```

>从这个提示中可以看出，对于**引用类型的变量，我们不光要声明它**，还要为它**分配内容**空间，否则我们的值放在哪里去呢？这就是上面错误提示的原因。

对于值类型的声明不需要，是因为已经默认帮我们分配好了。

要分配内存，就引出来今天的new和make。

























