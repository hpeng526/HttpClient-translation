### 第一章 基础

#### 1.1 执行请求

HttpClient 最重要的功能就是执行 HTTP 方法。一个 HTTP 方法的执行涉及到多个 HTTP 请求和响应，通常是 HttpClient 内部处理。用户只是提供一个请求对象去执行，而 HttpClient 是发送该请求到目标服务器并返回对应的返回对象，或者抛出异常，如果执行不成功的话。
自然的，HttpClient API 的主要切入点就是上面描述的 HttpClient 的接口。
下面是个最简单的请求处理的例子

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/");
CloseableHttpResponse response = httpclient.execute(httpget);
try {
    <...>
} finally {
    response.close();
}
```

##### 1.1.1 HTTP 请求

所有的 HTTP 请求都有包含方法名，请求 URI 和 HTTP 协议版本的请求行。
HttpClient 支持 HTTP/1.1 规范的所有 HTTP 方法的方法块: GET，HEAD，POST，PUT，DELETE，TRACE 和 OPTIONS。为每一个方法类型提供了一个特定的类: HttpGet，HttpHead，HttpPost，HttpPut，HttpDelete，HttpTrace 和 HttpOptions。
Request-URI 是统一资源标识符，被用来标识其应用请求的资源。HTTP 请求 URIs 包括了请求协议，主机名，可选端口，资源路径，可选查询和可选片段。

```java
HttpGet httpget = new HttpGet(
     "http://www.google.com/search?hl=en&q=httpclient&btnG=Google+Search&aq=f&oq=");
```

HttpClient 提供了 URIBuilder 工具类来简化请求 URI 的创建和修改

```java
URI uri = new URIBuilder()
        .setScheme("http")
        .setHost("www.google.com")
        .setPath("/search")
        .setParameter("q"，"httpclient")
        .setParameter("btnG"，"Google Search")
        .setParameter("aq"，"f")
        .setParameter("oq"，"")
        .build();
HttpGet httpget = new HttpGet(uri);
System.out.println(httpget.getURI());
```

输出 >

```bash
http://www.google.com/search?q=httpclient&btnG=Google+Search&aq=f&oq=
```

##### 1.1.2 HTTP 响应

HTTP 响应是指在接受和执行了请求消息之后，返回给客户端的消息。该消息的第一行包含了协议的版本后面跟着数字类型的状态码和相关的文本短语。

```java
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1,
HttpStatus.SC_OK，"OK");

System.out.println(response.getProtocolVersion());
System.out.println(response.getStatusLine().getStatusCode());
System.out.println(response.getStatusLine().getReasonPhrase());
System.out.println(response.getStatusLine().toString());
```

输出 >

```bash
HTTP/1.1
200
OK
HTTP/1.1 200 OK
```

##### 1.1.3

HTTP消息可以包含多个描述该消息的属性标题的诸如内容长度，内容类型等。HttpClient 提供方法来检索，添加，删除和枚举头。

```java
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1,
    HttpStatus.SC_OK，"OK");
response.addHeader("Set-Cookie",
    "c1=a; path=/; domain=localhost");
response.addHeader("Set-Cookie",
    "c2=b; path=\"/\"，c3=c; domain=\"localhost\"");
Header h1 = response.getFirstHeader("Set-Cookie");
System.out.println(h1);
Header h2 = response.getLastHeader("Set-Cookie");
System.out.println(h2);
Header[] hs = response.getHeaders("Set-Cookie");
System.out.println(hs.length);
```

输出 >

```bash
Set-Cookie: c1=a; path=/; domain=localhost
Set-Cookie: c2=b; path="/"，c3=c; domain="localhost"
2
```

获取头最高效的方式是试用 HeaderIterator 接口.

```java
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1,
    HttpStatus.SC_OK，"OK");
response.addHeader("Set-Cookie",
    "c1=a; path=/; domain=localhost");
response.addHeader("Set-Cookie",
    "c2=b; path=\"/\"，c3=c; domain=\"localhost\"");

HeaderIterator it = response.headerIterator("Set-Cookie");

while (it.hasNext()) {
    System.out.println(it.next());
}
```

输出 >

```bash
Set-Cookie: c1=a; path=/; domain=localhost
Set-Cookie: c2=b; path="/"，c3=c; domain="localhost"
```

他还提供了方便的方法来解析 HTTP 头元素

```java
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1,
    HttpStatus.SC_OK，"OK");
response.addHeader("Set-Cookie",
    "c1=a; path=/; domain=localhost");
response.addHeader("Set-Cookie",
    "c2=b; path=\"/\"，c3=c; domain=\"localhost\"");

HeaderElementIterator it = new BasicHeaderElementIterator(
    response.headerIterator("Set-Cookie"));

while (it.hasNext()) {
    HeaderElement elem = it.nextElement();
    System.out.println(elem.getName() + " = " + elem.getValue());
    NameValuePair[] params = elem.getParameters();
    for (int i = 0; i < params.length; i++) {
        System.out.println(" " + params[i]);
    }
}
```

输出 >

```bash
c1 = a
path=/
domain=localhost
c2 = b
path=/
c3 = c
domain=localhost
```

##### 1.1.4 HTTP 实体

HTTP 消息可以在请求或返回中携带相应关联内容的实体。实体可以在一些请求和返回中找到，它们是可选的。使用了实体的请求被称为实体的封闭请求（enclosing requests）。HTTP 规范定义了两种封闭实体请求的方法：POST 和 PUT。响应通常包含封闭实体。这个规则也有很多例外，像是 HEAD 方法和 204 No Content， 304 Not Modified， 205 Reset Content 的响应。
HttpClient 根据它们的内容来源区分以下三种实体

* 流（streamed）：内容是从流中获取的或者是动态生成的。尤其是这种从 HTTP 响应接受回来的实体。流实体通常是不能重复的。
* 自包含（self-contained）：内容是位于内存或者是从独立链接或其他实体中获得。自包含实体通常是可以重复生成。这种类型通常被用于封闭请求的 HTTP 请求中。
* 包装（wrapping）：从另一个实体中获取。

当从 HTTP 响应处理内容时，这种区别对于连接的管理是很重要的。对于由应用程序创建并使用 HttpClient 发送的请求实体，流和自包含的区分是不重要的。在这种情况下，建议使用不可重复的实体如流，或者那别可重复的像自包含。

##### 1.1.4.1 可重复实体

一个可重复实体意味着内容可以重复的读。这只有在自包含实体中才有可能（例如：ByteArrayEntity 或者 StringEntity)

##### 1.1.4.2 使用 HTTP 实体
由于实体都可以表示二进制和字符内容，因此它对字符编码支持（对于后者，即字符内容的支持）。
当执行一个封闭请求或者已经成功的请求响应体被用于发回结果时，实体就会被创建。
为了从实体中读内容，可以从`HttpEntity#getContent()`方法来从输入流获取，他会返回`java.io.InputStream`对象。或者提供一个输出流到`Httpentity#writeTo(OutputStream)`方法，这会一次性返回所有内容，并写入给的流中。
当实体已经接受到传入的消息，方法`HttpEntity#getContentType()`和`HttpEntity#getContentLength()`可以用于读取通用元数据，例如`Content-Type`和`Content-Length`头部（如果它们是可用的）。当`Content-Type`头包含对文本`mime`字符编码像`text/plain`和`text/html`时，`HttpEntity#getContentEncoding()`方法可以用来读取这些信息。如果这些头是不可用的，此时长度`-1`会返回，内容类型将会返回`NULL`。如果`Content-Type`头是可用的，一个`Header`对象将会返回。
当为一个穿出消息创建实体时，元数据必须有实体创建者提供。

```java
StringEntity myEntity = new StringEntity("important message",
   ContentType.create("text/plain", "UTF-8"));

System.out.println(myEntity.getContentType());
System.out.println(myEntity.getContentLength());
System.out.println(EntityUtils.toString(myEntity));
System.out.println(EntityUtils.toByteArray(myEntity).length);
```

输出 >

```bash
Content-Type: text/plain; charset=utf-8
17
important message
17
```

##### 1.1.5 确保低级别的资源释放

为了确保正确释放系统的资源，必须关闭关联实体内容的流或者关闭响应自身。

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/");
CloseableHttpResponse response = httpclient.execute(httpget);
try {
    HttpEntity entity = response.getEntity();
    if (entity != null) {
        InputStream instream = entity.getContent();
        try {
            // do something useful
        } finally {
            instream.close();
        }
    }
} finally {
    response.close();
}
```

关闭内容流和关闭响应自身之间的差别在于，前者会尝试通过小号实习内容来保持下层连接可用而后者则会直接关系和丢弃连接。
请记住`HttpEntity#writeTo(OutputStream)`方法当实体被完全写出的时候需要确保系统资源正确的释放，如果此方法获得的实例`java.io.InputStream`调用了`HttpEntity#getContent()`的方法，它也有望在`finally`子句来关闭流。
当使用流实体时，可以使用`EntityUtils#consume(HttpEntity)`方法来确保实体内容被完全的消费同时依赖的流被关闭。
有这样一种情况，当只有一小部分返回内容需要被接受，消耗后面剩下的内容让连接可以重复使用相当影响性能，这种情况可以通过关闭响应来中止内容流。

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/");
CloseableHttpResponse response = httpclient.execute(httpget);
try {
    HttpEntity entity = response.getEntity();
    if (entity != null) {
        InputStream instream = entity.getContent();
        int byteOne = instream.read();
        int byteTwo = instream.read();
        // Do not need the rest
    }
} finally {
    response.close();
}
```

这连接将不会被重复使用，而且所有层次的资源都会被正确的释放。

##### 1.1.6 消耗实体内容

消耗实体内容的推荐方式是使用它的`HttpEntity#getContent()`或者`HttpEntity#writeTo(OutputStream)`方法。HttpClient 也有 `EntityUtils` 类，它暴露几个静态方法让我们更加容易的从实体读取内容和信息。可以使用这个类里面的方法用字符串或者字节数组接收整个内容来替代字节从`java.io.InputStream`读取。但是`EntityUtils`只有在响应实体是来自受信的 HTTP 服务器和长度是已知有限的情况下才推荐使用。

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/");
CloseableHttpResponse response = httpclient.execute(httpget);
try {
    HttpEntity entity = response.getEntity();
    if (entity != null) {
        long len = entity.getContentLength();
        if (len != -1 && len < 2048) {
            System.out.println(EntityUtils.toString(entity));
        } else {
            // Stream content out
        }
    }
} finally {
    response.close();
}
```

在一些情况下，可能不止一次需要读取整个实体内容。这种情况下，实体内容必须以一种方式缓存起来在内存或者磁盘上。最简单的方式就是用`BufferedHttpEntity`来包装源实体。这会使得整个源实体都被读进内存缓冲区。其他任何方式都会包含源实体。

```java
CloseableHttpResponse response = <...>
HttpEntity entity = response.getEntity();
if (entity != null) {
    entity = new BufferedHttpEntity(entity);
}
```

##### 1.1.7 生成实体内容

HttpClient 提供了一些通过 HTTP 连接高效流出内容的类。这些类可以与封闭请求像 POST 和 PUT 请求实体关联，使得实体内容装入流出的 HTTP 请求。HttpClient 为大部分通用数据容器`string, byte array, input stream, and file`提供了类`StringEntity, ByteArrayEntity, InputStringEntity and FileEntity`。

```java
File file = new File("somefile.txt");
FileEntity entity = new FileEntity(file,
    ContentType.create("text/plain", "UTF-8"));        

HttpPost httppost = new HttpPost("http://localhost/action.do");
httppost.setEntity(entity);
```

请记住`InputStreamEntity`是不可重复的，因为它只能从底层数据流中读取一次。通常推荐实现一个自定义的`HttpEntity`类，这是自包含的类替代通用的`InputStreamEntity`。`FileEntity`可以是一个很好的开始。

##### 1.1.7.1 HTML 表单

许多应用需要模拟表单提交，例如，为了登陆一个网页应用或者提交输入数据。HttpClient 提供类实体类`UrlEncodedFormEntity`来简化此过程。

```java
List<NameValuePair> formparams = new ArrayList<NameValuePair>();
formparams.add(new BasicNameValuePair("param1", "value1"));
formparams.add(new BasicNameValuePair("param2", "value2"));
UrlEncodedFormEntity entity = new UrlEncodedFormEntity(formparams, Consts.UTF_8);
HttpPost httppost = new HttpPost("http://localhost/handler.do");
httppost.setEntity(entity);
```

`UrlEncodedFormEntity`实例会使用所谓的`URL encoding`来编码参数和生成以下内容

```bash
param1=value1&param2=value2
```

##### 1.1.7.2 内容分块

通常推荐让 HttpClient 选择基于 HTTP 消息属性最适当的传输方式。但是可以通过设置`HttpEntity#setChunked()`为`true`来告诉 HttpClient 使用该块编码。请注意 HttpClient 只会把这个标志当成一个提示。这个值将会被忽略当使用不支持块编码的 HTTP 协议，例如 HTTP/1.0。

```java
StringEntity entity = new StringEntity("important message",
        ContentType.create("plain/text", Consts.UTF_8));
entity.setChunked(true);
HttpPost httppost = new HttpPost("http://localhost/acrtion.do");
httppost.setEntity(entity);
```

##### 1.1.8 响应处理

最简单和最方便处理响应的方式就是使用`ResponseHandler`接口，它提供了`handleResponse(HttpResponse response)`方法。这个方法完全让用户不用担心连接管理。当使用`ResponseHandler`，HttpClient 会自动接管连接的释放无论是请求执行成功了或是导致异常了。

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/json");

ResponseHandler<MyJsonObject> rh = new ResponseHandler<MyJsonObject>() {

    @Override
    public JsonObject handleResponse(
            final HttpResponse response) throws IOException {
        StatusLine statusLine = response.getStatusLine();
        HttpEntity entity = response.getEntity();
        if (statusLine.getStatusCode() >= 300) {
            throw new HttpResponseException(
                    statusLine.getStatusCode(),
                    statusLine.getReasonPhrase());
        }
        if (entity == null) {
            throw new ClientProtocolException("Response contains no content");
        }
        Gson gson = new GsonBuilder().create();
        ContentType contentType = ContentType.getOrDefault(entity);
        Charset charset = contentType.getCharset();
        Reader reader = new InputStreamReader(entity.getContent(), charset);
        return gson.fromJson(reader, MyJsonObject.class);
    }
};
MyJsonObject myjson = client.execute(httpget, rh);
```

#### 1.2 HttpClient 接口

HttpClient 接口代表了 HTTP 请求执行的最重要的约定。它在请求执行中没有强加约束或特定的细节，让具体的连接管理，状态管理，认证和重定向留给各自的实现中。这会使得更容易的用额外的方法装饰接口，例如响应内容的缓存。
通常来说 HttpClient 的实现只是作为一个门面，负责特定方面的 HTTP 协议，例如重定向，身份认证，对连接保持和连接存活长短做抉择。这使得用户可以选择性的用自定义的，应用相关的来替换默认的实现。

```java
ConnectionKeepAliveStrategy keepAliveStrat = new DefaultConnectionKeepAliveStrategy() {

    @Override
    public long getKeepAliveDuration(
            HttpResponse response,
            HttpContext context) {
        long keepAlive = super.getKeepAliveDuration(response, context);
        if (keepAlive == -1) {
            // Keep connections alive 5 seconds if a keep-alive value
            // has not be explicitly set by the server
            keepAlive = 5000;
        }
        return keepAlive;
    }

};
CloseableHttpClient httpclient = HttpClients.custom()
        .setKeepAliveStrategy(keepAliveStrat)
        .build();
```

##### 1.2.1 HttpClient 的线程安全

HttpClient 的实现是线程安全的。建议同一个实例可以用于多个请求的执行。

##### 1.2.2 HttpClient 资源的重新分配

当一个`CloseableHttpClient`的实例已经不在需要，和即将超过连接管理关联的范围时，必须使用`CloseableHttpClient#close()`方法来关闭它。

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
try {
    <...>
} finally {
    httpclient.close();
}
```

#### 1.3 HTTP 执行上下文

HTTP 起初被设计为无状态的，面向请求-响应的协议。但是实际上，应用通常要保持状态信息在逻辑相关的请求响应过程中。为了使应用保持一个处理状态，HttpClient允许在一个特定的执行上下文中执行 HTTP 请求，这叫 HTTP 上下文（HTTP context）。多个逻辑相关的请求可以在一个逻辑会话中参与，如果同一个context在连续的HTTP请求中被重复使用。Http 上下文的功能有点像 `java.uitl.Map<String, Object>`。它是简单的任意值的集合。一个应用程序可以在请求前填充上下文，或者在执行完成后检查上下文。
HttpContext可以包含任意的对象，因此它在线程共享时是不安全的。建议每一个线程在执行时保持自己的上下文。
在 HTTP 请求的过程中，HttpClient会添加下列属性到执行上下文里：
* HttpConnetion 表示到目标服务器的实际连接。
* HttpHost 表示连接目标
* HttpRoute 表示完整的连接路由
* HttpRequest 表示实际的 HTTP 请求。在执行上下文中最后的 HttpRequest对象始终表示消息状态，它才是被发往目标服务器的对象。每一个默认的 HTTP/1.0 HTTP/1.1 使用相对的请求 URI，但是如果请求是通过代理或者非隧道模式模式，则 URI 是绝对的。
* java.lang.Boolean 对象表示实际是否已经完全传输到连接目标的标志。
* RequestConfig 对象表示实际请求的配置。
* java.util.List<URI> 对象，表示所有重定向请求执行过程中的地址集合。
可以使用`HttpClientContext`适配器类来简化与上下文状态的交互。

```java
HttpContext context = <...>
HttpClientContext clientContext = HttpClientContext.adapt(context);
HttpHost target = clientContext.getTargetHost();
HttpRequest request = clientContext.getRequest();
HttpResponse response = clientContext.getResponse();
RequestConfig config = clientContext.getRequestConfig();
```

在逻辑相关的绘画中多个顺序的请求应该在同一个 HttpContext 实例中来确保在请求间自动保持的会话上下文传播和状态信息。
下面的例子中，在初始请求中的请求配置将会在执行上下文中保持，并传播到连续请求中共享相同的上下文。

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
RequestConfig requestConfig = RequestConfig.custom()
        .setSocketTimeout(1000)
        .setConnectTimeout(1000)
        .build();

HttpGet httpget1 = new HttpGet("http://localhost/1");
httpget1.setConfig(requestConfig);
CloseableHttpResponse response1 = httpclient.execute(httpget1, context);
try {
    HttpEntity entity1 = response1.getEntity();
} finally {
    response1.close();
}
HttpGet httpget2 = new HttpGet("http://localhost/2");
CloseableHttpResponse response2 = httpclient.execute(httpget2, context);
try {
    HttpEntity entity2 = response2.getEntity();
} finally {
    response2.close();
}
```

#### 1.4 HTTP 协议拦截器

HTTP 协议拦截器是一个实现来 HTTP 协议特定方面的程序。通常协议拦截器是工作在在一个特定的头或者一组特定的头在进来的消息中，或者位于在出去的消息中的一个特定头或者一组特定头。协议拦截器还可以操作附带信息的内容实体，内容的压缩解压就是一个很好的例子。通常这是使用装饰模式来完成，利用装饰实体类被用于装饰原始的实体。几个协议拦截器可以组合起来形成一个逻辑单元。
协议拦截器可以通过像通过 HTTP 执行上下文处理状态来共享信息合作。协议拦截器可以使用 HTTP 上下文为一个或者多个连续的请求来存储处理状态。
通常拦截器的执行顺序是没关系的，只要它们不依赖特效的执行上下文状态。如果协议拦截器有内部依赖，因此必须在一种特定的顺序中执行，它们将会以一个期望的顺序被加在一个协议处理器中。
协议拦截器必须是线程安全的。跟 servlet 类似，协议拦截器不应该使用实例除非获取这些变量需要同步的方式。
这是一个如果本地上下文在多个连续请求中被用于保持处理状态的例子
```java
CloseableHttpClient httpclient = HttpClients.custom()
        .addInterceptorLast(new HttpRequestInterceptor() {

            public void process(
                    final HttpRequest request,
                    final HttpContext context) throws HttpException, IOException {
                AtomicInteger count = (AtomicInteger) context.getAttribute("count");
                request.addHeader("Count", Integer.toString(count.getAndIncrement()));
            }

        })
        .build();

AtomicInteger count = new AtomicInteger(1);
HttpClientContext localContext = HttpClientContext.create();
localContext.setAttribute("count", count);

HttpGet httpget = new HttpGet("http://localhost/");
for (int i = 0; i < 10; i++) {
    CloseableHttpResponse response = httpclient.execute(httpget, localContext);
    try {
        HttpEntity entity = response.getEntity();
    } finally {
        response.close();
    }
}
```

#### 1.5 异常处理

HTTP 协议处理器可以抛出两种类型的异常：`java.io.IOException` I/O失败，像 socket 超时或者 socket 被重置；`HttpException` 标志着 HTTP 失败像违反 HTTP 协议。一般来说 I/O 错误是被认为非致命性的和可恢复的，而 HTTP 协议错误这被认为致命性的和不可自动恢复的。请记住，HttpClient 将 HttpException 实现成了可抛出的类 `ClientProtocolException`，它是`java.io.IOException`的子类。这样用户就能在一个catch代码块中同时处理 I/O 错误和违反协议错误了。

##### 1.5.1 HTTP 传输安全

很重要的一点，HTTP 协议并不适用与所有类型的应用程序。HTTP 是一个简单的面向请求响应的协议，最初被设计成支持静态的或者动态生成内容的检索。它未被打算支持事物操作。例如，HTTP 服务认为接收和处理请求，生成响应并发送状态码回客户端就完成了约定的内容。如果客户端不能接收响应因为超市，请求取消了，或者系统崩溃了，服务器不会回滚事务。如果客户端打算重发相同的请求，服务端将会不可避免的多次执行相同的事务。某些情况下这回导致应用数据出错或者不一致的应用状态。
尽管 HTTP 从未被设计成支持事务，它仍可以被用作在特定条件下满足关键任务应用的传输协议。为了保证 HTTP 传输层的安全性，系统必须保证应用层中的 HTTP 方法是幂等的。


##### 1.5.2 幂等方法。

HTTP/1.1 规范定义了幂等方法
【 N > 0 次的相同多次请求等同与单次请求的效果（除了错误或者过期问题），这样的方法具有幂等性】。
换句话说应用程序必须保证处理可能是多种请求的但是相同含义的方法情况。例如，通过提供一个独特的事务id和通过避免相同逻辑操作的执行来实现。请注意，这个问题不局限于 HttpClient，基于浏览器的应用程序都会面临同样的问题因为 HTTP 方法的非幂等性。
默认情况下 HttpClient 假定唯一的非实体方法，如 GET 和 HEAD 是幂等的实体方法，POST 和 PUT 为了兼容则不是幂等的。

##### 1.5.3 自动恢复异常

默认情况下 HttpClient 回尝试从 I/O 异常中自动恢复，默认的自动恢复机智只限于少数已知的安全的异常。
* HttpClient 不会尝试从任何逻辑上或者 HTTP 协议错误中恢复（派生于`HttpException`类）
* HttpClient 会自动重试那些被认为是幂等的方法。
* HttpClient 会自动重试 HTTP 请求正在传输到目标服务器由于传输异常导致失败的异常。（就是请求没有完全被传输到服务器）

##### 1.5.4 请求重试处理

为了确保自定义异常恢复机制，必须为`HttpRequestRetryHandler`接口提供一个实现。

```java
HttpRequestRetryHandler myRetryHandler = new HttpRequestRetryHandler() {

    public boolean retryRequest(
            IOException exception,
            int executionCount,
            HttpContext context) {
        if (executionCount >= 5) {
            // Do not retry if over max retry count
            return false;
        }
        if (exception instanceof InterruptedIOException) {
            // Timeout
            return false;
        }
        if (exception instanceof UnknownHostException) {
            // Unknown host
            return false;
        }
        if (exception instanceof ConnectTimeoutException) {
            // Connection refused
            return false;
        }
        if (exception instanceof SSLException) {
            // SSL handshake exception
            return false;
        }
        HttpClientContext clientContext = HttpClientContext.adapt(context);
        HttpRequest request = clientContext.getRequest();
        boolean idempotent = !(request instanceof HttpEntityEnclosingRequest);
        if (idempotent) {
            // Retry if the request is considered idempotent
            return true;
        }
        return false;
    }

};
CloseableHttpClient httpclient = HttpClients.custom()
        .setRetryHandler(myRetryHandler)
        .build();
```

请注意，可以用StandardHttpRequestRetryHandler代替使用默认的，为了处理安全自动的重试那些在 RFC-2616 协议中定义的请求方法GET，HEAD，PUT，DELETE，OPTIONS 和 TRACE。

#### 1.6 中断请求

某些情况下，HTTP 请求围在预期时间框架内完成，因为目标服务器的高负载或者客户端有太多并发的请求。在这种情况下，可能需要提前中止请求，并开启阻塞在 I/O 操作的线程。通过`HttpUriRequest#abort()`方法可以在任何阶段中止由 HttpClient 执行的 HTTP 请求。这个方法是线程安全的，可以在任意线程调用。当一个 HTTP 请求被中断他的执行线程，即使在一个 I/O 操作中阻塞，也可以通过抛出`InterruptedIOException`保持畅通。

#### 1.7 重定向处理

HttpClient 会自动处理所有类型的重定向，除了那些被 HTTP 规范明确禁止的需要用户干预的。
考虑到其他的在 HTTP 规范中定义的 POST 和 PUT 请求的重定向成 GET 请求的重定向（状态码303），可以使用自定义的重定向策略来减少 HTTP 规范规定的 POST 方法重定向的限制。

```java
LaxRedirectStrategy redirectStrategy = new LaxRedirectStrategy();
CloseableHttpClient httpclient = HttpClients.custom()
        .setRedirectStrategy(redirectStrategy)
        .build();
```

HttpClient 通常需要重写请求消息在执行过程中。默认的，HTTP/1.0 和 HTTP/1.1 通常使用相对的URI。同样的，原始请求可能被重定向到其他地址多次。最终解释的绝对 HTTP 地址可以用初始请求和上下文来构建。实用方法`URIUtil#resolve`可以被用来构建生成最终请求的解释绝对URI。该方法包括从重定向请求或者原始请求的最后一个片段标识符。

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpClientContext context = HttpClientContext.create();
HttpGet httpget = new HttpGet("http://localhost:8080/");
CloseableHttpResponse response = httpclient.execute(httpget, context);
try {
    HttpHost target = context.getTargetHost();
    List<URI> redirectLocations = context.getRedirectLocations();
    URI location = URIUtils.resolve(httpget.getURI(), target, redirectLocations);
    System.out.println("Final HTTP location: " + location.toASCIIString());
    // Expected to be an absolute URI
} finally {
    response.close();
}
```




z
