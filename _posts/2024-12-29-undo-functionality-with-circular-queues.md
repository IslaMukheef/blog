---
layout: post
title:  "Implementing the Undo (Ctrl + Z) Functionality with Circular Queues in C"
date:   2024-12-29 
categories: coding, low-level, c
summary: How to Implement ctrl + z for a text editor in c!
---

## Introduction
I’m working on a text editor called [Teex](https://github.com/IslaMukheef/teex). While working on the project, I wanted to share some of the implementations I did for specific features.. If you check the github source things may be different as I do change things from time to time but here i will share the general idea of how to implement undo function for a text editor.

## Approach
The idea is as the following;

As I mentioned, I recommend tracking only recent changes, as keeping a history of all changes could become inefficient. You can limit the tracked history to the last 50 or 1000 characters, depending on your use case. Deciding on this number helps prevent errors and inefficiencies. Another implementation approach can be used, I will talk about it at the end.

## Circular Queues
We gonna use circular queue to achieve our goal.


<div style="background:#fff; display:inline-block; padding:12px; border-radius:6px;">
  <img src="/assets/img/undo-func-teex/1.webp" alt="DIE detecting the file type">
</div>
*Figure 1: [javatpoint](https://www.javatpoint.com/circular-queue)-  circular-queue.*

A circular queue is an efficient data structure for managing a fixed-size buffer. It can be implemented with either an array or a linked list and allows us to track recent changes while ensuring that we don’t exceed a fixed size. If you want to understand more about this you can use this [javapoint](https://www.javatpoint.com/circular-queue) it will help you understand the topic or you can follow along.

## How it work!
You start with a front which is the head of your list or whatever the first element is at the moment. Rear points to the next available position for inserting new data. Let say you got 5 elements

[1, 2, 3, 4, 5] // front = 0, rear = 4, size = 5

now let say that you wanna push another char, the first element where front points will be freed for our use and front will move by 1 so it will keep pointing to the new first element.

[6, 2, 3, 4, 5] // front = 1, rear = 1, size = 5

now front is pointing to 1 which holds the value (2) and our rear points to 1 as well. Why rear points to 1 you may ask not 0. The answer is that rear points to the next available position after it got used. So before we said rear=4 where it was actually 0 after it added (5) to position 4.

If you’re confused, don’t worry! You’ll better understand how the circular queue works as we walk through the code.

Now before we start with the code I’m gonna first show the array based implementation then show the linked list one and explain which one may be fit for you. So what we need to have for our Queue(or stack)?

Array based Circular Queues

- char to track recent chars
- row,col the position of the char
- front, rear
- current_size track the size of the queue

## Queue Struct

```c
#define Queue_SIZE 5 // change it to what you want
typedef struct{
    char node_char[Queue_SIZE];
    int x_row[Queue_SIZE];
    int y_col[Queue_SIZE];
    int front;
    int rear;
    int current_size;
}Queue;
```

There is not much to explain here as it just declaration.

## Initiate Tracking
Now we need to initiate your tracking but you can do it in a different way i just made a function for it in case i want to add a turn on-off for this implementation.

```c
void initTracking(Queue *stack){
    stack->front = 0;
    stack->rear = 0;
    stack->current_size = 0;
}
```

Here we just initiated the trackers to 0 so we can start using them.

## Push Function
Now we need to push all the new typed chars to our Queue.

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

The function push expect the following

Queue *stack: This is a pointer to a Queue structure.(we will define stack in main later)

char new_char: just the new char that will be added to our queue

int new_x,new_y: the position of the char(where it was written in the screen)

```c
if(stack->current_size == Queue_SIZE){
        stack->front = (stack->front +1) % Queue_SIZE; // move it by one when the last elemetn is reached
    }
```

we check if the current_size of our queue has reached the limit we set for it if yes then we move the stack front pointer by one (stack->front +1 ) so we can use the old place it was pointing to. the (% Queue_SIZE) makes sure that front will reset when it reaches the last place.

ex: if front is at 4(last place) then the code will look like this

stack->front = (4 + 1) % 5; the results of this will be 0 so it will reset when it hit the limit we did set for it.

```c
else{
        stack->current_size++;
    }
```

If the the current_size did not hit our limit means we need to increment by 1 to update our size with the new char that we will add. The reason why we have it in an else statement is to not increment in all cases just when current_size is still not equal to our limit which is 5.

```c
stack->node_char[stack->rear] = new_char; // store new_char in rear current positon
stack->x_row[stack->rear] = new_x;
stack->y_col[stack->rear] = new_y;
stack->rear = (stack->rear +1) % Queue_SIZE; // move rear to next element
```

here we just char, row, col to the free place that rear points to. stack->rear is a pointer to a free space or free space.

the last line is setting rear to the next free position. same as we did for front when we moved it by one. We also do % Queue_SIZE to make sure that rear will reset to point to position 0 when it hit the last place.

## Pop Function

```c
void pop(Queue *stack){
    if(stack->current_size ==0){
        printf("Empty list \n");
        return;
    }
    int index = (stack->rear -1 + Queue_SIZE) % Queue_SIZE;
    printf("Popped: %c at (%d, %d)\n",
           stack->node_char[index],
           stack->x_row[index],
           stack->y_col[index]);

           
           stack->current_size--;
}
```

Here is our pop function where the chars will be deleted from the screen.

function expect a pointer to the stack(Queue *stack)

```c
if(stack->current_size ==0){
        printf("Empty list \n");
        return;
    }
```

here we just say we can’t pop(remove) from the queue if it is empty

```c
int index = (stack->rear -1 + Queue_SIZE) % Queue_SIZE;
printf("Popped: %c at (%d, %d)\n",
           stack->node_char[index],
           stack->x_row[index],
           stack->y_col[index]);

           
           stack->current_size--;
```

The index here helps us find the last element we want to pop. See the rear always point to the next free place so we need to get the one behind it not what it currently pointing to as we want to remove a char not add it.

let say rear is pointing to the 3 place and we want to pop the last element which is 2 below is how it gonna work.

index = (3–1 + 5) % 5
= (2 + 5) % 5
= 7 % 5
= 2

index here will hold the the last element position in our queue.

now if you look at the printf it reaches the last element using the index we just talked about for each type we had for our structure.

lastly for current_size we decrement as we poped a char from it so it should free a space(there is no freeing we just gonna ignore what is there till we enter a new char and it will be in use again).

in my project i had a function which deals with deleting chars from the screen and files so you should handle it here or call another function to remove the char from where you want. The point of this code is to store information about the new added chars so you can use whatever lib or stuff you are doing to delete it(check my project to see how i did the other stuff if you want).

## Display function
The below function is just a way to show our results you can use it to test the code

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

now our main function

```c
void main(){
 Queue stack;
    initTracking(&stack);

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
    pop(&stack);
    display(&stack);

    pop(&stack);
    display(&stack);

}
```

Queue stack; we create an instance of Queue

initTracking(&stack); initiate our tracking

the reset is just a test case for our work

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


void pop(Queue *stack){
    if(stack->current_size ==0){
        printf("Empty list \n");
        return;
    }
    int index = (stack->rear -1 + Queue_SIZE) % Queue_SIZE;
    printf("Popped: %c at (%d, %d)\n",
           stack->node_char[index],
           stack->x_row[index],
           stack->y_col[index]);

           
           stack->current_size--;
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
    initTracking(&stack);

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
    pop(&stack);
    display(&stack);

    pop(&stack);
    display(&stack);

}
```

Why Array not Linked List
No need for dynamic allocating, better indexing as you saw in pop function. Since this text editor may need to handle a large amount of input in the future, using an array ensures faster access and manipulation of elements. For these reasons, an array-based approach is a better choice for now.

Implementation of redo

<a class="ref-card" href="{% post_url 2024-12-30-redo-functionality-c %}">
  <span class="ref-label">&rarr; Related reading</span>
  <span class="ref-title">Implementing the redo(Ctrl + Y) Functionality with Circular Queues in C</span>
</a>

## Conclusion
This article demonstrated how to implement an Undo (Ctrl+Z) function using circular queues in C. This approach offers a balance of simplicity and performance.

For additional code and updates, check out the [Teex](https://github.com/IslaMukheef/teex) project repository.