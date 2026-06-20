---
layout: post
title:  "Implementing the redo(Ctrl + Y) Functionality with Circular Queues in C"
date:   2024-12-30 
categories: coding, low-level, c
summary: This is a continuation of the previous undo implementation article for a text editor in c!
---

## Introduction
I now will talk about the redo functionality in Teex editor. If you saw this article first you should read the

<a class="ref-card" href="{% post_url 2024-12-29-undo-functionality-with-circular-queues %}">
  <span class="ref-label">&rarr; Related reading</span>
  <span class="ref-title">Implementing the Undo(Ctrl + Z) Functionality with Circular Queues in C</span>
</a>

implementation first as the redo heavily relies on it. Now we have the undo in our editor how can we redo what it did.

## Approach
Mostly everything will be the same as the undo but with some changes!

We will create a new instance of our Circular Queues name it reStack as we will push them back to the stack. Whenever pop is called we will push but for our reStack instance so we can keep track of the deleted chars with undo!

Before showing any code i want to say that in Teex things may look different, I will add the snippet of the code below as well but i recommend that you read the whole code on github!.

the undo-redo full code will be at the end of the article as well!

## Queue struct

```c
#include <stdio.h>  
#define Queue_SIZE 5

typedef struct{
    char node_char[Queue_SIZE];
    int x_row[Queue_SIZE];
    int y_col[Queue_SIZE];
    int front;
    int rear;
    int current_size;
}Queue;
```

Everything is still the same in the struct we just going to reuse it (create a new instance in main)

## Main

```c
void main(){
    Queue stack;
    Queue reStack;
    initTracking(&stack);
    initTracking(&reStack);

    // Test cases
    rest of code ...

}
```

Here we created a second instance reStack of Queue and we called..

## initTracking

```c
void initTracking(Queue *stack){
    stack->front = 0;
    stack->rear = 0;
    stack->current_size = 0;
}
```

Here we just initialize front,rear and current_size to 0 for both instances

The Queue *stack is just a pointer to whatever was passed before so when we call initTracking(&stack); the pointer will be to stack but when we call initTracking(&reStack); the pointer will be to reStack. The name here does not matter i just kept it as it was you can change it to something else nothing will change.

### Push

```c
void push(Queue *stack,char new_char, int new_x, int new_y){
    if(stack->current_size == Queue_SIZE){
        stack->front = (stack->front +1) % Queue_SIZE; // move it by one when the last elemetn is reached
    }
    else{
        stack->current_size++;
    }
    stack->node_char[stack->rear] = new_char; // store new_char in rear current positon
    stack->x_row[stack->rear] = new_x;
    stack->y_col[stack->rear] = new_y;
    stack->rear = (stack->rear +1) % Queue_SIZE; // move rear to next element

}
```

Push will stay the same we going to use it in both cases as it is to push chars to our stack or reStack.

the Queue *stack is still just a pointer to what is being passed so stack here can be whatever instance that was passed!

## Pop

```c
void pop(Queue *stack, Queue *reStack){
    if(stack->current_size ==0){
        printf("Empty list \n");
        return;
    }
    int index = (stack->rear -1 + Queue_SIZE) % Queue_SIZE;
    push(reStack,stack->node_char[index], stack->x_row[index], stack->y_col[index]);

    printf("Popped: %c at (%d, %d)\n",
           stack->node_char[index],
           stack->x_row[index],
           stack->y_col[index]);

           
    stack->current_size--;
}
```

Here in pop we need to pass the two instance when we call the function Queue *stack, Queue *reStack

where before we only passed the *stack the only change here is


`push(reStack,stack->node_char[index], stack->x_row[index], stack->y_col[index]);`

when pop something from stack instance we need to push it to the restack instance as it should track what undo did so it can redo it(reprint it) if needed.

see here we did not pass reStack as pointer(*reStack) to the push function like before, the reason is that it is already a pointer in pop function if we add the * it will be double pointer.

after we used the index here to reach the last element in the stack and get it values.

now reStack instance will have the char we will pop from stack!

we print and make the current_size of stack -1.

## Redo

```c
void redo(Queue *reStack, Queue *stack){
    if(reStack->current_size == 0){
        return;
    }
    int index = (reStack->rear - 1 + Queue_SIZE) % Queue_SIZE; // get the index of last char
    char redo_char = reStack->node_char[index];
    int redo_row = reStack->x_row[index];
    int redo_col = reStack->y_col[index];
    push(stack, redo_char, redo_row, redo_col);// push the the char that we did rewrite to the stack so we can undo it later if want
    printf("redo: %c at (%d, %d)\n",
           stack->node_char[index],
           stack->x_row[index],
           stack->y_col[index]);
    reStack->rear = index ; 
    reStack->current_size--;
}
```

This is our new function that will work like pop but for reStack so it can reprint the chars to the screen if needed!

most of the code are the same as pop function except:

```c
    char redo_char = reStack->node_char[index]; // the char we will reprint
    int redo_row = reStack->x_row[index];       // it row
    int redo_col = reStack->y_col[index];       // it col
```

instead of using them directly we created a vars for them you can use the old way it is the same but this way may help you understand it better!

```c
push(stack, redo_char, redo_row, redo_col);
```

In here we did the same as in pop we pushed the char back to stack so it can track the chars. The idea here is that even if we used redo to restore a char that undo has deleted we should push the char back to stack instance so if we wanted to delete it again we can do so!

```c
reStack->rear = index ;
```

as we said rear points to the last(according to what current_size is) free position so after calling the redo function it need to go back by one as the place behind it is freed and it should point to it.

then we subtract 1 from current_size

## Display

```c
void display(Queue *stack) {
    if (stack->current_size == 0) {
        printf("Queue is empty\n");
        return;
    }

    printf("Queue elements:\n");
    int index = stack->front;
    for (int i = 0; i < stack->current_size; i++) {
        printf("%c at (%d, %d)\n",
               stack->node_char[index],
               stack->x_row[index],
               stack->y_col[index]);
        index = (index + 1) % Queue_SIZE;
    }
}
```

nothing changed here we will just pass a pointer to what instance we want to print it elements!

## Full code

```c
#include <stdio.h>  
#define Queue_SIZE 5

typedef struct{
    char node_char[Queue_SIZE];
    int x_row[Queue_SIZE];
    int y_col[Queue_SIZE];
    int front;
    int rear;
    int current_size;
}Queue;


void initTracking(Queue *stack){
    stack->front = 0;
    stack->rear = 0;
    stack->current_size = 0;
}

void push(Queue *stack,char new_char, int new_x, int new_y){
    if(stack->current_size == Queue_SIZE){
        stack->front = (stack->front +1) % Queue_SIZE; // move it by one when the last elemetn is reached
    }
    else{
        stack->current_size++;
    }
    stack->node_char[stack->rear] = new_char; // store new_char in rear current positon
    stack->x_row[stack->rear] = new_x;
    stack->y_col[stack->rear] = new_y;
    stack->rear = (stack->rear +1) % Queue_SIZE; // move rear to next element

}


void pop(Queue *stack, Queue *reStack){
    if(stack->current_size ==0){
        printf("Empty list \n");
        return;
    }
    int index = (stack->rear -1 + Queue_SIZE) % Queue_SIZE;
    push(reStack,stack->node_char[index], stack->x_row[index], stack->y_col[index]);

    printf("Popped: %c at (%d, %d)\n",
           stack->node_char[index],
           stack->x_row[index],
           stack->y_col[index]);

           
    stack->current_size--;
}
void redo(Queue *reStack, Queue *stack){
    if(reStack->current_size == 0){
        return;
    }
    int index = (reStack->rear - 1 + Queue_SIZE) % Queue_SIZE; // get the index of last char
    char redo_char = reStack->node_char[index];
    int redo_row = reStack->x_row[index];
    int redo_col = reStack->y_col[index];
    push(stack, redo_char, redo_row, redo_col);// push the the char that we did rewrite to the stack so we can undo it later if want
    printf("redo: %c at (%d, %d)\n",
           stack->node_char[index],
           stack->x_row[index],
           stack->y_col[index]);
    reStack->rear = index ; 
    reStack->current_size--;
}
void display(Queue *stack) {
    if (stack->current_size == 0) {
        printf("Queue is empty\n");
        return;
    }

    printf("Queue elements:\n");
    int index = stack->front;
    for (int i = 0; i < stack->current_size; i++) {
        printf("%c at (%d, %d)\n",
               stack->node_char[index],
               stack->x_row[index],
               stack->y_col[index]);
        index = (index + 1) % Queue_SIZE;
    }
}

void main(){
    Queue stack;
    Queue reStack;
    initTracking(&stack);
    initTracking(&reStack);

    // Test cases
    printf("Pushing elements:\n");
    push(&stack, 'A', 1, 1);
    push(&stack, 'B', 2, 2);
    push(&stack, 'C', 3, 3);
    push(&stack, 'D', 4, 4);
    push(&stack, 'E', 5, 5);
    display(&stack);

    printf("\nPushing F (causing overwrite):\n");
    push(&stack, 'F', 6, 6); // Overwrites 'A'
    display(&stack);

    printf("\nPushing G (causing overwrite):\n");
    push(&stack, 'G', 7, 7); // Overwrites 'B'
    display(&stack);

    printf("\nPopping rear elements (LIFO):\n");
    pop(&stack, &reStack); // Undo action
    display(&stack);

    pop(&stack, &reStack); // Undo action
    display(&stack);

    // Test redo functionality
    printf("\nRedoing actions:\n");
    redo(&reStack, &stack); // Redo last undo (reprint 'B')
    display(&stack);

    redo(&reStack, &stack); // Redo last undo (reprint 'A')
    display(&stack);
}
```

Main now has more test case for our new implementation.

## The Teex implementation of undo-redo

![](/assets/img/undo-func-teex/2.webp)
*Figure 1:*

For additional code and updates, check out the [Teex](https://github.com/IslaMukheef/teex) project repository.

## Conclusion
In this article, we explored the implementation of the redo functionality (Ctrl + Y) for the Teex text editor using circular queues in C. The redo feature heavily relies on the previously implemented undo functionality, and as shown, the core logic for both operations shares similarities. We introduced a new queue instance (reStack) to store undone operations, allowing the reapplication of these actions.

However, the implementation in Teex is slightly more complex due to the need to handle operations like character deletion, reprinting, and updating the cursor position. This adds additional complexity when compared to simpler undo-redo systems.

If you’re interested in seeing this functionality in action and exploring the code in more detail, I encourage you to check out the Teex source code on GitHub. Additionally, you can observe some of the differences in the image above, which visually highlights the key aspects of the implementation.