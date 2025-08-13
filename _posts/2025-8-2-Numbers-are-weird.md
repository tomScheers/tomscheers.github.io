---
layout: post
title: "Numbers are weird"
date: 2025-08-02
tags: [blog, tutorial, C]
---

> This article is an **indirect follow-up** of my previous [article](https://tomscheers.github.io/2025/07/29/writing-memory-efficient-structs-post.html) on how to write memory efficient structs in C, because many of the topics this article will address are direct lessons I've learnt from feedback I've gotten from you, so a big thank you for giving the feedback you've given!

One of the first things you've probably learnt when first getting into C is how to use basic integers. But since then so many different number variants and concepts have been introduced that it's hard to keep track, and some of these concepts aren't necessarily intuitive.

## The Basics
It might be smart to first look at how a basic integer actually works before getting into the weird stuff. You likely already know that computers store data in a binary format, ones and zeros, but how do our integers actually look in memory? Well say you define a number like this:
```c
int a = 32;
```
This, as you likely see, assigns the value of 32 to an integer variable called a. This is how this integer (likely, we'll get into that in a ***bit***) will look like in memory:
```
00000000 00000000 00000000 00100000
```
As you can see I've added some spaces for formatting purposes. So now our memory is neatly laid out into sets of bytes (8 bits). In this case we have 4 bytes, but this isn't actually guaranteed by the C standard. In the case of variable sizes the C standard only gives this:
```
sizeof(char) <= sizeof(short) <= sizeof(int) <= sizeof(long) <= sizeof(long long)
```
Where a `char` must be equal or greater than 1 byte, a `short` and `int` must not be less than 2 bytes, a `long` must not be less than 4 bytes and a `long long` must contain at least 8 bytes This means that, theoretically, all of the listed types above can be as large as 8 bytes. In practise these sizes are usually the same across systems but it's handy to know that just saying an int is 4 bytes is not always true. For example many 16-bit systems use integers with a 2 byte size.

### long Ambiguity
Most of the types mentioned above are generally the same, except for `long`. The difference depends mainly on the architecture of the operating system. If you run a 32-bit machine a long will generally be 4 bytes, whilst on a 64-bit machine a long is generally 8 bytes. The difference is subtle, but it's important to know about if you want to store large integers in your program for example and want it to be portable across systems. The same goes for `size_t`, since it's defined as an `unsigned long`, it differs from architecture to architecture what the size is. However for a `size_t` this makes more sense, since its primary purpose is storing data related to memory size, it doesn't make sense for it to be larger than the size of a pointer on the system.
As an example: on a 32-bit system a pointer is stored in 4 bytes of memory, so you can *only* have 4,294,967,296 unique pointers in your program. Consequently that's also the max size of a `size_t`, so if you're keeping track of the amount of values in an array on a 32-bit system, the size of that array will never exceed 4,294,967,296.

## Unsigned VS Signed
You can split all integer types up into two groups; those who are signed and those who aren't. But what exactly does it mean for an integer to be *signed*? Before getting into the memory, let's first look at what the difference is in value.
In basic terms: a signed integer can represent negative and positive values, whilst an unsigned integer can only represent positive values. This is how the range looks when comparing an unsigned 32-bit integer to a signed 32-bit integer:
Signed: -2,147,483,648 to 2,147,483,647
Unsigned: 0 to 4,294,967,295
The unsigned 32-bit integer can represent nearly double the amount of positive integers the signed 32-bit integer can, why is this.
To find out we have to look into memory. Let's go with a -32 signed integer as an example:
```
11111111 11111111 11111111 11100000
```

What? That's so much different than how 32 is stored in memory? Let's look at this with a step per step example.

The most significant bit (MSB) is the sign, so if this is 1 it's negative and if it's 0 it's positive, so in this case we're dealing with a negative number.

But this doesn't actually explain the rest of the integer. What does explain it is Two's Complement (method used for storing signed integers). This is the method for calculating any given negative number:
1. starting with the absolute binary representation of the number, with the leading bit being a sign bit;
2. inverting (or flipping) all bits – changing every 0 to 1, and every 1 to 0;
3. adding 1 to the entire inverted number, ignoring any overflow. Accounting for overflow will produce the wrong value for the result.

So using this on our -32 bit example we get:

1. We take the absolute value of -32, which is 32 and turn it into binary accounting for the signed bit; `0100000`
2. Then we invert all bits; `1011111`
3. Then, we add one to this: `1100000`

So -32 in binary is just `1100000`, and since our integer is 32-bit we just add all the zeros (in this case flipped, so all the ones), in between the MSB and the second MSB to get the result we got above.

### Comparison
Given the following code:
```c
unsigned int a = 5;
int b = -2;
if (a > b)
	printf("%d is greater than %d\n", a, b);
else
	printf("%d is less than or equal to %d\n", a, b);
```
What would you assume the output of this code to be? Well let's run it:
```bash
> clang main.c -o main
> ./main
  5 is less than or equal to -2
```
Well that's obviously not true. Why does this happen? This happens because in this case you're comparing a `signed int` to an `unsigned int`, and when you compare two variables of different integral types they're performed within a so-called "common" type, defined by so called [usual arithmetic conversion](https://rgambord.github.io/c99-doc/sections/6/3/1/8/index.html). In this case the `unsigned int` is the common type, so the `signed int` gets treated as an unsigned integer. Remember our -32 integer from before? That binary value now gets treated as an unsigned value, so instead of seeing if 5 is greater than -32 you're actually looking if 5 is greater than 4,294,967,264, since `11111111 11111111 11111111 11100000` represents -32 if signed, but 4,294,967,264 if unsigned. However this bug is completely preventable by just using some compiler flags:
```bash
> clang -Wall -Wextra -Werror main.c -o main
main.c:6:11: error: comparison of integers of different signs: 'unsigned int' and 'int' [-Werror,-Wsign-compare]
    6 |     if (a > b)
      |         ~ ^ ~
1 error generated.
```
So here's the takeaway:
> Use your compiler's warning to your advantage!

### UB or Wrap?
Another *not* so obvious behaviour of unsigned and signed integers is how they handle overflow. When an unsigned integer overflows, or underflows for that matter, it wraps around, meaning that it goes from `UINT_MAX` to `UINT_MIN`or vice versa. In the case of a signed integer it leads to undefined behaviour (UB). Meaning the behaviour is compiler-dependent and context-dependent. This also means that I made an error in my previous article. Here's the problematic part:
```c
struct Monster {
	unsigned int health; // Storing the health of a monster
}
```
My code was ***supposed*** to use the `health` to see whether a `Monster` was alive or dead by looking if health was equal or less than zero. But considering the unsigned integer's wrap behaviour there is an error I overlooked before. Say our monster has 10 health points left, and it takes 11 hit points of damage, it isn't actually stored as -1, instead it's `UINT_MAX`. This also applies to subtraction done in if statements, for example:
```c
unsigned int health = 10;
unsigned int damage = 11;
if (health - damage <= 0) {
	printf("Dead\n");
} else {
	printf("Alive\n");
}
```
The above code would return alive, even though you would assume that `10 - 11` would be less than 0.

### Asymmetry
A signed integer in C is asymmetric, meaning it doesn't have the same amount of numbers less than 0 as the amount of numbers greater than 0. Take an 8-bit integer as an example (we'll get to that later), it can represent `2^8 = 256` unique values. But since it also has to include 0 you only get 255 unique numbers which can go on either side of the zero. Hence the range looks like this: `-128 to 127`. This applies to all signed integer types. Therefore `-INT_MAX` is allowed, but `-INT_MIN` will result in an overflow.

## Even More Types
Lucky for us there are even more integral types than just int and it's unsigned version. The only real difference in use case with most of these is their size and therefore their range. The primitive integer types are: `char`,` signed char`, `short`, `int`, `long`, `long long`, and their unsigned counterparts. I've already touched on most of these above, but one thing they all have in common is that their size isn't guaranteed (except for `char`, which is always 1 byte). This can lead to differences in code behaviour across platforms if you're program relies on a certain variable being a certain amount of bytes. Here's where the `stdint.h` header file comes into play. This header defines the following types:
- `int8_t`
- `uint8_t`
- `int16_t`
- `uint16_t`
- `int32_t`
- `uint32_t`
- `int64_t`
- `uint64_t`

The good things about these types is that their size is always the same across all systems. The type naming is also very intuitive, if it has a 'u' in front of it it's an unsigned integer, otherwise it's signed, and the number in the type name represents the amount of bits the number holds. So let's say you wanted to have an unsigned 4 byte integer, you'd just use `uint32_t`, which is easier than `unsigned int` and it guarantees to actually hold 4 bytes of data.
### Chars Are Types Too!
One of the things which tripped me up as a beginner was storing number data in a `char`. The name would suggest that `char` is only used for storing character, but this isn't entirely the case. Although it is true that `char` is often used to store character data in `ASCII` format, it is essentially just another integral type capable of storing 1 byte. This small size also means that it's range isn't that large, just 0 to 255 if unsigned and -128 to 127 if signed.
One thing which is unique about  `char` compared to it's other integral relatives is that the signed modifier isn't implied, neither is the unsigned modifier. `char` on it's own can either be signed or unsigned depending on compiler and architecture. If you want to check whether `char` is signed or unsigned on your system you can do so by checking if `CHAR_MIN`, defined in `limits.h`, is less than 0 it's signed, otherwise it's unsigned. This ambiguity can make it hard to use `char` directly as an integral value. That's why if you want to use `char` to store integers you should always implicitly use either `signed` or `unsigned` depending on the context. This makes the type work like any other integral type.

## Floating-points
There are also 3 primitive floating-point types, namely `float` (4 bytes), `double` (8 bytes) and `long double`, the last of which is a little bit more tricky in size, but it's at least 10 bytes in size, but usually greater depending on architecture. These types are used for storing floating-point data (as the name suggests) like 0.1 or 3.14 for example.
Floating-point types are a little bit more tricky to store in memory than integral types because of their nature. They use a [single-precision floating-point format](https://en.wikipedia.org/wiki/Single-precision_floating-point_format), where a signed float has the same maximum value as a signed integer. All integers with 7 or fewer decimal digits, and any whole digit 2^n where n ranges from -149 to 127 can be represented accurately in a float.
In memory a 32-bit float looks as follows:
- 1 sign bit
- 8 exponent bits
- 24 fraction bits (23 of which are explicitly stored)

We're familiar with the sign bit, but what are the exponent bits and the fraction responsible for? Well this will become way more obvious once we look at the formula for calculating the value in decimal representation:
let the sign bit be S, let the exponent bit be E and let the fraction be M. We can use the following formula to calculate the decimal number:
`(-1)^S ⋅ 2 ^ (E - 127) ⋅ 1.M`
Okay, this still is a big mess, so let's clean it up a bit.
The first factor is just `(-1)^S`, so if S is 0, meaning it's positive the result will be 1 and if it's 1 it'll stay -1.
Then we get this factor: `2 ^ (E - 127)`. Let's go by order of multiplication; First we subtract 127 from the exponent, we do this because we want this number to be small.
The last part is `1.M`, this is called the mantissa. However this leaves us with a binary number, which isn't really what we want. So to convert a floating-point binary number into decimal we just do the following:
Binary: `1.01011`
Now we just go from left to right and check what the bit says, we multiply the bit by `1/(2^bit_offset)`, so this looks like this:
`1⋅1+0⋅2/1​+1⋅4/1​+0⋅8/1​+1⋅16/1​+1⋅32/1​=1+0.25+0.0625+0.03125=1.34375`
So let's test this:
```
1    10000001 1000000000000000000000
^    ^        ^
sign exponent fraction
```
Before applying our formula let's convert our binary numbers to decimal for easy of use:
- S = 1b => 1
- E = 10000001b => 129
- M = 1.1b => 1.5

So applying our formula:
- First we do `(-1)^1`, which is -1
- Then we'll do `2 ^ (129 -127)`, which is the same as `2 ^ 2`, which is just 4
- Finally we'll handle the mantissa: `1.1b`, so `1 + 1 ⋅ 1 / 2 = 1.5`
Multiplying these we'll get:
$$
-1 ⋅ 4 ⋅ 1.5 = -6
$$
> Note that for normalized values, the IEEE 754 format assumes the significand always starts with a 1 before the decimal, so it doesn't need to store it. This gives us an extra bit of precision. That's why I said that there are 24 significant precision bits but only 23 of them are explicit.

### Special Bumbers
There are also some special cases with floating-point types, here's a list of some of them:
#### ±Zero
If both the exponent and the fraction are zero, the float holds the value zero. However, you can actually have -0.0 if it's a float. This is because floating-points are mainly based upon approximations, so if an integer from -1 is approaching 0 it'll become -0.0.

#### Subnormals
Subnormals are numbers which are too tiny to be stored in the regular format but still aren't quite 0. A number is subnormal when the exponent bit is zero and the fraction is non-zero. This is how the new formula of the subnormal looks like:
`value = (-1)^S ⋅ 2^(-126) ⋅ 0.M`
The difference being that the exponent is -126, whilst if you were to go off our original formula you'd assume it'd be -127, and the mantissa is 0.M instead of 1.M. This allows our subnormal to represent numbers smaller than the smallest normalized float (`1.18 × 10^-38`).

#### Infinity
The infinity value occurs whenever the exponent is 255 and the fraction is 0. This can be the result of division by zero or an overflow. And since it's a float it can also be signed so you can also have -infinity. You can get both of these values by using another special number I've mentioned before:
```c
1 / -0.0f == -infinity
1 / 0.0f == infinity
```

#### NaN
NaN, or not a number, is the result of an invalid mathematical calculation. In memory it's represented by the exponent being 255 and the fraction being non-zero. One of the calculations which return NaN is dividing by zero:
```c
0.0f / 0.0f == NaN
```

## Conclusion
Understanding how integers and floating-points are stored and behave is fundamental knowledge if you want to write reliable and efficient programs. Ambiguities like signed VS unsigned, why primitive type sizes aren't always the same and how special numbers look like and are stored can have a big impact on how your code looks.

I hope this article gave you a little insight on how different numbers are stored in memory and demystified some subtle differences between all the number types, because there are a lot. As always I'd highly recommend you to just leverage your compiler to its fullest capacity, and to use `stdint` integer types if you want to have fixed and predictable integer sizes across all platforms.

If you found this useful or have questions, feel free to leave feedback or reach out! Your input helps me improve future posts, and I'm here to learn.

Thanks for reading and see you **int** a *bit*.
