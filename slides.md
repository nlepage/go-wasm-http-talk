---
title: Deploy a Go HTTP server in your browser
theme: white
separator: ----
verticalSeparator: ---
enableMenu: false
enableChalkboard: false
enableTitleFooter: false
enableZoom: false
enableSearch: false
css: assets/styles.css
---

# Deploy a Go HTTP server in your browser

Notes:
- Hi everybody, my name is Nicolas Lepage, I am a developer at Zenika IT in France. I work mostly with Javascript, and I also like experimenting with Go.
- This is my first talk in english, so dont be surprised, I am a little out of practice.
- Today I am going to talk about, deploying a Go HTTP server in your browser.

---

## Why?

- For demonstration purposes
- Avoid deploying a server

Notes:
- First, you may be wondering, why? Why would I want to do that?
- Well it could be useful for demonstration puproses.
- I have this nice little Go project, it is a command line interface tool that helps me add a text at the bottom or at the top of a JPEG image.
- And it also has an http subcommand, which starts a little HTTP server, with just an HTML form where you can do the same.
- Now, I would like to put up a little demonstration server for my project.
- But I don't want to actually deploy a Go server for this.
- So that's how I started wondering if we could deploy a Go HTTP server in a browser.

---

## Build to WebAssembly

![WebAssembly Logo](assets/webassembly.svg) <!-- .element: style="width: 300px;" -->

- Since Go 1.11
- May be executed in a browser
- Has same limitations as JavaScript

Notes:
- Since Go 1.11, running a program written in Go in a browser is possible, if you build it to WebAssembly.
- You get a WebAssembly binary which can be downloaded and executed by a browser.
- If you haven't heard about WebAssembly, it uses a portable bytecode format executable by browsers, and compiled from higher level languages such as Rust, C, C++, or Go.
- Of course, Go code executed in a browser has exactly the same limitations as any JavaScript code executed in the same browser.
- For example, it will not be able to use the `os` package to access the client's file system.
- So, it will also not be able to actually start an HTTP server in the browser.
- However, when an HTTP request is sent from a web page, there are some cases when it will not actually reach the server.

---

## ServiceWorkers

![ServiceWorkers schema](assets/service-workers.png) <!-- .element: style="width: 800px;" -->

Notes:
- One case is when it is intercepted by a service worker, which usually allows web applications to work offline for example.
- Now I think you are starting to see where I am going with this.
- The question is, would it be possible to execute a Go WebAssembly binary in a service worker and use it to handle HTTP requests?
- Let's find out!

---

## ⚠️ Disclaimer ⚠️

Notes:
- A little warning before we go any further.
- When you are targetting WebAssembly, you have to make sure all the code you are trying to build is compatible.
- This means for example that you cannot rely on C bindings, or system dependencies, or a database server.
- Also you have to be carefull with Go's standard library, which for a large part can be built to WebAssembly, but will actually panic at runtime.
- That being said, today I'm going to focus on the HTTP side of things.

---

## Respond from a ServiceWorker

<!-- .slide: data-auto-animate -->

Notes:
- First, let's have a quick look at how it is possible to respond to an HTTP request from a service worker.
- When a service worker intercepts an HTTP request, it receives a `FetchEvent`.

---

## Respond from a ServiceWorker

<!-- .slide: data-auto-animate -->

```js [1,11|2-9|10|]
FetchEvent {
    request: {
        url: "http://example.com/",
        method: "GET",
        headers: {
            "Accept": "text/html",
        },
        body: ...
    },
    respondWith(Response | Promise<Response>),
}
```

Notes:
- The `FetchEvent` contains a `Request` object ▶️ which holds all the information we need about the request (method, headers), and also the body contents if any.
- The `FetchEvent` also has a `respondWith()` method ▶️, which accepts one parameter of type `Response` or `Promise` for a `Response`.
- ▶️ So, by reading the `Request` object and building a `Response` object to give to `respondWith()`, we are able to respond to an HTTP request from a ServiceWorker.
- However, what we want to do is delegate this task to a Go WebAssembly binary.

---

<!-- .slide: data-autoslide="1" -->

---

## Building a Go HTTP server

<!-- .slide: data-auto-animate -->

Notes:
- Now let's take a step back and review how we usually build an HTTP server in Go.
- The most straightforward way is to use the `http` package.

---

## Building a Go HTTP server

<!-- .slide: data-auto-animate -->

```go [1-5|7-9]
http.Handler("/foo", fooHandler)

http.HandleFunc("/bar", func(res http.ResponseWriter, req *http.Request) {
    // handle bar request
})

r := mux.NewRouter() // gorilla/mux
r.HandleFunc("/foo", handleFoo)
r.HandleFunc("/bar", handleBar)
```
<!-- .element: data-id="code" style="font-size: 0.42em;" -->

Notes:
- First we define HTTP handlers using `Handle()` which accepts a `Handler` or `HandleFunc()` which accepts a simple function.
- Indeed, handlers are simple functions which receive a `ResponseWriter` and a `Request`; so we can already see that this looks a lot like the service worker's `FetchEvent`.
- ▶️ We may also choose to define handlers using some third party libraries, such as `gorilla/mux`.

---

## Building a Go HTTP server

<!-- .slide: data-auto-animate -->

```go [11-13|]
http.Handler("/foo", fooHandler)

http.HandleFunc("/bar", func(res http.ResponseWriter, req *http.Request) {
    // handle bar request
})

r := mux.NewRouter() // gorilla/mux
r.HandleFunc("/foo", handleFoo)
r.HandleFunc("/bar", handleBar)

http.ListenAndServe(":8080", nil) // use http.DefaultServeMux
// or
http.ListenAndServe(":8080", r) // use mux router
```
<!-- .element: data-id="code" style="font-size: 0.42em; overflow: hidden;" -->

Notes:
- Then, once our handlers are defined, in most cases we will call `ListenAndServe()`, which will start listening for HTTP requests and use our handlers to respond to these.
- ▶️ In our case, we would like to reuse as much as possible of this code we wrote, but use it to respond to a request intercepted by a SW.
- Provided the handlers are WebAssembly compatible, we can keep and reuse them as they are.
- So this is nice, because the handlers are the main part of our code, this is where we declare all the logic.
- Of course, the one thing we are not going to be able to reuse is the call to `ListenAndServe()`.
- But, this is OK, because we are going to take a pretty radical shortcut.
- Ususally when we call `ListenAndServe()`, it takes care of a lot of things for us under the hood, and for each request it calls the `Handler` if we gave one in parameter or `DefaultServeMux`, which is the default `Handler`.
- So, what we can actually do is, directly call the `ServeHTTP()` method of the `Handler` or of `DefaultServeMux`.

---

## The plan

![Plan diagram](assets/plan.png)

Notes:
- So let's review the plan!
- First step, we intercept the HTTP request in the ServiceWorker, and send the Javascript Request object to the WebAssembly binary.
- Second step in the WebAssembly binary, we map the Javascript request into a Go request, and call the handler.
- Third step, once the handler has returned, we map the Go response into a Javascript Response Object, and send it back to the ServiceWorker.
- Fourth and last step in the ServiceWorker, we respond to the HTTP request with the Javascript Response Object.

---

## Adding `js/wasm` target

<!-- .slide: data-auto-animate -->

Notes:
- Now, ideally we still want to be able to build standard binaries of the server working for linux or mac OS for example, and also the WebAssembly binary.

---

## Adding `js/wasm` target

<!-- .slide: data-auto-animate -->

📄 `server.go`

```go []
// +build !js,!wasm

func startServer() {
    http.ListenAndServe(":8080", handler)
}
```

Notes:
- For this, we can move the `ListenAndServe()` call into its own file, and use build tags to tell the Go compiler that this file is not compatible with WebAssembly.

---

## Adding `js/wasm` target

<!-- .slide: data-auto-animate -->

📄 `server.go`

```go []
// +build !js,!wasm

func startServer() {
    http.ListenAndServe(":8080", handler)
}
```

📄 `server_js_wasm.go`
```go []
func startServer() {
    wasmhttp.Serve(handler)
}
```

Notes:
- Then we can create a specific file for WebAssembly, this time using the file naming convention instead of the build tags, to tell the compiler that this file is compatible only with WebAssembly.
- And in this file we are going to use our own API, which will probably look something like this, so a `wasmhttp.Serve()` function which needs only the `Handler` parameter.

---

<!-- .slide: data-autoslide="1" -->

---

## Implementing `wasmhttp.Serve()`

<!-- .slide: data-auto-animate -->

```go []
import (
    "http"
)

func Serve(handler http.Handler) {

}
```
<!-- .element: data-id="code" style="font-size: 0.42em;" -->

Notes:
- Now let's dive in the implementation of this `Serve()` function.
- If you remember the first step of the plan, it needs to receive the Javascript request objects.
- So from the Javascript point of view, the ServiceWorker needs to call the Webassembly binary with each `Request` object.
- At the moment, Go Webassembly binaries have no way to export functions or other values to Javascript.
- So, in order to work around this, the `Serve()` function will have to give a callback function to the ServiceWorker.

---

## Implementing `wasmhttp.Serve()`

<!-- .slide: data-auto-animate -->

```go []
import (
    "http"
    "syscall/js"
)

func Serve(handler http.Handler) {
    callback := js.FuncOf(func(_ js.Value, args []js.Value) interface{} {

    })
}
```
<!-- .element: data-id="code" style="font-size: 0.42em;" -->

Notes:
- The `syscall/js` package allows to create such callback functions using `FuncOf()`.
- `FuncOf()` takes a Go function, and creates a JavaScript function from it.
- The `js.Value` type represents a Javascript value for Go.
- The first underscored parameter is the `this` value of the Javascript function, which isn't useful for us.

---

## Implementing `wasmhttp.Serve()`

<!-- .slide: data-auto-animate -->

```go [8-10]
import (
    "http"
    "syscall/js"
)

func Serve(handler http.Handler) {
    callback := js.FuncOf(func(_ js.Value, args []js.Value) interface{} {
        jsReq := args[0]

        // FIXME: return a Promise for a Response
    })
}
```
<!-- .element: data-id="code" style="font-size: 0.42em;" -->

Notes:
- The callback function needs only the first argument, which is the Javascript request object.
- And it will return one value, which will have to be a Javascript `Promise` for a Javascript `Response` object.

---

## Implementing `wasmhttp.Serve()`

<!-- .slide: data-auto-animate -->

```go [13]
import (
    "http"
    "syscall/js"
)

func Serve(handler http.Handler) {
    callback := js.FuncOf(func(_ js.Value, args []js.Value) interface{} {
        jsReq := args[0]

        // FIXME: return a Promise for a Response
    })

    js.Global().Get("setGoCallback").Invoke(callback)
}
```
<!-- .element: data-id="code" style="font-size: 0.42em;" -->

Notes:
- Then the `Serve` function can register the callback with the ServiceWorker, by calling a setter function, which has to be previously declared in the ServiceWorker's global scope.

---

## Sending requests to Go

📄 `sw.js`
```js [|6-8|]
let goCallback
self.setGoCallback = v => {
    goCallback = v
}

self.addEventListener('fetch', e => {
    goCallback(e.request)
})
```
<!-- .element: style="font-size: 0.46em;" -->

Notes:
- From the ServiceWorker's point of view, we are now able to forward the FetchEvent's request to the WebAssembly binary.
- ▶️ In the event handler, we just have to call the callback function with the `FetchEvent`'s request.
- ▶ The `self` variable is a reference to the ServiceWorker's global scope, so this is handy for making the `setGoCallback()` function available for WebAssembly the binary.
- The actual code is a little more complex, because we have to use a Promise for the callback, otherwise a FetchEvent might occur before the callback is defined.

---

## `Promise` for a `Response`

<!-- .slide: data-auto-animate -->

```go []
callback := js.FuncOf(func(_ js.Value, args []js.Value) interface{} {
    jsReq := args[0]

    // FIXME: return a Promise for a Response
})
```
<!-- .element: data-id="code" style="font-size: 0.42em;" -->

Notes:
- First step of the plan is done, now step two on the Go side, let's focus on implementing this callback function.
- We said that this callback function must return a `Promise` for a Javascript `Response` object.

---

## `Promise` for a `Response`

<!-- .slide: data-auto-animate -->

```go [4-8]
callback := js.FuncOf(func(_ js.Value, args []js.Value) interface{} {
    jsReq := args[0]

    resPromise, resolve, reject := NewPromise()

    // FIXME resolve Response Promise

    return resPromise
}
```
<!-- .element: data-id="code" style="font-size: 0.42em;" -->

Notes:
- So let's create a new `Promise`, and return it.
- The `NewPromise` function I am using here is not part of the `syscall/js` package, it is just a utility function to ease the creation of a new JavaScript `Promise`, which can be a little cumbersome using `syscall/js`.
- Returning a promise means the callback function is asynchronous, so we need to start a new goroutine...

---

## `Promise` for a `Response`

<!-- .slide: data-auto-animate -->

```go [6-8]
callback := js.FuncOf(func(_ js.Value, args []js.Value) interface{} {
    jsReq := args[0]

    resPromise, resolve, reject := NewPromise()

    go func() {
        // FIXME resolve Response Promise
    }()

    return resPromise
}
```
<!-- .element: data-id="code" style="font-size: 0.42em;" -->

Notes:
- ...otherwise the ServiceWorker would be blocked.
- And starting a new goroutine actually makes sense, because this is how a Go HTTP server usually works, it starts a new goroutine for each request.
- And this is it for the callback function, the rest of the work will be done in the new goroutine.

---

<!-- .slide: data-autoslide="1" -->

---

## JS Request to Go Request

<!-- .slide: data-auto-animate -->

```go []
import (
    "net/http"
    "syscall/js"
)

func JSRequestToGoRequest(jsReq js.Value) http.Request {
    
}
```
<!-- .element: data-id="code" -->

Notes:
- Now the first thing we need to do in this new goroutine, is create an instance of a Go `http.Request`, from the Javascript `Request` object.
- We could use the `NewRequest()` function from the `http` package, but if you read carefully the documentation, it says that this function is suitable only for outgoing requests, but what we actually want is to emulate an incoming request.

---

## JS Request to Go Request

<!-- .slide: data-auto-animate -->

```go []
import (
    "net/http"
    "net/http/httptest"
    "syscall/js"
)

func JSRequestToGoRequest(jsReq js.Value) http.Request {
    req := httptest.NewRequest(
        jsReq.Get("method").String(),
		jsReq.Get("url").String(),
		// body io.Reader
    )
}
```
<!-- .element: data-id="code" -->

Notes:
- Thankfully, the `httptest` package has the same `NewRequest()` function, which creates requests suitable for passing to an HTTP handler.
- Usually this is usefull for testing purposes, but this is exactly what we want! So let's use this.
- The `NewRequest()` function takes 3 parameters.
- The first two parameters are the request method and URL, which we can simply read from the Javascript `Request` object's properties.
- The third parameter is going to be a little bit trickier, it is an `io.Reader` for the request body.
- How are we going to copy the binary data of the request body from Javascript to Go?

---

## `CopyBytesToGo`

```go
func CopyBytesToGo(dst []byte, src js.Value) int

// src must be a Uint8Array

```

Notes:
- Luckily for us, the `syscall/js` package has a `CopyBytesToGo()` function, just for that.
- It takes a bytes slice as destination, and a reference to a Javascript typed array of unsigned 8 bit integers as source.
- So this is OK, with just a few more plumbing we should be able to copy the body content from Javascript to Go.

---

## JS Request to Go Request

<!-- .slide: data-auto-animate -->

```go [1,2,12|1,3,12|1,4-5,12|1,7-11,12]
func JSRequestToGoRequest(jsReq js.Value) http.Request {
    arrayBuffer := Await(jsReq.Call("arrayBuffer"))
    jsBody := js.Global().Get("Uint8Array").New(arrayBuffer)
	body := make([]byte, jsBody.Get("length").Int())
    js.CopyBytesToGo(body, jsBody)
    
    req := httptest.NewRequest(
        jsReq.Get("method").String(),
		jsReq.Get("url").String(),
		bytes.NewBuffer(body),
    )
}
```
<!-- .element: data-id="code" style="font-size: 0.46em;" -->

Notes:
- We call the `arrayBuffer()` method of the Javascript `Request`, which returns a `Promise` for an `ArrayBuffer`
- We wait for this `Promise` to be resolved, ▶ then we can wrap the `ArrayBuffer`, into an `Uint8Array`.
- ▶ Now we can just create a bytes slice of the same length, and finally call `CopyBytesToGo()`.
- A bytes buffer will do just fine for the body parameter of `NewRequest()`.
- Now we have a Go request, the only important information missing on this request, is the headers.

---
## JS Request to Go Request

<!-- .slide: data-auto-animate -->

```go []
func JSRequestToGoRequest(jsReq js.Value) http.Request {
    // ...

    headersIt := jsReq.Get("headers").Call("entries")
	for {
		entry := headersIt.Call("next")
		if entry.Get("done").Bool() {
			break
		}
		v := entry.Get("value")
		req.Header.Set(v.Index(0).String(), v.Index(1).String())
	}

	return req
}
```
<!-- .element: data-id="code" style="font-size: 0.46em;" -->

Notes:
- The headers are stored in a simple map of strings, both in Javascript and Go, so we just iterate over these and set each header on the Go `Request`.
- And the Go `Request` is now complete.

---

<!-- .slide: data-autoslide="1" -->

---

## Calling the HTTP Handler

<!-- .slide: data-auto-animate -->

```go []
go func() {
    handler.ServeHTTP(/* ResponseWriter */, JSRequestToGoRequest(jsReq))

    // FIXME resolve Response Promise
}()
```
<!-- .element: data-id="code" style="font-size: 0.44em;" -->

Notes:
- We are almost with the step 2 of our plan, actually we are already starting step 3.
- In order to call the `Handler`, we need a value to act as `ResponseWriter`.

---

## Calling the HTTP Handler

<!-- .slide: data-auto-animate -->

```go []
go func() {
    resRecorder := httptest.NewRecorder()

    handler.ServeHTTP(resRecorder, JSRequestToGoRequest(jsReq))

    res := resRecorder.Result()

    // FIXME resolve Response Promise
}()
```
<!-- .element: data-id="code" style="font-size: 0.44em;" -->

Notes:
- For this, we can use the `ResponseRecorder` type from the `httptest` package, which implements `ResponseWriter` and records the response.
- And now we are able to call the `Handler`'s `ServerHTTP()` method.
- Once the `Handler` returns, the `Result()` method of the `ResponseRecorder` allows us to get the HTTP response written by the handler.
- Now the step 2 is really done!
- Step 3, we need to build a Javascript `Response` object from the Go response, in fact the opposite of what we did with the request.

---

<!-- .slide: data-autoslide="1" -->

---

## Go Response to JS Response

<!-- .slide: data-auto-animate -->

```go []
func GoResponseToJSResponse(res *http.Response) js.Value {
	return js.Global().Get("Response").New(
        // body BufferSource = ArrayBuffer | TypedArray
        // init Object
    )
}
```
<!-- .element: data-id="code" style="font-size: 0.38em;" -->

Notes:
- In order to build a Javascript `Response` object, we can use the `Response` constructor which takes 2 parameters, the response body and an init object for additional information such as status code and headers.
- The first parameter accepts several types, one of them is `BufferSource`, actually this is not a real type but either an `ArrayBuffer` or a `TypedArray`.
- `TypedArray` is fine for us, it will allow us to use the `CopyBytesToJS()` function from the `syscall/js` package, which works just like `CopyBytesToGo()` but in the opposite direction.

---

## Go Response to JS Response

<!-- .slide: data-auto-animate -->

```go [1,2-5,13|1,6-12,13]
func GoResponseToJSResponse(res *http.Response) js.Value {
    b, err := ioutil.ReadAll(res.Body)
    if err != nil {
        panic(err)
    }
    body := js.Global().Get("Uint8Array").New(len(b))
    js.CopyBytesToJS(body, b)

	return js.Global().Get("Response").New(
        body,
        // init Object
    )
}
```
<!-- .element: data-id="code" style="font-size: 0.38em;" -->

Notes:
- First we have to read all of the response's body content into a bytes slice.
- ▶ Then create a new `Uint8Array()` of the same length as the slice, and finally call `CopyBytesToJS()`.

---

## Go Response to JS Response

<!-- .slide: data-auto-animate -->

```go [1,9-15,21|1,17-21]
func GoResponseToJSResponse(res *http.Response) js.Value {
    b, err := ioutil.ReadAll(res.Body)
    if err != nil {
        panic(err)
    }
    body := js.Global().Get("Uint8Array").New(len(b))
    js.CopyBytesToJS(body, b)

    init := map[string]interface{}{
        "status": res.StatusCode,
        "headers": make(map[string]interface{}, len(res.Header)),
    }
    for k := range res.Header {
        init["headers"][k] = res.Header.Get(k)
    }

	return js.Global().Get("Response").New(
        body,
        init,
    )
}
```
<!-- .element: data-id="code" style="font-size: 0.38em;" -->

Notes:
- In order to build the init object, we can use a map of string to empty interface, which the `syscall/js` is able to transform to a new Javascript object.
- We only add two values, one for the response status code, and one for the headers, for which we can also use a map of string to empty interface.
- ▶ And finally we can call the `Response` constructor.

---

## Resolve Response Promise

```go []
go func() {
    resRecorder := httptest.NewRecorder()

    handler.ServeHTTP(resRecorder, JSRequestToGoRequest(jsReq))

    res := resRecorder.Result()

    resolve(GoResponseToJSResponse(res))
}()
```
<!-- .element: style="font-size: 0.48em;" -->

Notes:
- Back to the goroutine responsible for handling the request, we can finally resolve the Promise with the Javascript Response we just built.
- And we are done with step 3.

---

## Sending back the Response

<!-- .slide: data-auto-animate -->

📄 `sw.js`
```js []
let goCallback
self.setGoCallback = v => {
    goCallback = v
}

self.addEventListener('fetch', e => {
    goCallback(e.request) // returns a Promise for Response
})
```
<!-- .element: data-id="code" style="font-size: 0.46em;" -->

Notes:
- Back in the ServiceWorker, now we know that the `goCallback()` function returns a promise for a response.
- For the last step, we have to send back the response to the page.

---

## Sending back the Response

<!-- .slide: data-auto-animate -->

📄 `sw.js`
```js []
let goCallback
self.setGoCallback = v => {
    goCallback = v
}

self.addEventListener('fetch', e => {
    e.respondWith(goCallback(e.request))
})
```
<!-- .element: data-id="code" style="font-size: 0.46em;" -->

Notes:
- For that we can directly give the return value of the `goCallback()` function to the `FetchEvent`'s `respondWith()` method.
- And WE ARE DONE!
- Now, the question is, does this actually work?
- Let's find out, with a simple example.

---

## JSON hello example

![Hello example diagram](assets/hello.png)

Notes:
- We have a small index.html page, which sends a POST request to /api/hello, with a JSON body containing a name property.
- This request should be handled by the api WebAssembly binary, which must return a JSON response with a message property.
- And this message will be displayed in an alert

---

## JSON hello example

```go [|12-24|26-27]
package main

import (
	"encoding/json"
	"fmt"
	"net/http"

	wasmhttp "github.com/nlepage/go-wasm-http-server"
)

func main() {
	http.HandleFunc("/hello", func(res http.ResponseWriter, req *http.Request) {
		params := make(map[string]string)
		if err := json.NewDecoder(req.Body).Decode(&params); err != nil {
			panic(err)
		}

		res.Header().Add("Content-Type", "application/json")
		if err := json.NewEncoder(res).Encode(map[string]string{
			"message": fmt.Sprintf("Hello %s!", params["name"]),
		}); err != nil {
			panic(err)
		}
	})

	wasmhttp.Serve(nil)
	select {} // block the main goroutine
}
```
<!-- .element: style="font-size: 0.28em;" -->

Notes:
- On the Go side, ▶️ we only have one `HandleFunc`, which decodes the request body, then formats a hello message in the response body.
- ▶️ And of course the call to `wasmhttp.Serve()`.
- We must also add something to block the main goroutine, here I used an empty select, which is not pretty, but works.

---

## JSON hello example

👇 Click the gopher! 👇

[![Hang gliding gopher](assets/hang_glider_gopher_purple.png) <!-- .element: style="width: 400px;" -->](https://nlepage.github.io/go-wasm-http-server/hello/)

Notes:
- Let's try this out!

---

## 😺 Catption example

<!-- .slide: data-auto-animate -->

Notes:
- Now let's come back to my little project, which was the catption server.
- The hello example only used the WebAssembly binary to exchange JSON messages.
- But this time the WebAssembly binary will actually serve the HTML page containing the form.

---

## 😺 Catption example

<!-- .slide: data-auto-animate -->

![Hello example diagram](assets/catption.png)

Notes:
- So what we can do is create a small HTML file, which will only be responsible for registering the ServiceWorker.
- Once the ServiceWorker is activated, it will trigger a reload of the same address, which will now be served from the WebAssembly binary.

---

## 😺 Catption example

👇 Click the gopher! 👇

[![](assets/slide_gopher_blue.png) <!-- .element: style="width: 300px;" -->](https://nlepage.github.io/catption/wasm/)

Notes:
- Let's see if this works.

---

## Stateless vs Stateful

Notes:
- So far, in the examples I have been using, the server is stateless.
- 🖵 This means it can be stopped and restarted as much as we want.
- And this is actually a good thing, because the lifecycle of ServiceWorkers is event based.
- The browser will start the ServiceWorker only when it is necessary, for example when a FetchEvent is received.
- Then if no more events are received, after some time the browser may decide to stop the ServiceWorker and kill the WebAssembly binary.
- So if my server is stateful, the state will be lost.
- So how can we work around this? Well there is no real solution here.
- The ServiceWorkers specification does not allow to keep a ServiceWorker alive if it has no clients.
- This means we need at least one page to be loaded in the scope of the ServiceWorker, if we want to be able to keep it alive.
- So the most we can do, is send periodic messages from the page to the ServiceWorker, in order to keep the browser from stopping the ServiceWorker as long as the page is loaded.
- 🖵 In summary, it is not really possible to have a stateful server leaving in a ServiceWorker.

---

## The project

[https://github.com/nlepage/go-wasm-http-server/](https://github.com/nlepage/go-wasm-http-server/#readme)

Notes:
- More information is available on the github project page.
- Including a usage section, to help you do the same with your own project.
- If you give it a try please let me know, I'll be glad to have your feedback.

---
## Conclusion

Notes:
- 🖵 In conclusion, as you would expect, it is not really possible to deploy a Go HTTP server in a browser, however it is possible to execute Go HTTP handlers in a ServiceWorker.
- Using build conditions allows to reuse most of the code we usually write for building a Go HTTP server, but targetting WebAssembly requires this code to be compatible.
- And finally, we saw that deploying a long-running stateful server in a ServiceWorker is not a good idea, because of the lifecycle of ServiceWorkers.

---
## Thank you

Notes:
- Thank you for listening, thank you to FOSDEM organizers, and to the Go devroom organizers.
- And a big thank you to all those who helped me prepare this talk.
