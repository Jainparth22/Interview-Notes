# Study Volume 9: Programming and Systems (C++, Linux, Compilers, Computer Architecture)

Ninth volume of the in-depth study set, same template as before: the idea, how it works, a worked example, the loopholes where people go wrong, interview questions, and a **Revise in 30 seconds** box. This volume covers the last items of your checklist: **C++ essentials, Linux and the command line, compilers and program execution, and computer architecture**. These topics are asked in technical rounds to check that you understand what happens **below** your Python, Flask and SQL, and C++ is the language many OAs and product-company interviews expect you to read.

**How this page was checked:** the C++ programs were compiled with `g++ 13.3` (C++20) and run, the shell sessions were run in a real Linux shell, the compiler pipeline was executed step by step with `gcc`, `nm`, `readelf`, `size` and `ldd`, and the little compiler and virtual machine were fuzz-tested against Python's own evaluator. **The outputs quoted are real.** Three honest caveats: (1) exact addresses, timings and process ids change from run to run and machine to machine, so look at the **pattern**, not the digits, (2) the sandbox had a single CPU and reported different cache sizes on two runs, so cache sizes are not quoted as facts and you should check `lscpu` on your own machine, and (3) some tools were not installed here (`gdb`, `valgrind`, `strace`, `perf`), so those are described, not demonstrated.

## Contents of this volume

- Part 1: C++ essentials: memory layout, pointers and references, RAII and smart pointers, move semantics, virtual functions, struct padding, the STL, undefined behaviour and sanitizers, threads and data races
- Part 2: Linux and the shell: files and permissions, links, text pipelines, redirection, exit codes, quoting, signals, `/proc`, limits
- Part 3: Compilers and program execution: preprocess, compile, assemble, link, load, optimisation, linker errors, and a tiny compiler with a virtual machine
- Part 4: Computer architecture: data representation, floating point, caches and locality, branch prediction, virtual memory, parallel speedup
- Part 5: Interview bank and the formula sheet

## Part 1: C++ essentials

### 1.1 Why C++ shows up in interviews

C++ gives you **direct control over memory and performance**, so questions probe whether you understand **object lifetime, ownership, and what the machine does**. You rarely need exotic features. You need a precise model of **stack versus heap, copies versus moves, who frees what, and what is undefined behaviour**. For OAs it is also the fastest mainstream language, and the STL (standard template library) gives you containers and algorithms.

**Sizes of the basic types** on this 64-bit Linux machine (they are implementation-defined, so never assume them blindly):

```text
sizeof(Bad)=24  offsets a=0 b=8 c=16 d=20  alignof=8
sizeof(Good)=16  (same members, reordered)
sizeof(Packed)=14 (no padding, but misaligned access)
sizeof(NoPoly)=4 vs sizeof(Poly with a virtual function)=16  (the hidden vptr)
sizeof: char 1 short 2 int 4 long 8 long long 8 pointer 8 double 8
```

### 1.2 Memory: where things live (verified)

A running program has regions (Volume 1 covered the idea). This program prints one address from each region:

```cpp
#include <cstdio>
#include <cstdlib>
#include <cstdint>
#include <sys/mman.h>
int initialised_global = 42;          // .data
int uninitialised_global;             // .bss
const char* literal = "hello";        // the string bytes live in .rodata
int depth(int n, uintptr_t* first, uintptr_t* last) {   // recursion shows which way the stack grows
    int local; if (n == 0) { *first = (uintptr_t)&local; } 
    if (n == 3) { *last = (uintptr_t)&local; return 0; }
    return depth(n + 1, first, last);
}
int main() {
    int on_stack = 1;
    int* on_heap = (int*)malloc(sizeof(int));
    void* big = mmap(nullptr, 1 << 20, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    uintptr_t first = 0, last = 0; depth(0, &first, &last);
    auto kb = [](uintptr_t a) { return a; };
    printf("code   (main)               %#lx\n", kb((uintptr_t)&main));
    printf("rodata (string literal)     %#lx\n", kb((uintptr_t)literal));
    printf("data   (initialised global) %#lx\n", kb((uintptr_t)&initialised_global));
    printf("bss    (uninit global)      %#lx\n", kb((uintptr_t)&uninitialised_global));
    printf("heap   (malloc 4 bytes)     %#lx\n", kb((uintptr_t)on_heap));
    printf("mmap   (1 MiB anonymous)    %#lx\n", kb((uintptr_t)big));
    printf("stack  (local in main)      %#lx\n", kb((uintptr_t)&on_stack));
    printf("stack grows %s: frame 0 at %#lx, frame 3 at %#lx\n", last < first ? "DOWN (deeper calls have lower addresses)" : "UP", first, last);
    return 0;
}
```

Output of the non-PIE build (a build with fixed code and data addresses, so the ordering is easy to read). The stack, heap and mmap addresses change on every run:

```text
code   (main)               0x401226
rodata (string literal)     0x402008
data   (initialised global) 0x404030
bss    (uninit global)      0x404044
heap   (malloc 4 bytes)     0x3b5a22a0
mmap   (1 MiB anonymous)    0x7f0884700000
stack  (local in main)      0x7ffc7ccc3404
stack grows DOWN (deeper calls have lower addresses): frame 0 at 0x7ffc7ccc33e4, frame 3 at 0x7ffc7ccc3324
```

**What it shows:**
- **Code** (`.text`), **read-only data** (`.rodata`, the string literal), **initialised globals** (`.data`), **uninitialised globals** (`.bss`) sit at **low addresses** and come from the executable file, in that order.
- The **heap** (`malloc`, `new`) comes next, growing **upward**. Large allocations (1 MiB here) are served by **`mmap`** instead of growing the heap, and land in a separate high region.
- The **stack** is at the **top** and grows **downward**: each deeper recursive call has a **lower** address (the last line confirms it).
- With a **position-independent executable (PIE)** (the default on modern systems) and **ASLR** (address space layout randomisation), the base addresses are **different on every run**, which makes memory-corruption exploits harder. Do not hard-code addresses.

**Stack versus heap in one table:**

| | Stack | Heap |
|---|---|---|
| Lifetime | Automatic: ends when the function returns | Manual (or smart pointer): until freed |
| Speed | Very fast (move a pointer) | Slower (allocator work, possible fragmentation) |
| Size | Small and fixed (typically 8 MiB per thread), **stack overflow** on deep recursion or huge local arrays | Large, limited by memory |
| Typical use | Local variables, function frames | Objects whose size or lifetime is dynamic |

**Classic memory bugs, with the tool that catches them** (the sanitizers are in 1.7):
- **Leak:** allocated, never freed. **Dangling pointer or use-after-free:** using memory after it was freed. **Double free:** freeing twice. **Buffer overflow:** writing past the end of an array. **Uninitialised read.** **Returning a pointer or reference to a local variable** (the local dies when the function returns).

### 1.3 Pointers, references, const

- A **pointer** holds an address, can be null, can be reassigned, supports arithmetic. A **reference** is an alias for an existing object: it must be initialised, cannot be null, cannot be reseated. Use a **reference for 'this must exist'** and a **pointer for 'maybe nothing'** or when you need to reassign.
- **Pass by value** copies (cheap for `int`, expensive for a big object). **Pass by `const T&`** avoids the copy and forbids modification (the default for non-trivial read-only parameters). **Pass by `T&`** to modify the caller's object. **Pass by `T&&`** to take ownership of a temporary (move).
- **`const` correctness:** `const int* p` (pointer to constant data), `int* const p` (constant pointer), `const int* const p` (both). Read right to left. Mark methods that do not change the object `const`.
- **Never return a reference or pointer to a local variable.** Returning by value is fine and efficient (the compiler elides the copy or moves).
- **Arrays decay to pointers** when passed to functions, losing their size, so pass a length or use `std::array` or `std::span`. `sizeof(arr)` inside the callee is the size of a pointer.
- **Pointer arithmetic** scales by the element size: `p + 1` advances `sizeof(*p)` bytes.

### 1.4 RAII and smart pointers (verified)

**RAII (resource acquisition is initialisation):** acquire a resource in a constructor, release it in the destructor, so cleanup happens **automatically at scope exit, even when an exception is thrown**. Files (`std::fstream`), locks (`std::lock_guard`), and memory (smart pointers) all follow it. It is C++'s answer to `try/finally` and Python's `with`.

**The smart pointers:**
- **`std::unique_ptr<T>`:** **sole ownership**, movable, not copyable, no overhead. Create with `std::make_unique<T>(...)`. **This is the default choice.**
- **`std::shared_ptr<T>`:** **shared ownership** by **reference counting**. The object dies when the last owner goes. Create with `std::make_shared`. It has overhead (a control block, atomic counts).
- **`std::weak_ptr<T>`:** a **non-owning** observer of a `shared_ptr`, used to **break reference cycles** and to check whether an object still exists.
- **Raw pointers** are fine for **non-owning** access. **Avoid `new` and `delete` in application code**, prefer `make_unique` and containers.

This program logs constructors, copies, moves and destructors:

```cpp
#include <cstdio>
#include <memory>
#include <vector>
#include <string>
#include <utility>
struct Noisy {
    std::string name;
    Noisy(std::string n) : name(std::move(n)) { printf("  ctor %s\n", name.c_str()); }
    Noisy(const Noisy& o) : name(o.name + "(copy)") { printf("  COPY %s\n", name.c_str()); }
    Noisy(Noisy&& o) noexcept : name(std::move(o.name)) { printf("  MOVE -> %s\n", name.c_str()); }
    ~Noisy() { printf("  dtor %s\n", name.c_str()); }
};
struct Node { std::shared_ptr<Node> next; std::weak_ptr<Node> prev; int id; ~Node() { printf("  Node %d destroyed\n", id); } };
struct Leaky { std::shared_ptr<Leaky> other; int id; ~Leaky() { printf("  Leaky %d destroyed\n", id); } };
int main() {
    printf("1) RAII: destructor runs at scope exit, even for early exits\n");
    { Noisy a("a"); printf("  inside scope\n"); }
    printf("2) unique_ptr: sole owner, movable, not copyable\n");
    { auto p = std::make_unique<Noisy>("u"); auto q = std::move(p); printf("  p is %s after the move, q owns %s\n", p ? "non-null" : "null", q->name.c_str()); }
    printf("3) shared_ptr reference counting\n");
    { auto s = std::make_shared<Noisy>("s"); { auto t = s; printf("  use_count = %ld\n", s.use_count()); } printf("  use_count = %ld\n", s.use_count()); }
    printf("4) vector<Noisy>: push_back of an lvalue copies, of an rvalue moves, reallocation relocates\n");
    { std::vector<Noisy> v; v.reserve(1); Noisy x("x"); v.push_back(x); v.push_back(Noisy("y")); printf("  (second push_back reallocated: elements moved because the move ctor is noexcept)\n"); }
    printf("5) shared_ptr cycle leaks, weak_ptr breaks it\n");
    { auto a = std::make_shared<Leaky>(); auto b = std::make_shared<Leaky>(); a->id = 1; b->id = 2; a->other = b; b->other = a; printf("  leaving scope with a cycle... (no destructor message follows: a leak)\n"); }
    { auto a = std::make_shared<Node>(); auto b = std::make_shared<Node>(); a->id = 3; b->id = 4; a->next = b; b->prev = a; printf("  leaving scope with weak_ptr back-link...\n"); }
    return 0;
}
```

```text
1) RAII: destructor runs at scope exit, even for early exits
  ctor a
  inside scope
  dtor a
2) unique_ptr: sole owner, movable, not copyable
  ctor u
  p is null after the move, q owns u
  dtor u
3) shared_ptr reference counting
  ctor s
  use_count = 2
  use_count = 1
  dtor s
4) vector<Noisy>: push_back of an lvalue copies, of an rvalue moves, reallocation relocates
  ctor x
  COPY x(copy)
  ctor y
  MOVE -> y
  MOVE -> x(copy)
  dtor 
  dtor 
  (second push_back reallocated: elements moved because the move ctor is noexcept)
  dtor x
  dtor x(copy)
  dtor y
5) shared_ptr cycle leaks, weak_ptr breaks it
  leaving scope with a cycle... (no destructor message follows: a leak)
  leaving scope with weak_ptr back-link...
  Node 3 destroyed
  Node 4 destroyed
```

**Reading the output:**
1. The destructor ran at the **end of the scope**.
2. After `std::move`, the original `unique_ptr` is **null** and the new one owns the object.
3. `use_count` rose to 2 while a second `shared_ptr` existed, then fell back to 1.
4. `push_back(x)` of an **lvalue made a COPY**, `push_back(Noisy("y"))` of a **temporary MOVED**, and the second insertion **reallocated the vector**, which **moved the existing element** (because the move constructor is `noexcept`; otherwise the vector would copy for exception safety). The blank `dtor` lines are the **moved-from** objects being destroyed.
5. **Two `shared_ptr`s pointing at each other (a cycle) were never destroyed: a leak.** Their reference counts can never reach zero. The version with a `weak_ptr` back-link **did** destroy both nodes. Remember: **ownership edges point one way, back-references use `weak_ptr`**.

### 1.5 Rule of zero, three, five, and move semantics

- If a class manages a resource and you define **one of** the destructor, copy constructor, copy assignment, you almost certainly need **all three** (the **rule of three**). With move operations it becomes the **rule of five** (add move constructor and move assignment).
- **The best rule is the rule of zero:** hold resources in members that already manage themselves (`std::vector`, `std::string`, smart pointers) and **define none of the five**, so the compiler-generated ones are correct.
- **Move semantics:** an **rvalue** (a temporary, or something you cast with `std::move`) can have its resources **stolen** instead of copied. A move **leaves the source valid but unspecified** (do not use its value). `std::move` itself **does not move anything**, it just **casts to an rvalue reference** so overload resolution picks the move constructor.
- Mark move constructors and move assignment **`noexcept`**, so containers use them during reallocation (shown above).
- **Return by value** and let **copy elision** and moves do the work. Writing `return std::move(local);` is usually **worse**, it blocks elision.
- **Common bug:** using an object **after** `std::move`d from it.

### 1.6 Virtual functions, the vtable, and object slicing (verified)

- A **virtual function** is resolved at **run time** by the object's **dynamic type**. The compiler adds a hidden **vptr** (pointer to the class's **vtable**, a table of function pointers) to every object of a class with virtual functions. The measured sizes above show it: `sizeof(NoPoly)` is **4** bytes, the same class content with one virtual function, `sizeof(Poly)`, is **16** (an 8-byte vptr plus 4 bytes of data, padded to 8-byte alignment).
- **A class used as a polymorphic base needs a `virtual` destructor.** Deleting a derived object through a base pointer whose destructor is **not virtual is undefined behaviour** (the derived destructor is skipped, leaking what it owns).
- **`override`** makes the compiler check that you really override a base function, **catching signature typos**. **`final`** forbids further overriding. **`= 0`** makes a function **pure virtual** (an abstract class cannot be instantiated).
- **Object slicing:** assigning or passing a derived object **by value** to a base **cuts off** the derived part. Pass **by reference or pointer** for polymorphism.
- **Constructor and destructor order:** bases construct first, then members, then the class body. Destruction is the reverse. **Virtual calls inside a constructor or destructor resolve to the class being constructed**, not the most-derived one.
- **Multiple inheritance and the diamond:** two paths to the same base duplicate it unless you use **virtual inheritance** (`struct B : virtual A`). Prefer interfaces (pure abstract classes) plus composition.

```cpp
#include <cstdio>
#include <memory>
struct BaseBad { ~BaseBad() { printf("  ~BaseBad\n"); } };
struct DerivedBad : BaseBad { int* buf = new int[4]; ~DerivedBad() { delete[] buf; printf("  ~DerivedBad (frees buf)\n"); } };
struct BaseGood { virtual ~BaseGood() { printf("  ~BaseGood\n"); } virtual const char* who() const { return "BaseGood"; } };
struct DerivedGood : BaseGood { ~DerivedGood() override { printf("  ~DerivedGood\n"); } const char* who() const override { return "DerivedGood"; } };
int main() {
    printf("virtual dispatch through a base pointer: %s\n", std::unique_ptr<BaseGood>(new DerivedGood())->who());
    printf("virtual destructor, delete via base pointer:\n"); { BaseGood* p = new DerivedGood(); delete p; }
    printf("(a NON-virtual destructor deleted through a base pointer is undefined behaviour: ~DerivedBad would not run, leaking buf)\n");
}
```

```text
virtual dispatch through a base pointer: DerivedGood
  ~DerivedGood
  ~BaseGood
virtual destructor, delete via base pointer:
  ~DerivedGood
  ~BaseGood
(a NON-virtual destructor deleted through a base pointer is undefined behaviour: ~DerivedBad would not run, leaking buf)
```

### 1.7 Struct layout, padding, alignment (verified)

The compiler **aligns** each member to a multiple of its size and **pads** to keep that true, and pads the **end** so arrays of the struct stay aligned. **Member order changes the size**:

```cpp
#include <cstdio>
#include <cstddef>
struct Bad  { char a; double b; char c; int d; };          // naive order
struct Good { double b; int d; char a; char c; };          // largest first
struct Packed { char a; double b; char c; int d; } __attribute__((packed));
struct Poly { virtual void f() {} int x; };
struct NoPoly { int x; };
int main() {
    printf("sizeof(Bad)=%zu  offsets a=%zu b=%zu c=%zu d=%zu  alignof=%zu\n", sizeof(Bad), offsetof(Bad,a), offsetof(Bad,b), offsetof(Bad,c), offsetof(Bad,d), alignof(Bad));
    printf("sizeof(Good)=%zu  (same members, reordered)\n", sizeof(Good));
    printf("sizeof(Packed)=%zu (no padding, but misaligned access)\n", sizeof(Packed));
    printf("sizeof(NoPoly)=%zu vs sizeof(Poly with a virtual function)=%zu  (the hidden vptr)\n", sizeof(NoPoly), sizeof(Poly));
    printf("sizeof: char %zu short %zu int %zu long %zu long long %zu pointer %zu double %zu\n", sizeof(char), sizeof(short), sizeof(int), sizeof(long), sizeof(long long), sizeof(void*), sizeof(double));
}
```

```text
sizeof(Bad)=24  offsets a=0 b=8 c=16 d=20  alignof=8
sizeof(Good)=16  (same members, reordered)
sizeof(Packed)=14 (no padding, but misaligned access)
sizeof(NoPoly)=4 vs sizeof(Poly with a virtual function)=16  (the hidden vptr)
sizeof: char 1 short 2 int 4 long 8 long long 8 pointer 8 double 8
```

Same members, **24 bytes in the naive order and 16 when sorted largest-first**, although the members add up to only 14 bytes of data. The offsets show where padding went: 7 bytes after `a` so that `b` (a double) can sit at offset 8, `c` at 16, then 3 bytes of padding so that `d` (an int) lands at offset 20. `__attribute__((packed))` removes padding (size 14, exactly the sum of the members) but causes **misaligned accesses** that are slower or fault on some CPUs. **Rule of thumb: declare members from largest to smallest.** It matters for memory use and cache behaviour (Part 4), and `sizeof` of a struct is a favourite interview question.

### 1.8 The STL (verified), and what to remember about complexity

```cpp
#include <cstdio>
#include <vector>
#include <map>
#include <unordered_map>
#include <queue>
#include <algorithm>
#include <string>
int main() {
    std::vector<int> v; size_t last = 99; printf("vector capacity growth: ");
    for (int i = 0; i < 40; i++) { v.push_back(i); if (v.capacity() != last) { last = v.capacity(); printf("%zu ", last); } } printf("\n");
    std::vector<int> w = {1, 2, 3}; int* before = w.data(); w.reserve(1000); printf("iterator invalidation: data() pointer %s after reserve\n", before == w.data() ? "unchanged" : "CHANGED (old iterators and pointers are now dangling)");
    std::map<std::string,int> m = {{"pear",1},{"apple",2},{"fig",3}}; printf("std::map iterates in key order:"); for (auto& [k, val] : m) printf(" %s", k.c_str()); printf("\n");
    std::priority_queue<int> pq; for (int x : {3, 9, 1, 7}) pq.push(x); printf("priority_queue is a MAX-heap, pops:"); while (!pq.empty()) { printf(" %d", pq.top()); pq.pop(); } printf("\n");
    std::priority_queue<int, std::vector<int>, std::greater<int>> mn; for (int x : {3, 9, 1, 7}) mn.push(x); printf("with std::greater it is a MIN-heap, top = %d\n", mn.top());
    std::vector<int> d = {5, 3, 8, 1, 9, 2}; std::sort(d.begin(), d.end(), [](int a, int b) { return a > b; }); printf("sort descending with a lambda:"); for (int x : d) printf(" %d", x); printf("\n");
    std::vector<int> s = {1, 3, 5, 7, 9}; printf("lower_bound(5) index %ld, upper_bound(5) index %ld, binary_search(4) %d\n", std::lower_bound(s.begin(), s.end(), 5) - s.begin(), std::upper_bound(s.begin(), s.end(), 5) - s.begin(), (int)std::binary_search(s.begin(), s.end(), 4));
    std::unordered_map<int,int> um; um[1]++; um[1]++; size_t size_before = um.size(); (void)um[99]; printf("operator[] inserts a default value: size before reading um[99] = %zu, after = %zu\n", size_before, um.size());
}
```

```text
vector capacity growth: 1 2 4 8 16 32 64 
iterator invalidation: data() pointer CHANGED (old iterators and pointers are now dangling) after reserve
std::map iterates in key order: apple fig pear
priority_queue is a MAX-heap, pops: 9 7 3 1
with std::greater it is a MIN-heap, top = 1
sort descending with a lambda: 9 8 5 3 2 1
lower_bound(5) index 2, upper_bound(5) index 3, binary_search(4) 0
operator[] inserts a default value: size before reading um[99] = 1, after = 2
```

| Container | Operations | Notes |
|---|---|---|
| `vector` | index O(1), `push_back` amortised O(1), insert in the middle O(n) | Contiguous, **capacity doubles** (1, 2, 4, 8, ... above), **reallocation invalidates all iterators, pointers and references** (shown above) |
| `deque` | `push_front` and `push_back` O(1) | Not contiguous |
| `list` | insert and erase O(1) at a known position | Poor cache behaviour, rarely the right choice |
| `map`, `set` | insert, find, erase O(log n) | Balanced tree, **sorted iteration**, needs `<` |
| `unordered_map`, `unordered_set` | average O(1), worst case O(n) | Hash table, **no order**, needs a hash and `==` |
| `priority_queue` | push and pop O(log n), top O(1) | **Max-heap by default**, use `std::greater` for a min-heap |
| `stack`, `queue` | O(1) | Adapters over `deque` |

**Things interviewers like you to know:**
- **`operator[]` on a `map` or `unordered_map` inserts a default element if the key is missing.** Use `find` or `count` to test without inserting, as the last line of output demonstrates.
- **`lower_bound(x)`** returns the first element `>= x`, **`upper_bound(x)`** the first element `> x` (binary search, the container must be sorted). `binary_search` only answers yes or no.
- **`std::sort`** is O(n log n) (introsort), not stable. **`std::stable_sort`** keeps equal elements in order. The comparator must be a **strict weak ordering** (`<` not `<=`), or you get undefined behaviour.
- **Erasing while iterating** invalidates iterators, use the iterator returned by `erase` or the erase-remove idiom.
- **`reserve(n)`** avoids reallocations, **`emplace_back`** constructs in place.
- Competitive-coding habits: `ios::sync_with_stdio(false); cin.tie(nullptr);` for fast input, `long long` for sums that can exceed 2^31, and `'\n'` instead of `endl` (which flushes).

### 1.9 Lambdas, templates, and modern syntax in one page

- **Lambda:** `[captures](params) { body }`. **`[=]`** captures **by value** (a copy), **`[&]`** by reference, `[x, &y]` mixes. **A lambda that captures a local by reference and outlives the function holds a dangling reference** (a classic bug). Capture by value when a lambda is stored or run later.
- **Templates** are compile-time code generation: the compiler instantiates a version per type. Errors appear at instantiation and can be long. **`auto`** deduces types, **range-for** (`for (auto& x : v)`), **structured bindings** (`auto [k, val] = *it;`), **`constexpr`** (compute at compile time).
- **Header files and the one-definition rule:** declarations go in headers (with **include guards** or `#pragma once`), **definitions in exactly one `.cpp`** (Part 3 shows the linker error when you break this).
- **`static`** has several meanings: a local that persists across calls, a file-private global (internal linkage), a class member shared by all objects.

### 1.10 Undefined behaviour and the sanitizers (verified)

**Undefined behaviour (UB)** means the language places **no requirements** on what happens. The compiler assumes UB **never occurs**, and optimises on that assumption, so the symptoms are bizarre: it may seem to work, crash later, or behave differently at `-O2`. Common causes: **signed integer overflow**, out-of-bounds access, use after free, null dereference, reading an uninitialised variable, **data races**, violating strict aliasing, modifying a `const` object, and a non-virtual destructor through a base pointer. (**Unsigned** overflow is **well-defined** and wraps, as Part 4 shows.)

**Sanitizers are compiler flags that detect these at run time.** Build with `-g -fsanitize=...` (and `-O0` or `-O1`) while testing:

| Flag | Detects |
|---|---|
| `-fsanitize=address` (ASan) | heap and stack buffer overflows, use-after-free, double free, leaks |
| `-fsanitize=undefined` (UBSan) | signed overflow, null dereference, bad shifts, misaligned access |
| `-fsanitize=thread` (TSan) | data races |
| `-fsanitize=memory` (MSan, clang) | uninitialised reads |

Real messages from small test programs (a use-after-free, an out-of-range array read, and a signed overflow):

```text
==86==ERROR: AddressSanitizer: heap-use-after-free on address 0x502000000014 at pc 0x5584a950322a ...
READ of size 4 at 0x502000000014 thread T0
0x502000000014 is located 4 bytes inside of 16-byte region [0x502000000010,0x502000000020)
freed by thread T0 here:
previously allocated by thread T0 here:

==94==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x7fd027e00050 at pc 0x5584ae68f429 ...
READ of size 4 at 0x7fd027e00050 thread T0
Address 0x7fd027e00050 is located in stack of thread T0 at offset 80 in frame

ubsan_ovf.cpp:3:49: runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'
```

Note how ASan tells you **what kind** of bug, **where the memory was allocated and freed**, which is far more useful than a bare crash. Without a sanitizer, the use-after-free above would probably **just work**, which is why you must not trust 'it ran fine' as evidence of correctness.

### 1.11 Threads and data races (verified, with an honest surprise)

```cpp
std::thread t([&]{ ... });  t.join();           // start and wait
std::mutex mu;  std::lock_guard<std::mutex> g(mu);   // RAII lock
std::atomic<int> counter{0};  counter.fetch_add(1);  // atomic read-modify-write
std::scoped_lock lock(mu1, mu2);                // locks several mutexes without deadlock
```

**Experiment 1:** 8 threads each add 1 to a shared counter 3,000,000 times, **no synchronisation**, built with `-O0`. Three runs:

```text
expected 24000000, got 13218755 (UPDATES LOST)
expected 24000000, got 3771218 (UPDATES LOST)
expected 24000000, got 13682693 (UPDATES LOST)
```

**Millions of increments were lost**, and the result changed on every run. This happened **even though the machine has one CPU**: threads are time-sliced and can be switched out between the read and the write of `counter = counter + 1` (the same lost-update race as Volume 1).

**Experiment 2 (the surprise):** the same kind of program with a plain `int`, 8 threads, 200,000 increments each, built with **`-O1`**, ran **correctly three times in a row**, together with the `std::atomic` and mutex versions:

```text
expected 1600000 | plain int 1600000 | std::atomic 1600000 | mutex-protected 1600000
```

**Do not conclude the plain `int` was safe.** At `-O1` the compiler is allowed to keep the variable in a register for the whole loop and add the total at the end, precisely **because a data race is undefined behaviour and it may assume there is none**. The bug was hidden by the optimiser, on one machine, in one configuration. **ThreadSanitizer found the race immediately** in that very program:

```text
WARNING: ThreadSanitizer: data race (pid=141)
  Write of size 4 at 0x7fff2e08235c by thread T4:
  Previous write of size 4 at 0x7fff2e08235c by thread T3:
```

**Lessons:** (1) a data race is **undefined behaviour**, not just 'a wrong number sometimes'. (2) **Passing tests does not prove thread safety**, use **TSan** and design for correctness (atomics, mutexes, message passing). (3) Use **`std::atomic`** for a single shared counter or flag, a **mutex** for a multi-step critical section (and RAII lock guards so you never forget to unlock). (4) Locking several mutexes in different orders deadlocks, **`std::scoped_lock`** acquires them together. (5) Memory ordering (`std::memory_order_*`) exists for expert lock-free code, **the default sequentially consistent ordering is correct and fine for almost everything.** (6) `volatile` is **not** a synchronisation tool in C++ (it is for memory-mapped hardware).

### Loopholes and common wrong answers in C++

- 'A reference is just a pointer': semantically it cannot be null or reseated, and it has different syntax and rules.
- 'Use `new` and remember `delete`': prefer `make_unique` and containers, avoid manual ownership.
- Passing a polymorphic object **by value** (slicing), or deleting through a base pointer **without a virtual destructor**.
- Assuming `sizeof(struct)` is the sum of its members, or that `int` is always 4 bytes.
- 'It ran without crashing, so it is correct' (UB, races, use-after-free often appear to work).
- Using `std::move` and then reading the moved-from object.
- Iterating a `vector` while calling `push_back` (iterator invalidation).
- `<=` in a sort comparator.

### Interview questions on C++

- **Stack versus heap, and where do globals live?** Automatic versus manual lifetime, small and fast versus large. Globals are in `.data` (initialised) or `.bss` (zero-initialised), string literals in read-only data.
- **Pointer versus reference?** Nullable and reseatable versus alias that cannot be null or reseated.
- **What is RAII and why does it matter?** Tying resource lifetime to scope so cleanup is automatic and exception-safe, the basis of smart pointers and lock guards.
- **`unique_ptr` versus `shared_ptr` versus `weak_ptr`?** Sole ownership, shared reference-counted ownership, non-owning observer that breaks cycles. We saw a cycle leak and a `weak_ptr` fix.
- **What is the rule of three, five and zero?** If you manage a resource you need destructor, copy and move operations. Prefer zero by using self-managing members.
- **What does `std::move` do?** Casts to an rvalue so a move constructor can be chosen, it does not itself move.
- **How do virtual functions work, and why a virtual destructor?** A vptr to a per-class vtable gives run-time dispatch (hence 16 bytes versus 4 in our measurement). Deleting through a base pointer without one is undefined behaviour.
- **Why is `sizeof` of this struct 24 when its members add up to 14 bytes?** Alignment padding. Reordering largest-first gave 16.
- **What is undefined behaviour, give examples, and how do you catch it?** No guarantees, the compiler assumes it never happens. Signed overflow, out-of-bounds, use-after-free, data races. Sanitizers.
- **How do you prevent a data race?** Atomics, mutexes, or avoid sharing. We measured millions of lost updates without protection.
- **`vector` growth and iterator invalidation?** Capacity doubles (1, 2, 4, ...), reallocation invalidates pointers, iterators and references.
- **`map` versus `unordered_map`?** Sorted tree O(log n) versus hash O(1) average. `operator[]` inserts a default.

### Revise in 30 seconds: C++

- Layout: code, rodata, data, bss low, heap grows up, stack at the top grows down (verified), ASLR randomises bases. Stack automatic and small, heap manual and large.
- Pointer nullable and reseatable, reference is an alias. Pass `const T&` for big read-only arguments. Never return a reference to a local.
- RAII ties cleanup to scope. Default to `unique_ptr`, `shared_ptr` counts owners (cycles leak, `weak_ptr` breaks them). Rule of zero. Moves: `noexcept`, `std::move` is a cast, do not use moved-from objects.
- Virtual means vtable and vptr (4 bytes became 16), polymorphic bases need a virtual destructor, `override`, avoid slicing. Struct size depends on member order (24 versus 16).
- STL: vector doubles and invalidates iterators, map sorted O(log n), unordered O(1), `priority_queue` is a max-heap, `operator[]` inserts, `lower_bound` versus `upper_bound`.
- Undefined behaviour (signed overflow, out-of-bounds, use-after-free, data races) can look fine, use ASan, UBSan and TSan. Plain-int races lost millions of updates at `-O0` and were hidden at `-O1`, TSan found them. Atomics for counters, mutexes for critical sections, `scoped_lock` for several.

## Part 2: Linux and the shell

### 2.1 The mental model

- **Everything is a file** (or looks like one): regular files, directories, devices (`/dev`), pipes, sockets, even **kernel and process state** (`/proc`). The same tools (`cat`, `grep`, redirection) therefore work on all of them.
- A **process** has **file descriptors**: small integers pointing at open files. **0 is standard input, 1 is standard output, 2 is standard error.** A pipe connects one process's descriptor 1 to another's descriptor 0.
- The **shell** is an ordinary program that reads a line, then does **`fork` and `exec`** to run the command (Volume 1, Part 1), after arranging the descriptors for redirections and pipes. When you type `ls | wc -l`, the shell starts **both** processes at once and wires the pipe between them.
- **`PATH`** is a colon-separated list of directories searched for a command name. **Environment variables** are inherited by child processes. `type name` shows whether something is a builtin, an alias, a function or a file.

Key directories: `/etc` (configuration), `/var/log` (logs), `/home` (users), `/tmp` (temporary, cleared on reboot), `/usr/bin` and `/bin` (programs), `/proc` (processes and kernel), `/dev` (devices), `/opt` (add-on software).

### 2.2 Files, permissions, links (verified)

Every file has an **owner**, a **group**, and **three sets of permission bits** (owner, group, others), each `r` (read, 4), `w` (write, 2), `x` (execute, 1). **Add them to get the octal digit.** For a directory, `x` means 'may enter and access names inside', `r` means 'may list names', `w` means 'may create or delete entries'. The listing below was produced by real commands. Note the `inode` numbers: a **hard link** has the **same inode** as the original, a **symbolic link** has its own.

```text
$ awk '{print $4}' access.log | sort | uniq -c | sort -rn     # requests per status code
      4 200
      2 201
      1 500
      1 409
      1 404

$ cut -d" " -f1 access.log | sort | uniq -c | sort -rn | head -2   # busiest clients
      4 10.0.0.1
      3 10.0.0.2

$ sort -k5,5nr access.log | head -2                              # slowest requests
10.0.0.3 GET /api/slots 500 250
10.0.0.2 GET /api/slots 200 90

$ awk '{s[$3]+=$5; n[$3]++} END {for (p in s) printf "%-14s avg %.1f ms (%d)\n", p, s[p]/n[p], n[p]}' access.log | sort
/api/book      avg 45.0 ms (2)
/api/slots     avg 170.0 ms (2)
/api/tasks     avg 20.8 ms (4)
/api/tasks/7   avg 3.0 ms (1)

$ grep -cE " (4|5)[0-9]{2} " access.log                           # error responses
3

$ chmod 755 script.sh; chmod 640 data.txt; stat -c "%A %a %n" script.sh data.txt
-rwxr-xr-x 755 script.sh
-rw-r----- 640 data.txt

$ umask   # default mask; a new file starts at 666 and a new directory at 777, minus the mask
0022
-rw------- 600 private.txt   (created under umask 077)

$ ln orig.txt hard.txt; ln -s orig.txt soft.txt; ls -li orig.txt hard.txt soft.txt
574315 -rw-r--r-- 2 hard.txt
574315 -rw-r--r-- 2 orig.txt
574316 lrwxrwxrwx 1 soft.txt -> orig.txt
$ rm orig.txt; cat hard.txt; cat soft.txt
hi
soft.txt: cat: soft.txt: No such file or directory

$ ls /nonexistent access.log > out.txt 2> err.txt
out.txt: access.log
err.txt: ls: cannot access '/nonexistent': No such file or directory

# exit codes: 0 means success, anything else is a failure
true -> 0
false -> 1
grep with no match -> 1
pipeline status without pipefail: 0
pipeline status with pipefail: 1

# word splitting breaks on spaces in file names
for f in $(ls): 3 iterations for 2 files
for f in * : 2 iterations
xargs -0 got: ./my report.txt
xargs -0 got: ./notes.txt

# signals
caught SIGTERM, cleaning up
exit status after the trap handler: 0
SIGKILL: exit status of the killed process = 137 (128 + 9), no trap can intercept it

# /proc exposes kernel state as files (the reading process is "self", here the grep command)
Name:	grep
State:	R (running)
PPid:	99
Threads:	1
559a7a2a3000-559a7a2c4000 [heap]
7ffecd0c3000-7ffecd0e4000 [stack]
open descriptors of the shell: 0 1 2 255 3 
limits: open files 20000, stack KB 8192
```

**What the output shows (and the facts to remember):**
- **`chmod 755`** gives `-rwxr-xr-x` (owner `7 = 4+2+1`, group and others `5 = 4+1`). `640` gives `-rw-r-----`. **Scripts need `x` to run directly** (`./script.sh`).
- **`umask`** subtracts bits from the defaults (files start at 666, directories at 777). With the usual `0022` a new file is `644`, and under `umask 077` it became `600` (owner only).
- **A hard link is another name for the same inode.** Both names are equally 'real', and the data survives until **every** hard link is removed. After `rm orig.txt`, `hard.txt` **still reads `hi`**. A **symbolic link** stores a **path**, so when the target is removed it **dangles** (`No such file or directory`). Hard links cannot cross file systems or point at directories, symlinks can.
- **Special bits:** **setuid** (a program runs with its owner's privileges, as `passwd` does, a classic security risk), **setgid**, and the **sticky bit** on a directory like `/tmp` (only a file's owner may delete it).
- **`rm` is permanent.** There is no recycle bin. Quote carefully, and never run `rm -rf $VAR/` where the variable could be empty.
- 'Permission denied' checklist: do the **directory** permissions (`x` along the whole path) and the **file** permissions allow it, who is the **owner**, is the file system **mounted read-only**, and is a security module (SELinux) involved.

### 2.3 Text pipelines: the daily tools (verified above)

The first part of the output is a real analysis of a small web log, built entirely from small tools joined by pipes. Each tool does one job on a stream of text:

| Tool | Does | Example from the output |
|---|---|---|
| `grep` | select lines (`-E` extended regex, `-v` invert, `-c` count, `-i` ignore case, `-r` recurse) | `grep -cE ' (4\|5)[0-9]{2} '` counted 3 error responses |
| `cut` | pick columns by delimiter | `cut -d' ' -f1` extracted the client IPs |
| `awk` | field-based processing, arithmetic, aggregation with arrays | per-path average latency |
| `sed` | stream editing, mostly `s/old/new/g` | `sed 's/10\.0\.0\./ip-/'` |
| `sort` | order lines (`-n` numeric, `-r` reverse, `-k` by column, `-u` unique) | `sort -k5,5nr` found the slowest request |
| `uniq -c` | count **adjacent** duplicates | requests per status code |
| `tr`, `wc`, `head`, `tail`, `tee`, `xargs`, `find` | translate characters, count, slice, copy a stream to a file and onward, build commands from input, search the file system | |

**The most-used idiom in operations work:** `... | sort | uniq -c | sort -rn | head`. It counts how many times each distinct value occurs and shows the most frequent. **`uniq` only collapses adjacent duplicates, so you must `sort` first** (the most common mistake). Sort numerically with `-n`, otherwise `10` sorts before `9`.

**`awk` in one line:** `awk '{print $4}'` prints the fourth whitespace-separated field, `$NF` is the last, `-F,` sets the delimiter, `NR` is the line number, and `awk '{s[$3]+=$5} END {for (k in s) print k, s[k]}'` aggregates by key, exactly the 'group by' of SQL.

### 2.4 Redirection, exit codes, and chaining (verified above)

- `cmd > file` overwrites stdout, `>>` appends, `2> file` redirects **stderr**, `2>&1` sends stderr to wherever stdout goes, `< file` feeds stdin, `<<EOF ... EOF` (a here-doc) feeds inline text. The output showed `ls` writing the listing to `out.txt` and the error to `err.txt` separately. **Order matters:** `cmd > file 2>&1` captures both, `cmd 2>&1 > file` does not.
- Every command returns an **exit status**: **0 is success, non-zero is failure** (`$?` holds the last one). `grep` returns 1 when nothing matched, which is not an error condition but **will trip `set -e`**.
- **`&&`** runs the next command only on success, **`||`** only on failure, `;` always. `cmd1 && cmd2 || cmd3` is **not** an if-else (`cmd3` also runs if `cmd2` fails).
- **A pipeline's status is the status of the last command by default.** The output proved it: `false | true` reported **0**, and with **`set -o pipefail`** it reported **1**. This is why robust scripts use `set -euo pipefail`.
- Command substitution `$(cmd)` captures output, arithmetic `$((7 * 6))`, and **process substitution** `<(cmd)` presents output as a file.

### 2.5 Quoting and word splitting (verified above)

**The shell splits unquoted expansions on whitespace and expands wildcards.** The output showed it: `for f in $(ls)` iterated **3 times for 2 files**, because `my report.txt` was split into `my` and `report.txt`, while `for f in *` iterated 2 times, and `find -print0 | xargs -0` handled the name correctly. **Rules to live by:**
1. **Always quote variable expansions**: `"$file"`, `"$@"`. Unquoted `$var` is a bug waiting for a space or an asterisk.
2. Never parse `ls` output. Use globs (`for f in *`) or `find ... -print0 | xargs -0`.
3. **Single quotes** prevent all expansion, **double quotes** allow `$` and backticks.
4. Test strings with `[[ ... ]]` in bash (safer than `[ ... ]`), and compare numbers with `-eq`, `-lt`.

### 2.6 Processes and signals (verified above)

- **View:** `ps aux` or `ps -ef` (a snapshot), `top` or `htop` (live), `pgrep -f name`. Process states in `ps`: `R` running, `S` sleeping, `D` uninterruptible (usually disk I/O), `T` stopped, `Z` zombie.
- **Signals** are small asynchronous notifications. **`SIGTERM` (15)** is the polite 'please exit' that a program can **catch and handle** (clean up, flush, exit), and it is what `kill pid` sends and what platforms send first on a deploy. **`SIGINT` (2)** is Ctrl+C. **`SIGKILL` (9)** **cannot be caught or ignored**, the kernel ends the process immediately with no cleanup. **`SIGHUP` (1)** traditionally means 'terminal closed', often reused as 'reload your configuration'.
- The output demonstrated both: a script with **`trap` caught `SIGTERM`**, printed its cleanup message and exited with status 0, while a process killed with **`kill -9` reported status 137, which is 128 + 9** (the convention is 128 plus the signal number). This ties to the deployment lesson in Volume 5: apps must **handle SIGTERM** to drain and exit cleanly, or they get SIGKILLed after the grace period.
- **Job control:** `cmd &` runs in the background, `jobs`, `fg`, `bg`, `wait`, **`nohup cmd &`** keeps it alive after logout, `disown`. For real services use **systemd** (or a container platform), not `nohup`.

### 2.7 `/proc`, limits and system inspection (verified above)

`/proc` exposes live kernel state as readable text. `/proc/<pid>/status` (state, parent, threads, memory), `/proc/<pid>/maps` (the memory map, with the `[heap]` and `[stack]` regions seen in Part 1), `/proc/<pid>/fd` (open descriptors, here `0 1 2 255 3`), `/proc/cpuinfo`, `/proc/meminfo`. `/proc/self` means **the process reading it**, which in the output was the `grep` command itself (note `Name: grep`).

**Limits** (`ulimit -a`): the output showed **open files 20000** and **stack size 8192 KB** (the 8 MiB stack from Part 1). 'Too many open files' (`EMFILE`) in a server means leaked connections or a limit that is too low (a pile of `CLOSE_WAIT` sockets, Volume 3, is the usual cause).

**The commands you reach for, by question** (the ones marked with a star were not run in the authoring environment, so check your own system's output):

| Question | Command |
|---|---|
| Is the disk full? Where is the space? | `df -h`, `du -sh * \| sort -h`, `lsof +L1` (*deleted files still held open*) |
| Is memory the problem? | `free -h`, `top` or `htop`, `ps aux --sort=-%mem \| head`, `dmesg \| grep -i oom` (*OOM killer*) |
| High CPU or load? | `uptime` (load average), `top`, `pidstat` (*), `py-spy top --pid` for Python |
| Who is using this port? | `ss -ltnp`, `lsof -i :8000` |
| What is this process doing? | `strace -p PID` (*system calls*), `lsof -p PID` (*open files*), `cat /proc/PID/status` |
| Why did it crash? | `journalctl -u service -e` (*systemd logs*), `dmesg`, the exit code (137 SIGKILL or OOM, 139 segfault, 143 SIGTERM) |
| Network path or DNS problem? | `ping`, `traceroute`, `dig`, `curl -v`, `nc -vz host port` (Volume 3) |
| Search for text or files | `grep -rn 'text' .`, `find . -name '*.log' -mtime -1`, `locate` |
| Schedule something | `cron` (`m h dom mon dow command`, e.g. `*/5 * * * *` every 5 minutes), or a systemd timer |
| Run a service reliably | **systemd** unit file with `ExecStart`, `Restart=on-failure`, then `systemctl start\|stop\|status\|enable service` |

**Common production problems in one place:** a `command not found` is a `PATH` problem or a missing package. A script that fails with a strange `^M` or `bad interpreter` has **Windows line endings (CRLF)**, fix with `dos2unix`. A `Permission denied` on a script means a missing `x` bit or a noexec mount. **A full disk can be caused by a deleted-but-still-open log file** (`du` does not see it, `df` does), restart the process or truncate it. A script working by hand but not from `cron` is almost always a **different `PATH` or environment**, so use absolute paths.

### 2.8 A robust script skeleton (verified)

Most shell bugs come from **continuing after an error, using unset variables, and not cleaning up**. This skeleton avoids them:

```bash
#!/usr/bin/env bash
# A robust shell script skeleton: fail fast, clean up, validate arguments
set -euo pipefail                      # -e exit on error, -u error on unset variables, -o pipefail fail if any pipeline stage fails
tmpdir=$(mktemp -d)
cleanup() { rm -rf "$tmpdir"; echo "cleaned up $tmpdir"; }
trap cleanup EXIT                      # runs on normal exit, on error, and on SIGINT or SIGTERM
usage() { echo "usage: $0 <logfile>" >&2; exit 2; }
[[ $# -eq 1 ]] || usage
[[ -r "$1" ]] || { echo "cannot read $1" >&2; exit 1; }
awk '{print $4}' "$1" | sort | uniq -c | sort -rn > "$tmpdir/status.txt"
echo "top status code: $(head -n1 "$tmpdir/status.txt")"
echo "variable set to a default when unset: ${UNSET_VARIABLE:-default-value}"
```

```text
$ ./robust.sh /tmp/lx2/access.log
top status code:       4 200
variable set to a default when unset: default-value
cleaned up /tmp/tmp.pHXmGNAWic
exit status: 0

$ ./robust.sh      # no argument
usage: ./robust.sh <logfile>
cleaned up /tmp/tmp.KNaLZOJtav
exit status: 2

$ ./robust.sh /no/such/file
cannot read /no/such/file
cleaned up /tmp/tmp.60LwN6s3we
exit status: 1
```

`set -e` stops at the first failing command, `set -u` turns a typo in a variable name into an error instead of an empty string, `pipefail` catches a failure anywhere in a pipeline, `trap cleanup EXIT` removes the temporary directory **however** the script ends (the demo shows it cleaning up on success, on a usage error with status 2, and on an unreadable file with status 1), and `${VAR:-default}` supplies a default. **Caveat:** `set -e` has surprising exceptions (inside `if` conditions and `||` chains), so it is a safety net, not a substitute for checking the results you care about. For anything beyond about 50 lines or real logic, **use Python** instead of shell.

### Loopholes in Linux answers

- 'Hard link and symbolic link are the same': a hard link is another name for the same inode, a symlink is a path that can dangle.
- Parsing `ls`, unquoted variables, forgetting `sort` before `uniq`, sorting numbers lexically.
- Assuming `kill` always stops a process (SIGTERM can be caught, only SIGKILL cannot, and an uninterruptible `D` process ignores even that until its I/O completes).
- `chmod 777` as a fix (it hides the real ownership problem and is a security hole).
- Believing `/proc/self` is the shell (it is whichever process reads it).
- Running a long task with `&` and expecting it to survive logout.

### Interview questions on Linux

- **What happens when you type `ls | wc -l`?** The shell forks two processes, connects stdout of the first to stdin of the second with a pipe, and each does `exec`. They run concurrently.
- **Hard link versus symbolic link?** Same inode and independent of the original name, versus a pointer to a path that breaks if the target goes. We saw `hard.txt` survive deleting `orig.txt` and `soft.txt` dangle.
- **What does `chmod 640` mean?** Owner read and write, group read, others nothing.
- **What is the difference between SIGTERM and SIGKILL?** Catchable request to terminate versus uncatchable immediate kill. Exit status 143 and 137.
- **How do you find what is using port 8000? How do you find why the disk is full?** `ss -ltnp` or `lsof -i :8000`. `df -h` then `du -sh * | sort -h`, and check for deleted-but-open files with `lsof`.
- **What is `set -euo pipefail` for?** Fail on errors, unset variables and failures inside pipelines.
- **How would you count the most common IPs in a log?** `awk '{print $1}' log | sort | uniq -c | sort -rn | head`.
- **What do file descriptors 0, 1, 2 mean, and what does `2>&1` do?** stdin, stdout, stderr. It sends stderr to the current destination of stdout.
- **A script works in my terminal but not from cron. Why?** Different environment and `PATH`, working directory, and no terminal. Use absolute paths and set the environment explicitly.

### Revise in 30 seconds: Linux

- Everything is a file, descriptors 0, 1, 2, the shell does fork and exec, pipes connect processes. `PATH` finds commands, `/proc` exposes process state.
- Permissions `rwx` = 4, 2, 1 (755 = `rwxr-xr-x`), `umask` removes bits, hard link = same inode (survives deleting the original), symlink = a path (dangles).
- Pipelines: `grep`, `cut`, `awk`, `sed`, `sort` (use `-n`), `uniq -c` needs sorted input, `sort | uniq -c | sort -rn | head`.
- Exit status 0 = success, pipeline status is the last command's unless `pipefail`. `&&`, `||`, `2>&1`, here-docs. Quote every `"$var"`, never parse `ls`, use `find -print0 | xargs -0`.
- SIGTERM is catchable (trap), SIGKILL is not (status 137), 143 is SIGTERM. Use systemd for services. Script skeleton: `set -euo pipefail` plus `trap cleanup EXIT`.
- Diagnose with `df`, `du`, `free`, `top`, `ss -ltnp`, `lsof`, `journalctl`, `strace`, and the exit codes.

## Part 3: Compilers and program execution

### 3.1 The whole journey, from source to a running process

```mermaid
flowchart LR
    A[hello.c source] -->|preprocessor| B[Expanded source .i]
    B -->|compiler: parse, check, optimise| C[Assembly .s]
    C -->|assembler| D[Object file .o]
    D -->|linker + libraries| E[Executable ELF]
    E -->|kernel execve + dynamic loader| F[Running process]
```

This is the real session for a small C program, run step by step. Read the comments as you go:

```c
#include <stdio.h>
#define GREETING "hello"
#define SQUARE(x) ((x) * (x))
int counter = 5;
int sum_to_ten(void) { int s = 0; for (int i = 0; i < 10; i++) s += i; return s; }
int main(void) { printf("%s %d %d\n", GREETING, SQUARE(counter), sum_to_ten()); return 0; }
```

```text
$ cat hello.c
#include <stdio.h>
#define GREETING "hello"
#define SQUARE(x) ((x) * (x))
int counter = 5;
int sum_to_ten(void) { int s = 0; for (int i = 0; i < 10; i++) s += i; return s; }
int main(void) { printf("%s %d %d\n", GREETING, SQUARE(counter), sum_to_ten()); return 0; }

$ gcc -E hello.c -o hello.i      # 1. preprocess: headers pasted in, macros expanded
hello.c: 6 lines  ->  hello.i: 819 lines
int main(void) { printf("%s %d %d\n", "hello", ((counter) * (counter)), sum_to_ten()); return 0; }

$ gcc -S -O0 -masm=intel hello.c   # 2. compile to assembly (sum_to_ten, no optimisation)
sum_to_ten:
.LFB0:
	endbr64
	push	rbp
	mov	rbp, rsp
	mov	DWORD PTR -8[rbp], 0
	mov	DWORD PTR -4[rbp], 0
	jmp	.L2
.L3:
	mov	eax, DWORD PTR -4[rbp]
	add	DWORD PTR -8[rbp], eax
	add	DWORD PTR -4[rbp], 1
.L2:
	cmp	DWORD PTR -4[rbp], 9
	jle	.L3
	mov	eax, DWORD PTR -8[rbp]
	pop	rbp
	ret
.LFE0:

$ gcc -S -O2 -masm=intel hello.c   # same function with optimisation
sum_to_ten:
.LFB23:
	endbr64
	mov	eax, 45
	ret
.LFE23:

$ gcc -c hello.c -o hello.o        # 3. assemble to an object file
$ nm hello.o                        # symbols: T code here, D data here, U needed from elsewhere
0000000000000000 D counter
000000000000002d T main
                 U printf
0000000000000000 T sum_to_ten
$ size hello.o                      # section sizes in bytes
   text	   data	    bss	    dec	    hex	filename
    250	      4	      0	    254	     fe	hello.o

$ gcc hello.o -o hello           # 4. link (adds startup code, resolves printf from libc)
hello 25 45
$ readelf -h hello | grep Type
Type:                              DYN (Position-Independent Executable file)
$ ldd hello                        # shared libraries loaded at run time
	linux-vdso.so.1
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
	/lib64/ld-linux-x86-64.so.2
$ size hello
   text	   data	    bss	    dec	    hex	filename
   1504	    604	      4	   2112	    840	hello

$ gcc -static hello.c -o hello_static; ls -l hello hello_static
16040 hello
785376 hello_static

# linker errors
$ gcc undef.c -o undef
undef.c:(.text+0x9): undefined reference to `helper'
error: ld returned 1 exit status
$ gcc dup1.c dup2.c -o dup
/tmp/ccTUX4OF.o:(.data+0x0): multiple definition of `value'; /tmp/ccZUMnrq.o:(.data+0x0): first defined here
```

### 3.2 What each stage does

1. **Preprocessing (`gcc -E`):** text substitution before the compiler proper. `#include` pastes the header's text (**6 lines became 819** in the run, because `stdio.h` pulls in more headers), `#define` macros are expanded, `#ifdef` selects code. The last line printed shows `main` with the macros already expanded: `GREETING` became the string and `SQUARE(counter)` became `((counter) * (counter))`. **Macros are blind text replacement**, which causes classic bugs:

```text
SQUARE_BAD(1+2) = 5 (expands to 1+2 * 1+2)
SQUARE_OK(1+2)  = 9
SQUARE_OK(j++) : j++ was evaluated twice, j is now 5 (and the result 12 is undefined behaviour)
```

`SQUARE_BAD(1+2)` gave **5, not 9**, because the text `x * x` became `1+2 * 1+2`. Always **parenthesise macro parameters and the whole body**, and prefer **`inline` functions, `constexpr` and `const`** in C++. A macro that uses its argument twice **evaluates side effects twice** (`j++` ran twice, and that is undefined behaviour).

2. **Compilation:** the front end does **lexing** (characters to tokens), **parsing** (tokens to a syntax tree, according to the grammar), and **semantic analysis** (types, scopes, declarations are checked, which is where 'undeclared identifier' and type errors come from). It then lowers to an **intermediate representation (IR)**, applies **optimisations**, and the **back end** does **instruction selection and register allocation** to emit **assembly** for the target CPU. 3.6 builds a miniature version of this.
3. **Assembly to an object file (`gcc -c`):** the assembler converts assembly to **machine code** in an **object file** (ELF format on Linux). It contains **sections** (`.text` code, `.data` initialised globals, `.bss` zero-initialised globals which take **no space in the file**, `.rodata` constants and string literals), a **symbol table** (`nm` above: `T` code defined here, `D` data defined here, **`U` undefined, to be found elsewhere**), and **relocation entries** (placeholders for addresses not yet known). In the output, `printf` is `U`, the compiler cannot know where it lives.
4. **Linking:** the linker **combines object files and libraries**, **resolves every `U` symbol** to its definition, assigns final addresses, and applies relocations. It also adds **startup code** (`_start`, which calls the C runtime, which calls `main`).
   - **Static linking** copies library code into the executable (**self-contained, large**). **Dynamic linking** records which **shared libraries** (`.so`) are needed and **loads them at run time** (small executable, libraries shared between processes and updatable, but the right versions must be present, and `ldd` lists what is required). The run showed the difference: the dynamic executable was about **16 KB** and the static one about **785 KB** (it contains its own copy of the C library).
5. **Loading and running:** `./hello` makes the shell `fork` and then **`execve`**. The kernel maps the executable's segments into a **fresh address space**, sets up the stack, and (for a dynamic executable) starts the **dynamic loader (`ld.so`)**, which maps the shared libraries and resolves symbols, using the **PLT and GOT** (jump tables) so that library calls such as `printf` are bound **lazily**, on first use. Then `_start` runs, `main` runs, and `exit` ends the process. **PIE and ASLR** randomise where it is all placed (Part 1.2).

### 3.3 Optimisation, seen in the assembly (verified above)

The function `sum_to_ten` loops from 0 to 9 adding. At **`-O0`** (no optimisation) the compiler emits a **literal translation**: variables live on the stack, with a compare-and-jump loop (you can see the loop label, the `cmp`, the `jle` and the memory loads and stores). At **`-O2`** the same function becomes a **single instruction that loads the constant 45** into the return register. The compiler **evaluated the whole loop at compile time**.

**Optimisations worth naming:** **constant folding and propagation** (compute at compile time), **dead code elimination**, **inlining** (replace a call with the function body, enabling further optimisation), **loop unrolling and invariant code motion**, **common subexpression elimination**, **strength reduction** (replace a multiply by a shift), **vectorisation** (use SIMD instructions), and **register allocation** (keep values in registers, not memory).

**Consequences that matter:**
- **Benchmark and debug builds differ:** use `-O2` for performance numbers, `-O0 -g` (debug symbols, no optimisation) for debugging, and `-Wall -Wextra` always. **Sanitizers** (Part 1.10) work best at `-O0` or `-O1`.
- **The optimiser assumes your program has no undefined behaviour**, and may **delete code that checks for it**. The data race in Part 1.11 vanished at `-O1` for this reason. A signed-overflow check written as `if (x + 1 < x)` can be optimised away entirely, because signed overflow 'cannot happen'.
- Adding `volatile`, or printing in a benchmark loop, may be needed to stop the compiler removing the code you are trying to measure (the sums in the cache experiments are printed for that reason).

### 3.4 Linker errors, name mangling, and the one-definition rule (verified above)

```text
undefined reference to `helper'                       <- declared but never defined: forgot to link the file or library
multiple definition of `value'                        <- defined in two translation units: the one-definition rule is broken
```

- **Undefined reference** means the **declaration was found** (so it compiled) but **no definition was linked**. Causes: a missing `.cpp` in the build, a missing library (`-lm`), the wrong order of libraries on the command line, or a mismatch between declaration and definition (**remember that C++ encodes the signature in the symbol name**).
- **Multiple definition** means a **non-inline definition lives in a header** that more than one `.cpp` includes. Put **declarations in headers** and **definitions in one `.cpp`**, or mark small functions `inline`. **Include guards** (`#pragma once` or `#ifndef`) stop one translation unit from including the same header twice, which is a different error (redefinition).
- **Name mangling** lets C++ overload functions: the compiler encodes the parameter types in the symbol. `nm` showed `_Z3addii` for `add(int, int)` and `_Z3adddd` for `add(double, double)`, and `nm -C` demangles them. A function declared **`extern "C"`** keeps the plain name `add_c`, which is how C++ code exposes a C-compatible interface or calls C libraries.

### 3.5 Interpreters, bytecode, JIT

- A **compiler** translates ahead of time to machine code (C, C++, Rust, Go). An **interpreter** executes the program directly step by step. **Most real languages are a mixture.** **Python** compiles source to **bytecode** (what the `.pyc` files hold) that the **CPython virtual machine** interprets. **Java** compiles to bytecode, and the **JIT (just-in-time) compiler** translates hot methods to machine code at run time, using measurements the ahead-of-time compiler did not have.
- You can see Python's bytecode with `dis`. The last output in 3.6 shows that `a * 2 + b` becomes a few stack-machine instructions (`LOAD_FAST a`, `LOAD_CONST 2`, `BINARY_OP *`, `LOAD_FAST b`, `BINARY_OP +`, `RETURN_VALUE`), which is the same idea as the toy virtual machine below.
- **Trade-offs:** ahead-of-time compilation gives speed and no start-up cost, interpreters give portability, dynamic features and fast edit-run cycles, JITs aim for both after a warm-up. **Garbage-collected** languages free memory automatically (reference counting in CPython, plus a cycle collector, or tracing collectors in Java and Go), at the price of some pauses and memory overhead.

### 3.6 A tiny compiler, end to end (verified, fuzz-tested)

To make the stages concrete, here is a complete miniature compiler for arithmetic expressions with `+`, `-`, `*`, parentheses and unary minus: a **lexer**, a **recursive-descent parser** that builds an **AST**, a **code generator** that emits **stack-machine bytecode**, and a **virtual machine** that runs it. Operator **precedence is encoded in the grammar** (one parsing function per level: `expr` for `+ -`, `term` for `*`, `unary`, `atom`), which is how precedence is usually implemented.

```python
import re, random

# ---- 1. lexer: characters -> tokens ----
TOKEN = re.compile(r'\s*(?:(\d+)|(.))')
def lex(src):
    tokens = []
    for num, op in TOKEN.findall(src):
        if num: tokens.append(('NUM', int(num)))
        elif op.strip(): tokens.append((op, op))
    return tokens + [('END', None)]

# ---- 2. parser: tokens -> AST (recursive descent, one function per precedence level) ----
class Parser:
    def __init__(self, tokens): self.t, self.i = tokens, 0
    def peek(self): return self.t[self.i][0]
    def eat(self, kind):
        tok = self.t[self.i]
        if tok[0] != kind: raise SyntaxError(f'expected {kind}, got {tok[0]}')
        self.i += 1; return tok
    def expr(self):                                   # expr := term (('+' | '-') term)*
        node = self.term()
        while self.peek() in ('+', '-'):
            op = self.eat(self.peek())[0]; node = (op, node, self.term())
        return node
    def term(self):                                   # term := unary ('*' unary)*   (binds tighter than + and -)
        node = self.unary()
        while self.peek() == '*':
            self.eat('*'); node = ('*', node, self.unary())
        return node
    def unary(self):                                  # unary := '-' unary | atom
        if self.peek() == '-': self.eat('-'); return ('neg', self.unary())
        return self.atom()
    def atom(self):                                   # atom := NUM | '(' expr ')'
        if self.peek() == 'NUM': return ('num', self.eat('NUM')[1])
        self.eat('('); node = self.expr(); self.eat(')'); return node
def parse(src):
    p = Parser(lex(src)); tree = p.expr(); p.eat('END'); return tree

# ---- 3. code generation: AST -> stack-machine bytecode ----
def codegen(node, out=None):
    out = [] if out is None else out
    if node[0] == 'num': out.append(('PUSH', node[1]))
    elif node[0] == 'neg': codegen(node[1], out); out.append(('NEG', None))
    else: codegen(node[1], out); codegen(node[2], out); out.append(({'+': 'ADD', '-': 'SUB', '*': 'MUL'}[node[0]], None))
    return out

# ---- 4. virtual machine: executes the bytecode ----
def run(code):
    st = []
    for op, arg in code:
        if op == 'PUSH': st.append(arg)
        elif op == 'NEG': st.append(-st.pop())
        else:
            b, a = st.pop(), st.pop(); st.append({'ADD': a + b, 'SUB': a - b, 'MUL': a * b}[op])
    return st.pop()

src = '2 + 3 * (4 - 1) - -5'
print('source  :', src); print('tokens  :', lex(src)); tree = parse(src); print('AST     :', tree)
code = codegen(tree); print('bytecode:', code); print('result  :', run(code), '| Python eval says', eval(src))

# ---- fuzz against Python's own evaluator ----
R = random.Random(7)
def gen(d=0):
    if d > 3 or R.random() < 0.3: return str(R.randint(0, 20))
    r = R.random()
    if r < 0.15: return '-' + gen(d + 1)
    if r < 0.3: return '(' + gen(d + 1) + ')'
    return gen(d + 1) + R.choice([' + ', ' - ', ' * ']) + gen(d + 1)
for _ in range(3000):
    e = gen(); assert run(codegen(parse(e))) == eval(e), e
print('fuzz: 3000 random expressions, the compiler and VM agree with Python eval on every one')

# ---- Python bytecode, for comparison ----
import dis
def f(a, b): return a * 2 + b
print('\nCPython compiles to bytecode too (dis.dis of f(a, b): return a * 2 + b):'); dis.dis(f)
```

Output:

```text
source  : 2 + 3 * (4 - 1) - -5
tokens  : [('NUM', 2), ('+', '+'), ('NUM', 3), ('*', '*'), ('(', '('), ('NUM', 4), ('-', '-'), ('NUM', 1), (')', ')'), ('-', '-'), ('-', '-'), ('NUM', 5), ('END', None)]
AST     : ('-', ('+', ('num', 2), ('*', ('num', 3), ('-', ('num', 4), ('num', 1)))), ('neg', ('num', 5)))
bytecode: [('PUSH', 2), ('PUSH', 3), ('PUSH', 4), ('PUSH', 1), ('SUB', None), ('MUL', None), ('ADD', None), ('PUSH', 5), ('NEG', None), ('SUB', None)]
result  : 16 | Python eval says 16
fuzz: 3000 random expressions, the compiler and VM agree with Python eval on every one

CPython compiles to bytecode too (dis.dis of f(a, b): return a * 2 + b):
 75           0 RESUME                   0
              2 LOAD_FAST                0 (a)
              4 LOAD_CONST               1 (2)
              6 BINARY_OP                5 (*)
             10 LOAD_FAST                1 (b)
             12 BINARY_OP                0 (+)
             16 RETURN_VALUE
```

It was checked against Python's own `eval` on **3,000 random expressions**, and every result matched. **What it demonstrates:** a grammar becomes a parser almost mechanically, an AST makes later stages simple, and **code generation is a post-order walk of the tree** (children first, then the operator, which is exactly how a stack machine wants its operands). Real compilers add **type checking, a symbol table, an optimising IR (often SSA, static single assignment), instruction selection and register allocation (a graph-colouring problem)**.

**Grammar facts to know:** **BNF** describes syntax with rules. A grammar is **ambiguous** if a string has two parse trees (the dangling `else` is the classic example), solved by precedence and associativity rules. **LL parsers** (top-down, like the one above) and **LR parsers** (bottom-up, generated by tools such as yacc and bison) are the two main families. A **lexer** is built from regular expressions (finite automata), a **parser** from context-free grammars (pushdown automata).

### Interview questions on compilers

- **Walk through what happens from `gcc hello.c` to the program running.** Preprocess, compile, assemble, link, then the kernel's `execve` and the dynamic loader, `_start`, `main`.
- **What is the difference between compiling and linking?** Compiling translates each source file separately into an object file with unresolved symbols. Linking combines them and resolves the symbols.
- **Static versus dynamic linking?** Self-contained and large (785 KB versus 16 KB in our run) versus shared libraries loaded at run time.
- **What does `undefined reference` mean, and `multiple definition`?** Declared but not defined in anything linked, versus defined more than once (put definitions in one `.cpp`).
- **Why can optimisation change the behaviour of a buggy program?** The compiler assumes no undefined behaviour and transforms accordingly (deleted overflow checks, hidden data races).
- **What do `.text`, `.data`, `.bss` and `.rodata` hold?** Code, initialised globals, zero-initialised globals (no file space), constants and string literals.
- **What is a macro pitfall?** Textual substitution, missing parentheses (`1+2 * 1+2 = 5`), double evaluation of arguments.
- **Is Python compiled or interpreted?** Both: compiled to bytecode, executed by a virtual machine.
- **What is name mangling and `extern "C"`?** Encoding signatures in symbol names to support overloading, and disabling it for C interoperability.
- **What does a JIT do?** Compiles hot code to machine code at run time using profile information.

### Revise in 30 seconds: compilers and execution

- Source, preprocess (text: includes and macros), compile (lex, parse, check, optimise, assembly), assemble (object file with `T`, `D`, `U` symbols and sections), link (resolve symbols, add startup code), load (`execve`, dynamic loader, PLT and GOT, ASLR), run.
- Sections: `.text` code, `.data` initialised, `.bss` zero-filled (no file space), `.rodata` constants. Static link 785 KB versus dynamic 16 KB.
- `-O0` is a literal translation, `-O2` folded the loop to the constant 45. The optimiser assumes no undefined behaviour. Debug builds use `-O0 -g`, benchmarks `-O2`, always `-Wall`.
- Macros are text: parenthesise, avoid double evaluation (`1+2 * 1+2 = 5`). Undefined reference means no definition linked, multiple definition means defined twice. C++ mangles names (`_Z3addii`), `extern "C"` disables it.
- Compilers: lexer, parser (precedence in the grammar), AST, codegen as a post-order walk, VM. Python compiles to bytecode, JITs compile hot code at run time.

## Part 4: Computer architecture

### 4.1 How a CPU runs your program

- A CPU repeats the **fetch, decode, execute, write-back** cycle. It works on **registers** (a few dozen very fast storage slots), executes instructions from an **instruction set architecture (ISA)** (x86-64 and ARM64 are the common ones), and is paced by a **clock** (the frequency in GHz). **Performance = instructions per program x cycles per instruction x seconds per cycle**, so faster clocks, fewer cycles per instruction, or fewer instructions all help.
- **RISC** (ARM, RISC-V: simple fixed-size instructions, load and store only touch memory) versus **CISC** (x86: richer variable-length instructions, internally translated into simpler micro-operations). The distinction matters less today than it used to.
- **Function calls and the stack (the System V x86-64 convention, as seen in the `-O0` assembly above):** the first integer arguments go in registers (`rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`), the result comes back in `rax`, `call` pushes the **return address**, and the function begins with `push rbp; mov rbp, rsp` to set up its **stack frame** (saved registers, locals). A **buffer overflow** that overwrites the saved return address lets an attacker redirect execution, which is why compilers add **stack canaries** (a guard value checked before returning), the **NX bit** marks the stack non-executable, and **ASLR** randomises addresses (Part 1.2).

### 4.2 Pipelining, hazards, and branch prediction (verified)

**Pipelining** overlaps the stages of consecutive instructions (like a car assembly line), so ideally one instruction **completes every cycle** even though each takes several. Modern cores are also **superscalar** (several instructions per cycle) and **out of order** (they execute whatever is ready, then retire results in program order).

**Hazards** stall the pipeline: a **data hazard** (an instruction needs a result that is not ready yet, partly solved by **forwarding**), a **structural hazard** (two instructions need the same unit), and a **control hazard** (a branch: the CPU does not know which instruction comes next until the condition is evaluated).

**Branch prediction** attacks control hazards. The CPU **guesses** the direction of each branch and **speculatively executes** ahead. A correct guess is free, a **misprediction flushes the pipeline** and wastes on the order of 10 to 20 cycles. (Speculative execution is also the root of the Spectre family of security vulnerabilities.)

**The classic experiment, run for real:** sum only the values `>= 128` over a million random bytes, 100 times, then **sort the data and repeat**. It is the **same work and the same result**, only the order differs, so a sorted array makes the branch almost perfectly predictable.

```text
-O0 (a real conditional branch in the loop):
random order 974 ms | sorted 348 ms | random is x2.8 slower  (same result 10047555700 = 10047555700)
random order 935 ms | sorted 318 ms | random is x2.9 slower
-O2 (the compiler removes the data-dependent branch):
random order 94 ms | sorted 92 ms | random is x1.0 slower
```

**Two lessons:** (1) **unpredictable branches are expensive** (about 2.8 times slower here), and sorting data can speed up a loop even though sorting itself costs time. (2) **The effect disappeared at `-O2`**, because the compiler replaced the branch with **branchless code** (a conditional move or vectorised instructions), so micro-benchmarks depend on compiler flags. In the interview, say both: the mechanism, **and** that you would **measure with the production optimisation level** before drawing conclusions. (At `-O1` in a separate run the effect was also absent, about x1.0.)

### 4.3 The memory hierarchy and cache locality (verified)

Memory is **organised by speed versus size**: registers, then **L1, L2, L3 caches**, then RAM, then disk (latency figures are in Volume 6). A cache holds copies of recently used memory, and it moves data in fixed blocks called **cache lines (64 bytes on this machine, confirmed with `getconf LEVEL1_DCACHE_LINESIZE`)**, so reading one byte **brings in its 63 neighbours**. **Pages** are 4,096 bytes (`getconf PAGESIZE`). Caches work because programs show **locality of reference**: **temporal** (reuse recently used data) and **spatial** (use nearby addresses next).

**Experiment 1: traversal order.** Sum a 4096 x 4096 matrix of ints (64 MiB) stored row by row, once **walking along rows** and once **walking down columns**. Same additions, same result (`33554432`):

```text
run 1: row-by-row 17 ms | column-by-column 165 ms | slowdown x9.6
run 2: row-by-row 11 ms | column-by-column 137 ms | slowdown x12.2
```

Walking along a row uses all 16 ints in each cache line before moving on. Walking down a column jumps 16 KiB between accesses, so **each access touches a different cache line (and often a different page)**, and the line fetched is mostly wasted. **About 10 times slower for identical arithmetic.** This is why C and C++ matrices are traversed row-major, why **arrays of structs versus structs of arrays** matters, and why **a `vector` usually beats a linked list** even when big-O says otherwise (list nodes are scattered, every step is a cache miss).

**Experiment 2: stride.** Touch 2^20 elements of a 256 MiB array, changing only the **distance between consecutive accesses** (stride, in ints):

```text
stride    1:  1.43 ms    (consecutive: every line fully used, and the prefetcher streams ahead)
stride    4:  2.94 ms
stride   16:  6.61 ms    (16 ints = 64 bytes: a new cache line on every access)
stride   64: 11.72 ms
stride  256:  9.53 ms
stride 1024:  7.39 ms
```

The time **grows several-fold as the stride reaches and passes one cache line**, then levels off once nearly every access misses. The exact numbers varied a lot between runs of this experiment on the shared sandbox (an earlier run gave 1.2 ms at stride 1 and about 24 ms at larger strides), and several effects overlap (cache lines, the hardware prefetcher, and the **TLB** at strides of 1,024 ints = one page per access), so **trust the pattern, not the digits**. The conclusion for code: **sequential access is dramatically faster than scattered access**, even when the work looks identical.

**Cache organisation, briefly:** a cache is divided into **sets**. **Direct-mapped** (each address maps to one place, simple, conflict misses), **set-associative** (each address may go in any of N ways in its set, the common design), **fully associative** (anywhere, expensive). On a miss, a **replacement policy** (usually approximate LRU) picks a victim. **Write-through** writes to memory immediately, **write-back** marks the line dirty and writes it later (faster, needs care). **The three kinds of miss:** compulsory (first touch), capacity (the working set does not fit), conflict (too many addresses compete for one set).

**Multicore effects (not run here: the sandbox had a single CPU):** each core has its own L1 and L2, so the hardware keeps them consistent with a **cache coherence protocol (MESI: Modified, Exclusive, Shared, Invalid)**. **False sharing** happens when two threads write **different variables that sit on the same cache line**: the line ping-pongs between cores and performance collapses, although the program is logically correct. The fix is to **pad or align** per-thread data to separate cache lines (`alignas(64)`). Atomic operations and locks are also slower when many cores contend for one line.

### 4.4 Virtual memory and the TLB, in one paragraph

Every address a program uses is **virtual** and translated to a physical address by the MMU through **page tables** (Volume 1, Part 7). Translating on every access would be slow, so a small cache of recent translations, the **TLB**, makes it nearly free **as long as the working set spans few pages**. A scattered access pattern touching many different 4 KiB pages causes **TLB misses**, each needing a page-table walk, and **huge pages** (2 MiB) reduce that pressure for large datasets such as databases. **Page faults** (Volume 1) cost milliseconds when they need the disk, so keep the working set in RAM.

### 4.5 Data representation (verified)

All of this is real output from one program:

```text
endianness: 0x01020304 is stored in memory as 04 03 02 01 (little endian)
two's complement: int8 -1 is the bit pattern 0xff (255 unsigned); -128..127 is the int8 range
unsigned overflow wraps (well-defined): 4294967295 + 1 = 0
IEEE 754 float 0.1f = bits 0x3dcccccd: sign 0 exponent 123 mantissa 0x4ccccd, stored value 0.10000000149011611938
0.1 + 0.2 == 0.3 ? false (0.1 + 0.2 = 0.30000000000000004441)
fabs(a-b) < 1e-9 comparison: equal within tolerance | float has ~7 digits: 16777217.0f prints as 16777216.0 | NaN != NaN: 1
```

- **Endianness:** the byte order of multi-byte values. **Little endian** (x86, most ARM) stores the least significant byte first (`04 03 02 01`), **big endian** stores the most significant first, and network protocols use **big endian ('network byte order')** (`htons`, `htonl` convert). It matters when reading binary files or sending structs over a network.
- **Two's complement** represents negative numbers: to negate, **invert all bits and add 1**. `-1` is all ones (`0xff` in 8 bits). A signed `n`-bit integer ranges over `-2^(n-1)` to `2^(n-1) - 1` (-128 to 127 for 8 bits), so there is **one more negative value than positive**, and `-INT_MIN` overflows. **Signed overflow is undefined behaviour in C and C++, unsigned overflow wraps** (shown: `4294967295 + 1 = 0`). Right-shifting a negative signed number is typically an **arithmetic** shift (sign-extending), an unsigned one is **logical**. Python's integers are unbounded and have none of these problems.
- **IEEE 754 floating point:** a value is `(-1)^sign x 1.mantissa x 2^(exponent - bias)`. A **32-bit float** has 1 sign bit, 8 exponent bits (bias 127) and 23 mantissa bits (about **7 decimal digits**), a **64-bit double** has 11 exponent bits and 52 mantissa bits (about **15 to 16 digits**). `0.1` is **not exactly representable in binary**, just as 1/3 is not in decimal: the float nearest to 0.1 is `0.100000001490116...` (bits `0x3dcccccd`, exponent field 123 so the unbiased exponent is 123 - 127 = -4: 0.1 = 1.6 x 2^-4). That is why **`0.1 + 0.2 == 0.3` is false** (the sum is `0.30000000000000004441`).
- **Rules for floating point:** never compare with `==`, compare with a **tolerance** (`fabs(a - b) < eps`, `math.isclose`). A 32-bit float **cannot represent every integer above 2^24 = 16,777,216**, shown by `16777217.0f` printing as `16777216.0`. **Floating-point addition is not associative**, so summing in a different order (including in parallel) changes the last digits. **NaN is not equal to anything, including itself** (`NaN != NaN` printed 1), and `inf` and `-inf` are legitimate values. Very small numbers become **denormals** and lose precision. **Never store money in binary floating point**, use integers in minor units or a decimal type (Volume 4).
- **Characters:** ASCII is 7 bits, **Unicode** assigns code points to every character, **UTF-8** encodes them in 1 to 4 bytes and is backward compatible with ASCII, so a 'character' and a 'byte' differ, and string length depends on what you count.
- **Bit tricks** from Volume 8 (`x & (x - 1)`, `x & -x`, XOR) rely directly on two's complement.

### 4.6 Parallelism and Amdahl's law (computed)

Adding cores helps only for the **parallelisable fraction** `p` of the work. **Amdahl's law:** `speedup = 1 / ((1 - p) + p / n)` on `n` cores, so the serial fraction `(1 - p)` caps the gain **no matter how many cores you add**:

```text
parallel fraction 50%: max 2x on infinite cores;   8 cores give 1.78x
parallel fraction 90%: max 10x on infinite cores;  8 cores give 4.71x
parallel fraction 95%: max 20x on infinite cores;  8 cores give 5.93x
parallel fraction 99%: max 100x on infinite cores; 8 cores give 7.48x
```

Check the 90 percent row by hand: `1 / (0.1 + 0.9 / 8) = 1 / 0.2125 = 4.71`. **Practical consequences:** find the **serial bottleneck** (locks, a single-threaded stage, I/O, a shared counter) before adding hardware. **Synchronisation and cache coherence add overhead** that Amdahl's formula ignores, so real speedups are lower. **Gustafson's law** is the optimistic counterpart (a bigger problem on more cores keeps the serial part proportionally small). Related ideas: **SMT / hyper-threading** (two hardware threads share one core's resources), **SIMD** (one instruction operates on a vector of values, which the optimiser uses for loops), and **GPUs** (thousands of simple cores for data-parallel work). The earlier concurrency volume and the thread experiment in Part 1.11 cover correctness, this section covers speed.

### 4.7 Reasoning about performance like an engineer

- **Big-O is necessary but not sufficient.** Two O(n) loops can differ by 10 times because of **memory access patterns** (experiments above), **branch predictability**, and **allocation**. Constant factors and cache behaviour decide between algorithms of the same order, and sometimes a worse big-O wins for small inputs.
- **Measure first.** Profile (`perf`, `py-spy`, `cProfile`, Volume 5), find the hot spot, change one thing, measure again. Benchmark with **representative data** and **production compiler flags**, repeat runs, and report the spread, as the varying timings above should remind you.
- **Know the order of magnitude of costs:** a register access is under a nanosecond, an L1 hit a few cycles, a main-memory access roughly a hundred nanoseconds, a system call hundreds of nanoseconds to microseconds, a context switch microseconds, an SSD read tens to hundreds of microseconds, a network round trip within a data centre about half a millisecond, across continents about a hundred (Volume 6).
- **Reduce work, then reduce memory traffic, then add parallelism.** In that order.

### Loopholes and common wrong answers in architecture

- 'Faster clock means faster program': instructions per cycle, caches and memory latency matter as much.
- 'Cache is just a small fast memory': it moves 64-byte lines, so layout and access order matter.
- 'More cores always means proportionally faster': Amdahl's law and contention.
- 'Floating point is imprecise randomly': it is deterministic and exactly specified, just not able to represent most decimals.
- Treating signed and unsigned overflow the same, or assuming `int` is 32 bits everywhere.
- Quoting a micro-benchmark result without the compiler flags (the branch experiment reversed at `-O2`).

### Interview questions on architecture

- **What is a cache line, and why does traversal order matter?** The 64-byte unit caches move. Row-major walks use every byte, column walks waste most of each line (we measured roughly 10 to 12 times slower).
- **What is pipelining, and what is a hazard?** Overlapping instruction stages, stalls from data, structural or control dependencies.
- **What is branch prediction, and why was a sorted array faster?** Speculating on branch direction. Predictable branches are free, mispredictions flush the pipeline (2.8 times slower at `-O0`, not visible at `-O2`).
- **What is false sharing?** Different variables on the same cache line written by different cores, causing coherence traffic.
- **Explain little endian versus big endian.** Byte order of multi-byte values. x86 is little, networks use big.
- **How are negative numbers represented, and what is the range of an 8-bit signed integer?** Two's complement, -128 to 127.
- **Why is `0.1 + 0.2 != 0.3`?** 0.1 and 0.2 are not exactly representable in binary floating point, so the sum is 0.30000000000000004. Compare with a tolerance.
- **What does Amdahl's law say?** Speedup is limited by the serial fraction: 90 percent parallel can never exceed 10 times (4.71 times on 8 cores).
- **Why can a vector beat a linked list despite equal big-O?** Contiguous memory means cache-friendly, prefetchable access, list nodes are scattered.
- **What is the TLB?** A small cache of virtual-to-physical translations that avoids page-table walks.
- **What is a stack frame, and how does a buffer overflow exploit it?** Locals and the saved return address live on the stack, overwriting the return address redirects execution, mitigated by canaries, NX and ASLR.

### Revise in 30 seconds: computer architecture

- CPU: fetch, decode, execute. Pipelines overlap stages, hazards stall them, branch misprediction flushes (sorted data ran 2.8 times faster at `-O0`, no difference at `-O2` where the branch was removed).
- Hierarchy: registers, L1, L2, L3, RAM, disk. Cache lines are 64 bytes, pages 4,096. Row-major traversal was about 10 to 12 times faster than column-major, strides beyond a cache line slowed access several-fold, sequential beats scattered. False sharing: different variables, same line.
- Virtual memory: page tables plus TLB, huge pages cut TLB misses.
- Little endian (x86), network order is big endian. Two's complement: invert and add 1, range -128 to 127, signed overflow is undefined, unsigned wraps.
- IEEE 754: 0.1 is not exact, `0.1 + 0.2 = 0.30000000000000004`, compare with a tolerance, floats lose integers above 2^24, NaN != NaN, never use floats for money.
- Amdahl: `1 / ((1 - p) + p / n)`, 90 percent parallel gives 4.71 times on 8 cores and never more than 10. Measure with production flags, find the serial bottleneck, reduce memory traffic before adding cores.

## Part 5: The interview bank and cheat sheet

### 5.1 Rapid-fire questions across the whole volume

1. **Where does each kind of variable live (local, global, `static`, `malloc`'d, string literal)?** Stack, `.data` or `.bss`, `.data` or `.bss` (a function-local `static`), heap, `.rodata`.
2. **Why do deeper recursive calls have lower addresses?** The stack grows downward (shown in the layout output).
3. **What is a memory leak, a dangling pointer, a double free?** Never freed, used after being freed, freed twice. ASan finds them.
4. **When would you use `shared_ptr`?** Genuinely shared ownership. Default to `unique_ptr`, and use `weak_ptr` to break cycles.
5. **What is the output of copying versus moving a `std::vector<Noisy>`?** Copy duplicates every element, move steals the buffer, and reallocation moves elements only if the move constructor is `noexcept`.
6. **Why must a polymorphic base class have a virtual destructor?** Otherwise `delete base_ptr` skips the derived destructor (undefined behaviour).
7. **Explain what happens on `std::sort` with `<=` as the comparator.** Undefined behaviour: the comparator must be a strict weak ordering.
8. **How do you debug a crash with no information?** Rebuild with `-g -fsanitize=address,undefined`, reproduce, read the report, then use a debugger or logging.
9. **What is the difference between `fork` and `exec`?** Duplicate the process versus replace the program image (Volume 1).
10. **How would you find the five largest files in a directory tree?** `find . -type f -exec du -b {} + | sort -rn | head -5` (or `du -ah . | sort -rh | head`).
11. **Why does `for f in $(ls)` break?** Word splitting on spaces. Use globs or `find -print0 | xargs -0`.
12. **What does exit status 137 mean?** Killed by SIGKILL (128 + 9), often the OOM killer.
13. **What do the sections `.text`, `.data`, `.bss` contain?** Code, initialised globals, zero-initialised globals.
14. **Why can `-O2` break code that works at `-O0`?** Undefined behaviour the optimiser exploits, or timing-dependent bugs revealed or hidden.
15. **Why is column-major traversal slower?** Cache lines and prefetching.
16. **Why are floats bad for currency?** Binary cannot represent most decimals exactly, errors accumulate.
17. **What limits parallel speedup?** The serial fraction (Amdahl), contention, and cache coherence costs.

### 5.2 One-page cheat sheet

- **Types and sizes:** `char` 1, `short` 2, `int` 4, `long` 8 (on 64-bit Linux), pointer 8, `double` 8. Padding: align to the member size, order members largest-first.
- **Memory:** `.text` `.rodata` `.data` `.bss`, heap up, stack down, ASLR randomises, `mmap` for big allocations.
- **C++ ownership:** `unique_ptr` default, `shared_ptr` counted, `weak_ptr` breaks cycles, rule of zero, `noexcept` moves, `virtual ~Base()`, `override`, no slicing.
- **UB list:** signed overflow, out-of-bounds, use-after-free, null dereference, uninitialised read, data race, non-virtual delete through base. Tools: `-fsanitize=address,undefined,thread`.
- **Shell:** `set -euo pipefail`, quote `"$var"`, `sort | uniq -c | sort -rn | head`, `2>&1`, exit 0 means success, `kill` = SIGTERM, `kill -9` = status 137, permissions 4 2 1.
- **Build:** `gcc -E` preprocess, `-S` assembly, `-c` object, link, `nm` shows `T D U`, `ldd` shows libraries, `-O0 -g` to debug, `-O2` to benchmark, `-Wall -Wextra` always.
- **Architecture:** cache line 64 B, page 4 KiB, little endian, two's complement, IEEE 754 (float 24-bit significand, double 53-bit), Amdahl `1 / ((1 - p) + p / n)`.

## What Volume 9 covers, and what comes next

Volume 9 covers the programming and systems items of your checklist in depth, with programs that were compiled and run: C++ memory layout, RAII and smart pointers, move semantics, virtual functions, struct padding, the STL, undefined behaviour and sanitizers, and a data race that lost millions of updates (and was hidden by the optimiser); Linux file, permission, link, pipeline, signal and `/proc` behaviour from real sessions; the full compiler pipeline run step by step plus a fuzz-tested miniature compiler and VM; and computer architecture measured on the machine: cache locality, strides, branch prediction, floating point and Amdahl's law.

That completes the nine study volumes. The last piece is the **final revision map**: one page per round (OA, technical round, interview) that links every 'Revise in 30 seconds' box in the order you should reread them, with a last-week schedule.
