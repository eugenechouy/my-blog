
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
### initializer_list

C++ 11 引入新的型態 `<initializer_list>`，這個型態簡化了 C++ STL 的初始化，以往我們針對不同的 STL 容器的初始化方式都不同

```
# vector 1
int arr[] = {1, 2, 3, 4, 5};
vector<int> vec(arr, arr+sizeof(arr));

# vector 2
vector<int> arr2;
for(int i=1 ; i<=5 ; i++)
    arr2.emplace_back(i); # emplace 後面會提到

# map
map<string, string> mp;
mp["one"] = "Taipei";
mp["two"] = "Taoyuan";
```

可以看到要嘛要先建一個陣列要嘛一個一個放，運用`<initializer_list>`的特性就簡化許多

```
vector<int> vec = {1, 2, 3, 4, 5};
map<string, string> mp = {
    "one": "Taipei",
    "two": "Taoyuan",
    ...
}
```

除了初始化 container 外也可以用來初始化我們自己寫的 class ，我們只需要在 class 中定義一個　`initialize list constructor`

```
class MagicFoo {
public:
    std::vector<int> vec;
    MagicFoo(std::initializer_list<int> list) {
        for (std::initializer_list<int>::iterator it = list.begin(); 
             it != list.end(); ++it)
            vec.push_back(*it);
    }
};

...

MagicFoo = {1, 2, 3, 4, 5};
```

基本上只要用到大括號諸如初始化、函式參數，就會自動建構一個 `initialize list`，可以把它看成一個常量，當你複製`initialize list` 並不會真的複製而是引用而已。

### Structured binding

C++ 11 後引入 `<tuple>` 讓我們可以把不同值包在一起，但是在 C++17 後才能方便的使用，如下
```
tuple<int, double, string> foo() {
	return make_tuple(1, 1.1, "111");
}
...

auto [x, y, z] = foo();
```

## Type inference

C++ 跟 C 的巨大區別之一就是引入了 `auto` 和 `decltype`，`auto` 大家都很熟就不多說了，只是有兩點要注意

* 不能用來當作函式參數的型態
```
int add(auto x, auto y); // error
```
* 不能用於宣告陣列
```
auto arr[10]; // error
```

會有以上兩種限制也是很自然的，上述狀況都有關記憶體分配，如果指定為 auto  程式會不知道該分配多少大小的位址。

### decltype

`decltype` 用來計算某表達式的類型 -> ```decltype(expression)```

```
auto x = 1;
auto y = 2;
decltype(x+y) z;
```

或是搭配 `is_same<T, U>::value` 判斷兩個類型是否相等

```
if (is_same<decltype(x), int>::value)
    cout << "type x == int" << endl;
```

另外在 C++ 中是不能把類型當作參數的，所以如果你嘗試印出類型會噴錯

```
cout << decltype(x) << endl; // error
```

所以如果你真的想要印出某個變數的類型可以透過 template

```
template <class T>
void printType(const T&){
    cout << __PRETTY_FUNCTION__ << endl;
}
...
printType(x);

// output
$ void printType(const T&) [with T = int]
```

### tail type inference

前面提到說 `auto` 不能當作函式參數的類型，那能不能當作返回型態呢，C++ 14 後可以

```
template<typename T, typename U>
auto add(T x, U y){
    return x + y;
}
```

`tail type inference` 是在 C++14 之前因為函數不具備返回值推導，所以需要這個機制來指定返回型態

```
template<typename T, typename U>
auto add(T x, U y) -> decltype(ｘ+y){
    return x + y;
}
```

至於為什麼不把返回型態寫成 `decltype(ｘ+y)` 就好是因為讀到前面是程式還不知道 x 和 y 是什麼

### decltype(auto) ??

作者表示要看完下一章比較好理解
