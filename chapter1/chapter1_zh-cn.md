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

##### 1.2 HttpClient 接口











z
