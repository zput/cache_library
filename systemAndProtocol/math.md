

## hight math

https://www.latexlive.com/##

#### function-limitation-continuous


- 极限的存在：根据定义，当$x->x_0$ 的时候$|f(x)-A|<\varepsilon$.
  - 这里是趋向一个常数A。;且这里不要把连续的一些概念混淆了，比如左右极限需相等啥的。

>sequence limit
>function limitation
>> $x->x_0$
>>> ![极限：x->x0](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210714195652.png)
>> $x->\infty$
>> ![极限：x->∞](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210714201724.png)


---

QA:
- 为什么 $|x-x_0|>0$ ,不能等于 $x==x_0$?
   - 定义中 $|x-x_0|>0$ 表示 $x!=x_0$ ,所以即使 x 在 $x_0$ 没有定义，f(x) 在 $x_0$ 点也可以有极限。因为我们研究极限是研究 $x->x_0$ 过程中 f(x) 的变化趋势，而与 f(x) 在 $x_0$ 处有无定义没有关系。
- $\delta$ 的取值与 $\varepsilon$ 有关，且 $\delta$ 不唯一。也就是说，任意给一个 $\varepsilon$ 就会有一个 $\delta$ 与之对应，换一个 $\varepsilon$ 就换一个 $\delta$ ，这一点和数列极限中的 N 是一样的。



---
![20210707170541](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210707170541.png)

- 保号性第一条：
  - 根据极限的定义，$|f(x)-A|<\varepsilon$ 它这个就是一个<号，不会有<=.
  - 然后根据一些 $f(x)=1/(1+x^2)$ 函数它的 $x->\infty$时候，极限为0，但是f(x)>0
  - 一些函数在 $x_0$ 处不需要有定义.



<img src="https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210707172054.png" width = "60%"  alt="效果图" align=right />

---

- 保号性的第三条：f(x)>=g(x), 且$\lim\limits_{x\rightarrow x_0}f(x)$, $\lim\limits_{x\rightarrow x_0}g(x)$存在, 则$\lim\limits_{x\rightarrow x_0}f(x)$ >= $\lim\limits_{x\rightarrow x_0}g(x)$
  - 使用F(x) = f(x)-g(x) >= 0, 则根据第二条，得到 $\lim\limits_{x\rightarrow x_0}$F(x)>=0, 推导得到f(x)>=g(x)

