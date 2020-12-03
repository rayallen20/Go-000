学习笔记

# Go 语言实践 - error

## PART0. Introduction

3层架构下的日志打印问题:

![Image text](https://github.com/rayallen20/Go-000/blob/main/Week02/img/%E4%B8%89%E5%B1%82%E6%9E%B6%E6%9E%84%E4%B8%8B%E7%9A%84%E6%97%A5%E5%BF%97%E6%89%93%E5%8D%B0%E9%97%AE%E9%A2%98.jpg)

**error的核心:wrap error,把error包装起来,像一个堆栈一样往上层抛,最后只在1个地方记录日志,这件事是本节课程的核心**

## PART1. Error vs Exception

### 1.1 标准库error在设计时想要规避的问题

```go
// The error built-in interface type is the conventional interface for
// representing an error condition, with the nil value representing no error.
// 内置的error接口类型是常见的用于表示一个错误情况的接口,若值为nil则意味着没有错误.
type error interface {
	Error() string
}
```

该接口定义了一个`Error()`方法.

我们经常使用`errors.New()`来返回一个`error`对象

```go
// errorString is a trivial implementation of error.
// errorString类型是error的一个细节的实现
// TODO:trivial翻译为"细节的"或许不太准确,回头问问别人的
type errorString struct {
	s string
}

func (e *errorString) Error() string {
	return e.s
}
```

```go
// New returns an error that formats as the given text.
// Each call to New returns a distinct error value even if the text is identical.
// New返回一个error接口的实现,这个error接口的实现将给定的文本格式化.
// 每次调用New均会返回一个不同的error值即使给定的文本内容是相同的
func New(text string) error {
	return &errorString{text}
}
```

// TODO

Question1:为什么`New()`返回的是私有的`errorString`类型呢?

GO的基础库中大量使用预定义错误(sentinel error)

```go
// bufio/bufio.go
var (
	ErrInvalidUnreadByte = errors.New("bufio: invalid use of UnreadByte")
	ErrInvalidUnreadRune = errors.New("bufio: invalid use of UnreadRune")
	ErrBufferFull        = errors.New("bufio: buffer full")
	ErrNegativeCount     = errors.New("bufio: negative count")
)
```

这样外部的包在使用时,只要import该包,并在处理error时做等值判断即可.

sentinel error的信息书写规则:`当前包名: message`

而`errors.New()`返回的是包内部`errorString`对象的指针.

Question2: 为什么`errors.New()`返回包内部`errorString`对象的指针?

demo1.go:

```go
package main

import (
	"errors"
	"fmt"
)

// Create a named type for our new error type
// 为我们自己的error创建一个自己命名的类型
type errorString string

// Implement type error interface
// 实现error接口
func (e errorString) Error() string {
	return string(e)
}

// New creates interface values of type error
// New()函数创建一个error接口的实现
func New(text string) error {
	// 注意此处和errors包不同,没有做取指针的操作!
	return errorString(text)
}

// 自定义错误类型(errorString)
var ErrNameType = New("EOF")
// 内置错误类型(errors.errorString)
var ErrStructType = errors.New("EOF")

func main() {
	if ErrNameType == New("EOF") {
		fmt.Println("Named Type Error")
	}

	if ErrStructType == errors.New("EOF") {
		fmt.Println("Struct Type Error")
	}
}
```

运行结果:

```
go build demo1.go 
./demo1 
Named Type Error
```

Answer2:首先,我们自己实现的`errorString`实际上就是给`string`类型取了个别名,也就是说该结构体本质上就是string.那么,在这个大前提下,若`New()`返回时不取指针,则等值判断时,比对的是值域,也就是比较的是2个string的字面量是否相同;但是标准库的`errorString`是用了一个struct把string包起来了,并且在返回的时候取了地址,这样每次在调用`errors.New()`时,每次返回的都是一个新的对象,等值判断时比较的是2个对象的地址是否相同,而非字面量是否相同.

实际上语言的设计者如此设计,为的就是避免在开发过程中, developerA和developerB以相同的`string`创建了`errorString`时,在等值判断上会出现混乱.A的本意是对自己的errorString做等值判断并handle,但是却和B的errorString等值判断结果为true,这样就会造成混乱.

这也就是为什么官方库一定要把error message用struct包起来并取地址的原因.

**看源码的时候尝试理解作者的意图,作者想要避免何种情况**

TODO:如果return的时候给string也做取址操作,等值判断时会如何呢?(整理完了自己试试)

Question3: 如果从定义上也仿照官方标准库,使用一个struct将string包起来,但只是New()的时候不取地址,等值判断时会怎么样?

demo2.go:

```go
package main

import "fmt"

type errorString struct {
	s string
}

func (e errorString) Error() string {
	return e.s
}

func NewError(text string) error {
	return errorString{s: text}
}

var ErrType = NewError("EOF")

func main() {
	if ErrType == NewError("EOF") {
		fmt.Println("Error:", ErrType)
	}
}
```

运行结果:

```
go build demo2.go 
./demo2          
Error: EOF
```

Answer3:创建时不取地址,则预定义的sentinel error和New()创建出的error做等值判断时,判定结果还是为true.对2个struct做等值判断,实际上有些像浅拷贝,比较的是2个struct中的每个field的字面量是否相等.那么此时我们自定义的`errorString`中只有1个field,且字面量相同(都是EOF),所以相同了.因此New()的时候一定要取地址,因为以取址的形式返回,等值判断时,判断的是2个对象的内存地址是否相同.

### 1.2 Error VS Exception

各个语言的演进历史:

C:单返回值,一般通过传递指针作为入参,返回值int表示成功还是失败.

```c
ngx_int_t ngx_create_path(ntx_file_t *file, ngx_path_t *path)
```

C语言是单返回值,那么假设此时要创建一个对象/文件,传入一个路径要返回一个file,那么此时要么把这个file作为返回值返回,要么就是入参定义为指针类型,在函数内部通过该指针来修改结构体中的某些属性,最后返回.因为返回的int要用来表示操作结果(success or fail),所以被影响的对象只能以指针的形式传入.

C++:引入了exception,但是无法得知被调用方会抛出何种异常.也就是说,我调用一个函数时,就只能以非代码的方式(文档/函数签名)获知被调用方有可能抛出何种exception.

JAVA:引入了checked exception,方法的所有者必须声明,调用者必须处理.在启动时抛出大量异常是司空见惯的事情,并在它们的调用堆栈中尽职地记录下来.JAVA异常不再是异常,而是变得司空见惯了.它们从良性到灾难性都有使用,异常的严重性有函数的调用者来区分.这样导致调用者为了方便,基于Exception是一个根对象,就直接cache Exception,然后忽略.或者调用者在处理异常时,直接throw一个`java.Error`或`java.RuntimeException`,直接绕过具体的Exception.

checked exception:在方法签名中,告知调用者,本方法有可能抛出何种异常.

```java
try{} catch(e Exception) {// ignore}
```

GO:处理异常逻辑时,不引入exception,而是支持多参数返回,所以很容易在函数签名中带上实现了error接口的对象,然后交由调用者来判定.

**如果一个函数返回了(value, error),则调用者不能对这个value做任何假设,必须先判定error.唯一可以忽略error的情况是:你连value也不关心.**

通常error不为nil时,value是不可用的(如果error不为nil时,value还可用,就很奇怪).

demo3:

```go
package main

import "fmt"

func handle() (int, error) {
	return 1, nil
}

func main() {
	i, err := handle()
	if err != nil {
		return
	}

	fmt.Println(i)
}
```

TODO: 如果我call一个func,我连它的返回值都不关心,也忽略它的error,那么我call它的目的何在呢?

GO中有panic的机制,如果你认为和其他语言的exception一样,那你就错了,当我们throw exception的时候,相当于你把exception扔给了调用者来处理.例如:在C++中,把string转为int,若转换失败,会抛出异常.或在java中转换string为date失败时,会抛出异常.

但是GO的panic机制,意味着fatal error(也就是这个进程就挂了).不能假设调用者来解决panic,因为panic意味着代码不能继续运行.使用多个返回值和一个简单的约定,GO解决了让程序员知道何时出现了问题,并且在这个大前提下为真正的异常情况保留了panic.

**GO的panic,不能假设由调用者来处理!**实际上在业务代码中几乎不太可能抛panic的.通常在http服务框架或grpc服务框架中,第1个middleware就是recover panic并打印异常的.最终这个HTTP请求的响应一定是5XX,被中止掉.不对这个请求做还原或异常恢复(TODO: 请求还原的意思是指再curl一次这个请求吗?).

对于在线服务来说,panic有可能是由于某1个接口里的索引越界等细节问题触发的,对于Request-Driven级别的服务,不能因为1个请求导致的panic就挂掉,因此是对这个服务要有保护的.此处所说的保护,就是下文的这种recover机制,对于造成panic的请求,以5XX的形式响应,但同时不妨碍进程响应其他未触发panic的请求.同时由于异常已经打印了,所以事后还是可以复现和处理的.

https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/recovery.go:

```go
package blademaster

import (
	"fmt"
	"net/http/httputil"
	"os"
	"runtime"

	"github.com/go-kratos/kratos/pkg/log"
)

// Recovery returns a middleware that recovers from any panics and writes a 500 if there was one.
func Recovery() HandlerFunc {
	return func(c *Context) {
		defer func() {
			var rawReq []byte
			if err := recover(); err != nil {
				const size = 64 << 10
				buf := make([]byte, size)
				buf = buf[:runtime.Stack(buf, false)]
				if c.Request != nil {
					rawReq, _ = httputil.DumpRequest(c.Request, false)
				}
				pl := fmt.Sprintf("http call panic: %s\n%v\n%s\n", string(rawReq), err, buf)
				fmt.Fprintf(os.Stderr, pl)
				log.Error(pl)
				c.AbortWithStatus(500)
			}
		}()
		c.Next()
	}
}
```

1. **异步的goroutine中触发了panic时,在主线程里是recover不住的!**
2. **recover必须defer才能生效**

建议对goroutine的panic采取如下处理方式:

在kit库中,创建sync包.

```
package sync

import (
	"log"
	"sync"
)

func Go(x func()) {
	wg := &sync.WaitGroup{}
	wg.Add(1)
	go func() {
		defer func() {
			if e := recover(); e != nil {
				log.Printf("%v\n", e)
				wg.Done()
			}
		}()
		x()
		wg.Done()
	}()
	wg.Wait()
}
```

TODO:如果goroutine中有参数怎么办?

开启goroutine时,以如下方式开启:

```
package main

import (
	"class2/demo4/kit/sync"
	"fmt"
)

func main() {
	sync.Go(triggerPanic)
}

func triggerPanic() {
	panic("trigger panic")
}

func noPanic() {
	fmt.Println("no panic")
}
```

如果面对请求要开启goroutine,1个请求开1个goroutine是不靠谱的.而是把要处理的逻辑封装成一个message,对于调用方来讲,将这个message传入channel,同时在后台开了几个goroutine,这样可以避免大量创建goroutine.(这些goroutine要用工作池管理起来)

TODO:这个代码写的很糙,要改造!

```gopackage main

import (
	"fmt"
	"log"
	"net/http"
	"sync"
)

type User struct {
	UserName string
	Password string
}

var UserChannelInput chan *User = make(chan *User, 10)

var UserChannelOutput chan *User = make(chan *User, 10)

func main() {
	wg := &sync.WaitGroup{}
	wg.Add(1)
	go handleUser()
	http.HandleFunc("/index", index)
	err := http.ListenAndServe(":8081", nil)
	if err != nil {
		log.Printf("listen and serve failed: %v\n", err)
	}
	wg.Wait()
}

func index(w http.ResponseWriter, r *http.Request) {
	username := r.FormValue("username")
	password := r.FormValue("password")
	user := &User{
		UserName: username,
		Password: password,
	}
	UserChannelInput <- user
	user = <- UserChannelOutput
	fmt.Fprintf(w, "%s\n%s",user.UserName, user.Password)
}

func handleUser() {
	defer func() {
		if e := recover(); e != nil {
			log.Printf("handle user error:%v\n", e)
		}
	}()
	for {
		user := <- UserChannelInput
		user.UserName += "1"
		user.Password += "2"
		UserChannelOutput <- user
	}
}
```

实际上panic的本质是**不可被处理的**,所以以上的策略实际上是"兜底"性质的,还是要立刻去fix bug.

那么,何种情况下,panic可以不recover呢?

1. main()函数中,有一些资源的初始化工作是这段代码/程序强依赖的,若初始化不成功,则这段代码对外提供服务时,会大量报错.
2. 对于一些配置文件中,若值写的不对时(e.g:超时控制时间本来是5s,但是配置文件中读取到的时间是50s),此时就该抛出panic,告知对方配置文件参数不符合预期.这样可以避免你的代码工作在一个不正确的上下文信息中.因为如果不对配置文件做预期性检测,那么当出现故障时是无迹可寻,很难排查的.
3. init()函数中有一些初始化资源的操作,若初始化不成功,则panic

TODO:1和3有啥区别?

Question4:什么叫强依赖? 什么叫弱依赖?

Answer4:此处举2个实际例子.但具体的强依赖和弱依赖还是要看业务场景,读写缓存策略等环境相关的条件才能判别.

例1:

Q: 现有一**读多写少**的服务,此时DAO的DB连接失败,但cache可连接,请问此时该服务是否该启动?

A: 若能够hit cache,是可以启动该服务的.虽然会有DB宕机 -> 主从切换 -> 需要时间 -> 这段时间内无法启动的情况产生,但若是能够hit cache,是可以对外提供读请求的服务的.写请求虽然会失败,但是会抛出5XX的错误,也不会影响数据的一致性,所以问题是不大的.因此可以启动该服务.

例2:

Q: 现有一底层gRPC服务挂掉,其他依赖该底层服务(简称A服务)的上游服务发版均起不来,对于这些上游服务而言,是否应该将对A服务的依赖看做是弱依赖?

A: 此处首先要讲gRPC Client的连接策略:

1. blocking grpc:即连接失败会一直hang在创建Client的代码处,直到连接成功
2. non-blocking:即使连接失败,也会返回Client,但是内部会尝试自动重连

如果采用第2种策略,则在Listen时,会出现一种情况:当Listen之后流量进来的时候,会因为Client还未准备就绪,而报一小部分的错.此时该如何解决这个问题呢?

因此产生了第3种策略:二者结合的模式.即non-blocking + 超时控制.比如给Client10s的时间去准备连接,若连接失败则以non-blocking的状态对外提供服务,这样就可以实现既尝试自动重连,又不会无限等待.

刚刚的内容或许还不够说明GO的异常处理是足够好用的,因此我们来看一些demo.

demo6.go

```go
package main

import "fmt"

// Positive returns true if the number is positive,false if it is negative
// Positive 在n为正数时返回true,在n为负数时返回false
func Positive(n int) bool {
	return n > -1
}

func Check(n int) {
	if Positive(n) {
		fmt.Println(n, "is positive")
	} else {
		fmt.Println(n, "is negative")
	}
}

func main() {
	Check(1)
	Check(0)
	Check(-1)
}
```

运行结果:

```
go build
./demo6 
1 is positive
0 is positive
-1 is negative
```

很明显这段代码是有BUG的.因为0非正非负.

因此我们fix了一个版本:

demo7.go

```go
package main

import "fmt"

// Positive returns true if the number is positive, false if it is negative.
// The second return value indicates if the result is valid, which in the case of
// n == 0, is not valid.
// Positive 在给定的数字为正数时返回true,在给定的数字为负数时返回false
// 第2个返回值表示表示返回值是否有效,因为当n==0时返回值是无效的
func Positive(n int) (bool, bool) {
	if n == 0 {
		return false, false
	}
	return n > -1, true
}

func Check(n int) {
	pos, ok := Positive(n)
	if !ok {
		fmt.Println(n, "is neither")
		return
	}

	if pos {
		fmt.Println(n, "is positive")
	} else {
		fmt.Println(n, "is negative")
	}
}

func main() {
	Check(1)
	Check(0)
	Check(-1)
}
```

运行结果:

```
go build
./demo7 
1 is positive
0 is neither
-1 is negative
```

对于go来讲,通常不会用bool来做判定逻辑.而是用标准库errors来表示错误.

demo8.go

```go
package main

import (
	"errors"
	"fmt"
)

// Positive returns true if the number is positive,false if it is negative
// Positive 在n为正数时返回true,在n为负数时返回false
func Positive(n int) (bool, error) {
	if n == 0 {
		return false, errors.New("undefined")
	}
	return n > -1, nil
}

func Check(n int) {
	pos, err := Positive(n)
	if err != nil {
		fmt.Println(n, err)
		return
	}

	if pos {
		fmt.Println(n, "is positive")
	} else {
		fmt.Println(n, "is negative")
	}
}

func main() {
	Check(1)
	Check(0)
	Check(-1)
}
```

运行结果:

```
go build
./demo8 
1 is positive
0 undefined
-1 is negative
```

对于`Positive()`的调用者来讲,必须先判断error,才能确定值是否可用.

有人提出了如下的解决方案,返回一个*bool,用空指针的形式向调用者表示n==0的情况.

demo9.go

```go
package main

import "fmt"

func Positive(n int) *bool {
	if n == 0 {
		return nil
	}

	r := n > -1
	return &r
}

func Check(n int) {
	pos := Positive(n)
	if pos == nil {
		fmt.Println(n, "is neither")
		return
	}

	if *pos {
		fmt.Println(n, "is positive")
	} else {
		fmt.Println(n, "is negative")
	}
}

func main() {
	Check(1)
	Check(0)
	Check(-1)
}
```

运行结果:

```
go build
./demo9 
1 is positive
0 is neither
-1 is negative
```

Question5: 若DAO层查询记录查不到,此时该返回一个空指针,还是返回error?

Answer5: 建议返回error.因为如果返回一个空指针,则对于调用者而言,要先判断返回值是否为空,从可读性上来讲,空指针的含义是不明确的.对于调用方而言,无法在判断nil的这一位置上告知其代码的读者,对nil的判定表示何种意义.进而读者只能去翻看上下文,才能确定此处对nil的判定表示DB查询结果不存在.

还有人提出以panic()的形式fix bug.

demo10.go

```go
package main

import "fmt"

// Positive returns true if the number is positive,false if it is negative
// In the case that n is 0, Positive will panic
// Positive 在n为正数时返回true,在n为负数时返回false.
// 当n为0时,Positive 将会panic
func Positive(n int) bool {
	if n == 0 {
		panic("undefined")
	}
	return n < -1
}

func Check(n int) {
	defer func() {
		if recover() != nil {
			fmt.Println(n, "is undefined")
		}
	}()

	if Positive(n) {
		fmt.Println(n, "is positive")
	} else {
		fmt.Println(n, "is negative")
	}
}

func main() {
	Check(1)
	Check(0)
	Check(-1)
}
```

运行结果:

```
go build
./demo10 
1 is negative
0 is undefined
-1 is negative
```

这种相当于是用panic去处理逻辑上的问题.这个是不行的.

对于真正意外的情况和那些不可恢复的程序错误,例如索引越界、不可恢复的环境问题、栈溢出,我们才使用panic.对于其他的错误情况,我们应该使用error来进行判定.

you only need to check the error value if you care about the result.  -- Dave

如果关注结果,必须判定error.

This blog post from Microsoft’s engineering blog in 2005 still holds true today, namely:
My point isn’t that exceptions are bad. My point is that exceptions are too hard and I’m not smart enough to handle them.

我不认为异常是坏事.我认为异常只是很难处理,难的程度已经到了我认为我不够聪明以至于我无法处理.

我的意思不是说异常不好.我的观点是,异常太难了,我不够聪明,无法处理它们.(谷歌机翻)

```
item = getFromDB()
item.Value() = 400
saveToDB(item)
item.Text = 'price changed'
```

若`saveToDB(item)`时若报错,抛出异常.则item是处于不一致的状态的.但对于GO来讲,可以抛出error,对于调用者而言一定是先判断error,进而告知调用方,item此时处于不一致的状态.

TODO:"不一致的状态"是什么意思?

1. 简单
2. 考虑失败,而非成功(Plan for failure, not success)
3. 没有隐藏的控制流(因为如果有隐藏的控制流,代码可能就执行到另外一个地方去了,比如上文的2个wg.Done()当然那个case是panic的情况)
4. 完全交给你来控制error
5. Error are values(error在go中就是一个值类型,可以自己做操作的)

## PART2. Error Type



## PART3. Handling Error
## PART4. Go 1.13 errors
## PART5. Go 2 Error Inspection
