---
title: "Test1"
date: 2020-05-16T14:28:27+03:00
draft: true
tags: [a.b, c.d]
---

So if we write something like this, we'll get undefined beheviour (UB).

{{<admonition bug "UB" false>}}
```cpp
#include <iostream>

int main() {
  int a;
  std::cout << *(&a + 1) << std::endl;
}
```
{{</admonition>}}

So this is UB
