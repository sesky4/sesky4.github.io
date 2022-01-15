# Atomic

## 参考:
 - [CppCon 2017: Fedor Pikus “C++ atomics, from basic to advanced. What do they really do?”](https://www.youtube.com/watch?v=ZQFzMfHIxng)

## Atomic
- C++中Atomic不支持乘法，因为硬件通常都不支持

## CAS(Compare-and-swap)操作

CAS操作大概解释为: 如果变量 value == 我期望的值 old_v，那么 value = new_value ，否则什么都不做，伪代码：
```
value = value == old_v ? new_v : value
```

但是由于这个步骤会被拆分为多条机器指令，因此在多线程环境下会有问题

解决该问题就要加锁使得`value = value == old_v ? new_v : value`这行代码不能被多个cpu核心同时执行，伪代码：
```
bool compare_and_swap(T& old_v, T& new_v) {
  Lock(); // Lock机制由cpu内部提供
  
  T tmp = value;
  
  if (tmp != old_v) {
    old_v = tmp; // 失败的时候把当前值返回，好让程序知道接下来该怎么做
    Unlock();
    return false; 
  }
  
  value = new_v;
  Unlock();
  return true;
}
```

由于Lock()操作相较于value的read操作而言更为昂贵，因此可以进一步优化为：
```
bool compare_and_exchange(T& old_v, T& new_v) {
  T tmp = value;
  
  // fail fast: 
  // 先尝试读取一下，如果失败了就直接返回了，避免了昂贵的Lock()
  if (tmp != old_v) {
    old_v = tmp;
    return false; 
  }
  
  Lock();
  
  // double check:
  // value的值可能在上一次读取之后被改变了，因此这里要重新读取一次值
  tmp = value;
  
  if (tmp != old_v) {
    old_v = tmp;
    Unlock();
    return false; 
  }
  
  value = new_v;
  Unlock();
  return true;
}
```

C++中CAS操作有`atomic_compare_exchange_weak`和`atomic_compare_exchange_strong`，二者有什么区别？

`atomic_compare_exchange_strong`就如同上面的`compare_and_exchange`伪代码实现

`atomic_compare_exchange_weak`伪代码如下：
```
bool atomic_compare_exchange_weak(T& old_v, T& new_v) {
  T tmp = value;
  
  if (tmp != old_v) {
    old_v = tmp;
    return false; 
  }
  
  // 区别在这里：
  // Lock操作需要当前核心和其他核心通信，达成一致，这需要耗时，而且时间不固定
  // 把它想像成一个http请求，客户端可以设定超时时间，如果超时也认为失败
  if ( !Lock(timeout) ) {
    return false;
  }
  
  tmp = value;
  
  if (tmp != old_v) {
    old_v = tmp;
    Unlock();
    return false; 
  }
  
  value = new_v;
  Unlock();
  return true;
}
```
