---
layout: post
title: "Syntax Highlighting Test"
date: 2024-01-15
author: "Andrii Zinoviev"
tags: [test, swift, c]
---

# Syntax Highlighting Test

This post tests if Swift and C syntax highlighting is working properly.

## Swift Code Example

```swift
import Foundation

class ExampleClass {
    var property: String = "Hello, World!"
    
    func greet(name: String) -> String {
        return "Hello, \(name)!"
    }
    
    func calculateSum(_ numbers: [Int]) -> Int {
        return numbers.reduce(0, +)
    }
}
```

## C Code Example

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    printf("Hello, World!\n");
    
    int numbers[] = {1, 2, 3, 4, 5};
    int sum = 0;
    
    for (int i = 0; i < 5; i++) {
        sum += numbers[i];
    }
    
    printf("Sum: %d\n", sum);
    return 0;
}
```

## Plaintext Example

```plaintext
This is plain text without syntax highlighting.
```

The syntax highlighting should now work for Swift and C code blocks. 