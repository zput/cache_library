


## test函数的种类 

- ```func TestXxx(t *testing.T) { ... }```
  - 注意下这个Xxx需要大写
- ```func BenchmarkXxx(b *testing.B) { ... }```
  - 注意下这个Xxx需要大写
- ```ExampleXxx```
  - ```prints output to os.Stdout```
  - "Output:"
  - "Unordered output:"



- ```go help testfunc```

```sh
The 'go test' command expects to find test, benchmark, and example functions
in the "*_test.go" files corresponding to the package under test.

A test function is one named TestXxx (where Xxx does not start with a
lower case letter) and should have the signature,

        func TestXxx(t *testing.T) { ... }

A benchmark function is one named BenchmarkXxx and should have the signature,

        func BenchmarkXxx(b *testing.B) { ... }

An example function is similar to a test function but, instead of using
*testing.T to report success or failure, prints output to os.Stdout.
If the last comment in the function starts with "Output:" then the output
is compared exactly against the comment (see examples below). If the last
comment begins with "Unordered output:" then the output is compared to the
comment, however the order of the lines is ignored. An example with no such
comment is compiled but not executed. An example with no text after
"Output:" is compiled, executed, and expected to produce no output.

Godoc displays the body of ExampleXxx to demonstrate the use
of the function, constant, or variable Xxx. An example of a method M with
receiver type T or *T is named ExampleT_M. There may be multiple examples
for a given function, constant, or variable, distinguished by a trailing _xxx,
where xxx is a suffix not beginning with an upper case letter.

Here is an example of an example:

        func ExamplePrintln() {
                Println("The output of\nthis example.")
                // Output: The output of
                // this example.
        }

Here is another example where the ordering of the output is ignored:

        func ExamplePerm() {
                for _, value := range Perm(4) {
                        fmt.Println(value)
                }

                // Unordered output: 4
                // 2
                // 1
                // 3
                // 0
        }

The entire test file is presented as the example when it contains a single
example function, at least one other function, type, variable, or constant
declaration, and no test or benchmark functions.

See the documentation of the testing package for more information.
```


## go test工具

> 测试结果输出结果：```test status ('ok' or 'FAIL')```   ```package name```   ```elapsed time```

- ```go test [build/test flags] [packages] [build/test flags & test binary flags]```
  - 当前目录模式: 如果当前没有go.mod，那么会自动生成[一个临时_path:_/tmp/hello][1]
    - ```go test```
    - ```go test -v```
  - 包列表模式
    - ```go test <module name>```
    - ```go test .```
    - ```go test ./...```
      - 打印更详细的信息
        - ```go test -v .```
        - ```go test -bench .```





```
[root@iZf8z14idfp0rwhiicngwqZ hello]# tree
.
├── main.go
└── main_test.go

0 directories, 2 files
[root@iZf8z14idfp0rwhiicngwqZ hello]# go test
PASS
ok  	_/tmp/hello	1.001s

// --------------------------------------------------------------------------- //

[root@iZf8z14idfp0rwhiicngwqZ hello]# tree
.
├── go.mod
├── main.go
└── main_test.go

0 directories, 3 files
[root@iZf8z14idfp0rwhiicngwqZ hello]# cat go.mod
module example.com/hello

go 1.13

// --------------------------------------------------------------------------- //

[root@iZf8z14idfp0rwhiicngwqZ hello]# go test example.com/hello/hello
can't load package: package example.com/hello/hello: module example.com/hello/hello: Get https://proxy.golang.org/example.com/hello/hello/@v/list: dial tcp 172.217.160.113:443: i/o timeout
[root@iZf8z14idfp0rwhiicngwqZ hello]# go test example.com/hello
ok  	example.com/hello	1.002s
[root@iZf8z14idfp0rwhiicngwqZ hello]# go test hello
can't load package: package hello: malformed module path "hello": missing dot in first path element
[root@iZf8z14idfp0rwhiicngwqZ hello]# go test .
ok  	example.com/hello	(cached)
[root@iZf8z14idfp0rwhiicngwqZ hello]# go test ./...
ok  	example.com/hello	(cached)

// --------------------------------------------------------------------------- //

[root@iZf8z14idfp0rwhiicngwqZ hello]# go test -v .
=== RUN   TestHello
--- PASS: TestHello (1.00s)
PASS
ok  	example.com/hello	1.001s
[root@iZf8z14idfp0rwhiicngwqZ hello]# go test -bench .
PASS
ok  	example.com/hello	1.001s
```




- ```go help test```

```sh
usage: go test [build/test flags] [packages] [build/test flags & test binary flags]

'Go test' automates testing the packages named by the import paths.
It prints a summary of the test results in the format:

        ok   archive/tar   0.011s
        FAIL archive/zip   0.022s
        ok   compress/gzip 0.033s
        ...

followed by detailed output for each failed package.

'Go test' recompiles each package along with any files with names matching
the file pattern "*_test.go".
These additional files can contain test functions, benchmark functions, and
example functions. See 'go help testfunc' for more.
Each listed package causes the execution of a separate test binary.
Files whose names begin with "_" (including "_test.go") or "." are ignored.

Test files that declare a package with the suffix "_test" will be compiled as a
separate package, and then linked and run with the main test binary.

The go tool will ignore a directory named "testdata", making it available
to hold ancillary data needed by the tests.

As part of building a test binary, go test runs go vet on the package
and its test source files to identify significant problems. If go vet
finds any problems, go test reports those and does not run the test
binary. Only a high-confidence subset of the default go vet checks are
used. That subset is: 'atomic', 'bool', 'buildtags', 'nilfunc', and
'printf'. You can see the documentation for these and other vet tests
via "go doc cmd/vet". To disable the running of go vet, use the
-vet=off flag.

All test output and summary lines are printed to the go command's
standard output, even if the test printed them to its own standard
error. (The go command's standard error is reserved for printing
errors building the tests.)

Go test runs in two different modes:

The first, called local directory mode, occurs when go test is
invoked with no package arguments (for example, 'go test' or 'go
test -v'). In this mode, go test compiles the package sources and
tests found in the current directory and then runs the resulting
test binary. In this mode, caching (discussed below) is disabled.
After the package test finishes, go test prints a summary line
showing the test status ('ok' or 'FAIL'), package name, and elapsed
time.

The second, called package list mode, occurs when go test is invoked
with explicit package arguments (for example 'go test math', 'go
test ./...', and even 'go test .'). In this mode, go test compiles
and tests each of the packages listed on the command line. If a
package test passes, go test prints only the final 'ok' summary
line. If a package test fails, go test prints the full test output.
If invoked with the -bench or -v flag, go test prints the full
output even for passing package tests, in order to display the
requested benchmark results or verbose logging. After the package
tests for all of the listed packages finish, and their output is
printed, go test prints a final 'FAIL' status if any package test
has failed.

In package list mode only, go test caches successful package test
results to avoid unnecessary repeated running of tests. When the
result of a test can be recovered from the cache, go test will
redisplay the previous output instead of running the test binary
again. When this happens, go test prints '(cached)' in place of the
elapsed time in the summary line.

The rule for a match in the cache is that the run involves the same
test binary and the flags on the command line come entirely from a
restricted set of 'cacheable' test flags, defined as -cpu, -list,
-parallel, -run, -short, and -v. If a run of go test has any test
or non-test flags outside this set, the result is not cached. To
disable test caching, use any test flag or argument other than the
cacheable flags. The idiomatic way to disable test caching explicitly
is to use -count=1. Tests that open files within the package's source
root (usually $GOPATH) or that consult environment variables only
match future runs in which the files and environment variables are unchanged.
A cached test result is treated as executing in no time at all,
so a successful package test result will be cached and reused
regardless of -timeout setting.

In addition to the build flags, the flags handled by 'go test' itself are:

        -args
            Pass the remainder of the command line (everything after -args)
            to the test binary, uninterpreted and unchanged.
            Because this flag consumes the remainder of the command line,
            the package list (if present) must appear before this flag.

        -c
            Compile the test binary to pkg.test but do not run it
            (where pkg is the last element of the package's import path).
            The file name can be changed with the -o flag.

        -exec xprog
            Run the test binary using xprog. The behavior is the same as
            in 'go run'. See 'go help run' for details.

        -i
            Install packages that are dependencies of the test.
            Do not run the test.

        -json
            Convert test output to JSON suitable for automated processing.
            See 'go doc test2json' for the encoding details.

        -o file
            Compile the test binary to the named file.
            The test still runs (unless -c or -i is specified).

The test binary also accepts flags that control execution of the test; these
flags are also accessible by 'go test'. See 'go help testflag' for details.

For more about build flags, see 'go help build'.
For more about specifying packages, see 'go help packages'.

See also: go build, go vet.
```

### 运行文件的单个测试单元

- 运行某个test文件的某个test例子。
  - ```go test -v -run="Regrex$" main_test.go```

```go
// tree

.
├── go.mod
├── main.go
└── main_test.go

0 directories, 3 files

// cat main_test.go
package hello

import (
	"testing"
	"time"
)

func TestHello(t *testing.T) {
    want := "Hello, world."
    if got := Hello(); got != want {
        t.Errorf("Hello() = %q, want %q", got, want)
    }
    time.Sleep(time.Second)

}

func TestRegrex(t *testing.T) {
    want := "Hello, world. --- regrex"
    if got := Hello(); got != want {
        t.Errorf("Hello() = %q, want %q", got, want)
    }
    time.Sleep(time.Second)

}

// cat go.mod
module example.com/hello

go 1.13
```

```sh
[root@iZf8z14idfp0rwhiicngwqZ hello]# go test -v -run="Regrex$" main_test.go
# command-line-arguments [command-line-arguments.test]
./main_test.go:10:15: undefined: Hello
./main_test.go:19:15: undefined: Hello
FAIL	command-line-arguments [build failed]
FAIL



[root@iZf8z14idfp0rwhiicngwqZ hello]# go test -v -run="Regrex$" main_test.go main.go
=== RUN   TestRegrex
--- FAIL: TestRegrex (1.00s)
    main_test.go:20: Hello() = "Hello, world.", want "Hello, world. --- regrex"
FAIL
FAIL	command-line-arguments	1.002s
FAIL



[root@iZf8z14idfp0rwhiicngwqZ hello]# go test -v -run="Regrex$" .
=== RUN   TestRegrex
--- FAIL: TestRegrex (1.00s)
    main_test.go:20: Hello() = "Hello, world.", want "Hello, world. --- regrex"
FAIL
FAIL	example.com/hello	1.001s
FAIL



[root@iZf8z14idfp0rwhiicngwqZ hello]# go test -v -run="Regrex$" example.com/hello
=== RUN   TestRegrex
--- FAIL: TestRegrex (1.00s)
    main_test.go:20: Hello() = "Hello, world.", want "Hello, world. --- regrex"
FAIL
FAIL	example.com/hello	1.001s
FAIL
```





### test文件生成二进制文件


- ```go test [build/test flags] [packages] [build/test flags & test binary flags]```
  - ```test binary flags```


```go
// ls  ------------------------------------------------------------
bench.test   go.mod       main.go      main_test.go

// cat go.mod  ----------------------module is bench---------------
module bench

go 1.13

// cat main.go   --------------------------------------------------
package main

func fib(n int) int {
        if n == 0 || n == 1 {
                return n
        }
        return fib(n-2) + fib(n-1)
}

// cat main_test.go -----------------------------------------------
package main

import "testing"

func BenchmarkFib(b *testing.B) {
        for n := 0; n < b.N; n++ {
                fib(30) // run fib(30) b.N times
        }
}
```


```sh

go test bench -c -o test_binary -bench .

go test -bench="Fib$" . 
goos: darwin
goarch: amd64
pkg: bench
BenchmarkFib-4               224           5329575 ns/op
PASS
ok      bench   1.739s

./bench.test -test.bench="Fib$"
goos: darwin
goarch: amd64
pkg: bench
BenchmarkFib-4               223           5320568 ns/op
PASS
```



### benchmark

- go test 命令默认不运行 benchmark 用例的，如果我们想运行 benchmark 用例，则需要加上 -bench 参数。
  - ```-bench```参数支持传入一个正则表达式，匹配到的用例才会得到执行。

- ```-cpu```
  - GOMAXPROCS，默认等于 CPU 核数。可以通过 -cpu 参数改变 GOMAXPROCS，-cpu 支持传入一个列表作为参数

- 提升测试准确度的一个重要手段就是增加测试的次数。
  - 我们可以使用```-benchtime 和 -count```两个参数达到这个目的。
    - ```-benchtime```的默认时间是 1s，那么我们可以使用```-benchtime```指定为 5s。
    - 还可以是具体的次数。```-benchtime=30x`(执行30次)
  - ```-count``` 参数可以用来设置 benchmark 的轮数。例如，进行 3 轮 benchmark。

- ```-benchmem```参数可以度量内存分配的次数。

#### 基准测试的测试次数

> b *testing.B，有个属性 b.N 表示这个用例需要运行的次数。b.N 对于每个用例都是不一样的。
>> 那这个值是如何决定的呢？b.N 从 1 开始，如果该用例能够在 1s 内完成，b.N 的值便会增加，再次执行。b.N 的值大概以 1, 2, 3, 5, 10, 20, 30, 50, 100 这样的序列递增，越到后面，增加得越快。


#### 精确


- ResetTimer

```go
func BenchmarkFib(b *testing.B) {
	time.Sleep(time.Second * 3) // 模拟耗时准备任务
	b.ResetTimer() // 重置定时器
	for n := 0; n < b.N; n++ {
		fib(30) // run fib(30) b.N times
	}
}
```



- StopTimer & StartTimer

```go
// sort_test.go
package main

import (
	"math/rand"
	"testing"
	"time"
)

func generateWithCap(n int) []int {
	rand.Seed(time.Now().UnixNano())
	nums := make([]int, 0, n)
	for i := 0; i < n; i++ {
		nums = append(nums, rand.Int())
	}
	return nums
}

func bubbleSort(nums []int) {
	for i := 0; i < len(nums); i++ {
		for j := 1; j < len(nums)-i; j++ {
			if nums[j] < nums[j-1] {
				nums[j], nums[j-1] = nums[j-1], nums[j]
			}
		}
	}
}

func BenchmarkBubbleSort(b *testing.B) {
	for n := 0; n < b.N; n++ {
		b.StopTimer()
		nums := generateWithCap(10000)
		b.StartTimer()
		bubbleSort(nums)
	}
}
```



## 附录

[1]:https://blog.golang.org/using-go-modules "https://blog.golang.org/using-go-modules"



