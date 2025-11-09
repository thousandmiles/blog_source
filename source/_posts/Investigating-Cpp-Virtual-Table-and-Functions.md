---
title: Investigating C++ Virtual Table and Functions
categories:
  - Programming
tags:
  - C++
  - Virtual Functions
  - Memory Model
  - Object-Oriented Programming
date: 2025-11-09 13:08:08
---

## Problem/Challenge

Understanding how C++ implements runtime polymorphism through virtual functions is crucial for mastering object-oriented programming. However, the underlying mechanism—the virtual table (vtable)—remains a mystery to many developers. How can we actually observe and verify the vtable structure, locate virtual function pointers, and understand the object memory layout?

<!-- more -->

## Solution

We'll explore three key aspects of C++ virtual tables:

1. How to get the virtual table and pointer addresses
2. How to verify these results are correct
3. How to understand the object address model

### 1. Getting Virtual Table and Pointer Addresses

The virtual table pointer (vptr) is typically stored as the first member of an object with virtual functions. Here's how to access it:

```cpp
#include <iostream>
#include <cstdint>

class Base {
public:
    virtual void func1() { std::cout << "Base::func1()" << std::endl; }
    virtual void func2() { std::cout << "Base::func2()" << std::endl; }
    virtual void func3() { std::cout << "Base::func3()" << std::endl; }
    virtual ~Base() {}

    int baseData = 10;
};

class Derived : public Base {
public:
    void func1() override { std::cout << "Derived::func1()" << std::endl; }
    void func2() override { std::cout << "Derived::func2()" << std::endl; }

    int derivedData = 20;
};

int main() {

    // Get object address
    Base* basePtr = new Base();
    Base* derivedPtr = new Derived();

    std::cout << "Base object address: " << basePtr << std::endl;
    std::cout << "Derived object address: " << derivedPtr << std::endl;

    // Get vptr (first 8 bytes on 64-bit system)
    uintptr_t* baseVptr = reinterpret_cast<uintptr_t*>(basePtr);
    uintptr_t* derivedVptr = reinterpret_cast<uintptr_t*>(derivedPtr);

    std::cout << "\nVirtual table pointer addresses:" << std::endl;
    std::cout << "Base vptr: 0x" << std::hex << *baseVptr << std::endl;
    std::cout << "Derived vptr: 0x" << std::hex << *derivedVptr << std::endl;

    // Get vtable entries (function pointers)
    uintptr_t* baseVtable = reinterpret_cast<uintptr_t*>(*baseVptr);
    uintptr_t* derivedVtable = reinterpret_cast<uintptr_t*>(*derivedVptr);

    std::cout << "\nVirtual table entries (function pointers):" << std::endl;
    std::cout << "Base vtable[0] (func1): 0x" << baseVtable[0] << std::endl;
    std::cout << "Base vtable[1] (func2): 0x" << baseVtable[1] << std::endl;
    std::cout << "Base vtable[2] (func3): 0x" << baseVtable[2] << std::endl;
    std::cout << "Derived vtable[0] (func1): 0x" << derivedVtable[0] << std::endl;
    std::cout << "Derived vtable[1] (func2): 0x" << derivedVtable[1] << std::endl;
    std::cout << "Derived vtable[2] (func3): 0x" << derivedVtable[2] << std::endl;

    delete basePtr;
    delete derivedPtr;

    return 0;
}
```

**Real Output Analysis:**

Running this code produces the following output:

```
Base object address: 0000025D266B5460
Derived object address: 0000025D266B1A60

Virtual table pointer addresses:
Base vptr: 0x7ff755c8bc30
Derived vptr: 0x7ff755c8bc88

Virtual table entries (function pointers):
Base vtable[0] (func1): 0x7ff755c81195
Base vtable[1] (func2): 0x7ff755c81474
Base vtable[2] (func3): 0x7ff755c81550
Derived vtable[0] (func1): 0x7ff755c81519
Derived vtable[1] (func2): 0x7ff755c811d6
Derived vtable[2] (func3): 0x7ff755c81550
```

**Key Observations from the Output:**

```
Object Base
+----------------+  <-- basePtr
| vptr        ---|----> +----------------+
+----------------+      | &Base::func1   |  <-- baseVtable[0]
| baseData1      |      +----------------+
+----------------+      | &Base::func2   |  <-- baseVtable[1]
| baseData2      |      +----------------+
+----------------+      | &Base::func3   |  <-- baseVtable[2]
                        +----------------+
```

1. **Object Locations (Heap Memory)**:

   - Base object: `0x25D266B5460` - allocated on the heap via `new`
   - Derived object: `0x25D266B1A60` - also on the heap
   - These addresses are in the user process heap space (relatively low addresses)

2. **Virtual Table Locations (Read-Only Memory)**:

   - Base vtable: `0x7ff755c8bc30`
   - Derived vtable: `0x7ff755c8bc88`
   - These addresses are much higher (`0x7ff7...` range), indicating they're in the program's code/data segment
   - The vtables are **88 bytes apart** (`0x7ff755c8bc88 - 0x7ff755c8bc30 = 0x58 = 88 bytes`)

3. **Function Pointer Addresses**:

   - All function addresses are in the same general region (`0x7ff755c81...`), confirming they're in the code segment
   - Notice that Derived's function addresses for func1 and func2 are **different** from Base's, proving that polymorphism works by having different vtable entries
   - `Derived::func3` has the exact same address `as Base::func3` (`0x7ff755c81550`)
   - This is because Derived **didn't override** func3, so it inherits Base's implementation
   - The compiler optimizes by reusing the same function pointer instead of creating duplicate code

4. **Polymorphism vs Inheritance**:

   This output beautifully demonstrates the difference:

   | Function | Base Address   | Derived Address | Overridden?                  |
   | -------- | -------------- | --------------- | ---------------------------- |
   | func1    | 0x7ff755c81195 | 0x7ff755c81519  | ✅ Yes - Different addresses |
   | func2    | 0x7ff755c81474 | 0x7ff755c811d6  | ✅ Yes - Different addresses |
   | func3    | 0x7ff755c81550 | 0x7ff755c81550  | ❌ No - Same address!        |

   **Key Insight**: When a derived class doesn't override a virtual function, the vtable simply points to the base class implementation. This is efficient—no code duplication, just pointer reuse!

5. **Memory Region Summary**:
   ```
   Heap (Objects):        0x00025Dxxxxxxxx  ← Object instances
   Code/Data (vtables):   0x7ff755c8bxxx   ← Virtual tables (static)
   Code (Functions):      0x7ff755c81xxx   ← Function implementations
   ```

### 2. Verifying the Results

We can verify that the vtable pointers are correct by calling the functions through the vtable directly:

```cpp
#include <iostream>
#include <cstdint>

class Base {
public:
    virtual void func1() { std::cout << "Base::func1()" << std::endl; }
    virtual void func2() { std::cout << "Base::func2()" << std::endl; }
    virtual void func3() { std::cout << "Base::func3()" << std::endl; }
    virtual ~Base() {}
};

class Derived : public Base {
public:
    void func1() override { std::cout << "Derived::func1()" << std::endl; }
    void func2() override { std::cout << "Derived::func2()" << std::endl; }
};

// Function pointer type matching virtual function signature
typedef void (*VirtualFunc)(void*);

int main() {
    Base* basePtr = new Base();
    Base* derivedPtr = new Derived();

    std::cout << "=== Normal virtual function calls ===" << std::endl;
    basePtr->func1();
    derivedPtr->func1();
    derivedPtr->func3();

    std::cout << "\n=== Manual vtable function calls ===" << std::endl;

    // Get vptr
    uintptr_t* baseVptr = *reinterpret_cast<uintptr_t**>(basePtr);
    uintptr_t* derivedVptr = *reinterpret_cast<uintptr_t**>(derivedPtr);

    // Call func1 through vtable (first entry)
    VirtualFunc baseFunc1 = reinterpret_cast<VirtualFunc>(baseVptr[0]);
    VirtualFunc derivedFunc1 = reinterpret_cast<VirtualFunc>(derivedVptr[0]);

    std::cout << "Calling through Base vtable: ";
    baseFunc1(basePtr);  // Pass object pointer as 'this'

    std::cout << "Calling through Derived vtable: ";
    derivedFunc1(derivedPtr);  // Pass object pointer as 'this'

    // Verify func3 as well
    VirtualFunc baseFunc3 = reinterpret_cast<VirtualFunc>(baseVptr[2]);
    VirtualFunc derivedFunc3 = reinterpret_cast<VirtualFunc>(derivedVptr[2]);

    std::cout << "Calling through Base vtable: ";
    baseFunc3(basePtr);  // Pass object pointer as 'this'

    std::cout << "Calling through Derived vtable: ";
    derivedFunc3(derivedPtr);  // Pass object pointer as 'this'

    delete basePtr;
    delete derivedPtr;

    return 0;
}
```

**Why Pass the Object Pointer?**

The line `baseFunc1(basePtr)` might seem confusing, but it reveals how C++ member functions actually work.

**Key Concept: The Hidden `this` Pointer**

When you write:

```cpp
basePtr->func1();  // Normal call
```

The compiler transforms it into something like:

```cpp
Base::func1(basePtr);  // What actually happens (conceptually)
```

Every non-static member function has a **hidden first parameter** called `this`—a pointer to the object being operated on. When we manually call through the vtable, we must explicitly pass this `this` pointer ourselves.

**Example with Member Data:**

If `func1` accessed member variables, it would look like:

```cpp
class Base {
    int data = 42;
public:
    virtual void func1() {
        std::cout << "data = " << data << std::endl;  // Uses 'this->data'
    }
};

// What the compiler sees (conceptual):
void Base::func1(Base* this) {
    std::cout << "data = " << this->data << std::endl;
}
```

**Our Manual Call:**

```cpp
typedef void (*VirtualFunc)(void*);  // First param is 'this'
VirtualFunc baseFunc1 = reinterpret_cast<VirtualFunc>(baseVptr[0]);
baseFunc1(basePtr);  // Explicitly pass 'this' = basePtr
```

**Why `void*` for the function signature?**

We use `void (*VirtualFunc)(void*)` because:

- The actual signature is `void Base::func1(Base* this)`
- But vtable entries are stored as raw function pointers
- Using `void*` allows us to pass any object pointer type without complex casting

**What Happens If You Don't Pass It?**

```cpp
baseFunc1();  // ERROR! Missing the 'this' pointer
// The function would try to access members at undefined memory
// Result: crash or undefined behavior
```

**Real Output Analysis:**

Running this verification code produces:

```
=== Normal virtual function calls ===
Base::func1()
Derived::func1()
Base::func3()

=== Manual vtable function calls ===
Calling through Base vtable: Base::func1()
Calling through Derived vtable: Derived::func1()
Calling through Base vtable: Base::func3()
Calling through Derived vtable: Base::func3()
```

**What This Proves:**

1. **Manual dispatch works identically to normal calls**: The functions invoked through manual vtable traversal produce the exact same output as compiler-generated virtual calls, confirming our vtable address extraction is correct.

2. **Polymorphism verified**:

   - `Derived::func1()` is called (not `Base::func1()`) when using the Derived vtable, proving the vtable entry points to the overridden implementation
   - Both manual and normal calls respect the dynamic type

3. **Function inheritance confirmed**:

   - Both `Base vtable` and `Derived vtable` call `Base::func3()`
   - This proves our earlier observation: since `func3` wasn't overridden, both vtables contain the same function pointer (`0x7ff755c81550`)
   - No code duplication—the compiler efficiently reuses the base implementation

4. **The vtable mechanism is transparent**: The behavior is identical whether you use normal virtual function syntax or manually dereference the vtable. This demonstrates that virtual function calls are simply syntactic sugar for vtable lookups.

### 3. Understanding Object Address Model

The C++ object memory layout with virtual functions follows this structure:

```cpp
#include <iostream>
#include <cstdint>

class Base {
public:
    virtual void func() {}
    int baseData1 = 100;
    int baseData2 = 200;
};

class Derived : public Base {
public:
    void func() override {}
    int derivedData1 = 300;
    int derivedData2 = 400;
};

void printMemoryLayout(void* obj, size_t size) {
    unsigned char* ptr = static_cast<unsigned char*>(obj);
    std::cout << "Memory layout (hex bytes):" << std::endl;
    for (size_t i = 0; i < size; i++) {
        std::cout << std::hex << (int)ptr[i] << " ";
        if ((i + 1) % 8 == 0) std::cout << std::endl;
    }
    std::cout << std::dec << std::endl;
}

int main() {
    Base base;
    Derived derived;

    std::cout << "=== Base Object Memory Layout ===" << std::endl;
    std::cout << "Object size: " << sizeof(Base) << " bytes" << std::endl;
    std::cout << "Object address: " << &base << std::endl;

    // Show member offsets
    std::cout << "\nMember offsets:" << std::endl;
    std::cout << "vptr offset: 0" << std::endl;
    std::cout << "baseData1 offset: " << offsetof(Base, baseData1) << std::endl;
    std::cout << "baseData2 offset: " << offsetof(Base, baseData2) << std::endl;

    std::cout << "\n=== Derived Object Memory Layout ===" << std::endl;
    std::cout << "Object size: " << sizeof(Derived) << " bytes" << std::endl;
    std::cout << "Object address: " << &derived << std::endl;

    std::cout << "\nMember offsets:" << std::endl;
    std::cout << "vptr offset: 0 (inherited)" << std::endl;
    std::cout << "baseData1 offset: " << offsetof(Derived, baseData1) << std::endl;
    std::cout << "baseData2 offset: " << offsetof(Derived, baseData2) << std::endl;
    std::cout << "derivedData1 offset: " << offsetof(Derived, derivedData1) << std::endl;
    std::cout << "derivedData2 offset: " << offsetof(Derived, derivedData2) << std::endl;

    // Visualize memory
    std::cout << "\nBase object raw memory:" << std::endl;
    printMemoryLayout(&base, sizeof(Base));

    std::cout << "\nDerived object raw memory:" << std::endl;
    printMemoryLayout(&derived, sizeof(Derived));

    return 0;
}
```

**Real Output Analysis:**

Running this code produces:

```
=== Base Object Memory Layout ===
Object size: 16 bytes
Object address: 000000CCD9F3F558

Member offsets:
vptr offset: 0
baseData1 offset: 8
baseData2 offset: 12

=== Derived Object Memory Layout ===
Object size: 24 bytes
Object address: 000000CCD9F3F588

Member offsets:
vptr offset: 0 (inherited)
baseData1 offset: 8
baseData2 offset: 12
derivedData1 offset: 16
derivedData2 offset: 20

Base object raw memory:
Memory layout (hex bytes):
30 bc df b6 f7 7f 0 0
64 0 0 0 c8 0 0 0

Derived object raw memory:
Memory layout (hex bytes):
58 bc df b6 f7 7f 0 0
64 0 0 0 c8 0 0 0
2c 1 0 0 90 1 0 0
```

**Detailed Memory Analysis:**

**1. Object Sizes:**

- Base: 16 bytes = 8 (vptr) + 4 (baseData1) + 4 (baseData2)
- Derived: 24 bytes = 8 (vptr) + 4 (baseData1) + 4 (baseData2) + 4 (derivedData1) + 4 (derivedData2)

**2. Member Offsets (The Memory Layout):**

| Offset | Base Object | Derived Object        |
| ------ | ----------- | --------------------- |
| 0-7    | vptr        | vptr (inherited)      |
| 8-11   | baseData1   | baseData1 (inherited) |
| 12-15  | baseData2   | baseData2 (inherited) |
| 16-19  | -           | derivedData1          |
| 20-23  | -           | derivedData2          |

**3. Raw Memory Breakdown:**

**Base Object (`30 bc df b6 f7 7f 0 0 | 64 0 0 0 c8 0 0 0`):**

```
Bytes 0-7:   30 bc df b6 f7 7f 00 00
             └────────────────────┘
             vptr = 0x00007ff7b6dfbc30 (little-endian)
             Points to Base vtable

Bytes 8-11:  64 00 00 00
             └─────────┘
             baseData1 = 0x64 = 100 (decimal)

Bytes 12-15: c8 00 00 00
             └─────────┘
             baseData2 = 0xc8 = 200 (decimal)
```

**Derived Object (`58 bc df b6 f7 7f 0 0 | 64 0 0 0 c8 0 0 0 | 2c 1 0 0 90 1 0 0`):**

```
Bytes 0-7:   58 bc df b6 f7 7f 00 00
             └────────────────────┘
             vptr = 0x00007ff7b6dfbc58 (little-endian)
             Points to Derived vtable (different from Base!)

Bytes 8-11:  64 00 00 00
             └─────────┘
             baseData1 = 0x64 = 100 (inherited)

Bytes 12-15: c8 00 00 00
             └─────────┘
             baseData2 = 0xc8 = 200 (inherited)

Bytes 16-19: 2c 01 00 00
             └─────────┘
             derivedData1 = 0x12c = 300 (decimal)

Bytes 20-23: 90 01 00 00
             └─────────┘
             derivedData2 = 0x190 = 400 (decimal)
```

**4. Key Observations:**

1. **Little-Endian Byte Order**:

   - The vptr `30 bc df b6 f7 7f 00 00` is stored with least significant byte first
   - When reconstructed: `0x00007ff7b6dfbc30`

2. **Different VPTRs Confirm Distinct Vtables**:

   - Base vptr: `0x7ff7b6dfbc30`
   - Derived vptr: `0x7ff7b6dfbc58`
   - These point to different vtables, enabling polymorphism

3. **Data Values Visible in Memory**:

   - `baseData1 = 100` appears as `64 00 00 00` (hex 0x64 = decimal 100)
   - `baseData2 = 200` appears as `c8 00 00 00` (hex 0xc8 = decimal 200)
   - `derivedData1 = 300` appears as `2c 01 00 00` (hex 0x12c = decimal 300)
   - `derivedData2 = 400` appears as `90 01 00 00` (hex 0x190 = decimal 400)

4. **Memory Layout is Contiguous**:

   - No gaps or padding between members (in this case)
   - vptr always comes first, then base members, then derived members

5. **Stack Allocation**:
   - Object addresses (`0xCCD9F3F558`, `0xCCD9F3F588`) are stack addresses (local variables)
   - Much different from heap addresses we saw earlier with `new`

**Memory Layout Visualization:**

```
Base Object (16 bytes on 64-bit):
Offset  |  Content          |  Hex Bytes              |  Value
--------|-------------------|-------------------------|------------------
0-7     | vptr              | 30 bc df b6 f7 7f 00 00 | → Base vtable
8-11    | baseData1 (int)   | 64 00 00 00             | 100 (decimal)
12-15   | baseData2 (int)   | c8 00 00 00             | 200 (decimal)

Derived Object (24 bytes on 64-bit):
Offset  |  Content          |  Hex Bytes              |  Value
--------|-------------------|-------------------------|------------------
0-7     | vptr              | 58 bc df b6 f7 7f 00 00 | → Derived vtable
8-11    | baseData1 (int)   | 64 00 00 00             | 100 (inherited)
12-15   | baseData2 (int)   | c8 00 00 00             | 200 (inherited)
16-19   | derivedData1      | 2c 01 00 00             | 300 (decimal)
20-23   | derivedData2      | 90 01 00 00             | 400 (decimal)
```

**Visual Diagram:**

```
Base Object (16 bytes):
┌─────────────────────┐  Offset 0
│ vptr: 0x7ff7b6dfbc30│  ← Points to Base vtable
├─────────────────────┤  Offset 8
│ baseData1: 100      │
├─────────────────────┤  Offset 12
│ baseData2: 200      │
└─────────────────────┘  Offset 16

Derived Object (24 bytes):
┌─────────────────────┐  Offset 0
│ vptr: 0x7ff7b6dfbc58│  ← Points to Derived vtable (different!)
├─────────────────────┤  Offset 8
│ baseData1: 100      │  ← Inherited from Base
├─────────────────────┤  Offset 12
│ baseData2: 200      │  ← Inherited from Base
├─────────────────────┤  Offset 16
│ derivedData1: 300   │  ← Derived-specific
├─────────────────────┤  Offset 20
│ derivedData2: 400   │  ← Derived-specific
└─────────────────────┘  Offset 24
```

**Why vptr Comes First:**

The vptr is placed at offset 0 for important reasons:

1. **Consistent Access**: Regardless of the inheritance hierarchy, the vptr is always at the same location (beginning of object)
2. **Efficient Dispatch**: The compiler knows to look at offset 0 to find the vtable, no calculation needed
3. **Type Safety**: When casting `Base* → Derived*`, the vptr remains accessible at the same offset

**What If We Had Multiple Inheritance?**

With multiple inheritance, you might have multiple vptrs at different offsets. But for single inheritance (our case), there's only one vptr at offset 0, inherited and overwritten by derived classes.

## Explanation

### How Virtual Tables Work

1. **vptr (Virtual Pointer)**: Each object with virtual functions contains a hidden pointer (typically the first member) that points to its class's vtable.

2. **vtable (Virtual Table)**: A static array of function pointers, one per virtual function, stored in the program's read-only memory. Each class with virtual functions has its own vtable.

3. **Dynamic Dispatch**: When calling a virtual function through a pointer/reference, the compiler generates code that:
   - Dereferences the vptr to get the vtable address
   - Indexes into the vtable to get the function pointer
   - Calls the function through that pointer
