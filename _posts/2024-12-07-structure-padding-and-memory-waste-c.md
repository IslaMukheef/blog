---
layout: post
title:  "Structure padding and memory waste in C"
date:   2024-12-07 
categories: coding, low-level, c
summary: Explaing what is structured padding and why compilers add paddings!
---

## What is Structure padding ?
It’s when the compiler adds empty byte/s between the members of the structure to align them in memory. The point of this alignment is to improve the CPU access speed.

## Why dose padding matter?
Padding wastes memory. If your code involves handling a large number of structures, you probably going to waste a lot of memory because of it. I would say in normal cases where you have small amount of structures you the memory waste would not be that big of problem but it’s still important to understand padding to optimize your code when necessary.

For example

```c
#include <stdio.h>  

// first struct
struct first{
  int myInt; // 4 bytes
  char myChar1; //1 byte
  char myChar2; // 1 byte
};
//second struct
struct  second{
  char myChar1;//1 byte 
  int myInt; // 4 bytes
  char myChar2;//1 byte
};

int main(){
  struct first struc1;
  struct second struc2;
  printf("%zu\n",sizeof(struc1));
  printf("%zu\n", sizeof(struc2));
  return 0;
}
```

`Guess the Output`

If we calculate the size of each structure manually:

- int is 4 bytes.
- char is 1 byte.
for struct first, you might expect the size to be: 4 (int) + 1 (char) + 1 (char) = 6 bytes. However, due to padding, the actual output is:

`8`
`12`

### Why Does This Happen?
If we look the memory layout for each structure it will help us understand it more.

```c
//struct first layout

myInt  : [0][1][2][3]  (4 bytes for the int)
myChar1: [4]           (1 byte for the char)
myChar2: [5]           (1 byte for the char)
Padding: [6][7]        (2 bytes of padding to align the struct to 4 bytes boundary)
```
Total size is 8 due to the padding that was added.

you see here there was only two bytes padding to align the struct to a 4-byte boundary.

but if we look at the struc2

```c
//struct first layout

myChar1: [0]           (1 byte for the char)
Padding: [1][2][3]     (3 bytes of padding to align to myint)
myInt  : [4][5][6][7]  (4 bytes for the int)
myChar2: [8]           (1 byte for the char)
Padding: [9][10][11]   (3 bytes of padding to align the struct to 4 bytes boundary)
```
Total size is 12 due to the padding that was added.

## Conclusion
The order of members in a struct significantly affects padding and memory usage.

To reduce padding, order the members by size, from largest to smallest.

Padding ensures that members are aligned according to their size requirements, but it can lead to wasted memory if not optimized.