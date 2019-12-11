---
title: Apache HttpClient
date: 2019-12-03 22:35:26
tags: Java
---

# HttpClient example

```java
import java.io.IOException;

import org.apache.http.HttpEntity;
import org.apache.http.HttpHeaders;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;

public class HttpClientExample {

    public static void main(String[] args) throws IOException {

        HttpGet request = new HttpGet("https://httpbin.org/get");

        // add request headers
        request.addHeader(HttpHeaders.USER_AGENT, "Googlebot");

        try (CloseableHttpClient httpClient = HttpClients.createDefault();
            CloseableHttpResponse response = httpClient.execute(request)) {

            // Get HttpResponse Status
            System.out.println(response.getProtocolVersion());              // HTTP/1.1
            System.out.println(response.getStatusLine().getStatusCode());   // 200
            System.out.println(response.getStatusLine().getReasonPhrase()); // OK
            System.out.println(response.getStatusLine().toString());        // HTTP/1.1 200 OK

            HttpEntity entity = response.getEntity();
            if (entity != null) {
                // return it as a String
                String result = EntityUtils.toString(entity);
                System.out.println(result);
            }
        }
    }
}
```
<!--more-->

# HttpClient <- CloseableHttpClient <- MinimalHttpClient

MinimalHttpClient contains the following fields:

```java
private final HttpClientConnectionManager connManager;
private final MinimalClientExec requestExecutor;
private final HttpParams params;
```

- HttpParams params
- MinimalClientExec requestExecutor

It represents a complete execution process. It's using decorator pattern.

- HttpClientConnectionManager connManager

It maintains a connectio pool. Manage lifecycle of connections. 

The status of `Connection` when created is idle and it's managed by the connection pool. When the `Connection` is used, the first step is to check if it's in `open` status. If not, it starts connecting. Based on which schema is used (http/https), socket (ssl or plain) is created.


Let's take a look at the core method

```java
    @Override
    protected CloseableHttpResponse doExecute(
            final HttpHost target,
            final HttpRequest request,
            final HttpContext context) throws IOException, ClientProtocolException {
        Args.notNull(target, "Target host");
        Args.notNull(request, "HTTP request");
        HttpExecutionAware execAware = null;
        if (request instanceof HttpExecutionAware) {
            execAware = (HttpExecutionAware) request;
        }
        try {
            final HttpRequestWrapper wrapper = HttpRequestWrapper.wrap(request);
            final HttpClientContext localcontext = HttpClientContext.adapt(
                context != null ? context : new BasicHttpContext());
            final HttpRoute route = new HttpRoute(target);
            RequestConfig config = null;
            if (request instanceof Configurable) {
                config = ((Configurable) request).getConfig();
            }
            if (config != null) {
                localcontext.setRequestConfig(config);
            }
            return this.requestExecutor.execute(route, wrapper, localcontext, execAware);
        } catch (final HttpException httpException) {
            throw new ClientProtocolException(httpException);
        }
    }
```

At last, it calls `requestExecutor.execute`. 

`requestExecutor` object is an instance of `MinimalClientExec` which implements `ClientExecChain`.


MinimalClientExec#execute:

```java
@Override
    public CloseableHttpResponse execute(
            final HttpRoute route,
            final HttpRequestWrapper request,
            final HttpClientContext context,
            final HttpExecutionAware execAware) throws IOException, HttpException {

        final ConnectionRequest connRequest = connManager.requestConnection(route, null);  // a request for a HttpClientConnection whose lifecycle is managed by a connection manager
        
        final RequestConfig config = context.getRequestConfig();

        final HttpClientConnection managedConn;  // send request and receive response
        try {
            final int timeout = config.getConnectionRequestTimeout();
            managedConn = connRequest.get(timeout > 0 ? timeout : 0, TimeUnit.MILLISECONDS);
        } //...

        context.setAttribute(HttpCoreContext.HTTP_TARGET_HOST, target);
        context.setAttribute(HttpCoreContext.HTTP_REQUEST, request);
        context.setAttribute(HttpCoreContext.HTTP_CONNECTION, managedConn);
        context.setAttribute(HttpClientContext.HTTP_ROUTE, route);
        // store Key-value pair. share data across different logics

        httpProcessor.process(request, context);
        final HttpResponse response = requestExecutor.execute(request, managedConn, context);
        httpProcessor.process(response, context);

        // requestExecutor is an object of HttpRequestExecutor

        // httpProcessor is an object of ImmutableHttpProcessor. It processes http protocol. It includes multiple protocol interceptors:
        /**
            private final HttpRequestInterceptor[] requestInterceptors;
            private final HttpResponseInterceptor[] responseInterceptors;
        */
        
    }
```

The core part is how http request is sent out?

HttpRequestExecutor#execute:

```java
HttpResponse response = doSendRequest(request, conn, context);
            if (response == null) {
                response = doReceiveResponse(request, conn, context);
            }
            return response;
```

what is inside `doSendRequest`?

```java
conn.sendRequestHeader(request);
conn.sendRequestEntity((HttpEntityEnclosingRequest) request);
```

As we can see, `HttpClientConnection` actually does the hard work, sending request and receiving response. `HttpRequestExecutor` encapsulates the process. 


Thee main parts:
- manage requests (ConnectionManager)
- execute requests (RequestExecutor)
- process request (HttpProcessor)



# HttpConnection <- HttpClientConnection

The most commonly used implementation is `DefaultBHttpClientConnection`

It has two fields:

```java
    private final HttpMessageParser<HttpResponse> responseParser;
    private final HttpMessageWriter<HttpRequest> requestWriter;
```

These two fieds are created based on `SessionOutputBuffer` and `SessionInputBuffer`
```java
        this.requestWriter = (requestWriterFactory != null ? requestWriterFactory :
            DefaultHttpRequestWriterFactory.INSTANCE).create(getSessionOutputBuffer());
        this.responseParser = (responseParserFactory != null ? responseParserFactory :
            DefaultHttpResponseParserFactory.INSTANCE).create(getSessionInputBuffer(), constraints);
```


# HttpClientConnectionManager <- BasicHttpClientConnectionManager

BasicHttpClientConnectionManager#connect
```java
 @Override
    public void connect(
            final HttpClientConnection conn,
            final HttpRoute route,
            final int connectTimeout,
            final HttpContext context) throws IOException {
        Args.notNull(conn, "Connection");
        Args.notNull(route, "HTTP route");
        Asserts.check(conn == this.conn, "Connection not obtained from this manager");
        final HttpHost host;
        if (route.getProxyHost() != null) {
            host = route.getProxyHost();
        } else {
            host = route.getTargetHost();
        }
        final InetSocketAddress localAddress = route.getLocalSocketAddress();
        this.connectionOperator.connect(this.conn, host, localAddress,
                connectTimeout, this.socketConfig, context);
    }
```

As we can see from the last line, `connectionOperator` create a socket and bind it to `HttpClientConnection`.


# HttpClientConnectionManager <- PoolingHttpClientConnectionManager

It maitains a connection pool (cpool). A pool usually has a factory (InternalConnectionFactory)

```java
 this.pool = new CPool(new InternalConnectionFactory(
                this.configData, connFactory), 2, 20, timeToLive, timeUnit);
```