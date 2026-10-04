# 🟢 测试程序在 MSVC 上链接不过：`cfg<override>` 找不到定义

> porthole，2026-10-04。saucer `c7dca16`，MSVC 14.44.35207（VS 18），Ninja，`-Dsaucer_tests=ON`。CI 的 Windows 两格本来就关着测试，所以 CI 看不见。

## 症状

```
main.cpp.obj : error LNK2001: unresolved external symbol "struct saucer::tests::co_runner cfg<struct boost::ext::ut::v2_3_1::override>"
（window / regression / smartview / webview 四个 .obj 同样）
tests\saucer-tests.exe : fatal error LNK1120: 1 unresolved externals
```

## 位置

`tests/include/runner.hpp:48-49`：

```cpp
template <>
inline auto boost::ut::cfg<boost::ut::override> = saucer::tests::co_runner{};
```

把 `auto` 换成具体类型（`inline saucer::tests::co_runner …`）**无效**，同样的链接错（试过、已撤回）。看着像 MSVC 对 inline 变量模板显式特化的处理问题。

## 影响

Windows 上没法跑 saucer 自己的测试 ⇒ WebView2 后端的行为只能靠下游验（`2026-10-04-porthole-wv2-reject-hangs.md` 那条就是这么漏过去的）。

## 建议

- 试 `-T ClangCL`（CI 第二格的配置）能不能链上，能的话让那一格打开测试；
- 或者把 `cfg<override>` 的定义挪进一个 .cpp（非 inline 的显式特化定义），头文件里只留声明。
