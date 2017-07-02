---
　　layout: default
　　title: 你好，世界
---
# C++11 Future和Promise解析

Future是用来存储异步操作的结果的一个模板类

* 获得future的方式有以下几种：
```c++
1:future from packaged_task:
    std::packaged_task<int()> task([](){ return 7; }); // wrap the function
    std::future<int> f1 = task.get_future();  // get a future
2:future from a async:
    std::future<int> f2 = std::async(std::launch::async, [](){ return 8; });
3:future from a promise
    std::promise<int> p;
    std::future<int> f3 = p.get_future();
```
摘自[cppreference.com](http://en.cppreference.com/w/cpp/thread/future)

* 重要的成员函数：
1. **get**：返回结果，当future的结果还未有效时，会等待future的结果变成有效。
2. **wait**：等待结果变成有效，这个函数可以用来等待同步。
3. **wait_for**:等待一段时间后返回一个future的当前的状态。
```c++
// future::wait_for
#include <iostream>       // std::cout
#include <future>         // std::async, std::future
#include <chrono>         // std::chrono::milliseconds

// a non-optimized way of checking for prime numbers:
bool is_prime (int x) {
  for (int i=2; i<x; ++i) if (x%i==0) return false;
  return true;
}

int main ()
{
  // call function asynchronously:
  std::future<bool> fut = std::async (is_prime,700020007); 

  // do something while waiting for function to set future:
  std::cout << "checking, please wait";
  std::chrono::milliseconds span (100);
  while (fut.wait_for(span)==std::future_status::timeout)
    std::cout << '.';

  bool x = fut.get();

  std::cout << "\n700020007 " << (x?"is":"is not") << " prime.\n";

  return 0;
}
```
摘自[cplusplus.com](http://www.cplusplus.com/reference/future/future/wait_for/)

4. **wait_until**:在某个时间段内future还没有有效的话，返回timeout;这个函数会阻塞调用的进程。
* **future**和**promise**的配对：promise通过get_future函数和future进行绑定。promise通过set函数（四种set_value、set_value_at_thread_exit、set_exception、set_exception_at_thread_exit）进行设置值，然后使得future有效。
* **shared_future**能够拥有多个wait函数可用于多个子线程的之间的同步等待。
示例
```c++
#include <iostream>
#include <future>
#include <chrono>
 
int main()
{   
    std::promise<void> ready_promise, t1_ready_promise, t2_ready_promise;
    std::shared_future<void> ready_future(ready_promise.get_future());
 
    std::chrono::time_point<std::chrono::high_resolution_clock> start;
 
    auto fun1 = [&, ready_future]() -> std::chrono::duration<double, std::milli> 
    {
        t1_ready_promise.set_value();
        ready_future.wait(); // waits for the signal from main()
        return std::chrono::high_resolution_clock::now() - start;
    };
 
 
    auto fun2 = [&, ready_future]() -> std::chrono::duration<double, std::milli> 
    {
        t2_ready_promise.set_value();
        ready_future.wait(); // waits for the signal from main()
        return std::chrono::high_resolution_clock::now() - start;
    };
 
    auto result1 = std::async(std::launch::async, fun1);
    auto result2 = std::async(std::launch::async, fun2);
 
    // wait for the threads to become ready
    t1_ready_promise.get_future().wait();
    t2_ready_promise.get_future().wait();
 
    // the threads are ready, start the clock
    start = std::chrono::high_resolution_clock::now();
 
    // signal the threads to go
    ready_promise.set_value();
 
    std::cout << "Thread 1 received the signal "
              << result1.get().count() << " ms after start\n"
              << "Thread 2 received the signal "
              << result2.get().count() << " ms after start\n";
}
```
摘自[cppreference.com](http://en.cppreference.com/w/cpp/thread/shared_future)