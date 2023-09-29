## Lazy evaluation

马上初始化资源的，称为eager evaluation，延迟到需要时再初始化方式称为lazy evaluation。后者能够避免某些不必要的运算开销。

```
auto eager_get = [] (string_view path, audio default) {
    return audio::exists(path) ? audio::load(path) : default;
};
auto lazy_get = [] (string_view path, function<audio()> default) {
    return audio::exists(path) ? audio::load(path) : default();
};
auto fox_sound = eager_get("fox.wav", audio::load("default.wav"));
auto pig_sound = lazy_get("pig.wav", 
        []() { return audio::load("default.wav")} );
```

## 右值和eXpiring value

将右值的概念进行了进一步的划分：分为：纯右值（pure rvalue）、将亡值（eXpiring Value）；而将亡值是C++11新引入的概念，它依托于右值。


纯右值用于辨别临时变量和一些不跟对象管理的值。
* 非引用返回的函数返回的临时变量值
* 运算表达式产生的临时对象，比如1+2产
* 生的临时变量值
* 不跟对象关联的原始字面量，比如2，true，‘c’
* 类型转换函数的返回值
* lambda表达式等等

亡值(eXpiring value)，是C++11为了引入右值引用而提出的概念(因此传统C++中，纯右值和右值是同一个概念)，也就是即将被销毁、却能够被移动的值， 比如：
* 返回右值引用t&&的函数返回值
* std::move的返回值
* 转换为T&&的类型转换函数的返回值

`const T&`可以绑定到右值上。比如`const int& ct = 1;`但是不能够改变它，因为是const的。

右值引用可以绑定到右值上，延长lifetime。比如`const char*&& s = "rvalue"; s = "still rvalue";`

```
int main() {
    const int& ci = 1;
    // ci = 3; // cannot assign, const-qualified type
    int&& i = 1;
    i = 3; // ok to change
    cout << i << endl;
    const char*&& s = "rvalue";
    s = "still rvalue"; // ok to assign, s is just a ptr, we have just modified s itself but not characters.
    cout << s << endl;
}
```