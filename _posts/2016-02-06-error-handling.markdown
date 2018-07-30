---
layout: "post"
title: "Error Handling"
categories: blog
tags: ['program']
published: true
comments: true 
script: [post.js]
---
* Do not remove this line (it will not be displayed)
{: toc}

Programmers should _always_ check the error code returned by system-level functions. There are many subtle ways that things can go wrong, and it only makes sense to use the status information that the kernel provides us.

The idea is given some base system-level function _foo_, we define a wrapper function _Foo_ with identical arguments. The wrapper calls the base function and checks for errors. If it detects and error, the wrapper prints an informative message and terminates the process. Otherwise, it behaves like the base function.

Programmers could use different styles for returning errors, there lists three styles: __Unix-style__, __Posix-style__, and __DNS-style__.

# Unix-Style Error Handling

Functions such as `fork` and `wait` that were developed in early days of Unix overload the function return value with:

* useful return resutls
* error code: __errno__

This style has following form (take `wait` for example):

    if ((pid = wait(NULL)) < 0) {
        fprintf(stderr, "wait error: %s\n", strerror(errno));
        exit(0);
    }

# Posix-Style Error Handling

Most of the newer Posix functions such as Pthreads 

* use the return value to indicate success(0) or failure(1) only
* any useful results are returned in function arguments that are passed by reference

For example, `pthread_create` function indicates success or failure with its return value. And returns the ID of the newly created thread by reference in its first argument.

This style has following form (take `pthread_create` for example):

    if ((retcode = pthread_create(&tid, NULL, thread, NULL)) != 0) {
        fprintf(stderr, "pthread_create error: %s\n", sterror(retcode))
        exit(0);
    }

# DNS-Style Error Handling

The `gethostbyname` and `gethostbyaddr` functions that retrieve DNS host entires have another approch for returning errors. These functions return a NULL pointer on failre and st the global __h_errno__ variable.

This style has following form (take `gethostbyname` for example):
    
    if ((p = gethostbyname(name)) == NULL){
        fprintf(stderr, "gethostbyname error: %s\n", hstrerror(h_errno));
        exit(0);
    }

# Summary of Error-Reporting Functions

Following error-reporint funtions corresponds to the above three styles of error handling:

    void unix_error(char *msg){
        fprintf(stderr, "%s: %s\n", msg, strerror(errno));
    }

    void posix_error(int code, char *msg){
        fprintf(stderr, "%s: %s\n", msg, sterror(code));
    }

    void dns_errno(char *msg){
        fprintf(stderr, "%s: DNS error %d\n", msg, h_errno);
    }



