[TOC]

## :beer:前言

 Go有一套简单的`error`处理模型，但其实并不像看起来那么简单。本文中，我会提供一种好的方法去处理`error`，并用这个方法来解决在往后编程遇到的的类似问题。

 首先，我们会分析下Go中的`error`。接着我们来看看`error`的产生和`error`的处理，再分析其中的缺陷。最后，我们将要探索一种方法来解决我们在程序中遇到的类似问题。

----



## :beer:什么是error

看下`error`在内置包中的定义，我们可以得出一些结论：

```go
// error类型在内置包中的定义是一个简单的接口
// 其中nil代表没有异常
type error interface {
    Error() string
}
```

从上面的代码，我们可以看到`error`是一个接口，只有一个`Error`方法。

那我们要实现`error`就很简单了，看以下代码：

```go
type MyCustomError string

func (err MyCustomError) Error() string {
  return string(err)
}
```

下面我们用标准包`fmt`和`errors`去声明一些`error`：

```go
import (
  "errors"
  "fmt"
)

simpleError := errors.New("a simple error")
simpleError2 := fmt.Errorf("an error from a %s string", "formatted")
```

思考：上面的`error`定义中，只有这些简单的信息，就足够处理好异常吗？我们先不着急回答，下面我们去寻找一种好的解决方法。

----



## :beer:error处理流

现在我们已经知道了在Go中的`error`是怎样的了，下一步我们来看下`error`的处理流程。

为了遵循简约和DRY（避免重复代码）原则，我们应该只在一个地方进行`error`的处理。

我们来看下以下的例子：

```go
// 同时进行error处理和返回error
// 这是一种糟糕的写法
func someFunc() (Result, error) {
    result, err := repository.Find(id)
    if err != nil {
        log.Errof(err)
        return _, err
    }
    return result, nil
}
```

上面这段代码有什么问题呢？

我们首先打印了这个`error`信息，然后又将`error`返回给函数的调用者，这相当于重复进行了两次`error`处理。

很有可能你组里的同事会用到这个方法，当出现`error`时，他很有可能又会将这个`error`打印一遍，然后重复的日志就会出现在系统日志里了。

我们先假设程序有3层结构，分别是**<u>数据层</u>**，**<u>交互层</u>**和**<u>接口层</u>**：

```go
// 数据层:用了一个第三方orm库
func getFromRepository(id int) (Result, error) {
    result := Result{ID: id}
    err := orm.entity(&result)
    if err != nil {
        return _, err
    }
    return result, nil 
}
```

根据DRY原则，我们可以将`error`返回给调用的最上层接口层，这样我们就能统一的对`error`进行处理了。

但上面的代码有一个问题，Go的内置`error`类型是没有调用栈[^调用栈]的。另外，如果`error`产生在第三方库中，我们还需要知道我们项目中的哪段代码负责了这个`error`。

**github.com/pkg/errors** 可以使用这个库来解决上面的问题。

利用这个库，我对上面的代码进行了一些改进，加入了调用栈和加了一些相关的错误信息。

```go
import "github.com/pkg/errors"

// 数据层:使用了第三方的orm库
func getFromRepository(id int) (Result, error) {
    result := Result{ID: id}
    err := orm.entity(&result)
    if err != nil {
        return _, errors.Wrapf(err, "error getting the result with id %d", id);
    }
    return result, nil 
}
// 当error封装完后，返回的error信息将会是这样的
// err.Error() -> error getting the result with id 10
// 这就很容易知道这是来自orm库的error了
```

上面的代码对orm的`error`进行了封装，增加了调用栈，而且没有修改原始的`error`信息。

 然后我们再来看看在其他层是如何处理这个`error`的，首先是**<u>交互层</u>**：

```go
func getInteractor(idString string) (Result, error) {
  id, err := strconv.Atoi(idString)
  if err != nil {
    return _, errors.Wrapf(err, "interactor converting id to int")
  }
  return repository.getFromRepository(id)
}
```

接着是接口层：

```go
func ResultHandler(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    result, err := interactor.getInteractor(vars["id"])
    if err != nil {
        handleError(w, err)
    }
    fmt.Fprintf(w, result)
}

func handleError(w http.ResponseWriter, err error) {
    // 返回HTTO 500错误
    w.WriteHeader(http.StatusIntervalServerError)
    log.Errorf(err)
    fmt.Fprintf(w, err.Error())
}
```

现在我们只在最上层接口层处理了`error`，看起来很完美？并不是，如果程序中经常返回HTTP错误码500，同时将错误打印到日志中，像`result not found`这种没用的日志就会很烦人。

----



## :beer:error的三个宗旨

我们上面讨论到仅仅靠一个字符串是不足以处理好error的。我们也知道通过给error加一些额外的信息就能追溯到error的产生和最后的处理逻辑。

因此我定义了三个`error`处理的宗旨。

- 提供清晰完整的调用栈
- 必要时提供`error`的上下文信息
- 打印`error`到日志中（例如可以在框架层打印）

我们来创建一个`error`类型：

```go
const (
	NoType = ErrorType(iota)
	BadRequest
	NotFound
	// 可以加入你需要的error类型
)

// 自定义错误类型
type ErrorType uint

// 结构体：常见错误
type customError struct {
	// 错误类型
	errorType ErrorType
	// error源头
	originalError error
	// error具体内容
	context errorContext
}

// 结构体：错误信息
type errorContext struct {
	Field   string
	Message string
}

// 返回customError具体的错误信息
func (error customError) Error() string {
	return error.originalError.Error()
}

// 创建一个新的customError
func (errorType ErrorType) New(msg string) error {
	return customError{errorType: errorType, originalError: errors.New(msg)}
}

// 给customError自定义错误信息
func (errorType ErrorType) Newf(msg string, args ...interface{}) error {
	return customError{errorType: errorType, originalError: fmt.Errorf(msg, args...)}
}

// 对error进行封装
func (errorType ErrorType) Wrap(err error, msg string) error {
	return errorType.Wrapf(err, msg)
}

// 对error进行封装，并加入格式化信息
func (errorType ErrorType) Wrapf(err error, msg string, args ...interface{}) error {
	return customError{errorType: errorType, originalError: errors.Wrapf(err, msg, args...)}
}
```

 从上面的代码可以看到，我们可以创建一个新的`error`类型或者对已有的`error`进行封装。但我们遗漏了两件事情：1、我们不知道`error`的具体类型；2、是我们不知道怎么给这这个`error`加上下文信息。

为了解决以上问题，我们来对**github.com/pkg/errors**的方法也进行一些封装。

```go
// 创建一个NoType error
func New(msg string) error {
	return customError{errorType: NoType, originalError: errors.New(msg)}
}

// 创建一个加入了格式化信息的NoType error
func Newf(msg string, args ...interface{}) error {
	return customError{errorType: NoType, originalError: errors.New(fmt.Sprintf(msg, args...))}
}

// 给error封装多一层string
func Wrap(err error, msg string) error {
	return Wrapf(err, msg)
}

// 返回最原始的error
func Cause(err error) error {
	return errors.Cause(err)
}

// error加入格式化信息
func Wrapf(err error, msg string, args ...interface{}) error {
	wrappedError := errors.Wrapf(err, msg, args...)
	if customErr, ok := err.(customError); ok {
		return customError{
			errorType:     customErr.errorType,
			originalError: wrappedError,
			context:       customErr.context,
		}
	}

	return customError{errorType: NoType, originalError: wrappedError}
}
```

接着我们给`error`加入上下文信息：

```go
// AddErrorContext adds a context to an error
func AddErrorContext(err error, field, message string) error {
	context := errorContext{Field: field, Message: message}
	if customErr, ok := err.(customError); ok {
		return customError{errorType: customErr.errorType, originalError: customErr.originalError, context: context}
	}

	return customError{errorType: NoType, originalError: err, context: context}
}

// GetErrorContext returns the error context
func GetErrorContext(err error) map[string]string {
	emptyContext := errorContext{}
	if customErr, ok := err.(customError); ok || customErr.context != emptyContext {
		return map[string]string{"field": customErr.context.Field, "message": customErr.context.Message}
	}

	return nil
}

// GetType returns the error type
func GetType(err error) ErrorType {
	if customErr, ok := err.(customError); ok {
		return customErr.errorType
	}

	return NoType
}
```

现在将上述的方法应用在我们文章开头写的example中：

### 数据层

```go
import "github.com/our_user/our_project/errors"
// The repository uses an external depedency orm
func getFromRepository(id int) (Result, error) {
    result := Result{ID: id}
    err := orm.entity(&result)
    if err != nil {
        msg := fmt.Sprintf("error getting the  result with id %d", id)
        switch err {
        case orm.NoResult:
            err = errors.Wrapf(err, msg);
        default:
            err = errors.NotFound(err, msg);
        }
        return _, err
    }
    return result, nil 
}
// after the error wraping the result will be
// err.Error() -> error getting the result with id 10: whatever it comes from the orm
```

### 交互层

```go
func getInteractor(idString string) (Result, error) {
    id, err := strconv.Atoi(idString)
    if err != nil {
        err = errors.BadRequest.Wrapf(err, "interactor converting id to int")
        err = errors.AddContext(err, "id", "wrong id format, should be an integer)
        return _, err
    }
    return repository.getFromRepository(id)
}
```

### 接口层

```go
func ResultHandler(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    result, err := interactor.getInteractor(vars["id"])
    if err != nil {
        handleError(w, err)
    }
    fmt.Fprintf(w, result)
}

func handleError(w http.ResponseWriter, err error) {
    var status int
    errorType := errors.GetType(err)
    switch errorType {
    case BadRequest:
        status = http.StatusBadRequest
    case NotFound:
        status = http.StatusNotFound
    default:
        status = http.StatusInternalServerError
    }
    w.WriteHeader(status)

    if errorType == errors.NoType {
        log.Errorf(err)
    }
    fmt.Fprintf(w,"error %s", err.Error())

    errorContext := errors.GetContext(err)
    if errorContext != nil {
        fmt.Printf(w, "context %v", errorContext)
   }
}
```

通过简单的封装，我们可以明确的知道`error`的错误类型了，然后我们就能方便进行处理了。

读者也可以将代码运行一遍，或者利用上面的`errors`库写一些demo来加深理解。

----



## :beer:其他

### 备注

[^java error]: 译文中`error`可以理解为异常，但Go中的`error`和Java中的异常还是有很大区别的.
[^调用栈]: 没有调用栈，无法知道error的具体来源；

### 拓展

![GOLANG错误处理最佳方案](../../../assets/GOLANG错误处理最佳方案-1560317421309.png)