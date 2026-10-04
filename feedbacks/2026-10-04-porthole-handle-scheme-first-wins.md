# 🟢 `handle_scheme` 对同一个协议「先到先得」，第二次调用静默无效 —— 文档里没写

> porthole，2026-10-04。saucer `c7dca16`。不是 bug（四个后端一致，`embed()` 也依赖这个语义），是文档缺一句。

## 症状

porthole 先 `handle_scheme("porthole", 前端目录的 resolver)`，之后想再挂一个 resolver 伺候应用自己落盘的文件 ⇒ 第二次调用**什么都不发生**，
不报错、不替换。查了一阵才发现要先 `remove_scheme`（或者拼成一个 resolver 只挂一次）。

## 位置

- `src/wv2.webview.cpp:453`、`src/qt.webview.cpp:383`：`if (platform->schemes.contains(name)) return;`
- `src/wkg.scheme.impl.cpp:11`、`src/wk.scheme.impl.mm:9`：`m_callbacks.emplace(webview, …)`（`unordered_map`，键已在就不动）
- `src/webview.cpp:255` 的 `embed()` 每次都再调 `handle_scheme("saucer", …)`，靠的正是「已挂过就忽略」。
- `include/saucer/webview.hpp:197` 的声明没有注释说明这一点。

## 建议

在 `webview.hpp` 的 `handle_scheme` 上写一句：「同名协议已挂过时本调用无效；要替换先 `remove_scheme`」。
（要是想改成「替换」，记得 `embed()` 那条路会受影响。）
