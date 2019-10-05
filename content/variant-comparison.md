Title: Beating std::visit Without Really Trying
Date: 2019-10-04
Category: Programming
Tags: c++, dlang, assembly

One of my current projects is [a sum type implementation for the D programming
language][1]. It bills itself as having "zero overhead compared to hand-written
C," and while there are sound theoretical reasons to think that's true, I'd
never actually sat down and verified it empirically. Was I really getting the
optimal performance I thought I was?

I was also curious to see how my efforts stacked up next to C++17's
[`<variant>`][2] and D's [`Algebraic`][3], especially after reading a few [blog
posts][4] and [reddit comments][5] about the challenges faced by the C++
implementation. Could D, and my library, `SumType`, in particular, do better?

### Methodology

To answer these questions, I designed a simple program and implemented it in C,
in C++, and in D with both `Algebraic` and `SumType`, using the most natural
and idomatic code for each language and library. My goal was to find out how
well a production-grade, optimizing compiler would handle each sum type
implementation in typical use. The compilers I chose were `clang` 8, `clang++` 8,
and `ldc` 1.17.0, all of which use LLVM 8 for their backends. All programs were
compiled with optimization level `-O2`.

Each test program does the following things:

- Defines 10 empty `struct` types, named `T0` through `T9`.
- Defines a sum type with 10 members, one of each of those types.
- Defines a function that takes an instance of that sum type as an argument,
  and returns a unique integer for each possible type the sum type can hold.

For illustration's sake, here's the C++ version, slightly abridged:

```cpp
#include <variant>

// Source: https://en.cppreference.com/w/cpp/utility/variant/visit
template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

struct T0 {};
// ...
struct T9 {};

using example_variant = std::variant<T0, T1, T2, T3, T4, T5, T6, T7, T8, T9>;

int do_visit(example_variant v)
{
    return std::visit(overloaded {
        [](T0 val) { return 3; },
        [](T1 val) { return 5; },
        [](T2 val) { return 8; },
        // ...
        [](T9 val) { return 233; },
    }, v);
}
```

After compiling each program, I used `objdump` to disassemble them, and
compared the generated assembly for each version of `do_visit`.

All of the code used, along with the commands used for compilation and
disassembly, are available on Github in the repository
[pbackus/variant-comparison][6].

### Results

#### C

The C version uses a `union` with an `enum` tag as its sum type implementation,
and a switch statement for its "visit" function. It's the straightforward,
obvious implementation you'd use if you were writing the code by hand, without
worrying about making it generic, so it makes a good baseline to compare the
other versions to.

```text
00000000000000a0 <do_visit>:
  a0:    cmp    $0xa,%edi         # check if type index is in-bounds
  a3:    jae    b0
  a5:    movslq %edi,%rax
  a8:    mov    0x0(,%rax,4),%eax # get result from global array
  af:    retq   

  # Error path
  b0:    push   %rax
  b1:    mov    $0x0,%edi
  b6:    mov    $0x0,%esi
  bb:    mov    $0x0,%ecx
  c0:    mov    $0x45,%edx
  c5:    callq  ca                # __assert_fail
```

As you might expect, the compiler is able to optimize the entire thing down
into a single array lookup. It's so compact, in fact, that the code for calling
`libc`'s assertion-failure function takes up more space than the actual logic.

#### C++

The C++ version uses `std::variant` and `std::visit`, with the [`overloaded`
helper template][7] from cppreference.com to allow passing a set of labmda
functions as a visitor. This is standard-library code, so we should expect it
to be as well-optimized as the best and brightest C++ developers around can
make it.

```text
0000000000000000 <do_visit(std::__1::example_variant)>:
   0:    sub    $0x18,%rsp
   4:    mov    %rdi,%rax
   7:    mov    %rdi,0x8(%rsp)
   c:    shr    $0x20,%rax
  10:    mov    $0xffffffff,%ecx
  15:    cmp    %rcx,%rax        # check if type index is in-bounds
  18:    je     38
  1a:    mov    %rsp,%rcx
  1d:    mov    %rcx,0x10(%rsp)
  22:    lea    0x10(%rsp),%rdi
  27:    lea    0x8(%rsp),%rsi
  2c:    callq  *0x0(,%rax,8)    # call function pointer in global array
  33:    add    $0x18,%rsp
  37:    retq   

  # Error path
  38:    callq  3d               # __throw_bad_variant_access
```

The main thing to notice here is that, unlike in the C version, the compiler is
not able to inline the individual visitor functions. Instead, it generates an
indirect call to a function pointer stored in a global array. (The individual
lambda functions, of course, all consist of a single `mov` followed by a
return.)

I'm not enough of a C++ expert to understand the implementation of
`std::visit`, so I can only guess why this is the case. Maybe the optimizer
just gives up after too many layers of template-metaprogramming gunk?
Regardless, it's bad news for fans of zero-cost abstractions, since there's a
clear overhead here compared to the C assembly.

#### D, with Algebraic

This version uses the sum type implementation from D's standard library,
Phobos. Rather than create a dedicated sum type, the Phobos developers opted to
make `Algebraic` a thin wrapper around `Variant`, D's equivalent of C++17's
`std::any`. That choice turns out to have far-reaching repercussions, as we're
about to see.

```text
0000000000000000 <int dalgebraic.do_visit(ExampleAlgebraic)>:
   0:    jmpq   5 # int std.variant.visitImpl!(true, ExampleAlgebraic, ...)

0000000000000000 <int std.variant.visitImpl!(true, ExampleAlgebraic, ...)
   0:    push   %rbx
   1:    sub    $0x10,%rsp
   5:    cmp    0x0(%rip),%rdi    # check for uninitialized Variant
   c:    je     217               # if so, go to error path
  12:    mov    %rdi,%rbx

  # This part is repeated 10 times, once for each type
  15:    movq   $0x0,0x8(%rsp)    # prepare arguments for VariantN.handler
  1e:    lea    0x8(%rsp),%rdi
  23:    xor    %esi,%esi
  25:    xor    %edx,%edx
  27:    callq  *%rbx             # VariantN.handler: get TypeInfo for current type
  29:    mov    0x8(%rsp),%rsi
  2e:    mov    0x0(%rip),%rdi    # get TypeInfo for T0
  35:    callq  3a                # Object.opEquals: compare TypeInfos
  3a:    mov    %eax,%ecx
  3c:    mov    $0x3,%eax         # load return value for T0 into %eax
  41:    test   %cl,%cl           # check if Object.opEquals returned true
  43:    jne    211               # if so, go to return
 ...:    ...    ...

 # After checking for T9
 20f:    je     283               # if none of the types matched, assert(false)
 211:    add    $0x10,%rsp
 215:    pop    %rbx
 216:    retq   

 # Error path
 217:    mov    0x0(%rip),%rdi    # get ClassInfo for VariantException
 21e:    callq  223               # _d_allocclass: allocate VariantException
 223:    mov    %rax,%rbx
 226:    mov    0x0(%rip),%rax    # initialize VariantException vtable
 22d:    mov    %rax,(%rbx)
 230:    movq   $0x0,0x8(%rbx)
 238:    mov    0x0(%rip),%rax    # initialize VariantException
 23f:    movups 0x50(%rax),%xmm0
 243:    movups %xmm0,0x50(%rbx)
 247:    movups 0x10(%rax),%xmm0
 24b:    movups 0x20(%rax),%xmm1
 24f:    movups 0x30(%rax),%xmm2
 253:    movups 0x40(%rax),%xmm3
 257:    movups %xmm3,0x40(%rbx)
 25b:    movups %xmm2,0x30(%rbx)
 25f:    movups %xmm1,0x20(%rbx)
 263:    movups %xmm0,0x10(%rbx)
 267:    lea    0x0(%rip),%rdx    # get message for VariantException
 26e:    mov    $0x2f,%esi
 273:    mov    %rbx,%rdi
 276:    callq  27b               # VariantException.__ctor
 27b:    mov    %rbx,%rdi
 27e:    callq  283               # _d_throw_exception
 283:    lea    0x0(%rip),%rsi    # get error message for assert
 28a:    mov    $0x4b,%edi
 28f:    mov    $0xa1c,%edx
 294:    callq  299               # call D runtime assert function
```

The good part is that `ldc` is able to inline the lambdas into the body of
`visitImpl`. The bad part is, well, everything else.

Because `Algebraic` is implemented using `Variant`, it has to rely on runtime
type information (`TypeInfo`) to check the type of the current value. And
because it uses `TypeInfo`, rather than an integer index like the C and C++
versions, these checks can't be condensed down into a single array lookup, or
even a jump table. Finally, as if that wasn't enough, each `TypeInfo`
comparison requires a virtual method call to `Object.opEquals`.

Beyond these obvious pessimizations, the other thing that stands out is the
sheer volume of the generated code. It's an order of magnitude larger than both
the C and C++ versions, with significant bloat in both the normal and error
paths. Not only is this bad in isolation, since it puts unnecessary pressure on
the instrution cache, it also means that potential optimization opportunities
exposed by inlining the C and C++ versions of `do_visit` won't be available to
D code using `Algebraic`.

#### D, with SumType

This version uses my own implementation of a sum type in D. `SumType` does not
rely at all on runtime type information, and instead uses D's powerful
compile-time reflection and metaprogramming capabilities to try to generate
code as close to the C version as possible. Let's see how well it does.

```text
0000000000000000 <int dsumtype.do_visit(ExampleSumType)>:
   0:    push   %rax
   1:    movsbq 0x10(%rsp),%rax
   7:    cmp    $0xa,%rax          # check if type index is out-of-bounds
   b:    jae    19
   d:    lea    0x0(%rip),%rcx     # load address of global array
  14:    mov    (%rcx,%rax,4),%eax # get result from global array
  17:    pop    %rcx
  18:    retq   

  # Error path
  19:    lea    0x0(%rip),%rdx     # get error message
  20:    mov    $0x4ca,%edi
  25:    mov    $0x9,%esi
  2a:    callq  2f                 # __switch_error
```

As it turns out, it's almost exactly the same as the C version: a single array
lookup, plus some code for error handling. In fact, as far as I can tell, the
only difference has to do with the way the function argument is passed in
registers—the C version is able to avoid spilling anything to the stack,
whereas this version has a `push` and a `pop` at the beginning and end,
respectively.

Still, I think it's fair to say that this is a true zero-cost abstraction, and
that the claim of "zero overhead compared to hand-written C" is justified.

### Conclusions

If creating a zero-overhead generic sum type is so easy that even I can do it,
why doesn't C++ have one? Are the `libc++` developers a bunch of dummies?

No, of course not—but they are working with a significant handicap: C++.

The source code for `variant` is terrifyingly complex. In order to get the
results shown above, the authors have had to use every template metaprogramming
trick in the book, and then some. It's clearly the result of immense effort by
some very skilled programmers.

By contrast, the source code of `SumType` is almost embarrassingly simple. The
reason `match`, `SumType`'s equivalent of `std::visit`, is able to optimize
down to the same code as a C switch statement is that it literally is a switch
statement, with a `static foreach` loop inside to generate the cases.

To be honest, I wasn't even thinking about optimization when I coded `SumType`.
I wanted to make sure the interface was nice, and figured I could worry about
performance later. But D makes it so easy to write simple, straightforward code
that the compiler understands, I ended up [falling into the pit of success][8]
and beating `std::visit` entirely by accident.

[1]: https://code.dlang.org/packages/sumtype
[2]: https://en.cppreference.com/w/cpp/header/variant
[3]: https://dlang.org/phobos/std_variant.html#.Algebraic
[4]: https://bitbashing.io/std-visit.html
[5]: https://www.reddit.com/r/cpp/comments/9khij8/stdvariant_code_bloat_looks_like_its_stdvisit/
[6]: https://github.com/pbackus/variant-comparison
[7]: https://en.cppreference.com/w/cpp/utility/variant/visit
[8]: https://blog.codinghorror.com/falling-into-the-pit-of-success/
