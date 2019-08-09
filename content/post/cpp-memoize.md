---
title: "C++11實作Memoziation"
date: 2019-08-09T19:53:11+08:00
draft: true
tags:
  - cpp
  - memoization
description:
---
## 什麼是Memoization

Memoization是一種優化程式執行速度的技巧，大致上的概念是在函式運算結束後，將結果儲存在記憶體中，如果下次遇到同樣的輸入參數，可以直接回傳結果而不用重新計算。

基於以上的概念，能夠透過Memoization加速的函式，必須符合以下兩個條件：

- 函式回傳值與輸入參數一對一對應

也就是說在任何情況下，輸入相同的參數，必定得到相同的回傳值。這也是我們能夠透過儲存回傳值來避免重複運算的根本原因。

- 函式的運算時間必須足夠長

Memoization加速的原理在於透過傳入參數查詢對應回傳值，如果函式本身的運算時間低於「查詢」（通常是Hash Lookup）的話，那麼使用Memoization反而會使得函式執行時間變得更長。

## 從一個簡單例子開始

以下這個例子中，我們呼叫了運算時間長達兩秒的函式兩次，兩次所花的時間都差不多是兩秒。（我知道聽起來是廢話）

```cpp
#include <chrono>
#include <iostream>
#include <unistd.h>

class Handler {
public:
  int Process(int arg) {
    // 超級無敵複雜的運算
    usleep(2000);
    ret = arg + 1;
    return ret;
  }
};

int main()
{
  using namespace std::chrono;
  Handler handle;

  // 第一次呼叫
  auto start = high_resolution_clock::now();
  handle.Process(3);
  auto stop = high_resolution_clock::now();
  auto duration = duration_cast<microseconds>(stop - start);
  std::cout << "1st call: " << duration.count() << " ms" << std::endl;

  // 第二次呼叫
  start = high_resolution_clock::now();
  handle.Process(3);
  stop = high_resolution_clock::now();
  duration = duration_cast<microseconds>(stop - start);
  std::cout << "2nd call: " << duration.count() << " ms" << std::endl;

  return 0;
}
```

執行結果

```bash
1st call: 2152 ms
2nd call: 2148 ms
```

第一次呼叫`handle.Process(3)`得到回傳值以後，其實我們已經可以預期第二次使用同樣參數呼叫，必定得到相同的回傳值。然而在這個例子中，第二次呼叫`handle.Process(3)`時，依然必須承受超級無敵複雜運算的時間，明顯不是很有效率。

既然我們知道函式回傳值與傳入參數是一對一的對應關係，我們可以將第一次呼叫的結果儲存在`std::map`中，並且用傳入參數當作key，如此一來下次遇到先前算過的參數，就可以直接從`std::map`中找答案，而不用再次計算。

我們可以把`class Handler`做一點修改

```cpp
class Handler {
public:
  int Process(int arg) {
    auto Find = ProcessCache.find(arg);
    if (Find != ProcessCache.end()) {
      // 先前算過這個參數，直接將儲存的值回傳
      return Find->second;
    }

    // 超級無敵複雜的運算
    usleep(2000);
    auto ret = arg + 1;

    // 算完以後儲存起來以便之後使用
    ProcessCache[arg] = ret;

    return ret;
  }

private:
  // 增加一個map來記錄回傳值
  std::map<int, int> ProcessCache;
};
```

執行結果

```bash
1st call: 2091 ms
2nd call: 0 ms
```

第二次呼叫的時間變得超快，因為我們繞過了超級無敵複雜的運算，從`std::map`中直接得到了答案。
