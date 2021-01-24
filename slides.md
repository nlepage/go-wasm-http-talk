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
- Hi everybody, my name is Nicolas Lepage, I am a developer at Zenika IT in France. I work mainly with Javascript, and I also like experimenting with Go.
- So, today I am going to talk about, deploying a Go HTTP server in your browser.

## Why?

Demo gifs...

Notes:
- But first, you may be wondering, why? Why would you want to do that?
- Well, imagine I have this nice little project, it is a Go CLI tool called catption, that helps me add a text at the bottom or at the top of a JPEG image.
- And it also has an http subcommand, which starts a little server, with an HTML form where you can do the same.
- Now, I would like to put up a little demonstration server for my project.
- But I don't want to actually deploy a Go server for that.
- So that's how I started wondering if we could deploy a Go HTTP server in a browser.

---

## Deploy a Go HTTP server in your browser ?

Notes:
- Since Go 1.11, running a program written in Go in a browser is possible, if you build it to WebAssembly.
- (Quickly explain WebAssembly?)
- You get a WebAssembly binary which can be downloaded and executed by a browser.
- Of course, Go code executed in a browser has exactly the same limitations as any JavaScript code executed in the same browser.
- For example, it will not be able to use the `os` package to access the client's file system.
- So, it will also not be able to actually start an HTTP server in the browser.
- However, when an HTTP request is sent from a web page, there is one case when it will not actually reach the server.
- It may be intercepted by a service worker, which is usually allows web applications to work offline.
- Now I think you are starting to see where I am going with this.
- Would it be possible to execute a Go WebAssembly binary in a service worker and use it to handle HTTP requests?

---

## Disclaimer

Notes:
- A little warning before we go any further.
- When you are building to WebAssembly, you have to make sure all your code and dependencies are compatible.
- This means for example that you cannot rely on C bindings, or system dependencies, or a database server.
- That being said, today I'm going to focus on the HTTP side of things.

---

## Responding to an HTTP request from a service worker

Notes:
- First, let's have a quick look at how it is possible to respond to an HTTP request from a service worker.
- When a service worker intercepts an HTTP request, it receives a `FetchEvent`.
- The `FetchEvent` contains a `Request` object which holds all the information we need about the request (URL, method, headers), and also the body contents if any.
- The `FetchEvent` also has a `respondWith()` method, which accepts one parameter of type `Response` or `Promise` for a `Response`.
- So, by reading the `Request` object and building a `Response` object to give to `respondWith()`, we should be able to respond to an HTTP request from a service worker.
- However, what we want is to delegate this task to a Go WebAssembly binary.

---

## Building a Go HTTP server

Notes:
- Now let's take a step back and review how we usually build a Go HTTP server.
- The most straightforward way is to use the `net/http` package.
- First we define HTTP handlers using `Handle()` which accepts a `Handler` or `HandleFunc()` which accepts a simple function.
- Indeed, handlers are simple functions which receive a `Request` and a `ResponseWriter`; so we can already see that this looks a lot like the service worker's `FetchEvent`
- We may also choose to define handlers using some third party libraries, such as `gorilla/mux`.
- Then, once our handlers are defined, in most cases we will call `ListenAndServe()`, which will start listening for HTTP requests and use our handlers to respond to these.
- In our case, we would like to reuse as much as possible of this code we wrote, but use it to respond to a request intercepted by a SW.
- Of course, the one thing we are not going to be able to reuse is the call to `ListenAndServe()`.
- But, this is OK, because we are going to take a pretty radical(?) shortcut.
- Ususally when we call `ListenAndServe()`, it takes care of a lot of things for us under the hood, and for each request it calls the `Handler` if we gave one in parameter or `DefaultServeMux`, which is the default `Handler`.
- So, here is our shortcut, we are going to directly call the `ServeHTTP()` method of the `Handler` or of `DefaultServeMux`.

---

## Adding js/wasm target

Notes:
- This is nice, because provided our handlers are WebAssembly compatible, we can keep and reuse these as they are.
- Now, ideally we still want to be able to build standard binaries of our server working for linux, mac OS, or whatever, and also the WebAssembly binary.
- For this, we can move the `ListenAndServe()` call into its own file, and use build tags to tell the Go compiler that this file is not compatible with WebAssembly.
- Then we can create a specific file for WebAssembly, this time using the file naming convention instead of the build tags, to tell the compiler that this file is compatible only with WebAssembly.
- And in this file we are going to use our own API, which will probably look something like this, so a `Serve()` function which takes only one `Handler` parameter, but no address parameter.

---

## Implement `wasmhttp.Serve()`

Notes:
- Now let's dive in the implementation of this `Serve()` function.
- It needs to receive the Javascript request objects, so from the Javascript point of view, the ServiceWorker needs to have some callback from the Webassembly binary, in order to call it with each `Request` object.
- At the moment, Go Webassembly binaries have no way to export functions or other values.
- So in order to work around this, the `Serve()` function will have to register a callback function with the ServiceWorker.
- The `syscall/js` package allows to create such callback functions using `FuncOf()`.
- Our callback function will have one request parameter of type `js.Value`.
- And it will return one value, which will have to be a Javascript `Promise` for a Javascript `Response` object.
- Then the `Serve` function can register the callback with the ServiceWorker, by calling a setter function previously declared in the ServiceWorker's global scope.
- And this it, the ServiceWorker is now able to forward the FetchEvent's requests to the WebAssembly binary.

## Implement callback function

Notes:
- Next, we have to implement the callback function.
- We said that this callback function must return a `Promise` for a Javascript `Response` object.
- That means that this callback function is asynchronous.
- So we need to do two things: create and return a new `Promise`, and start a new goroutine that will be responsible for resolving this promise.
- In fact, we can create the new goroutine inside of the callback we give to the `Promise` constructor.
- Creating a new goroutine actually makes sense, because this is how a Go HTTP server usually works, it ceates a new goroutine for each request.
- And this is it for the callback function, the rest of the work will be carried out by the new goroutine we started.

## Javascript Request to Go Request

Notes:
- Now the first thing we need to do in this new goroutine, is create an instance of a Go `http.Request`, from the Javascript `Request` object.
- We could use the `NewRequest()` function from the `http` package, but if you read carefully the documentation, it says that this function is suitable only for outgoing requests, or client requests if you prefer.
- However the `httptest` package has the same `NewRequest()` function, which creates requests suitable for passing to an HTTP handler.
- So this is exactly what we want!
- The `NewRequest()` function takes 3 parameters.
- The first two parameters are the request method and URL, which we can simply read from the Javascript `Request` object's properties.
- The third parameter is going to be a little bit trickier, it is an `io.Reader` for the request body.
- How are we going to copy the raw binary data of the request body from Javascript to Go?
- Luckily for us, the `syscall/js` package has `CopyBytesToGo()` function for that.
- It takes a byte slice as destination, and a reference to a Javascript `Uint8Array` as source.
- So this is OK, with just a few more plumbing we should be able to copy the body content from Javascript to Go.
- We call the `arrayBuffer()` method of the Javascript `Request`, which returns a `Promise` for an `ArrayBuffer`, we wait for this `Promise` to be resolved, then we can wrap the `ArrayBuffer`, into an `Uint8Array`.
- Now we have a Go request, the only important information missing, is the headers of the request.
- The headers are stored in a simple map of strings, both in Javascript and Go, so we just iterate and set each header on our new Go `Request`.
- Now we are done!

## Calling the HTTP Handler

Notes:
- We are almost ready to call the `Handler`.
- We only need a value for its second parameter, which type is the `ResponseWriter` interface.
- For this, the `httptest` package has a `ResponseRecorder` type, which implements `ResponseWriter` and records the response.
- And now we are able to call the `Handler`'s `ServerHTTP()` method.
- Once the `Handler` returns, the last thing we have to do is read the result of the `ResponseRecorder`, which is an `http.Response` struct, and build a Javascript Object `Response` from it.

## Go response to Javascript Response

Notes:
 - FIXME