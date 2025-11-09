---
title: What If We Call main() in main()
categories:
  - Programming
tags:
  - C++
date: 2025-11-08 04:04:21
---

I saw this question on TikTok and couldn't stop thinking - "What happens if you call `main()` inside `main()`?" It's one of those things you never think about until someone asks, and then you _have_ to try it.

Turns out, it's a perfect example of how `main()` is just a function like any other, and what happens when you push recursion too far.

<!-- more -->

## The Experiment

In most programming languages, `main()` is the entry point of your program. But technically, it's just a function. So what stops us from calling it recursively?

### C/C++ - The Classic Example

In C/C++, calling `main()` recursively is **undefined behavior** according to the standard, but many compilers allow it:

```c
#include <stdio.h>

int count = 0;

int main() {
    count++;
    printf("Iteration %d: main() called\n", count);

    if (count < 10) {
        main();  // Recursive call to main()
    }

    printf("Returning from iteration %d\n", count);
    return 0;
}
```

**Output:**

```
Iteration 1: main() called
Iteration 2: main() called
Iteration 3: main() called
Iteration 4: main() called
Iteration 5: main() called
Iteration 6: main() called
Iteration 7: main() called
Iteration 8: main() called
Iteration 9: main() called
Iteration 10: main() called
Returning from iteration 10
Returning from iteration 10
Returning from iteration 10
Returning from iteration 10
Returning from iteration 10
Returning from iteration 10
Returning from iteration 10
Returning from iteration 10
Returning from iteration 10
Returning from iteration 10
```

**What's happening?**

- Each call to `main()` creates a new stack frame
- Since `count` is **global**, it keeps incrementing across all calls
- When recursion stops at count=10, all 10 stack frames unwind
- Each unwinding frame prints "Returning from iteration 10" (the current value)
- This demonstrates that global variables are shared across all recursive calls!

### Stack Overflow Demonstration

Let's remove the limit and see what happens:

```c
#include <stdio.h>

int count = 0;

int main() {
    count++;
    if (count % 1000 == 0) {
        printf("Depth: %d\n", count);
    }
    main();  // Infinite recursion
    return 0;
}
```

**Result:** Stack overflow: xxxx (typically after 10,000-100,000 calls depending on your system).

### Python - More Explicit

Python makes this clearer with its recursion limit:

```python
import sys

sys.setrecursionlimit(20)  # Set a low limit for demonstration
count = 0

def main():
    global count
    count += 1
    print(f"Iteration {count}")
    main()  # Recursive call

if __name__ == "__main__":
    try:
        main()
    except RecursionError as e:
        print(f"\nRecursionError after {count} calls")
        print(f"Maximum recursion depth: {sys.getrecursionlimit()}")
```

**Output:**

```
Iteration 1
Iteration 2
Iteration 3
Iteration 4
Iteration 5
Iteration 6
Iteration 7
Iteration 8
Iteration 9
Iteration 10
Iteration 11
Iteration 12
Iteration 13
Iteration 14
Iteration 15
Iteration 16
Iteration 17

RecursionError after 18 calls
Maximum recursion depth: 20
```

**Why not 20 iterations?**
Python's recursion limit includes internal function calls too. The initial `if __name__ == "__main__"` check and the try-except block add to the call stack, so the actual limit for our recursive function is slightly less than 20.

## Key Takeaway

Calling `main()` recursively is problematic:

1. **Violates C/C++ standards** - `main()` should only be called by the runtime
2. **Stack overflow risk** - Without a base case, guaranteed crash
3. **Poor code design** - Use a separate function if you need recursion

`main()` is just another function. Like any function, recursive calls consume stack space. The stack is finite, so infinite recursion = stack overflow.
