
## Constant

### nullptr

傳統 C++ 把 `NULL` 當成 `0` 或是 `(void*)0`，這可能會發生幾個問題

1. `NULL` = `(void*)0`: C++ 不允許將 `(void*)` 轉換為其他型態，所以以下的程式會有問題
```
char *str = NULL;
```
2. `NULL` = `0`: 雖然說將 NULL 定義成 0 可以解決上述問題，但是使用 overloading 時 
```
void foo(char*);
void foo(int);
```
`foo(NULL)` 會選擇 `void foo(int)`，因為通常我們使用 NULL 都是要傳給指標，結果卻是呼叫參數 `int`，十分不直觀

為了解決這些問題，C++11 印入了 **nullptr**，其型態為 **nullptr_t**，可以轉換成任何型態的指標

```
foo(0);       // call foo(int)
foo(NULL);    // error
foo(nullptr); // call foo(char*)
```


### constexpr

在 C++ 中宣告陣列時只能使用 constant expression 來指定陣列大小，即使是 const 變數也是非法的

```
int len = 10;
int arr[len]; //illegal

const in len2 = 10;
int arr2[len2]; //illegal
```

不過大家一定都用過上述第二種方法來指定陣列大小，而且 compiler 也都沒有噴過錯，那是因為大多數 compiler 都有優化，使得一些不合法的寫法也可以被接受

在 C++ 11 之前編譯器無法明確知道該變數或是函數回傳值就是個常量，所以在 C++11 中引入 constexpr 這個關鍵字告訴編譯器應該去驗證該變數或是函數回傳值在編譯期就是一個 constant expression，以下都是合法的

```
constexpr int len_foo_constexpr() {
    return 5;
}

...

constexpr int len = 10;
int arr[len];

constexpr int len2 = 1 + 2 + 3;
int arr2[len2];

int arr3[len_foo_constexpr()];
...

```

constexpr 的函數也可以用於遞迴

```
constexpr int fibonacci(const int n) {
    return n == 1 || n == 2 ? 1 : fibonacci(n-1) + fibonacci(n-2);
}
```

## Variables and initialization

### declare variable in if/switch

在 C++17 之前如果我們要找到 vector 中的特定值並修改他
```
const std::vector<int>::iterator itr = std::find(vec.begin(), vec.end(), 2);
if (itr != vec.end()) 
    *itr = 3;
```

這個地方的 `itr` 只是個暫時記錄用的變數，跟你寫 `for` 時裡面會用到的 `i`一樣，當你已經像上述宣告過 `itr`，`for` 迴圈的變數名稱就只能換一個用，在 C++17 後我們可以在 `if` 中宣告變數，讓開發者可以免去這種小煩惱

```
if (const std::vector<int>::iterator itr = std::find(vec.begin(), vec.end(), 3);
    itr != vec.end()) 
    *itr = 4;
```

---

持續更新中...
