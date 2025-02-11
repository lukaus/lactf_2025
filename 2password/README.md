# [pwn/2password](https://github.com/uclaacm/lactf-archive/tree/main/2025/pwn/2password) - by kaiphait

## The Challenge

Hint: 2Password > 1password

Provided files:
- [chall](https://github.com/uclaacm/lactf-archive/blob/main/2025/pwn/2password/chall) - executable with which we will interface
- [chall.c](https://github.com/uclaacm/lactf-archive/blob/main/2025/pwn/2password/chall.c) - source code for executable.
- [libc.so.6](https://github.com/uclaacm/lactf-archive/blob/main/2025/pwn/2password/libc.so.6) - glibc library from server.
- [ld-linux-x86-64.so.2](https://github.com/uclaacm/lactf-archive/blob/main/2025/pwn/2password/ld-linux-x86-64.so.2) - linux linker from server.
- [Dockerfile](https://github.com/uclaacm/lactf-archive/blob/main/2025/pwn/2password/Dockerfile) - configuration file for Docker container in which the chall .executable is running on the server 

We are also given the address to a socket with which to interface with the server -- where the actual flag file is stored.

## Starting Point - Exploration
Taking a look at [chall.c](https://github.com/uclaacm/lactf-archive/blob/main/2025/pwn/2password/chall.c):

Lines 18 - 26 read a "username", a first password, "password1", and a second password, "password2":
```c
...
  printf("Enter username: ");
  char username[42];
  readline(username, sizeof username, stdin);
  printf("Enter password1: ");
  char password1[42];
  readline(password1, sizeof password1, stdin);
  printf("Enter password2: ");
  char password2[42];
  readline(password2, sizeof password2, stdin);
...
```
<br>

Next, lines 27 - 33 read the flag we need from "flag.txt" -- notably, the flag is loaded before the username and passwords are checked:
```c
...
  FILE *flag_file = fopen("flag.txt", "r");
  if (!flag_file) {
    puts("can't open flag");
    exit(1);
  }
  char flag[42];
  readline(flag, sizeof flag, flag_file);
...
```
<br>

Finally, lines 34-43 check the username and passwords, and if correct, the flag is printed. Otherwise, an error message is given.
```c
...
  if (strcmp(username, "kaiphait") == 0 &&
      strcmp(password1, "correct horse battery staple") == 0 &&
      strcmp(password2, flag) == 0) {
    puts("Access granted");
  } else {
    printf("Incorrect password for user ");
    printf(username);
    printf("\n");
  }
}
```
<br>
Ah, so that's why flag.txt is loaded before the comparison -- the flag will only be printed if the value of "password2"... is the flag!
We can also see the hard-coded values for username and password1 that must be entered for the flag to be printed. But maybe we don't need to actually enter the correct username and passwords at all

## The Solve - Format Specifiers
An important observation for this challenge is that when the incorrect credentials are entered, the username the user enters is printed without any validation:


