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
- Self presentation
- Talk about...
- First, thanks (Go devroom, Z)
- Why? Explain. Example QR code generator. Catption demo?

---

## Deploy a Go HTTP server in your browser ?

Notes:
- Since Go 1.11, running a program written in Go in a browser is possible, if you build it to WebAssembly.
- (Quickly explain WebAssembly?)
- You get a wasm binary which can be downloaded and executed by a browser.
- Of course, Go code executed in a browser has exactly the same limitations as any JavaScript code executed in the same browser.
- For example, it will not be able to use the `io` package to access the client's file system.
- So, it will also not be able to actually start an HTTP server in the browser.
- However, when an HTTP request is sent from a web page, there is one case when it will not actually reach the server.
- It may be intercepted by a service worker, which in most cases will respond to the request using for example some cache, or forwarding it to the actual server.
- Now I think you are starting to see where I am going with this.
- Would it be possible to execute a Go wasm binary in a service worker and use it to handle HTTP requests?

---

## Responding to an HTTP request from a service worker

Notes:
- First, let's have a quick look at how it is possible to respond to an HTTP request from a service worker.
- When a service worker intercepts an HTTP request, it receives a `FetchEvent`.
- The `FetchEvent` contains a `Request` object which holds all the information we need about the request (URL, method, headers), and also the body contents if any.
- The `FetchEvent` also has a `respondWith()` method, which accepts one parameter of type `Response` or `Promise` for a `Response`.
- So, by reading the `Request` object and building a `Response` object to give to `respondWith()`, we should be able to respond to an HTTP request from a service worker.
- However, what we want is to delegate this task to a Go wasm binary.

---

## Building a Go HTTP server

Notes:
- Now let's take a step back and review how we usually build a Go HTTP server.
- The most straightforward way is to use the `net/http` package.
- First you define your HTTP handlers using `Handle()` which accepts a `Handler` or `HandleFunc()` which accepts a function.
- Indeed, handlers are simple functions which receive a `Request` and a `ResponseWriter` ; so we can already see that this looks a lot like the service worker's fetch event
- You may also choose to define your handlers using some third party libraries, such as `gorilla/mux`.
- Then, once your handlers are defined, in most cases you will call `ListenAndServe()`, which will start listening for HTTP requests and use your handlers to respond to these.
- In our case, we would like to reuse as much as possible of this code we wrote, but use it to respond to a request intercepted by a SW.
- Of course, the one thing we are not going to be able to reuse is the call to `ListenAndServe()`.
- But, this is OK, because we are going to take a pretty radical(?) shortcut.
- Ususally when we call `ListenAndServe()`, it takes care of a lot of things for us under the hood, and for each request it calls the `Handler` if we gave one in parameter or `DefaultServeMux`, which is the default `Handler`.
- So, here is our shortcut, we are going to directly call the `ServeHTTP()` method of our `Handler` or of `DefaultServeMux`.
- FIXME...