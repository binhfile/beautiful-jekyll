---
layout: post
title: Thực hiện singleton pattern trong C++ 
image: 
categories: [c++]
tags: [c++, template]
---

## Mục đích  
Thực hiện singleton pattern trong C++11.
## Thực hiện  
```cpp
#include <memory>
#include <mutex>
template <typename Derived> class singleton {
  public:
    singleton() = default;
    virtual ~singleton() = default;

    static std::shared_ptr<Derived> instance() {
        std::call_once(s_init_flag_, []() { s_inst_ = std::make_shared<Derived>(); });
        return s_inst_;
    }
    static void destroy() { s_inst_.reset(); }

  private:
    static std::once_flag s_init_flag_;
    static std::shared_ptr<Derived> s_inst_;
};
template <typename Derived> std::once_flag singleton<Derived>::s_init_flag_;
template <typename Derived> std::shared_ptr<Derived> singleton<Derived>::s_inst_;
```
## Sử dụng  
```cpp
class Foo : public singleton<Foo> {
  public:
    Foo() : singleton<Foo>() {}
    virtual ~Foo() = default;
    void Test() { std::cout << this << " - Foo: test" << std::endl; }
};
class Bar : public singleton<Bar> {
  public:
    Bar() : singleton<Bar>() {}
    virtual ~Bar() = default;
    void Test() { std::cout << this << " - Bar: test" << std::endl; }
};

#include <chrono>
#include <thread>
int main(int argc, const char **args) {
    std::this_thread::sleep_for(std::chrono::milliseconds(1));

    Foo::instance()->Test();
    Foo::instance()->Test();

    Bar::instance()->Test();
    Bar::instance()->Test();

    return 0;
}
// output
//  0x1c34c30 - Foo: test
//  0x1c34c30 - Foo: test
//  0x1c35060 - Bar: test
//  0x1c35060 - Bar: test
```
## Tham khảo  
- [Mã nguồn](https://github.com/binhfile/example-code/blob/master/cpp/singleton/singleton.cpp)  


