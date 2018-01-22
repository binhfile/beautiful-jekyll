---
layout: post
title: [Error] Pure virtual method called with std::thread
image: 
categories: [c++]
tags: [c++, error]
---

Pure virtual method called  

```c++
#include <iostream>
#include <thread>

using namespace std;

class A {
    public:
        A() = default;
        virtual ~A(){
            Wait();
        }
        
        void Run(){
            thread_ = std::thread([this]{ run(); });
        }
        void Wait(){    
            if(thread_.joinable()) 
                thread_.join();
        }
    protected:
        virtual void run() = 0;
    private:
        std::thread thread_;
};
class B : public A {
    public:
        B() = default;
        virtual ~B() = default;
        void run() override {
            std::cout<<"B"<<std::endl;
        }
};
int main()
{
    A* ptr = new B;    
    ptr->Run();    
    delete ptr;

    return 0;
}
```

Tạo ra lỗi khi runtime  
```bash
pure virtual method called
```

Lỗi ở đây là do thread_ ở object A sử dụng phương thức run của object B. Tuy nhiên ngay sau khi khởi tạo thread, object B bị hủy (trước khi A bị hủy ). Do đó thread_ không phân giải được địa chỉ của B::run. Giải pháp là kiểm tra chắc chắn thread_ đã được thiết lập thành công trước khi hủy B.  
```c++
class A {
    public:
		...
        void Wait(){    
            if(thread_.joinable()) 
                thread_.join();
        }
	...
int main()
{
    A* ptr = new B;    
    ptr->Run();    
    ptr->Wait();  //<- wait here
    delete ptr;

    return 0;
}
```

Lỗi này gặp ở GCC (đã test tới phiên bản 8.0) và Clang (đã test với phiên bản 5.0). Không gặp ở MSVC 2015.


## Tham khảo  
- https://tombarta.wordpress.com/2008/07/10/gcc-pure-virtual-method-called/
- https://wandbox.org  

