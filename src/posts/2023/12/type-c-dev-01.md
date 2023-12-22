---
title: "Type-C Dev Post 1"
date: "2023-12-22"
tags:
- type-c
- type-v
---
In a new series of blog post, I highlight the current status of type-c and the type-v VM. Since I am posting on an infrequent basis, expect the order of these posts to be chaotic.
<!-- excerpt -->

## Type-C
Type-C, does not aim to be a toy language. Even though it is the first large-scale language that I develop (scale is relative), I am trying to make it as efficient as possible. 
On the road of developments, choices have been made. Some of them are good, some of them are hard to deal with, let's talk about the foundations of the language:


1. Statically typed language:

Type-C is a statically typed language. Inspired heavily by typescript. The type system is structural, not nominal. This means that types are compatible if their structure is compatible, not if they have the same name.

> If it walks like a duck and it quacks like a duck, then it must be a duck.

However, duck typing is fixed to the interface level. Meaning, structs and interfaces both support duck typing, but classes do not. Classes are meant to be final, concrete and well-estbalished entities. 

2. Interface oriented programming:

Type-C is an interface oriented language. This means that classes cannot extend other classes. Instead, classes can implement interfaces. Interfaces are a set of methods that a class must implement. This is similar to the concept of interfaces in Java or Go. Interface can extend other interfaces, but not classes.

In today's post, I aim the address how structs are implemented in type-v.

As usual, anything written here might not necessarily end up in the final version of the language.

## Problem Definition

We all love javascript objects. They are easy to use, easy to create, and easy to pass around. However, under the hood is a hash function that maps keys to values. This is not very efficient. In order to maintain a minimal level of flexibility and maximum level of efficiency, Type-C uses allows structs to be used with duck typing.

In typescript, we can do the following:
```ts
let x = {x: 1, y: 2}
let y = {y: 2, x: 1}

function sumCoords(c: {x: number, y: number}) {
    return c.x + c.y
}

sumCoords(x)
sumCoords(y)
// both work just fine
```
Well javascript uses hashmap for fields. So order doesn't really matter. Type-C aims to be more efficient than that. How can we maintain compability between structs without using a hashmap? The answer is simple, one solution is using offsets. 


### Type-V Structs
Let's look at the following example:

```
strict type S = {x: i32, y: i32, z: {x: i32, y: i32}}

fn f() {
    let alpha: {x: i32, y: i32} = {y: 1, x: 6}
}

```

The keyword `strict` indicate that type will be matched using its name, which is `S` in this case. This keyword is used in purposes where you want to be explicit about the type. For instance, when you want to create a function that takes a struct as an argument. This feature doesn't have much impact on type inference, but rather is a syntactic sugar feature. It changes type checking from structural to nominal, but can be easily by passed through regular casting. However, this is just for demonstration purposes, it is not being used in this example.

What is important to notice in this example is how the type of `alpha` is inferred. Even though the type of alpha is not necessarily the value it is being assigned to, it is still compatible with it. This is because the type of `alpha` `{x: i32, y: i32}` is structurally compatible with `{y: i32, x: i32}`. 

The way type-c handles this, is by using two different data structures for structs. First being regular `struct`, second is `shadow struct`. A shadow struct, is merely a different view of the same structure, it is either the full view or a partial view.

In type-v, each struct has data block, which contains the entire data of the struct. This data block is a contiguous block of memory. Additionally, it also contains an array of offsets. Each offset is the offset of a field in the struct. This is used to access the field of the struct.

A shadow struct, is a struct that has a different view of the same data block. It's data points to the original data block, but it's offsets are different. 

So think of `y: 1, x: 6` as a struct:
```
tmp1 = {
    offsets: [0, 1]
    data: [1, 6],
}
```

a shadow copy with the type `{x: i32, y: i32}` would be:
```
tmp_2 = {
    offsets: [1, 0]
    data: tmp1.data
}
```

### Type-V Structs IR

Now let's have a look at the generated IR for the previous example:

{% asset_img "tc/graph-01.png" "Generated IR" %}


The IR is a bit hard to follow at first, but idea is that, `const_[type]` such as `const_i32` here creates a temporary variable, which will be used a register later on for the actual value. 
Let's look at the instruction we have here

| Instruction | Description | Arg 1 | Arg 2 | Arg 3 |
| ----------- | ----------- | ----- | ----- | ----- |
|`const_i32` | Creates a temporary variable of type i32 | name | value | |
|`local_ptr`|Assigned a pointer to a local variable| local target | tmp value | |
|`s_alloc`|Allocates a struct|Destination| Number of fields | total size in bytes |
|`s_set_offset`|Sets the offset of a given field index to the given value in the struct| struct | field index | offset value |
|`s_set_field_[type]`|Sets the value of the given index (through the offset) to the given value| struct | field index | value |
|`s_alloc_shadow`|Allocates a shadow struct| Original Struct (could also be a shadow) |Destination | Original Struct|Number of fields|
|`s_set_offset_shadow`|Sets the offset of a given field index (arg1) to the offset of the second given field index in the struct's original| struct | field index in the shadow struct| field index the original struct |

Now we can interpret the IR as follows:
1. Creates a struct and fills it 
2. Allocates shadow copy of the original struct, with swapped offsets

`s_set_offset_shadow struct dest_index src_index` will simply perform the following operation:

```
shadow.offsets[dest_index] = shadow.originalStruct.offsets[src_index]
```

Let's try to modify the value of the shadow struct now:

```
fn f() {
    let alpha: {x: i32, y: i32} = {y: 1, x: 6}
    alpha.x = 100
}
```

The new IR looks as follows:
    
{% asset_img "tc/graph-02.png" "Generated IR" %}

New instructions stores the constant 100 in `tmp4` and stores them into `alpha` struct.
The field index of `s_set_field_i32` is `0`, but if we go through the offset table.

Since the field 0 offset was swapped with the field 1 offset in the original, the actual offset of the field 0 in the shadow is 4, hence both structs are updated correctly.

### Conclusion

This post will hopefully turn into a series of posts that will highlight the development of type-c. I am not sure how frequent these posts will be, but I will try to keep them as frequent as possible.
With that being said, have a good one.