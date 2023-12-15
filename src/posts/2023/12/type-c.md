---
title: "Introducing Type-C"
date: "2023-12-08"
published: true

tags:
- type-c
- programming language
---

<!-- excerpt -->
{% asset_img "type-c.png" "typec" %}

### Notice
This is a work in progress. The language is not ready yet. I am still working on both the compiler and the VM.
I have no guarentee that everything written here will remain the same once the language is released.
However, The language design, philosophy, and syntax are pretty much in shape. The standard library
and VM instructions are the most likely to change.

# What is type-c
Type-C is my work-in progress programming language which I have been desining for some time now. 

Writing a language is something that I have been doing on and off for more than a decade. The challenge is not really to build a language per-se, but to build a language that is useful and that I can use to build other things. For instance, every time I can think of something that could make my language unique, some else is developed which does it better.

I will write about my history with creating languages some other time. At the moment, let's focus on Type-C.

This language is designed with the following goals in mind:
- Productivity: The language shouldn't slow you down
- Type safe: Reduce run time errors
- Efficiency: The language should be fast. As fast as possible
- Concurrency: The language should have inherent support for concurrency
- FFI: The language should have a good FFI, especially for C

Now that the goals have been set, let's look at the language itself.

# Type-C
Type is multi-paradgim language. It is a statically typed language with support for both interface oriented and functional programming. It is designed to be an application-level language, similar to python or modern JavaScript.

- Interface oriented programming (extend interface rather than classes)
- Functional programming (first class functions, closures, variadic types, etc)
- Static and Strong typing (type inference, type annotations)
- Concurrency (through processes, not to be confused with OS processes however)
- No Exceptions, Algebraic data types are used for error handling.
- FFI (to C, and other languages)

## Interface-what?
Interface oriented programming is a paradigm where the focus is on the interfaces between components. The idea is that the interfaces are more important than the implementation. This is in contrast to object oriented programming where the focus is on the objects and their implementation. 

You can still create classes in interface oriented programming, but classes are final by nature and cannot extend other classes. Instead, the default inheritance mechanism is through interfaces. An interface is a set of methods that a class must implement. This is similar to the concept of interfaces in Java or Go. 

Type-C Encourages the use of composition over inheritance. One of the key advantages of object-oriented programming is code reuse. Inheritance is one way to achieve code reuse. However, inheritance can lead to problems when used inappropriately. Type-C allows inheritance of interfaces, but encourages code-reuse when composition is not possible.

Also interfaces inhenrently provide encapsulation. This is because the implementation of the interface is hidden from the user. The user only sees the interface and not the implementation.

## No Exceptions?
Exceptions are a common way to handle errors in many languages. However, exceptions are not always the best way to handle errors. Exceptions are not always recoverable, and they can lead to unexpected behavior. Type-C uses algebraic data types to handle errors. 

Here is the tl;dr. Exceptions, as the name suggest, are exceptional cases. But you have them everywhere in your code, then they are no exceptional aren't they?

## Hello Type-C

### Hello World
```rust
import std.io 
fn main(args: string[] ) -> u32 
{ 
    io.print( "Hello, world! ") 
    return 0 
}
```

### Fib
```rust
fib(x: u32) -> u32 = 
    match x { 
        0 => 0, 
        1 => 1, 
        _ => fib(n-1) + fib(n-2) 
    }
```
If you have ligatures enabled, then you have to admit the code looks sexy.

### Data Types
```rust
// struct
type Point = {
    x: u32
    y: u32
}

// same as
type Point = struct {
    x: u32
    y: u32
}

// algebraic type, with a generic-cherry on top
type ServerResponse<T> = variant {
    Ok(data: T),
    Error(code: u32)
}

let p1: Point = {10, 10} // {x, y}
let p2: {x: u32} = p1

fn printPoint(p: {x: u32, y: u32}) -> void {
    print("x: ", p.x, " y: ", p.y)
}

printPoint({10, 10})
```

### FFI
FFI is not really a Type-C feature, but rather a featuer of the VM, called Type-V. 
The FFI design was massively inspired by Lua. However since Type-C is statically typed,
you will need to write both C code and Type-C headers. Here is an example of stdio FFI.

```rust
extern stdio from "stdio.so" = {
    print(number: u32) -> void
}
```

```c
void stdio_print(TypeV_Core *core) {
    uint32_t number = typev_api_stack_pop_u32(core);
    printf("%d", number);
}

static TypeV_FFIFunc stdio_lib[] = {
        (TypeV_FFIFunc)stdio_print, NULL
};

size_t typev_ffi_open(TypeV_Core* core){
    return typev_api_register_lib(core, stdio_lib);
}
```

## Cool! What's the state?
Everything is still under-development. The code generator is still being written. The VM needs major features such as GC, improved memory management, etc.
So expect any release in mid-2024. I will be posting updates on the language here. So stay tuned.

I will keep posting about both the language and the virtual machine, so stay tuned.
