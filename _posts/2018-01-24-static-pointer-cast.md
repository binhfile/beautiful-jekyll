---
layout: post
title: Thực hiện static_pointer_cast thay thế static_cast 
image: 
categories: [c++]
tags: [c++]
---

# Vấn đề  

Khi muốn cast các kiểu không có liên hệ (không kế thừa nhau), ta thường sử dụng  
```cpp
uint32_t a = 0x12345678;
uint8_t* p = (uint8_t*)&a;
```  
Cách này sẽ bị compiler warning và gợi ý sử dụng static_cast.  
```cpp
uint32_t a = 0x12345678;
uint8_t* p = static_cast<uint8_t*>(&a);
// gặp lỗi invalid static_cast from type ‘uint32_t* {aka unsigned int*}’ to type ‘uint8_t* {aka unsigned char*}’ 
// do kiểu uint32_t* và uint8_t* không có liên hệ với nhau. 
```  
Giải quyết bằng cách chuyển đổi kiểu về `void*` trước  
```cpp
uint32_t a        = 0x12345678;
uint8_t* p1       = static_cast<uint8_t*>(static_cast<void*>(&a));
const uint8_t* p2 = static_cast<const uint8_t*>(static_cast<const void*>(&a));
```  
Cách trên đòi hỏi thời gian typing lâu, nhàm chán ;) . Ta sẽ viết một API rút gọn các thao tác trên với dạng như sau  
```cpp
uint32_t a        = 0x12345678;
uint8_t* p1       = static_pointer_cast<uint8_t*>(&a);
const uint8_t* p2 = static_pointer_cast<const uint8_t*>(&a);
``` 

# Thực thi sử dụng std::enable_if và SFINE  
```cpp
#include <type_traits>
// DstType is const
template <typename DstType, typename SrcType>
DstType static_pointer_cast(
    typename std::enable_if<std::is_const<typename std::remove_pointer<DstType>::type>::value,
                            SrcType>::type &&src,
    SrcType arg_for_deduction = nullptr) {
    UNUSED(arg_for_deduction);
    return static_cast<DstType>(static_cast<const void *>(std::forward<SrcType>(src)));
}
// otherwise
template <typename DstType, typename SrcType>
DstType static_pointer_cast(SrcType &&src) {
    return static_cast<DstType>(static_cast<void *>(std::forward<SrcType>(src)));
}
```
Compiler sẽ chọn API trên cho `const` pointer và API dưới cho `non const` pointer.  

# Thực thi sử dụng std::conditional  
```cpp
#include <type_traits>
template <typename DstType, typename SrcType>
DstType static_pointer_cast(SrcType &&src){
	return static_cast<DstType>(
		static_cast<
			typename std::conditional<
				std::is_const<
					typename std::remove_pointer<DstType>::type
				>::value, const void*, void*
			>::type
		>(std::forward<SrcType>(src)));
}
```  
Phiên bản này có tính năng tương tự phiên bản sử dụng std::enable_if nhưng ngắn hơn ;)  

Tất cả các các thực thi trên đều hoạt động với phiên bản C++11.  

## Tham khảo  
- http://en.cppreference.com/w/cpp/language/sfinae
- http://en.cppreference.com/w/cpp/types/enable_if
- http://en.cppreference.com/w/cpp/types/conditional
- http://en.cppreference.com/w/cpp/utility/forward

