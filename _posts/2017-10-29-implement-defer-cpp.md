---
layout: post
title: Thực hiện defer trong C++ 
image: 
categories: [c++]
tags: [c++, template, macro]
---

## Mục đích  
Ngôn ngữ Go cung cấp cấu trúc defer expr với mục đích thực hiện expr khi ra khỏi phạm vi khai báo.  
```go
func Foo{
    defer log.Println("call defer")
    log.Println("enter")
    log.Println("processing")
    log.Println("exit")
}
// output:
//  enter
//  processing
//  exit
//  call defer
```
Ta sẽ thực hiện cấu trúc defer bằng ngôn ngữ C++11.  
## Thực hiện  
Các biểu thức trong defer sẽ được chứa trong 1 cấu trúc lambda với cấu trúc ví dụ như:  
```cpp
int index = 10;
DEFER([&index]{
    std::cout << "index=" << index << std::endl;
})
```
Định nghĩa 1 kiểu đối tượng chứa lambda: 
```cpp
template <typename Lambda>
class defer_impl {
  public:
    defer_impl(const Lambda &expr) : expr_(expr) {}
    virtual ~defer_impl() {
        expr_();
    }

  private:
    Lambda expr_;
};
```
Tại sao không sử dụng std::function: lời gọi std::function phức tạp và lâu hơn nhiều so với lambda.
Cùng một implement, std::function tạo ra ~2000 dòng code asm, trong khi lambda chỉ tạo ra ~200 dòng code.  

Sử dụng MACRO để tạo ra biến chứa đối tượng defer_impl tương ứng với lambda được khai báo. 
Điều kiện ở đây là mỗi lần khai báo DEFER cần tạo ra 1 biến riêng biệt. Điều này có thể thực hiện bằng cách sử dụng
MACRO `__COUNTER__` của compiler.  
```cpp
// clang-format off
#define TOKEN_CONCAT(x, y) x ## y
#define DEFER_IMPL(lname, vname, expr)                      \
    auto lname = expr;                                      \
    __hidden__::defer_impl<decltype(lname)> vname(lname);
#define DEFER_INDEX(index, expr) DEFER_IMPL(TOKEN_CONCAT(_hidden_defer_lambda_, index), TOKEN_CONCAT(_hidden_defer_impl_, index), expr)
#define DEFER(expr) DEFER_INDEX(__COUNTER__, expr)
// clang-format on
```
`__COUNTER__` sẽ được compiler tự động tăng sau mỗi lần gọi.

## Sử dụng  
```cpp
int main(int argc, const char **args) {
    DEFER([] {
        std::cout << "app - defer" << std::endl;
    })
    std::cout << "app - start" << std::endl;
    {
        int index = 1;
        DEFER([&index] {
            std::cout << index << " - defer" << std::endl;
        })
        std::cout << index << " - enter" << std::endl;

        std::cout << index << " - exit" << std::endl;
    }
    {
        int index = 2;
        DEFER([&index] {
            std::cout << index << " - defer 1 " << std::endl;
        })
        DEFER([&index] {
            std::cout << index << " - defer 2" << std::endl;
        })
        std::cout << index << " - enter" << std::endl;

        std::cout << index << " - exit" << std::endl;
    }
    std::cout << "app - stop" << std::endl;
    return 0;
}
```
Sẽ cho kết quả:  
```text
app - start
1 - enter
1 - exit
1 - defer
2 - enter
2 - exit
2 - defer 2
2 - defer 1
app - stop
app - defer
```

## Tham khảo  
- [Mã nguồn](https://github.com/binhfile/example-code/blob/master/cpp/defer/defer.cpp)  
- [CppCon 2014](https://github.com/CppCon/CppCon2014)  


