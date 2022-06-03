- [net/http](#nethttp)
- [http.Client](#httpclient)
  - [http.Get](#httpget)
  - [http.Post](#httppost)
  - [http.PostForm](#httppostform)
  - [http.Head](#httphead)
  - [(*http.Client).Do](#httpclientdo)
  - [http.Client Data Structures](#httpclient-data-structures)
  - [http.Transport Implementation](#httptransport-implementation)
  - [Transport.RoundTrip() Implementation](#transportroundtrip-implementation)
- [HTTP/HTTPS Request Handle](#httphttps-request-handle)
  - [HTTP Request Handle](#http-request-handle)
  - [HTTPS Request Handle](#https-request-handle)

# net/http

通過 `net.Dial()` 或 `net.DialTimeout` 函數來訪問基於 HTTP 協議的網路服務是完全沒問題的, 因為 HTTP 也是基於 TCP/IP 協議

但若通過 `net.Dial()` 函數進行 HTTP 程式開發, HTTP status code, packet header 和 packet body 的部分處理起來相當繁瑣

因此 Go standard library build-in 的 net/http package 涵蓋了 HTTP client 和 server 的具體實現

# http.Client

net/http package 提供了最簡潔的 HTTP client 實現, 不需借助第三方通訊 library(如 `libcurl`) 即可直接使用 `GET` 和 `POST` 發起 HTTP request

```go
func (c *Client) Do(req *Request) (*Response, error)
func (c *Client) Get(url string) (resp *Response, err error)
func (c *Client) Head(url string) (resp *Response, err error)
func (c *Client) Post(url, contentType string, body io.Reader) (resp *Response, err error)
func (c *Client) PostForm(url string, data url.Values) (resp *Response, err error)
```

## http.Get

調用 `http.Get()`方法並傳入請求 URL 即可發起一個 GET request:

```go
resp, err := http.Get("https://regy.dev") 
if err != nil {    // error handle ...    return }
defer resp.Body.Close() 
io.Copy(os.Stdout, resp.Body)
```

通過 `http.Get` 發起請求時, 默認調用的是上述 `http.Client` 默認物件的 `Get` 方法:

```go
func Get(url string) (resp *Response, err error) {    
    return DefaultClient.Get(url)
}
```

`DefaultClient` 默認指向 `http.Client` 物件實體:

```go
var DefaultClient = &Client{}
```

`http.Get()` 方法返回值有兩個, 第一個是 response 物件, 第二個是 `error` 物件, 若請求過程中出現錯誤, 則 `error` 物件不為 nil; 否則可以通過 response 物件獲得 response status code, response header, response body 等訊息

response 物件是 `http.Response` 類型實體, 一般可以通過 `resp.Body` 獲取 response body, 通過 `resp.Header` 獲得 response header, 通過 `resp.StatusCode` 獲取 response status 

獲得 response 成功後需記得調用 `resp.Body` 的 `Close()` 方法結束網路請求釋放系統資源

## http.Post

調用 `http.Post()` 方法並傳遞以下三個參數:
- 請求目標的 URL
- POST 請求資料的資源類型 (MIME Type)
- data byte stream ([]byte)

下列程式碼演示如何上傳使用者頭貼:

```go
resp, err := http.Post("https://regy.dev/avatar", "image/jpeg", &imageDataBuf) 
if err != nil {    // error handle    return }
if resp.StatusCode != http.StatusOK {     // error handle     return }
// ...
```

底層實現及返回值與 `http.Get()` 相同

## http.PostForm

`http.PostForm()` 方法實現了標準編碼格式 `application/x-www-form-urlencoded` 的 POST 表單提交

下列程式碼模擬 HTML 登錄表單提交:

```go
resp, err := http.PostForm("https://regy.dev/login", url.Values{"name":{"test-user"}, "password": {"test-passwd"}}) 
if err != nil {    // error handle    return } 
if resp.StatusCode != http.StatusOK {     // error handle     return } 
// ...
```

POST 請求參數通過 `url.Valuses` 進行編碼及封裝

底層實現及返回值與 `http.Get()` 相同

## http.Head

HTTP Head 請求表示只請求目標 URL 的 response header, 無需返回 response body

通過 `http.Head()` 方法發起 Head 請求, 與 `http.Get()` 方法相同, 只需傳入目標 URL 參數即可

下列程式碼用於請求首頁的 HTTP Response Header:

```go
resp, err := http.Head("https://regy.dev")
if err != nil {    
    fmt.Println("Request Failed: ", err.Error())    
    return
}
defer resp.Body.Close()

// print header info for 
key, value := range resp.Header  {    
    fmt.Println(key, ":", value)
}
```

## (*http.Client).Do

大多數應用場景 `http.Get`, `http.Post` 和 `http.PostForm` 就可以滿足, 但如果發起的 HTTP request 需要設定更多的自定義 request header, 比如:
- 設定自定義 `User-Agent` 而不是默認的 `Go http package`
- 傳遞 Cookie
- 發起其他方式 HTTP request, 比如 `PUT`, `PATCH`, `DELETE` 等

此時可以通過 `http.Client` 的 `Do()` 方法實現, 此時不再是通過默認的 `DefaultClient` 物件調用 `http.Client` 的方法, 而是需要手動實體化 `Client` 物件並傳入添加了自定義 request header 的請求物件來發起 HTTP request:

```go
// init client request object
req, err := http.NewRequest("GET", "https://regy.dev", nil)
if err != nil { return }

// add custom request header
req.Header.Add("Custom-Header", "Custom-Value")

// ... other config of request header
client := &http.Client{    // ... setup client property}
resp, err := client.Do(req)

if err != nil {    // ... error handle    return}

defer resp.Body.Close()
io.Copy(os.Stdout, resp.Body)
```

用於初始化請求物件的 `http.NewRequest()` 方法需要三個參數:
- Request method
- 目標 URL
- Request 實體 (只有 POST, PUT, DELETE 之類請求需要設置請求實體, 對於 HEAD, GET 只需傳入 nil 即可)

`http.NewRequest()` 方法返回的第一個值就是請求物件實體 `req`, 該實體類型為 `http.Request`

可以調用 `http.Request` 類型的 public methods 對請求物件進行自定義配置, 比如 request method, URL, request header 等

設置完成後將 request object 傳入 `client.Do()` 方法發起 HTTP request, 之後的操作與前四個基本方法一致

## http.Client Data Structures

`http.Get()`、`http.Post()`、`http.PostForm()` 和 `http.Head()` 方法其實都是在 http.DefaultClient 的基礎上調用的

`http.DefaultClient` 是 `net/http` package 提供的 HTTP client 默認實現:

```go
// DefaultClient is the default Client and is used by Get, Head, and Post.var DefaultClient = &Client{}
```

但還是可以基於 `http.Client` 自定義 HTTP client implementation

`Client` 類型的資料結構:

```go

type Client struct {
    // Transport 用於指定單次 HTTP request/response 完整流程
    // 默認值為 DefaultTransport 
    Transport RoundTripper

    // CheckRedirect 用於定義重定向處理策略
    // 它是一個函數類型，接收 req 和 via 兩個參數，分別表示即將發起的請求和已經發起的所有請求，越早的已發起請求在越前面
    // 若不為 nil, Client 將在跟蹤 HTTP 重定向前調用此函數
    // 若返回 error, Client 將直接返回 error, 不會再發起該 request
    // 若為 nil, Client 將採用確認策略, 會在 10 個連續 request 後終止
    CheckRedirect func(req *Request, via []*Request) error

    // Jar 用於指定 request/response header 中的 Cookie 
    //若 field 為 空, 則只有在 request 中顯式設定的 Cookie 才會被發送
    Jar CookieJar

    // 指定單次 HTTP request/response transaction timeout
    // 為設定的話使用 Transport 的默認設置，為零的話表示不設置 timeout
    Timeout time.Duration
```

> Transport

其中 `Transport` field 必須 implement `http.RoundTripper` interface, `Transport` 指定了一次 HTTP transaction (request/response) 的完整流程, 若不指定默認使用 `http.DefaultTransport` 這個默認實現, 比如 `http.DefaultClient`

> CheckRedirect

`CheckRedirect` 函數用於定義處理重定向策略

當使用 `http.DefaultClient` 提供的 `Get()` 或者 `Head()` 方法發送 HTTP request 時, 若 response status code 為 30x(比如 `301`, `302` 等), HTTP Client 會在遵循跳轉規則前先調用 `CheckRedirect` 函數

> Jar

`Jar` 可用於在 HTTP Client 中設置 Cookie, `Jar` 類型必須實現 `http.CookieJar` interface, 該 interface 預定義了 `SetCookies()` 和 `Cookies()` 兩個方法

若 HTTP Client 沒有設置 `Jar`, Cookie 將被忽略而不會發送到 Client, 一般會用 `http.SetCookie()` 方法來設置 Cookie

> Timeout

`Timeout` field 用於指定 `Transport` timeout 時間, 沒有指定的話使用 `Transport` 自定義的設置

## http.Transport Implementation

下面通過 `http.DefaultTransport` 實現來介紹 `http.Transport`, 沒有顯示設置 `Transport` field 時就會使用 `DefaultTransport`:

```go
func (c *Client) transport() RoundTripper {
	if c.Transport != nil {
		return c.Transport
	}
	return DefaultTransport
}
```

`DefaultTransport` 是 `Transport` 的默認實現, 對應的初始化程式碼如下:

```go
var DefaultTransport RoundTripper = &Transport{
	Proxy: ProxyFromEnvironment,
	DialContext: (&net.Dialer{
		Timeout:   30 * time.Second,
		KeepAlive: 30 * time.Second,
	}).DialContext,
	ForceAttemptHTTP2:     true,
	MaxIdleConns:          100,
	IdleConnTimeout:       90 * time.Second,
	TLSHandshakeTimeout:   10 * time.Second,
	ExpectContinueTimeout: 1 * time.Second,
}
```

這裡只設置 `Transport` 的部分屬性, `Transport` 類型完整的資料結構如下:

```go

type Transport struct {
    ...
    
    // 定義 HTTP Proxy 策略
    Proxy func(*Request) (*url.URL, error)
    
    // 用於指定創建未加密 TCP 連接的 context 參數 (透過 net.Dial() 創建連接時使用)
    DialContext func(ctx context.Context, network, addr string) (net.Conn, error)
    // deprecatede，使用 DialContext 替代
    Dial func(network, addr string) (net.Conn, error)
    // 創建加密 TCP 連接
    DialTLS func(network, addr string) (net.Conn, error)
    // 指定 tls.Client 所使用的 TLS 配置
    TLSClientConfig *tls.Config
    // TLS handsharking timeout
    TLSHandshakeTimeout time.Duration
    
    // 是否禁用 HTTP 長連接
    DisableKeepAlives bool
    // 是否對 HTTP 報文進行壓縮傳輸（gzip）
    DisableCompression bool
    
    // 最大空閒連接數（支持長連接時有效）
    MaxIdleConns int
    // 單個服務 (域名) 最大空閒連接數
    MaxIdleConnsPerHost int
    // 單個服務 (域名) 最大連接數
    MaxConnsPerHost int
    // 空閒連接 timeout 時間
    IdleConnTimeout time.Duration

    // 從 Client 將 request 完全提交給 server 到從 server 收到 reponse header timtout 時間
    ResponseHeaderTimeout time.Duration
    // 包含 "Expect: 100-continue" request header 的情況下從 Client 將 request 完全提交給 server 到從 server 收到 reponse header timtout 時間
    ExpectContinueTimeout time.Duration
    
    ...        
}
```

結合 `Transport` 資料結構觀察 `DefaultTransport` 的設置:
- `net.Dialer` 初始化 Dial context config, 默認 timeout 時間為 30 秒
- `MaxIdleConns` 指定最大空閒連接數為 100, 未顯式設定 `MaxIdleConnsPerHost` 和 `MaxConnsPerHost`, `MaxIdleConnsPerHost` 有默認值, 通過 `http.DefaultMaxIdleConnsPerHost` 設置, 默認值為 2
- 通過 `IdleConnTimeout` 指定最大空閒連接時間為 90 秒, 即當某個空閒連接超過 90 秒沒有被復用則銷毀, 空閒連接需要 `DisableKeepAlives` 為 `false` 的情況下才可用, 即 HTTP 長連接狀態下有效 (HTTP/1.1 以上版本支持長連接, 對應 request header `Connection:keep-alive`)
- 通過 `ExpectContinueTimeout` 指定 Client 想要使用 POST 請求把一個很大的報文體發送給 server 時, 先通過發送一個包含了 `Expect: 100-continue` 的 request header, 詢問 server 是否願意接收這個大報文體對應的超時時間，這裡默認設置為 1 秒

另外 `Transport` 包含了 `RoundTrip` 方法實現, 實現了 `RoundTripper` interface

接著來觀察 `Transport` 中 `RoundTrip` 方法的實現

## Transport.RoundTrip() Implementation

`http.RoundTripper` interface 具體定義:

```go
type RoundTripper interface {
    RoundTrip(*Request) (*Response, error)
}
```

`RoundTrip()` 方法用於執行一個獨立的 HTTP transaction, 接受傳入的 `*Request` 作為參數並返回對應的 `*Response`, 以及一個 `error`

底層 Go 通過 WHATWG Fetch API 實現了單次 HTTP request/response transaction:

```go
func (t *Transport) RoundTrip(req *Request) (*Response, error) {
  if useFakeNetwork() {
    return t.roundTrip(req)
  }

  ac := js.Global().Get("AbortController")
  if ac != js.Undefined() {
    // Some browsers that support WASM don't necessarily support
    // the AbortController. See
    // https://developer.mozilla.org/en-US/docs/Web/API/AbortController#Browser_compatibility.
    ac = ac.New()
  }

  opt := js.Global().Get("Object").New()
  // See https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/fetch
  // for options available.
  opt.Set("method", req.Method)
  opt.Set("credentials", "same-origin")
  if h := req.Header.Get(jsFetchCreds); h != "" {
    opt.Set("credentials", h)
    req.Header.Del(jsFetchCreds)
  }
  if h := req.Header.Get(jsFetchMode); h != "" {
    opt.Set("mode", h)
    req.Header.Del(jsFetchMode)
  }
  if ac != js.Undefined() {
    opt.Set("signal", ac.Get("signal"))
  }
  headers := js.Global().Get("Headers").New()
  for key, values := range req.Header {
    for _, value := range values {
      headers.Call("append", key, value)
    }
  }
  opt.Set("headers", headers)

  if req.Body != nil {
    // TODO(johanbrandhorst): Stream request body when possible.
    // See https://bugs.chromium.org/p/chromium/issues/detail?id=688906 for Blink issue.
    // See https://bugzilla.mozilla.org/show_bug.cgi?id=1387483 for Firefox issue.
    // See https://github.com/web-platform-tests/wpt/issues/7693 for WHATWG tests issue.
    // See https://developer.mozilla.org/en-US/docs/Web/API/Streams_API for more details on the Streams API
    // and browser support.
    body, err := ioutil.ReadAll(req.Body)
    if err != nil {
      req.Body.Close() // RoundTrip must always close the body, including on errors.
      return nil, err
    }
    req.Body.Close()
    a := js.TypedArrayOf(body)
    defer a.Release()
    opt.Set("body", a)
  }
  respPromise := js.Global().Call("fetch", req.URL.String(), opt)
  var (
    respCh = make(chan *Response, 1)
    errCh  = make(chan error, 1)
  )
  success := js.FuncOf(func(this js.Value, args []js.Value) interface{} {
    result := args[0]
    header := Header{}
    // https://developer.mozilla.org/en-US/docs/Web/API/Headers/entries
    headersIt := result.Get("headers").Call("entries")
    for {
      n := headersIt.Call("next")
      if n.Get("done").Bool() {
        break
      }
      pair := n.Get("value")
      key, value := pair.Index(0).String(), pair.Index(1).String()
      ck := CanonicalHeaderKey(key)
      header[ck] = append(header[ck], value)
    }

    contentLength := int64(0)
    if cl, err := strconv.ParseInt(header.Get("Content-Length"), 10, 64); err == nil {
      contentLength = cl
    }

    b := result.Get("body")
    var body io.ReadCloser
    // The body is undefined when the browser does not support streaming response bodies (Firefox),
    // and null in certain error cases, i.e. when the request is blocked because of CORS settings.
    if b != js.Undefined() && b != js.Null() {
      body = &streamReader{stream: b.Call("getReader")}
    } else {
      // Fall back to using ArrayBuffer
      // https://developer.mozilla.org/en-US/docs/Web/API/Body/arrayBuffer
      body = &arrayReader{arrayPromise: result.Call("arrayBuffer")}
    }

    select {
    case respCh <- &Response{
      Status:        result.Get("status").String() + " " + StatusText(result.Get("status").Int()),
      StatusCode:    result.Get("status").Int(),
      Header:        header,
      ContentLength: contentLength,
      Body:          body,
      Request:       req,
    }:
    case <-req.Context().Done():
    }

    return nil
  })
  defer success.Release()
  failure := js.FuncOf(func(this js.Value, args []js.Value) interface{} {
    err := fmt.Errorf("net/http: fetch() failed: %s", args[0].String())
    select {
    case errCh <- err:
    case <-req.Context().Done():
    }
    return nil
  })
  defer failure.Release()
  respPromise.Call("then", success, failure)
  select {
  case <-req.Context().Done():
    if ac != js.Undefined() {
      // Abort the Fetch request
      ac.Call("abort")
    }
    return nil, req.Context().Err()
  case resp := <-respCh:
    return resp, nil
  case err := <-errCh:
    return nil, err
  }
}
```

實現了 `http.RoundTripper` interface 的程式碼通常需要在多個 goroutine 中併發執行, 因此必須確保實現的同步安全性

以上就是 `http.Client` 底層實現的幾個核心組件及其默認實現, 重點關注 `http.Transport`, 其定義了一次 HTTP transaction 的完整流程

可以通過自定義 `Transport` 實現對 HTTP Client request 的訂製

# HTTP/HTTPS Request Handle

## HTTP Request Handle

使用 `net/http` package 提供的 `http.ListenAndServe()` 函數可以開啟一個 HTTP server, 並且在指定的 IP 和 port 上監聽 client request, 此函數簽名如下:

```go
func ListenAndServe(addr string, handler Handler) error
```

該函數有兩個參數:
- `addr`: 表示監聽的 IP 和 port
- `handler`: 表示 server 對應的處理程式, 通常為空, 意味著將調用 `http.DefaultServerMux` 處理

> server 業務邏輯處理程式 `http.Handle()` 或 `http.HandleFunc()` 默認會被注入到 `http.DefaultServerMux`

實現一個最基本的 HTTP server:

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/hello", func(writer http.ResponseWriter, request *http.Request) {
        params := request.URL.Query();
        fmt.Fprintf(writer, "Hello, %s", params.Get("name"))
    })
    err := http.ListenAndServe(":8080", nil)
    if err != nil {
        log.Fatalf("Failed to start HTTP server: %v", err)
    }
}
```

通過 `http.HandleFunc` 函數定義了一個 `/hello` 路由及其對應處理程式, 處理程式會返回一個 Hello string, 其中還引用了 client 傳送的 request parameter

再通過 `http.ListenAndServe` 函數啟動 HTTP server, 監聽 localhost 8080 port

當 client 請求 `http://127.0.0.1:8080/hello` URL 時, HTTP server 會將其轉發給默認的 `http.DefaultServeMux` 處理, 最後會調用 `http.HandleFunc` 函數定義的處理器 handle request 並 response

實現一個基本的 client request:

```go
req, err := http.NewRequest("GET", "http://127.0.0.1:8080/hello?name=regy", nil)
```

## HTTPS Request Handle

net/http package 還提供了 `http.ListenAndServeTLS()` 函數, 用於處理 HTTPS request:

```go
func ListenAndServeTLS(addr string, certFile string, keyFile string, handler Handler) error
```

`ListenAndServeTLS()` 和 `ListenAndServe()` 行為一致, 區別在於前者只處理 HTTPS request

Server 必須配置 SSL/TLS 憑證等相關文件, 比如 `certFile` 對應 SSL/TLS 憑證文件目錄, `keyFile` 對應憑證私鑰文件目錄
