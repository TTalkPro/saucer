# 🔴 WebView2：`scheme::executor::reject` 造出非法响应 —— 被拒的自定义协议请求**永远挂着**

> porthole，2026-10-04。saucer `c7dca16`（= porthole 的 pin，也是本仓 master），Windows 11 / WebView2 Runtime 154.0.4258.53，MSVC 14.44。
> ✅ **已在本仓工作树里修了（未提交）**：`src/wv2.webview.impl.cpp` 的 `reject`；用例 `tests/src/webview.test.cpp` 的 `scheme-reject`。

## 症状

处理函数对某个路径 `executor.reject(scheme::error::not_found)`（或任何错误）之后，页面上那条请求**既不 resolve 也不 reject**：
`fetch('myscheme://app/missing')` 永远 pending，`<img src=…>` 的 `onload` / `onerror` 都不触发。`resolve` 的路径正常。

porthole 里实测（同一页面，3–5 s 超时）：`fetch` 404 ⇒ TIMEOUT、`<img>` 404 ⇒ TIMEOUT、`fetch` 200 ⇒ 立刻 200。
后果：porthole 的白板示例等一张 404 的图等不到结果，自检不报 PASS 也不报 FAIL，一路连锁到进程被强杀、显卡上的模型没释放、进程卡死在内核里。

## 位置

`src/wv2.webview.impl.cpp` 的 `reject` lambda（原 :483）：

```cpp
environment->CreateWebResourceResponse(nullptr, status, L"", phrase.c_str(), &result);
```

两处错：

1. 签名是 `CreateWebResourceResponse(content, statusCode, reasonPhrase, headers, response)` —— **后两个参数传反了**：
   原因短语给了空串，headers 位置给了 `"Not Found"`（不是合法的 header 行）。
2. `scheme::error::failed` 的值是 **-1**（`include/saucer/scheme.hpp:20`），被原样当 HTTP 状态码传进去，同样非法。

返回值也没检查：创建失败时 `result` 是空的，照样 `put_Response(nullptr)`。

其余三个后端没有这个问题（wkg `webkit_uri_scheme_request_finish_error`、wk / qt 走各自的错误接口）。

## 修法（工作树里的改动）

- 参数摆正：`CreateWebResourceResponse(nullptr, status, phrase.c_str(), L"", &result)`；
- `status` 不是正数时（`failed`）按 500 报，原因短语 `Internal Server Error`；
- `SUCCEEDED(...)` 才 `put_Response`，否则不设响应直接 `Complete()`（交给 WebView2 按未处理走，至少不会挂住）。

## 验证

- ✅ 在 porthole 里验的（真 WebView2）：把这份修复放进 porthole 的 `deps/saucer`，并且把 porthole shim 里的绕开退掉（让 404 重新走
  `reject`）⇒ porthole 自检「自定义协议（404 答得回来）」**PASS**；未修的 saucer 同样配置 ⇒ **FAIL（timeout）**。
- ⚠️ `failed` ⇒ 500 那一支没有单独实测（porthole 的 shim 只会传 404）。
- ⚠️ 新用例 `scheme-reject` 在本机**跑不起来**：测试程序在 MSVC 上链接不过（见 `2026-10-04-porthole-tests-msvc-link.md`），
  而 CI 的 Windows 两格本来就是 `-Dsaucer_tests=OFF` ⇒ 这个用例今天只会在 Linux / macOS 上跑（那边只判「请求有结论」，
  Windows 上额外判 `404:500`）。

## porthole 那边怎么绕的

porthole 的 shim 对 status ≥ 400 改走 `resolve`（带真状态码），不再调 `reject`（porthole `8991a92`）。saucer 修了之后那一处可以保留
（`resolve` + 404 本来就比「网络错误」更贴近 HTTP），也可以退回去。
