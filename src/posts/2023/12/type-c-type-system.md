---
title: "Type-C: Type System"
date: "2023-12-24"
tags:
- Type-C
- Type-V
---
The name Type-C, actually comes from two things. The first is the fact that it is a statically typed language and rich in terms of data types, the second is C, the language that inspired it. In this post, I will be talking about the various data types that type-c supports.
Some interesting types are intentionally left out, such as `variant`, `string`, `array` and `process`. These will be addressed in a separate post. Hence in this post we focus on: Basic Types, Enums, Structs, Interfaces, Classes, Joins, Nullables and Functions. We finalize by looking into the strict type modifier.
<!-- excerpt -->
The name Type-C, actually comes from two things. The first is the fact that it is a statically typed language and rich in terms of data types, the second is C, the language that inspired it. In this post, I will be talking about the various data types that type-c supports.
Some interesting types are intentionally left out, such as `variant`, `string`, `array` and `process`. These will be addressed in a separate post. Hence in this post we focus on: Basic Types, Enums, Structs, Interfaces, Classes, Joins, Nullables and Functions. We finalize by looking into the strict type modifier.

## Data types:
Here are the various data types that type-c supports:
- Integers: `i8`, `i16`, `i32`, `i64`, `u8`, `u16`, `u32`, `u64`
- Floats: `f32`, `f64`
- Booleans: `bool`
- Enums: `enum`
- Strings: `string`: Implemented as built-in interface and part of the VM runtime library.
- Arrays: `T[]`: Implemented as built-in interface and part of the VM runtime library.
- Structs: `struct`
- Interfaces: `interface`
- Classes: `class`
- Type Join: `U & V` where `U` and `V` must be interfaces or joins themselves.
- Nullables: `T?` where T is a class, interface, join Type or Process.
- Variants: `variant`
- Processes: `process`
- Functions: `fn (x: U) -> V`

In this post, I will be talking about a few of those. Most likely, variants, strings, arrays, and processes will get their own post each. So we are simply addressing the rest.

## Integers & Floats
Integers are the basic building blocks of any programming language. Type-C supports 8, 16, 32 and 64 bit integers. Their size is fixed.
Since type-c is a statically and strongly typed language, some integers are not implicitly converted to other types, when there is a potential loss of precision. For example, the following code will not compile:
```
let x: i64 = 1
let y: i32 = x // error
```

However,
```
let x: i32 = 1
let y: i64 = x // ok
```

In the virtual machine, casting is done by using the following cast instructions:
| Instruction | Description |
| ----------- | ----------- |
|`cast_i8_u8`|Casts the value in register R from i8 to u8|
|`cast_u8_i8`|Casts the value in register R from u8 to i8|
|`cast_i16_u16`|Casts the value in register R from i16 to u16|
|`cast_u16_i16`|Casts the value in register R from u16 to i16|
|`cast_i32_u32`|Casts the value in register R from i32 to u32|
|`cast_u32_i32`|Casts the value in register R from u32 to i32|
|`cast_i64_u64`|Casts the value in register R from i64 to u64|
|`cast_u64_i64`|Casts the value in register R from u64 to i64|
|`cast_i32_f32`|Casts the value in register R from i32 to f32|
|`cast_f32_i32`|Casts the value in register R from f32 to i32|
|`cast_i64_f64`|Casts the value in register R from i64 to f64|
|`cast_f64_i64`|Casts the value in register R from f64 to i64|
|`upcast_i`|Upcasts the value in register R from given bytes to target bytes|
|`upcast_u`|Upcasts the value in register R from given bytes to target bytes|
|`upcast_f`|Upcasts the value in register R from given bytes to target bytes|
|`dcast_i`|Downcasts the value in register R from given bytes to target bytes|
|`dcast_u`|Downcasts the value in register R from given bytes to target bytes|
|`dcast_f`|Downcasts the value in register R from given bytes to target bytes|

As you can see, in the VM instructions there are no direct casts between some of the types, such as `u8` and `f32`. The idea is, to ensure safety, we need to be explicit about the cast. This is why, we need to cast to `i32` first, then to `f32`. This is done by the following code:
```ts
export type CastType = "u8" | "u16" | "u32" | "u64" | "i8" | "i16" | "i32" | "i64" | "f32" | "f64";

export function generateCastInstruction(from: CastType, to: CastType, reg: string): string[][] {
    const typeInfo = {
        u8: { size: 1, type: 'u' },
        u16: { size: 2, type: 'u' },
        u32: { size: 4, type: 'u' },
        u64: { size: 8, type: 'u' },
        i8: { size: 1, type: 'i' },
        i16: { size: 2, type: 'i' },
        i32: { size: 4, type: 'i' },
        i64: { size: 8, type: 'i' },
        f32: { size: 4, type: 'f' },
        f64: { size: 8, type: 'f' }
    };

    if (!(from in typeInfo) || !(to in typeInfo)) {
        throw new Error("Invalid type");
    }

    const instructions: string[][] = [];
    let currentType: CastType = from;

    // Handle special cases first
    if (from.startsWith('u') && to.startsWith('f')) {
        // Upcast to the size of the target floating point, if needed
        if (typeInfo[from].size < typeInfo[to].size) {
            instructions.push(["upcast_u", reg, typeInfo[from].size.toString(), typeInfo[to].size.toString()]);
            currentType = `u${typeInfo[to].size * 8}` as CastType;
        }
        // Convert to the corresponding signed type
        instructions.push([`cast_${currentType}_I${typeInfo[to].size * 8}`, reg]);
        // Convert to the target floating point
        instructions.push([`cast_i${typeInfo[to].size * 8}_${to}`, reg]);
        return instructions;
    }

    // Handle float to int conversion
    if (from.startsWith('f') && !to.startsWith('f')) {
        const intermediateType = `i${typeInfo[from].size * 8}` as CastType;
        instructions.push([`op_cast_${from}_${intermediateType}`, reg]);
        currentType = intermediateType;
    }

    // Handle int to float conversion
    if (!from.startsWith('f') && to.startsWith('f')) {
        const intermediateType = `i${typeInfo[to].size * 8}` as CastType;
        if (typeInfo[currentType].size !== typeInfo[to].size) {
            instructions.push(["upcast_i", reg, typeInfo[currentType].size.toString(), typeInfo[to].size.toString()]);
            currentType = intermediateType;
        }
        instructions.push([`cast_${currentType}_${to}`, reg]);
        return instructions;
    }

    // Handle upcasting or downcasting
    if (typeInfo[currentType].size !== typeInfo[to].size) {
        const op = typeInfo[currentType].size < typeInfo[to].size ? "op_upcast" : "op_dcast";
        const typeChar = currentType[0];
        instructions.push([`${op}_${typeChar}`, reg, typeInfo[currentType].size.toString(), typeInfo[to].size.toString()]);
        currentType = `${typeChar}${typeInfo[to].size * 8}`.toLowerCase() as CastType;
    }

    // Handle any remaining type conversions
    if (currentType !== to) {
        instructions.push([`op_cast_${currentType}_${to}`, reg]);
    }

    return instructions;
}
```

Let's try few samples:

```ts
console.log(generateCastInstruction("u8", "f32", "r0"))
(3) [Array(4), Array(2), Array(2)]
    0: (4) ["upcast_u", "r0", "1", "4"]
    1: (2) ["cast_u32_I32", "r0"]
    2: (2) ["cast_i32_f32", "r0"]
```

First we upcast u8 to u32, then we cast u32 to i32, finally we cast i32 to f32.

```ts
console.log(generateCastInstruction("f64", "u16", "r0"))
(3) [Array(2), Array(4), Array(2)]
    0: (2) ["op_cast_f64_i64", "r0"]
    1: (4) ["op_dcast_i", "r0", "8", "2"]
    2: (2) ["op_cast_i16_u16", "r0"]
```

## Booleans:
Booleans are treated as `u8`. Any instruction that checks for a boolean value will check if the value is `0` or not. If it is `0`, then it is false, otherwise it is true.
```ts
mv r0, 1
mv r1, 0

cmp r0 r1
jne false_label
je true_label
```


Same logic, we cast to from float to int, then downcast to i16, then cast to u16.

While this doesn't always guarentee the best performance, it reduces the number of instructions needed to perform the cast, and ensure a good level of safety.

## Enums
Enums are very basic, the most interesting part is how they are represented in the VM. Type-C allows you to choose a specific type on how to represent the enum.

```ts
type Colors = enum as u8 {
    Red,
    Green,
    Blue
}
```

You can only an integer type, signed or un signed. By default, it is `i32`. 

## Structs
Structs has been addressed previously, but they are flexible data structure, designed to be used for grouping data together.
```ts
type Point = struct {
    x: i32,
    y: i32
}

fn printPoint(p: {y: i32, x: i32}) {
    // do stuff
}
```
When checking for type compatibility, type-c checks for structural compatibility. This means that a function call to `printPoint` with a `Point` struct will work just fine. However, if the field have different types, then it will not work.

```
printPoint({x: f32, y: f32}) // error
```

## Interfaces, Classes, Join Types and Nullables

Interfaces are the most interesting part of type-c. They are the most powerful feature of the language. Type-C is not a fully object oriented language, but it does support some object oriented features.
Type-C many drops OOP in favor of interface oriented programming. This means that classes cannot extend other classes. Instead, classes can implement interfaces. Interfaces are a set of methods that a class must implement. This is similar to the concept of interfaces in Java or Go. Interface can extend other interfaces, but not classes.

```ts
type Duck = class {
    let color: Colors

    init(color: Colors) {
        this.color = color
    }

    quack() {
        // do stuff
    }

    walk() {
        // do stuff
    }

    fly() {
        // do stuff
    }

    swim() {
        // do stuff
    }
}
```
Now we can easily create an array of birds who can fly for example,

```ts
let birds: interface { fn fly() -> void }[] = [new Duck(Colors.Red), new Duck(Colors.Green)]
```

Types are allowed to be anynoumous, and inference is automatic. 

```ts
let birdsWhoFlyAndSwin: interface { fn fly() -> void, fn swim() -> void }[] = [new Duck(Colors.Red), new Duck(Colors.Green)]
```
Would be exactly the same as 

```ts
let birdsWhoFlyAndSwin: interface { fn fly() -> void } & interface { fn swim() -> void }[] = [new Duck(Colors.Red), new Duck(Colors.Green)]
```

The inference system is smart enough to group the interfaces together. This is called a join type. A join type is a type that is a combination of two or more interfaces. This is similar to the concept of intersection types in typescript. 

Now the last type is nullable. Nullable is a type that can be null. This is similar to the concept of nullable types in typescript. TypeC supports the denull operator `!!` and the optional chaining operator `?.`. 

```ts
let maybeNullDucks: Duck?[] = [new Duck(Colors.Red), null, new Duck(Colors.Green)]

maybeNullDucks[0]!!.quack() // ok
maybeNullDucks[1]?.quack() // ok
```

## Strict Types
The last category of types is strict types. `strict` is not really a data type but rather a type modifier. It is used to indicate that a type is strict. Strict type forces the compiler to check for strict structure equality. This means that the following code will not compile:

```ts
let x: strict {y: i32, x: i32} = {x: 1, y: 2}
```

This is because the type of `x` is not structurally compatible with the value it is being assigned to. This is similar to the concept of strict types in typescript.

Strict types are useful when dealing with FFI, since you will need to ensure that the data is exactly the same as the one you are expecting.

Strict types can be used with interfaces, classes and structs. Since at its core, `strict` is a feature designed to fix the layout, as long as the layout is guarenteed, strict can be very flexible:

```ts
let x: strict interface { fly() -> void, swim() -> void } = new Duck(Colors.Red) // OK
```

The previous code will work just fine, because `fly` and `swim` are aligned in the same order in the interface and the class.

```ts
let y1: strict {x: i32, y: i32} = {y: 1 as i32, x: 6 as i32} // ERROR
let y2: strict {x: i32, y: i32} = {a: 3.14 as f32, x: 4 as i32, y: 0 as i32} // OK
```

```ts
let z1: interface { beCool() -> void } & interface { isCool() -> bool } & strict interface { isAwesome() -> bool} = ...
let y2: strict interface { beCool() -> void } = z1 // OK
```

## Conclusion
Now is your chance to criticise the design of type-c type system. What do you like? What do you dislike? What would you change? Feel free to let me know.