## [errs] Yet Another Golang Error Package... (YAGEP?)

Errors are suposed to makes us better right? Well let's try to make errors better then...

This package is primarily a go-kit errors package. Finally no more needs for the `github.com/pkg/errors` if you need the caller and stack.

Advantages:
- Always provide real error caller
- Unclutter your logs and allows better error tracking
- Define logging activation by error Level
- Has an `error` compliant type
- Structured `*Err` type that ease unit testing even for customs or 'on the fly' defined error.

Let's asume an uppersace string service with this content check for example purpose:
```
func (stringService) Uppercase(s string) (string, error) {
	if _, err := strconv.Atoi(s); err == nil {
		return s, errs.New(http.StatusBadRequest, "uppercase some numbers dude, really??", errs.Info)
	}
    return strings.ToUpper(s), nil
}
```
The `errs.New()` function has a variadic input parameter where you can put an `error` type, `string` message, `int` for level (0-7) and an http.Status code to return.

You can define your error like this and they are some already ready to use inside `errs` package for your convenience (defined in `errors_def.go` file and please PR are accepted to include more of them) but they can be defined anywhere and customised as you wish. 
The included `Err` inside package:
```
var (
	// ErrInternalServer the basic 500 throw it all error...
	ErrInternalServer = &Err{
		Message: "an internal server error occurred please contact the server's administrator",
		Code:    http.StatusInternalServerError,
		Level:   Error,
	}

	ErrNotFound = &Err{
		Message: "not found",
		Code:    http.StatusNotFound,
		Level:   Info,
	}

	// ErrInvalidBody is used when the payload is wrong
	ErrInvalidBody = &Err{
		Message: "invalid body",
		Code:    http.StatusBadRequest,
		Level:   Info,
	}

	// ErrInvalidParameter throwed when a query required parameter is missing or wrong
	ErrInvalidParameter = &Err{
		Message: "invalid parameter",
		Code:    http.StatusBadRequest,
		Level:   Info,
	}

	// ErrEmpty is returned when an input string is empty.
	ErrEmptyParam = &Err{
		Message: "empty parameter",
		Code:    http.StatusBadRequest,
		Err:     errors.New("empty parameter"),
		Level:   Info,
	}
)
```

A simple straightforward usage out of the box would be:
```
return errs.New(errs.ErrEmptyParam)
```
would log this:
```json
{
    "caller": "/home/gp/go/src/github.com/gplume/gokit-error-handling/api/service_validation.go:21",
    "code": 400,
    "error": "empty parameter",
    "level": "Info",
    "message": "empty parameter",
    "http.method": "POST",
    "http.proto": "HTTP/1.1",
    "http.url": "/uppercase",
    "http.user_agent": "PostmanRuntime/7.6.0"
}
```
`caller` is clean –because 99.9% of the time you don't need the full stack– has the right caller and is ctrl-click-able into your console, it will open the appropriate file in you prefered IDE.

`code` will be translated in the right `http` status code to the client.

`message` contain the message displayed to the client under the field `error`

`error` contains the std go `err.Error()` string.

`level` is parametrable at api startup and is defaulted to only log under the Info level.


## Standard Error level (based on Standards):
| Standard | intLevel |
|----------|----------|
| Undefined | 0 |
| Fatal 	| 1 |
| Error 	| 2 |
| Warn      | 3 |
| Info 	    | 4 |
| Debug 	| 5 |
| Trace 	| 6 |


## Errs type
```
type Err struct {
	Err     error
	Message string
	Code    int
	Level   level // from 1 to 7
	Caller  string
	Stack   *Stack
}
```
At runetime you don't have to provide `Caller` and `Stack` it will be managed by `errs.New()` function.

The `*errs.Err` type is complient with the standard library `error` type by the trick of this line: 
```
var _ error = &Err{}
```

so actually:

```
var (
    ErrCustomInvalidParam = &errs.Err{
        Message: "invalid parameter",
        Code:    http.StatusBadRequest,
        Level:   errs.Info,
    }
)

func doSomeService(() error {
    return errs.New(ErrCustomInvalidParam)
}

```
...works !
That is very helpful to incorporate `errs` package in existing codebase and also `go-kit` that force you to use those `Endpoint` `error` type...


## Compositing

You can compose your own specific error at runtime like this:

```
return errs.New(err, "string message displayed to client", http.StatusNotAcceptable, errs.Error)

```
or:

```
return errs.New(errs.ErrEmptyParam, errs.Error) // here only the Level is overridden

```
So you can use a defined error and override some of its value if you put it alongside. Above the Error Level is overriden from `errs.Info` to `errs.Error` (and will therefore be logged).
Do whatever feels intuitive to you it will work!
Orders don't matters because of variadic. If you don't throw an http.Status code or message, these will be defaulted to the common error 500 code and message that you can find in the `errors_def.go` file.
Note that you are not forced to input somme `err` in `errs.New()`then in le log message the error will simply be nil.


## Still need the full stack?
You can! and we even made it better that `github.com/pkg/errors` by reverting the stack so the last line presents the actual needed caller with the addition of the Service name. Here's an example with the same error as above (but please don't use that in production :)

```
{
    "caller":"/home/gp/go/src/github.com/gplume/gokit-error-handling/api/service_validation.go:21",
    "code":400,
    "error":"empty parameter",
    "http.method":"POST",
    "http.proto":"HTTP/1.1",
    "http.url":"/uppercase",
    "http.user_agent":"PostmanRuntime/7.6.0",
    "level":"Info",
    "message":"empty parameter",
    "stack":"/usr/local/go/src/runtime/asm_amd64.s:1333: runtime.goexit:\n
    /usr/local/go/src/net/http/server.go:1847: net/http.(*conn).serve:\n
    /usr/local/go/src/net/http/server.go:2741: ...serverHandler.ServeHTTP:\n
    /usr/local/go/src/net/http/server.go:1964: ...HandlerFunc.ServeHTTP:\n
    /home/gp/go/src/github.com/gplume/gokit-error-handling/middle/recover.go:31: github.com/gplume/gokit-error-handling/middle.RecoverFromPanic.func1:\n
    /home/gp/go/src/github.com/gplume/gokit-error-handling/vendor/github.com/go-zoo/bone/bone.go:43: ...vendor/github.com/go-zoo/bone.(*Mux).ServeHTTP:\n
    /home/gp/go/src/github.com/gplume/gokit-error-handling/vendor/github.com/go-zoo/bone/helper.go:18: ...parse:\n
    /home/gp/go/src/github.com/gplume/gokit-error-handling/vendor/github.com/go-zoo/bone/route.go:148: ...(*Route).parse:\n
    /usr/local/go/src/net/http/server.go:1964: net/http.HandlerFunc.ServeHTTP:\n
    /home/gp/go/src/github.com/gplume/gokit-error-handling/middle/http_instrumenting.go:67: github.com/gplume/gokit-error-handling/middle.Metrics.func1.1:\n
    /usr/local/go/src/net/http/server.go:1964: net/http.HandlerFunc.ServeHTTP:\n
    /home/gp/go/src/github.com/gplume/gokit-error-handling/middle/notify.go:17: github.com/gplume/gokit-error-handling/middle.Notify.func1.1:\n
    /home/gp/go/src/github.com/gplume/gokit-error-handling/vendor/github.com/go-kit/kit/transport/http/server.go:110: ...vendor/github.com/go-kit/kit/transport/http.Server.ServeHTTP:\n
    /home/gp/go/src/github.com/gplume/gokit-error-handling/api/endpoints.go:19: ...api.MakeUppercaseEndpoint.func1:\n
    /home/gp/go/src/github.com/gplume/gokit-error-handling/api/service_validation.go:21: ...serviceValidation.Uppercase: empty parameter"
}
```

## Settings

In your `func main()` you can define how the `errs` package will behave:

```
	if err := errs.Settings(errs.End, false); err != nil {
		logger.Log("init_error", err)
		os.Exit(1)
	}
```
The first parameter define below which level the logging will be activated. `errs.End` means that all levels will be logged, good for development.
By default logging start at `errs.Warn`.

The second parameter is a boolean that activate the improved full stack you are used to with `github.com/pkg/errors` package, but do you really need that?
