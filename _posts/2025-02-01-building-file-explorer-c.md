---
layout: post
title:  "Building a Linux-Style File Explorer: Recreating the ‘ls’ Command in C"
date:   2025-02-01 
categories: coding, low-level, c
summary: A Step-by-Step Guide to coding a Linux-Inspired File Lister in c!
---

## Introduction
In this article I will try to explain how to use system calls to create a tool that can do ls command in linux. The article starts with explaining how to traverse files in a directories and moves after to doing system calls to get more details about the files. You can see in the picture below how the final results going to look like

1- no arguments: prints the files

2- with the -l argument it will print the long formatted version(this will cover most of how ls works).

![](/assets/img/building-file-explorer/1.webp)
*Figure 1:*

We will need the following

```c
#include <stdio.h> // for I/O

#include <dirent.h> // Traversing directories and reading their contents.

#include <string.h> // handle strings

#include <stdlib.h> // memory management

#include <sys/stat.h> // Retrieving info about the files

#include <sys/types.h> // needed for types that we will use

#include <time.h> // to format time

#include <pwd.h> // get user info

#include <grp.h> // get group info

```

i will not explain the ones that you use mostly but i will focus on the ones that we will need to traverse files and do the sys call etx.

## Main function

```c
#include <stdio.h>

int main(int argc, char *argv[]){
    char * path = ".";
    int long_format = 0; // flag to check if -l is provided
    if(argc > 1){
        if (strcmp(argv[1], "-l") == 0) { // check if the argument is -l
            long_format = 1; // set the flag to true
        }
    }
     traversfiles(path, long_format); // call traversfiles with the appropriate format

    
    return 0;
}
```

main function starts by declaring `path` as “.” which tell our current dir. This could be used to tell which directory to look into.

the `long_format` is a flag set to tell the app later if there was an argument passed or not.

as you see in the if statement it checks if there was an argument and if so it checks the content of `argv[1]` if it is the same as `“-l”` if the argument was -l then we set the `long_format` flag to 1. The flag here will be used to tell our app that it should print the longer version not just file names.

We call `traversfiles` function and pass the path and the `long_format` flag to it.

## Traversing files with dirent

```c
#include <stdio.h>
#include <dirent.h>
#include <string.h>
#include <stdlib.h>
#include <sys/stat.h>

int files_found = 0; // tracks the numbers of files
char *files_names[30]; //keeps track of at most 30 files!
struct stat buffer;


void traversfiles(char *path, int long_format){
    // search the dir for files and then print them
    struct dirent *dir_entry;
    DIR *dir = opendir(path);

    if(dir == NULL) // if dir return null break and return to main
    {
        printf("Dir returned NULL");
        return;
    }
    
    while((dir_entry=readdir(dir))!=NULL)
    {
        files_names[files_found] = strdup(dir_entry->d_name); // give a copy of string(it should be freed after)
        if (lstat(files_names[files_found], &buffer) == -1) {
            perror("lstat");
            continue;
        }
        //printf("%s: %lu\n", dir_entry->d_name, buffer.st_ino);
        if(long_format){
            Long_print(dir_entry->d_name);
        }
        files_found++;
        if(files_found>=30) break; // make sure it never pass 30 but it should be changed to something better!
    }
    closedir(dir);// close the directory

    if(!long_format){
        printfiles();
    }

    //free the space we allocated for file_names with strdup
    for(int i=0; i<files_found;i++){
        free(files_names[i]);
    } 

}

int main(...){
  ....
}
```

You can see that we added new libs to our code, these ones will be used to traverse files, handle strings, memory management , and the stat is the sys call we will use to get info on the files.

We declared 3 vars the first one `files_found` will help us keep track of the number of files that we did find.

`files_name` is an array that will hold at most 30 files(it is good practice to not write hard coded numbers)

`buffer` here is a struct. the man page of stat says that the lstat function wil return information about a file in the buffer pointed to by statbuf(buffer in our case). We will use it to store info about the files.
`man page ex:` int lstat(const char *pathname, struct stat *statbuf);

`traversfiles` function expect two argument as we set it in main.

these two:

struct dirent *dir_entry;

DIR *dir = opendir(path);

the first line returns a pointer to a structure representing the directory entry. The second line opens a directory stream to our path.

the if statement below it checks wither the dir stream is NULL or not if NULL return to main.

the WHile loop here` dir_entry=readdir(dir))!=NULL` loops through the whole dir until it is NULL.

`files_names[files_found] = strdup(dir_entry->d_name);` this line creates a copy of the file name that was found and it saves it in file names array(we will need to free this array later).

`if (lstat(files_names[files_found], &buffer) == -1)` this if statement makes sure that there was no error when using lstat to retrieve information about the file name and it will also store it in buffer.

`if(long_format)` In this part we check if our flag was set or not

`Long_print(dir_entry->d_name); `If the flag was 1 then we call the function Long_print and pass the file name to it`(dir_entry->d_name` returns the name of `current file).`

`files_found++; ` add to the tracker that there was a file found.

`if(files_found>=30)` break; this help us avoid overflow.

now the while loop is over we should close the directory stream with closedir(dir);

then we check if the `long_format` was 0 if so we call printfiles which will be just the file names.

after we need to free the space we allocated using strdup early when we copied the file names to our array

```c
for(int i=0; i<files_found;i++){

free(files_names[i]);

}
```

## Printing only the files name
if the `lonag_format` is 0 then the function `printfiles()` will be called

```c
.......
int files_found = 0; 
char *files_names[30]; 

void printfiles(){
    for( int i =0; i <files_found; i++)
    {
        printf("%s\n", files_names[i]);
    }
}

void traversfiles(...){
  .........
```

in this function we just loop through the files we found and we print them.

There is no more to it just loop and print.

## Printing the longer version

```c
........
struct stat buffer;

void Long_print(char file[30]){

    printf("%3o ",buffer.st_mode & 0777); 
    getUser();
    //printf("%d ", buffer.st_uid);         // prints the user id(owner)
    getGroup();
    //printf("%d ", buffer.st_gid);         // prints the group ID id(id of file group's)
    printf("%ld ", buffer.st_size);       // prints the size of the file
    convert_time(); // call function to convert the time and print
    //printf("%ld", buffer.st_mtime);
    printf("%s ", file);
    printf("\n");
}

................
.......
```

our goal in the function to print

`permissions , owner, group, size, last modification data, file name`

our function expect a string which will be our file name.

the first printf in our function will print the permissions so the type here is %3o which unsigned octal integer also will only print 3 places which is the permissions. The buffer.`st_mode` & 0777 here is getting the mode from buffer and it does a bitwise AND operation to isolates the file permissions from the other information stored in `st_mode`.

`getUser();` is a small function

```c
#include <pwd.h>
#include <sys/types.h>
void getUser(){
    struct passwd *pws;
    pws = getpwuid(buffer.st_uid);
    printf("%s ", pws->pw_name);
}
```

this function create pws pointer that will be used to retrieve information about the user from the pwd lib which it hold information about the user.

using `buffer.st_uid` we only get the ID of the user but we need to find their names so using pwd lib we can do that. `getpwuid` is pwd function which give us information about the user from their ID. now with our pws pointer we can access the pw_name(you can access more information about the user but in our case we only care about the name). After we just print the name as a string.

our in Long_print you can just use `printf(“%d “, buffer.st_uid);` instead to print the user ID only not the name

now back to our Long_print we now have

`getGroup();` function

```c
#include <grp.h>
#include <sys/types.h>
void getGroup(){
    struct group *gr;
    gr = getgrgid(buffer.st_gid);
    printf("%s ", gr->gr_name);
}
```

This function works the same as getuser it only uses grp lib here to get information on the file’s group from their ID. Same logic here we just use different function from grp (getgrid) to get information about the group from their ID and then print it.

back to Long_print function, now we have
`printf(“%ld “, buffer.st_size);` We use st_size to get the files size from buffer and then print it.

`convert_time()` function will convert the time for us to human readable.

```c
#include <time.h>
#include <sys/types.h>
void convert_time(){
    char mtime[80];
    time_t t = buffer.st_mtime; // hold the time in time_t type
    struct tm lt;
    localtime_r(&t, &lt); // convert to struct tm
    strftime(mtime, sizeof mtime, "%a, %d %b %Y %T", &lt);
    printf("%s ", mtime);
}
```

This function convert a time_t value to to local time value for humans to read then prints the time.

you can get the time from
`printf(“%ld”, buffer.st_mtime);`

but it’s not human readable so we need to covert it.

lastly we print the file and printf endline to move to the next line

`printf(“%s “, file);`

`printf(“\n”);`

## Conclusion
In this article, we’ve built a Linux-style file explorer in C, replicating the functionality of the ls command. By leveraging system calls and libraries like dirent, stat, and pwd, we’ve created a tool that can list files and display detailed file information. This project is a great way to understand how Linux commands work under the hood and how to interact with the file system in C.

This will be good start but you can extend this project by adding more features, such as sorting files, filtering by file type, or supporting additional flags.

Final code:[Github](https://github.com/IslaMukheef/LF) project repository.

The code on github may be different as i will keep working on this project there.

```c
#include <stdio.h>
#include <dirent.h>
#include <string.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <time.h>
#include <pwd.h>
#include <grp.h>



int files_found = 0; // tracks the numbers of files
char *files_names[30]; //keeps track of at most 30 files!
struct stat buffer;

void printfiles(){
    for( int i =0; i <files_found; i++)
    {
        printf("%s\n", files_names[i]);
    }
}
void getUser(){
    struct passwd *pws;
    pws = getpwuid(buffer.st_uid);
    printf("%s ", pws->pw_name);
}
void getGroup(){
    struct group *gr;
    gr = getgrgid(buffer.st_gid);
    printf("%s ", gr->gr_name);
}
void convert_time(){
    char mtime[80];
    time_t t = buffer.st_mtime; // hold the time in time_t type
    struct tm lt;
    localtime_r(&t, &lt); // convert to struct tm
    strftime(mtime, sizeof mtime, "%a, %d %b %Y %T", &lt);
    printf("%s ", mtime);
}
//Long format. Displays detailed file information, including permissions, ownership, size, and modification time.
void Long_print(char file[30]){
    printf("%3o ",buffer.st_mode & 0777); 
    getUser();
    //printf("%d ", buffer.st_uid);         // prints the user id(owner)
    getGroup();
    //printf("%d ", buffer.st_gid);         // prints the group ID id(id of file group's)
    printf("%ld ", buffer.st_size);       // prints the size of the file
    convert_time(); // call function to convert the time and print
    //printf("%ld", buffer.st_mtime);
    printf("%s ", file);
    printf("\n");
}
void traversfiles(char *path, int long_format){
    // search the dir for files and then print them
    struct dirent *dir_entry;
    DIR *dir = opendir(path);

    if(dir == NULL) // if dir return null break and return to main
    {
        printf("Dir returned NULL");
        return;
    }
    
    while((dir_entry=readdir(dir))!=NULL)
    {
        files_names[files_found] = strdup(dir_entry->d_name); // give a copy of string(it should be freed after)
        if (lstat(files_names[files_found], &buffer) == -1) {
            perror("lstat");
            continue;
        }
        if(long_format){
            Long_print(dir_entry->d_name);
        }
        files_found++;
        if(files_found>=30) break; // make sure it never pass 30 but it should be changed to something better!
    }
    closedir(dir);// close the directory

    if(!long_format){
        printfiles();
    }

    //free the space we allocated for file_names with strdup
    for(int i=0; i<files_found;i++){
        free(files_names[i]);
    } 

}


int main(int argc, char *argv[]){
    char * path = ".";
    int long_format = 0; // flag to check if -l is provided
    if(argc > 1){
        if (strcmp(argv[1], "-l") == 0) { // check if the argument is -l
            long_format = 1; // set the flag to true
        }
    }
     traversfiles(path, long_format); // call traversfiles with the appropriate format

    
    return 0;
}
```