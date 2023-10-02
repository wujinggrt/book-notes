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

## Proxy

使用Proxy对象，我们可以推迟执行一些耗时的计算或者申请新对象操作，避免计算一些不需要的结果。

例子如：有两个连接的字符串s1, s2，连接之后与另一个字符串s3相比，完全可以不用新申请一个std::string{s1 + s2}，而是使用迭代器，将s3前部分和s1比较，后部分与s2比较。

```
class String {
public:
    String() = default;
    String(std::string istr) : str_{std::move(istr)}{}
    std::string str_{};
};

class ConcatProx {
    const std::string& a;
    const std::string& b;
    operator String() const && { return String{a + b}; }
};

auto operator+(const String& a, const String& b) {
    return ConcatProxy{ a.str_, b.str_ };
}
```

例子如两个顶点，比较距离的时候，需要计算顶点x0-x1和y0-y1的平方，最后在开根。在只需要比较距离的场景，不需要开根操作即可比较，可以有省去计算缓慢开根的空间。

```
class Point {
public:
    Point(float x, float y) : x_{x}, y_{y} {}
    auto distance(const Point& p) const {
        return DistProxy{x_, y_, p.x_, p.y_}; 
    }
    float x_{};
    float y_{};
};

class DistProxy{
public:
    DistProxy(float x0, float y0, float x1, float y1)
        : dist_sqrd_{ std::pow(x0-x1, 2) + std::pow(y0-y1, 2) } {}
    auto operator<(float dist) const { return dist_sqrd_ < dist*dist; }
    auto operator<(const DistProxy& dp) const {
        return dist_sqrd_ < dp.dist_sqrd_; }
    // compare with real dist
    auto operator<(float dist) const {
        return dist_sqrd_ < dist*dist; }
    // Implicit cast to float
    operator float() const { return std::sqrt(dist_sqrd_); }
    // 如果为了避免用户混用成float的值，可以限制成仅在临时对象时候可以转换
    // operator float() const && { return std::sqrt(dist_sqrd_); }
private:
    float dist_sqrd_{};
};

Point a{x0, y0};
Point b{x1, y1};
Point spot{xs, ys};
// 隐式转换
float dist = a.distance(b); // Note that we cannot use auto here
auto dist_squared = a.distance(b);
float dist_float0 = dist_squared; // Assignment invoked std::sqrt()
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

#### 引用
* [1] *C++ High Performance*.