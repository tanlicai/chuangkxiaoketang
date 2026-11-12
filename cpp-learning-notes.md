# C++ 基础学习笔记

本文整理本次学习 ESP-IDF 项目时遇到的 C++ 语法和设计。

## 1. `app_main`：ESP-IDF 程序入口

```cpp
extern "C" void app_main(void)
```

- `app_main` 是 ESP-IDF 启动后自动调用的程序入口，类似普通 C/C++ 程序中的 `main()`。
- `void` 表示函数没有返回值。
- `(void)` 表示函数不接收参数。
- `extern "C"` 告诉 C++ 编译器使用 C 的函数名规则，避免 C++ 名称修饰，使 ESP-IDF 能找到并调用这个函数。

项目中的启动流程大致是：

```text
ESP32 启动
  -> ESP-IDF 初始化
  -> 调用 app_main()
  -> 初始化 NVS
  -> 获取 Application 实例
  -> 初始化应用
  -> 运行主事件循环
```

## 2. `public` 和 `private`

`public:` 和 `private:` 是类的访问权限控制符。

```cpp
class Application {
public:
    void Initialize();

private:
    void ConnectWifi();
};
```

- `public`：类外部可以直接访问。
- `private`：类外部不能直接访问，只有类自己的成员函数可以访问。

```cpp
Application& app = Application::GetInstance();
app.Initialize();  // 可以
app.ConnectWifi(); // 不可以，ConnectWifi 是 private
```

`private` 可以隐藏实现细节，让外部只使用安全、明确的公开接口。可以把它理解为：

```text
public  = 对外开放的按钮
private = 类内部的实现细节
```

对于 `class`，没有写权限标签时默认是 `private`；对于 `struct`，默认是 `public`。

## 3. `static Application& GetInstance()`

```cpp
static Application& GetInstance() {
    static Application instance;
    return instance;
}
```

这行函数定义可以从左到右理解：

```text
static        Application&        GetInstance()
静态函数      返回 Application 引用  函数名，无参数
```

### 3.1 成员函数前的 `static`

```cpp
static Application& GetInstance()
```

表示 `GetInstance()` 是静态成员函数，属于 `Application` 类本身，而不是某个具体对象。

因此可以在没有对象时，直接通过类名调用：

```cpp
Application::GetInstance();
```

`::` 是作用域运算符，表示在 `Application` 类的作用域中查找 `GetInstance`。

### 3.2 函数内部的 `static`

```cpp
static Application instance;
```

表示 `instance` 只在第一次执行到这一行时创建一次，并在函数结束后继续存在。

```cpp
Application::GetInstance(); // 第一次调用：创建 instance
Application::GetInstance(); // 后续调用：返回同一个 instance
```

这实现了单例模式：整个程序只使用一个 `Application` 对象。

## 4. 引用 `&`

`&` 的含义取决于它出现的位置。

### 4.1 表达式中：取地址

```cpp
int number = 10;
int* pointer = &number;
```

这里的 `&number` 表示获取 `number` 的内存地址。

### 4.2 类型声明中：引用

```cpp
int& alias = number;
```

这里的 `&` 表示 `alias` 是 `number` 的引用，也就是 `number` 的另一个名字。

```cpp
alias = 20;
// number 也变成 20
```

在下面的代码中：

```cpp
Application& app = Application::GetInstance();
```

`Application&` 是引用类型，不是取地址。`app` 是唯一 `Application` 对象的别名。

## 5. `auto& app` 和 `Application& app`

下面两种写法在这里完全等效：

```cpp
auto& app = Application::GetInstance();
Application& app = Application::GetInstance();
```

- `Application&`：明确写出类型。
- `auto&`：让编译器根据右侧表达式自动推断类型。

由于 `GetInstance()` 返回 `Application&`，所以 `auto&` 会被推断为 `Application&`。

## 6. 为什么不直接使用普通对象？

下面的写法会尝试复制一个 `Application` 对象：

```cpp
auto app = Application::GetInstance();
```

但 `Application` 类禁止复制：

```cpp
Application(const Application&) = delete;
Application& operator=(const Application&) = delete;
```

而且 `Application` 管理着音频、网络和事件等资源，复制它可能导致资源冲突。

使用引用：

```cpp
auto& app = Application::GetInstance();
```

不会创建新对象，只是给唯一实例起一个短名字。

也可以不声明 `app`，直接调用：

```cpp
Application::GetInstance().Initialize();
Application::GetInstance().Run();
```

但重复写 `Application::GetInstance()` 不够简洁，所以通常使用引用保存这个实例。

## 7. 引用和指针的区别

### 引用

```cpp
Application& app = Application::GetInstance();
app.Initialize();
```

- 是对象的别名。
- 正常情况下不能为 `nullptr`。
- 初始化后不能改为引用另一个对象。
- 使用 `.` 调用成员。

### 指针

```cpp
Application* app = &Application::GetInstance();

if (app != nullptr) {
    app->Initialize();
}
```

- 保存对象的内存地址。
- 可以是 `nullptr`。
- 可以改为指向另一个对象。
- 使用 `->` 调用成员。

`Application&` 中的 `&` 表示引用；`&Application::GetInstance()` 中的 `&` 表示取地址。

## 8. `Application::GetInstance()` 的完整含义

```cpp
Application::GetInstance()
```

可以拆成：

```text
Application       类名
::                作用域运算符
GetInstance()     调用获取实例的函数
```

它表示调用 `Application` 类中的静态函数，获取唯一的 `Application` 实例。

结合项目代码：

```cpp
auto& app = Application::GetInstance();
app.Initialize();
app.Run();
```

等价于：

```cpp
Application& app = Application::GetInstance();
app.Initialize();
app.Run();
```

整体意思是：

```text
获取 Application 的唯一实例
-> 用 app 引用它
-> 初始化应用
-> 运行应用主事件循环
```

