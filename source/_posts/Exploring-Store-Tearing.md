---
title: Exploring Store Tearing
categories:
  - Programming
tags:
  - C++
  - C
  - Memory-Model
  - Concurrency
date: 2025-11-30 10:00:00
---

Store tearing is a fascinating concurrency bug that occurs when a processor cannot update a memory location in a single atomic instruction. If a thread reads that memory location while the write is only partially complete, it sees a "torn" value—a mix of the old and new data.

Let's dive into what this looks like and how to prevent it.

<!-- more -->

## What is Store Tearing?

In modern multi-threaded programming, we often assume that writing to a variable is an instantaneous event. We think: "I set `x = 5`, so `x` is either the old value or `5`."

However, hardware doesn't always work that way. If the data type you are writing is larger than the processor's native word size, or if the data is not aligned properly in memory, the processor might have to issue multiple write instructions to update the value.

If a context switch happens or another core reads that memory address _between_ those write instructions, the reader will see a value that never actually existed in the program logic.

## A Conceptual Example

Imagine a 64-bit integer variable initialized to `0x0000000000000000`.
We have two threads:

1.  **Thread Writer**: Writes `0xFFFFFFFFFFFFFFFF` (-1) to the variable.
2.  **Thread Reader**: Reads the variable.

On a 32-bit architecture (or a 64-bit architecture handling an unaligned write), the processor might split the 64-bit write into two 32-bit stores:

1.  Store `0xFFFFFFFF` to the lower 32 bits.
2.  Store `0xFFFFFFFF` to the upper 32 bits.

If the **Reader** reads after step 1 but before step 2, it sees:
`0x00000000FFFFFFFFFF`

This value is neither the starting value (`0`) nor the ending value (`-1`). It is a "torn" write.

## Code Example in C++

Here is a C++ example that could theoretically exhibit store tearing. Note that reproducing this deterministically is difficult because it depends heavily on timing, OS scheduling, and hardware architecture.

```cpp
#include <iostream>
#include <thread>
#include <cstdint>

// A struct that might be larger than a single atomic store instruction or unaligned.
struct Data {
    uint64_t a;
    uint64_t b;
};

Data shared_data = {0, 0};
bool running = true;

void writer()
{
    for (int i = 0; i < 1000000; ++i)
    {
        if (!running)
            break;
        // Write Set A
        shared_data = {0, 0};
        // Write Set B
        shared_data = {0xFFFFFFFFFFFFFFFF, 0xFFFFFFFFFFFFFFFF};
    }
    running = false;
    std::cout << "Writer finished." << std::endl;
}

void reader() {
    while (running) {
        Data local = shared_data;
        // If we see a mismatch, or a mix of bits, we have tearing.
        if (local.a != local.b) {
            std::cout << "Tearing detected! a=" << std::hex << local.a
                      << ", b=" << local.b << std::dec << std::endl;
            running = false;
        }
        // Even within a single uint64_t, tearing could occur on 32-bit systems,
        // though checking a != b is an easier heuristic for this struct example.
    }
    std::cout << "Reader finished." << std::endl;
}

int main() {
    std::thread t1(writer);
    std::thread t2(reader);

    t1.join();
    t2.join();
    return 0;
}
```

In the example above, `shared_data` is a struct. The compiler will likely generate multiple instructions to copy the struct. If the reader interrupts the writer, `local.a` might be from the "all ones" set while `local.b` is from the "all zeros" set.

### A Hidden Bug: Data Race on `running`

Did you spot another issue in the code above?

The variable `bool running` is accessed by both threads without any synchronization. This is technically a **data race**, which is Undefined Behavior. While it often "just works" on x86 for simple booleans, the compiler is free to optimize the loop in `reader()` to assume `running` never changes (since it's not modified within that loop), potentially causing an infinite loop.

To fix this properly, `running` should also be `std::atomic<bool>`.

## How to Fix It

The solution to store tearing is to ensure **atomicity**.

### 1. Use `std::atomic`

In C++, `std::atomic` guarantees that loads and stores are atomic.

```cpp
#include <atomic>

struct Data {
    uint64_t a;
    uint64_t b;
};

std::atomic<Data> shared_data; // Now writes to this are atomic
```

_Note: `std::atomic` might use locks internally if the hardware cannot support atomic operations for the size of `Data`, but it guarantees safety._

_Note: Since `Data` is 16 bytes (128 bits), some compilers/architectures require linking against the atomic library. If you see linker errors like `undefined reference to __atomic_store_16`, compile with `-latomic`:_

```bash
g++ main.cpp -o main -latomic
```

### 2. Use Mutexes

If you cannot use atomic types, use a `std::mutex` to protect the critical section.

```cpp
#include <mutex>

std::mutex mtx;

void writer() {
    for (int i = 0; i < 1000000; ++i) {
        if (!running) break;
        // Use scopes to lock only for the duration of the write
        {
            std::lock_guard<std::mutex> lock(mtx);
            shared_data = {0, 0};
        }
        {
            std::lock_guard<std::mutex> lock(mtx);
            shared_data = {0xFFFFFFFFFFFFFFFF, 0xFFFFFFFFFFFFFFFF};
        }
    }
    running = false;
    std::cout << "Writer finished." << std::endl;
}

void reader() {
    while (running)
    {
        Data local;
        {
            std::lock_guard<std::mutex> lock(mtx);
            local = shared_data;
        }

        // If we see a mismatch, or a mix of bits, we have tearing.
        if (local.a != local.b)
        {
            std::cout << "Tearing detected! a=" << std::hex << local.a
                      << ", b=" << local.b << std::dec << std::endl;
            running = false;
        }
    }
    std::cout << "Reader finished." << std::endl;
}
```

## Conclusion

Store tearing is a subtle bug that reminds us that hardware implementation details matter in concurrent programming. Always use proper synchronization primitives or atomic types when sharing data between threads, rather than relying on assumptions about how the processor writes to memory.
