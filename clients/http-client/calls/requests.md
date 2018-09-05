---
title: Requests
caption: HTTP Client Requests
category: clients
permalink: /clients/http-client/calls/requests.html
---

## Simple requests

The basic usage is *super* simple: you just have to instantiate an `HttpClient` instance,
specifying an engine, for example [`Apache`](#apache), [`Jetty`](#jetty)
or [`CIO`](#cio), and start making requests using one of the many convenience methods available.

Since Ktor 0.9.3, you can omit the engine, and Ktor will choose an engine among the ones that are available
from the included artifacts using a ServiceLoader. 

First you need to instantiate the client:   

```kotlin
val client = HttpClient(Apache)
```

Then, to perform a `GET` request fully reading a `String`:

```kotlin
val htmlContent = client.get<String>("https://en.wikipedia.org/wiki/Main_Page")
```

And in the case you are interested in the raw bits, you can read a `ByteArray`:

```kotlin
val bytes: ByteArray = client.call("http://127.0.0.1:8080/").response.readBytes()
```

It is possible to customize the request a lot and
to stream the request and response payloads, but you can also just call a convenience
extension method like `HttpClient.get` to do a `GET` request to receive
the specified type directly (for example `String`).
{: .note}

## Custom requests

We cannot live only on *get* requests, Ktor allows you to build complex
requests with any of the HTTP verbs, with the flexibility to process responses in many ways.

### The `call` method

The HttpClient `call` method, returns an `HttpClientCall` and allows you to perform
simple untyped requests.

You can read the content using `response: HttpResponse`.
For further information, check out the [receiving content using HttpResponse](#HttpResponse) section. 

```kotlin
val call = client.call("http://127.0.0.1:8080/") {
    method = HttpMethod.Get
}
println(call.response.readText())
```

### The `request` method

In addition to call, there is a `request` method for performing a typed request,
[receiving a specific type](#receive) like String, HttpResponse, or an arbitrary class.
You have to specify the URL and the method when building the request. 

```kotlin
val call = client.request<String> {
    url(URL("http://127.0.0.1:8080/"))
    method = HttpMethod.Get
}
```

### The `post` and `get` methods

Similar to `request`, there are several extension methods to perform requests
with the most common HTTP verbs: `GET` and `POST`.

```kotlin
val text = client.post<String>("http://127.0.0.1:8080/")
```

When calling request methods, you can provide a lambda to build the request
parameters like the URL, the HTTP method (verb), the body, or the headers.
The `HttpRequestBuilder` looks like this:

```kotlin
class HttpRequestBuilder : HttpMessageBuilder {
    val url: URLBuilder
    var method: HttpMethod
    val headers: HeadersBuilder
    var body: Any = EmptyContent
    val executionContext: CompletableDeferred<Unit>
    fun header(key: String, value: String)
    fun headers(block: HeadersBuilder.() -> Unit)
    fun url(block: URLBuilder.(URLBuilder) -> Unit)
}
```

The `HttpClient` class only offers some basic functionality, and all the methods for building requests are exposed as extensions.\\
You can check the standard available [HttpClient build extension methods](https://github.com/ktorio/ktor/blob/master/ktor-client/ktor-client-core/src/io/ktor/client/request/builders.kt).
{: .note.api}

### Specifying custom headers
{: #custom-headers}

When building requests with `HttpRequestBuilder`, you can set custom headers.
There is a final property `val headers: HeadersBuilder` that inherits from `StringValuesBuilder`.
You can add or remove headers using it, or with the `header` convenience methods.

```kotlin
// this : HttpMessageBuilder

// Convenience method to add a header
header("My-Custom-Header", "HeaderValue")

// Calls methods from the headers: HeadersBuilder to manipulate the headers
headers.clear()
headers.append("My-Custom-Header", "HeaderValue")
headers.appendAll("My-Custom-Header", listOf("HeaderValue1", "HeaderValue2"))
headers.remove("My-Custom-Header")

// Applies the headers with the `headers` convenience method
headers { // this: HeadersBuilder
    clear()
    append("My-Custom-Header", "HeaderValue")
    appendAll("My-Custom-Header", listOf("HeaderValue1", "HeaderValue2"))
    remove("My-Custom-Header")
}
``` 


## Specifying a body for requests

For `POST` and `PUT` requests, you can set the `body` property:

```kotlin
client.post<Unit> {
    url(URL("http://127.0.0.1:8080/"))
    body = // ...
}
```

The `HttpRequestBuilder.body` property can be a subtype of `OutgoingContent` as well as a `String` instance:

* `body = "HELLO WORLD!"`
* `body = TextContent("HELLO WORLD!", ContentType.Text.Plain)`
* `body = ByteArrayContent("HELLO WORLD!".toByteArray(Charsets.UTF_8))`
* `body = LocalFileContent(File("build.gradle"))`
* `body = JarFileContent(File("myjar.jar"), "test.txt", ContentType.fromFileExtension("txt").first())`
* `body = URIFileContent(URL("https://en.wikipedia.org/wiki/Main_Page"))`

If you install the *JsonFeature*, and set the content type to `application/json`
you can use arbitrary instances as the `body`, and they will be serialized as JSON:

```kotlin
data class HelloWorld(val hello: String)

val client = HttpClient(Apache) {
    install(JsonFeature) {
        serializer = GsonSerializer {
            // Configurable .GsonBuilder
            serializeNulls()
            disableHtmlEscaping()
        }
    }
}

client.post<Unit> {
    url(URL("http://127.0.0.1:8080/"))
    contentType(ContentType.Application.Json) // Required
    body = HelloWorld(hello = "world")
}
```

Remember that your classes must be *top-level* to be recognized by `Gson`. \\
If you try to send a class that is inside a function, the feature will send a *null*.
{: .note}

## Uploading multipart/form-data
{: #multipart-form-data }

Right now, Ktor HttpClient doesn't include functionality to do this directly, but you can do something like this:

```kotlin
val result = client.post<HttpResponse>("http://127.0.0.1:$port/handler") {
    body = MultiPartContent.build {
        add("user", "myuser")
        add("password", "password")
        add("file", byteArrayOf(1, 2, 3, 4), filename = "binary.bin")
    }
}  
```

By including this small boilerplate to your code:

<https://github.com/ktorio/ktor-samples/blob/183dd65e39565d6d09682a9b273937013d2124cc/other/client-multipart/src/MultipartApp.kt#L57>