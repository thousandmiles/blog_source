---
title: "Understanding ABI: The Invisible Contract"
categories:
  - Programming
tags:
  - C++
  - C
  - ABI
date: 2025-11-30 15:00:00
---

We often talk about APIs (Application Programming Interfaces) when connecting software components. But there is a deeper, lower-level contract that keeps our programs running: the **ABI** (Application Binary Interface).

While an API defines _source code_ compatibility (function names, arguments), the ABI defines _binary_ compatibility—how those functions are actually executed by the processor.

Let's explore what makes up an ABI and why breaking it causes such headaches.

<!-- more -->

## API vs. ABI

- **API (Source Level):** Defines how you write code to interface with a library.

  - Example: `void print_msg(const char* msg);`
  - If you change the function signature, your code won't compile. This is an **API break**.

- **ABI (Binary Level):** Defines how the compiled machine code interacts.
  - Example: "The first argument goes into the `RDI` register. The return address is on the stack."
  - If you change the ABI (e.g., change the size of a struct used in a function), your code might compile fine but crash at runtime. This is an **ABI break**.

## Key Components of an ABI

### 1. Data Layout and Alignment

How are data types stored in memory?

- **Size:** How many bytes is an `int`? (4 bytes? 8 bytes?)
- **Alignment:** Does a `double` need to start at an address divisible by 8?
- **Padding:** How does the compiler insert gaps in structs to satisfy alignment?

**Example:**

```cpp
struct Point {
    char c;
    int x;
};
```

On many systems, `sizeof(Point)` is 8, not 5, because of 3 bytes of padding after `c` to align `x`. If Library A is compiled with one alignment setting and Application B with another, they will disagree on where `x` lives in memory.

### 2. Calling Conventions

When Function A calls Function B, they must agree on:

- **Parameters:** Are arguments passed in registers or on the stack? Which registers? In what order?
- **Return Values:** Where is the result stored? (`RAX` register?)
- **Stack Cleanup:** Who cleans up the stack arguments? The caller or the callee?

Common conventions include `cdecl`, `stdcall`, `fastcall`, and the System V AMD64 ABI (used by Linux).

### 3. Name Mangling

In C, function names in the object file are usually just the function name (e.g., `_printf`).
In C++, because of overloading and namespaces, names are "mangled" into unique strings (e.g., `_Z3fooi` for `foo(int)`).
Different compilers (GCC vs MSVC) use different mangling schemes, which is why you generally can't link C++ object files from different compilers.

## The "Fragile Binary" Problem

One of the most common ABI issues occurs when you modify a class or struct in a shared library.

**Version 1 (Library):**

```cpp
class Widget {
public:
    void draw();
private:
    int width;
};
```

**Version 2 (Library Update):**

```cpp
class Widget {
public:
    void draw();
private:
    int width;
    int height; // Added a new member!
};
```

If you update the library to Version 2 but don't recompile the application that uses it:

1.  The application thinks `sizeof(Widget)` is 4 bytes.
2.  The library thinks `sizeof(Widget)` is 8 bytes.
3.  When the application allocates a `Widget` on the stack, it reserves 4 bytes.
4.  When the library's `draw()` method runs, it might try to access `height`, reading/writing memory _outside_ the allocated 4 bytes.
5.  **Result:** Stack corruption, crashes, or silent data corruption.

## Conclusion

The ABI is the invisible glue holding our systems together. Understanding it helps explain why we can't just mix and match C++ libraries from different compilers, why "DLL Hell" exists, and why system updates have to be so careful about preserving binary compatibility.
