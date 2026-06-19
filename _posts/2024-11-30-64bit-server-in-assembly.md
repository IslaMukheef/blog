---
layout: post
title:  "Building a Simple 64-Bit Server in Assembly with NASM"
date:   2022-04-06 
categories: coding, low-level
summary: How to create a simple 64 bit server with NASM assembly!
---
With the use of [NASM](https://www.nasm.us/) assembly we can create low-level TCP server that will take a message, prints it out, and exits after handling it. Before we dive in, make sure you have NASM installed. While an IDE isn’t necessary, I use [SASM](https://github.com/Dman95/SASM)(check your package manager).

## Starting

To start we first going to need to understand how sockets works and what we need to create one. If you ever tried to create one in another language you know you need to:
1.Create a socket

2. Bind to the socket

3. Listen for incoming connections

4. Accept the connection

5. Receive data

6. Print the message

7. Close the socket and exit

For each step, I’ll provide the relevant code in its section. The complete code will be shared at the end to avoid any confusion!.

## Creating a socket

Before we start this will be our base code

```nasm
section .data ; will hold  variables later on

section .bss  ; will hold a reserved space for file descriptors later on 

section .text ; holds our code
global _start  ; where our program going to start
_start:  
```

To understand how we can create a socket we need to check the man page in Linux(or online from [GNU](https://doc.guix.gnu.org/guile/latest/en/html_node/Network-Sockets-and-Communication.html))

```bash
man socket
```

you can see that the [socket](https://man7.org/linux/man-pages/man2/socket.2.html) is created as the following according to the man page

```C
int socket(int domain, int type, int protocol);
```

so we understand from this that we need first to have a domain and you can see in the man page that each part is explained and has the all the types you can use but for us we going to use the IPv4 communication domain (AF_INET).

Type here is communication semantics that we need according to the man page(scroll down a bit) It’s the protocol we going to use to establish the connection in our case we going to use the TCP (SOCK_STREAM) and

Protocol will be 0 here as the man page says that in most cases it will be the default value.

Now that we now how we can create a socket we should look at the [Linux system call table for x64](https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/) to find the system call we need to create our program.

The system call for a socket in 64 is 41 and it require us to have

RDI = int family (our domain which going to be AF_INET)

RSI = int type (our TCP protocol SOCK_STREAM)

RDX = int protocol (this going to be 0 as the man page for socket said)

we also need to reserve a space that going to hold our socket descriptor so we can use it afterward.

```nasm
section .bss  ; will hold a reserved space for file descriptors later on 
  socket_fd resq 1;

section .text ; holds our code
global _start  ; where our program going to start
_start:
  
  ;creating a socket
    mov rax, 41 ; sys call for socket
    mov rdi, 2 ; 2 for AF_INET TCP
    mov rsi, 1 ; 1 for SOCK_STREAM
    xor rdx , rdx ; set rdx to 0 for default protocal
    syscall
    mov  [socket_fd], rax; move the socekt to socket_fd
```

lets breakdown the code:

First in .bss we reserved 8 bytes of memory for socket_fd so that we can store our socket descriptor in it after we create it.

For the .text we did move 41 to rax to initiate the sys call for socket then we need to pass it arguments that it need.

as we mentioned before we need to pass the domain first so moved 2 to rdi

but why 2 you may ask and the answer to this is that operation systems has standards which it will be defined in the headers of the file for example we are creating socket so we are doing [sys/socket](https://students.mimuw.edu.pl/SO/Linux/Kod/include/linux/socket.h.html) if you check it headers it define the AF_INET as 2 we going to use this number when we use this system call and other system calls as well.

For SOCK_STREAM it was defined as 1 so we going to move 2 to rsi.

For protocol we going to use the default which is 0 so we did the xor here.

syscall just do the call for the socket then returns result in rax

we moved whatever is in rax(our socket descriptors) to the reserved 8 bytes we created

## Bind to the socket
As we did before we check the man page for [bind](https://man7.org/linux/man-pages/man2/bind.2.html)

```bash
man bind
```
and we get this

```c
int bind(int sockfd, const struct sockaddr *addr,
                socklen_t addrlen);
```
for us to do the bind we need sockfd which is our socket that we just created, we also need struct sockaddr which we don’t have yet and addrlen which we going to talk about later.

```nasm
section .data
  sockaddr_in:
        dw 2       ; address family of AF_INET
        dw 0x901f  ; port 8080 in big-endian
        dd 0        ; 4 bytes value that represent the IP address

section .text ; holds our code
global _start  ; where our program going to start
_start:

   ; bind
    mov rax, 49 ; sys call for bind
    mov rdi , [socket_fd]; move the socket fd to rdi
    lea rsi, [sockaddr_in] ; pointer to sockaddr_in
    mov rdx, 16             ; size of sockaddr_in
    syscall
```

Let us breakdown this code

For `.data` we created a struct like definition to represent a sockaddr_in structure

dw 2 represent the family of AF_INET as we mentioned before

dw 0x901f is the port number which is 8080 and is converted to Hexadecimal(0x1f90) then with big-endian we get 0x901f

dd 0 it is our IP address that the server will use to bind to (ex 127.0.0.1) but here we used 0 to say that we want to bind to all network interfaces(INADDR_ANY)

For `.text` we moved 49 to rax to call the sys call for bind you can check the call from the [table](https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/) we talk about so you can understand what this call requires.

we did dereference the socket_fd pointer to rdi (loads the value of socket_fd to rdi)

For lea rsi, [sockaddr_in] it will load the address of sockaddr_in to rsi but you may wonder why did we use lea here not mov? the answer is that I used lea when I need an address of memory location(ex to pass buffer address or the struct in our case here to the syscall). where in mov I used to get a value stored at a memory location.

For mov rdx, 16 you may ask why did we move 16 bytes to rdx if the size of sockaddr_in is dw(2) + dw(2) + dd(4) add to 8 bytes. The answer is that sockaddr_in in networking is 16 bytes as it add another 8 bytes for padding and the reason for the padding is to align the size of this with sockaddr_in6 so it ensures compatibility. Then we just do the syscall

## Listen
As we always do check the man page for [listen](https://man7.org/linux/man-pages/man2/listen.2.html) so we get this

```c
int listen(int sockfd, int backlog);
```

we need our socket_fd that we created and the backlog which “defines the maximum length to which the queue of pending connections for sockfd may grow.” according to the man page

we check our sys call table for listen and its 50.

```nasm
section .text ; holds our code
global _start  ; where our program going to start
_start:

; create listen
    mov rax, 50 ; sys call for listen
    mov rdi , [socket_fd]
    mov rsi , 5 ; backlog 
    syscall
```
First moved 50 to rax to do the sys call for listen

Moved our socket_fd to rdi as the listen require a socket to listen to

and lastly we moved 5 to rsi which is the number of clients that can try to connect at the same time before getting error after we just do the syscall

## Accept

Now that we got our listen setup we can accept the incoming connection with accept. As always we check the man page for accept to see how it works and what we need

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```
we can see that we need sockfd which is our socket fd that we created and the sockaddr structure is filled with the address of the client which we don’t really need to use here and also the addrlen which we will not use as well. Now we check our sys call table and see that the accept sys call is 43 so we do the call now.

```nasm
section .bss
    socket_fd resq 1 ; space for the socket file descriptor
    client_fd resq 1; space for the client socket 

section .text ; holds our code
global _start  ; where our program going to start
_start:

; create accept
    mov rax, 43
    mov rdi, [socket_fd] ; sockfd
    xor rsi , rsi ; sockaddr
    xor rdx, rdx ; addrlen
    syscall
    mov [client_fd], rax;
```

First we if you see added another reserved space for client_fd in .bss. you don’t need to do this part you can just pass the results of rax to r8 to store it in there but I wanted to show you that you can.

`.text` we first did the sys call by moving 43 to rax as the sys call table said.

we moved our socket descriptor`([socket_fd])` to the rdi.

we put 0 in both rsi and rdx with xor as we mentioned before because we will not use them. After we just did the syscall

the instruction after the syscall is where we say that we want to store the results that rax has from the syscall which will be the client socket. You could move it to r8 register instead of a reserved space but it’s better to do it this way.

## Receive message

Now we are at the part where the connection is established and the client can send a message to our server, so we need to receive this message now.

as always check the man page for [recv](https://man7.org/linux/man-pages/man2/recv.2.html)

```c
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
```

- sockfd is the client socket not ours(check the man page)
- buf address of the buffer(we will add it)
- len this will be the buffer length
- flags will not be used here so we can set it to 0

```nasm
section .data
  sockaddr_in:
        dw 2       ; address family of AF_INET
        dw 0x901f  ; port 8080 in big-endian
        dd 0        ; 4 bytes value that represent the IP address
 msg db 128 dup(0)   ; buffer for client message

section .text ; holds our code
global _start  ; where our program going to start
_start:

; receive message 
    mov rax, 45 ; recv from sys call
    mov rdi, [client_fd] ; use client fd
    lea rsi , [msg] ; buffer for the message
    mov rdx , 128       ; message length
    syscall
    ;null terminate
    mov rcx, rax
    mov byte [msg + rcx], 0
```

First the `.data` we created a buffer msg of 128 bytes and all initialized to 0

for `.text` we started by moving 45 to to rax for the syscall recv

then after we moved client_fd to rdi (if you used r8 you can use it here instead)

after we passed the address of the buffer to rdi

after we said that the length of the message will be 128 by moving 128 to rdx

after we did the syscall.

if you see after we got a null terminate which is \0 will say that the string end here. If you skip this part you may have problems with the received message that is way we need to add the null terminate to the end of the string.

we copy the number of bytes from rax to rcx with (mov rcx, rax).

then we write 0 at the end of the last byte of the received data with (mov byte [msg + rcx], 0)

## Print
now e have the message we can use write to print it so we look at the man page of [write](https://man7.org/linux/man-pages/man2/write.2.html) (man 2 write)

```c
ssize_t write(int fd, const void *buf, size_t count);
```

- fd here will be 1 as we want to print to stdout(to our terminal) so we will not use file or anything just straight to the terminal.
- buf will be our msg that we received
- count will be the length of our message

```nasm
section .text ; holds our code
global _start  ; where our program going to start
_start:

;print the message
     mov rax, 1 ; sys call for stdout
     mov rdi, 1; stdout 
     lea rsi, [msg] ; buffer
     mov rdx , rcx; length of message
     syscall 
```

moved 1 to rax for write sys call

moved 1 to rdi to say that we want the output to be on stdout

we passed the address of msg to rsi as our buffer

lastly we said move rcx to rdx if you remember rcx still hold the number of bytes of our message from the receive when we tried to add the null terminate and this will be our length of the message.

syscall to call write. After this point youre server is done but you need to close the socket and end the connection and exit the program which is our next section.

## Close the socket and exit
if we check the man page for [close](https://man7.org/linux/man-pages/man2/close.2.html) or the table it only require us to pass the file descriptor

```c
int close(int fd);
```
so we going to pass our socket first then the client socket after we can exit our program.

```nasm
section .text ; holds our code
global _start  ; where our program going to start
_start: 
;close sockets
    ;client socket
    mov rax, 3 ; sys call for close
    mov rdi, [client_fd]; client fd
    syscall
    ;our socket
    mov rax, 3 ; sys call for close
    mov rdi, [socket_fd]; server fd socket
    syscall
    
    ; exit
    mov rax , 60; sys call for exit
    mov rdi , 0 ; for no error
    syscall
```

If you check the syscall table we see that 3 is for sys_close so we pass 3 to rax.

then we moved [client_fd] to rdi to say that we want to close this socket

and after we did the sys call.

Now the client socket is closed we can close our socket doing the same thing but moving [socket_fd] to rdi.

lastly for exit;

here we exit the whole program with 0 for no errors.

mov rax, 60 moves 60 to rax for exit if you check the sys call table

and we moved 0 to rdi to say that there was no error after we just do the syscall

now we just assemble and link the code an run it

Full code on [GitHub](https://github.com/IslaMukheef/ServerNasm)

```bash
nasm -f elf64 server.asm -o server.o
ld server.o -o server
```
now run

```bash
./server
```

and from another terminal try to send this command

```bash
echo "Hello, Server" | nc 0 8080
```

the server will get the message prints it and exit

## Closing Thoughts

Congratulations! You’ve just built a simple TCP server in assembly language using NASM. This project not only demonstrates the fundamentals of socket programming but also gives you a deeper understanding of how low-level system calls work in Linux. While this example is basic, it lays the groundwork for more complex networked applications in assembly. Feel free to experiment, expand, and optimize your server to handle multiple clients or add error handling. Assembly programming can be challenging but incredibly rewarding, happy coding!