由於寫了C++ 那麼多年雖然知道有C11, C14等標準慢慢被發展出來，但是始終沒有好好了解 C++ 和 C 上的差異，時常發現用C++ 寫出來的 code 和 C 差不多，所以打算花段時間來研究研究，接下來幾篇是我看 **Modern C++ Tutorial: C++11/14/17/20 On the Fly** 的筆記

作者提到希望我們不要有 "C++ is not a superset of C" 的想法，盡量在C++ 中避免 C style 的寫法，當你需要使用到 C 時要將 C code 和 C++ code 分開，並用不同的方式編譯再 include


### Deprecated Features
 
* 不該將 string literal constant 給 `char*` 應該用 `auto` 或 `const char*` 代替
```
char *str = "literal constant"  // warning
```
* `noexcept` 取代 `unexpected_handler`, `set_unexpected()` 等
* `unique_ptr` 取代 `auto_ptr`
* `register` 可以使用但不再有任何實際意義
* 不能對 `bool` 變數使用 `++`
* explicitly declaring a destructor will prevent the implicit generation of a move constructor and move assignment operator.
    至於為什麼要這個搞我們後續再討論
* 使用 `static_cast`, `reinterpret_cast`, `const_cast` 取代C語言風格的型態轉換 ( 在變數前面加(type) )
* C++17 中棄用了一些 C 函式庫，像是 `<cstdbool>`, `<ctgmath>` 等
* 還有很多很多 ...

當然你還是可以使用上述這些特性，Deprecated 只是要告訴你這些特性在未來可能會不在是標準，開發者最好先嘗試適應或改變習慣 ><

> Deprecation is not completely unusable, it is only intended to imply that programmers will disappear from future standards and should be avoided. However, the deprecated features are still part of the standard library, and most of the features are actually "permanently" reserved for compatibility reasons.

### 編譯

```
$ clang++ -std=c++2a
```