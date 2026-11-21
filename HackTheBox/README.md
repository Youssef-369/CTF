# Hack The Box — Impossible Password

![Platform](https://img.shields.io/badge/Platform-Hack%20The%20Box-green)
![Category](https://img.shields.io/badge/Category-Reverse%20Engineering-blue)
![Architecture](https://img.shields.io/badge/Architecture-x86--64-lightgrey)

**Challenge:** Impossible Password  
**Platform:** Hack The Box  
**Category:** Reverse Engineering  
**Binary:** `impossible_password.bin`  
**Architecture:** ELF 64-bit, x86-64

> **Spoiler warning:** This write-up contains the solution and flag.

## Table of Contents

- [1. Overview](#1-overview)
- [2. Identify the Executable](#2-identify-the-executable)
- [3. Binary Security Protections](#3-binary-security-protections)
- [4. Search for Interesting Strings](#4-search-for-interesting-strings)
- [5. Inspect the .rodata Section](#5-inspect-the-rodata-section)
- [6. Analyze the First Password Check](#6-analyze-the-first-password-check)
- [7. Analyze the Dynamic Password Generator](#7-analyze-the-dynamic-password-generator)
- [8. Recover the Flag Through XOR](#8-recover-the-flag-through-xor)
- [9. Flag](#9-flag)
- [10. Key Takeaways](#10-key-takeaways)

## 1. Overview

The challenge presents a password-protected Linux executable. The objective is to reverse-engineer its validation logic and recover the hidden flag.

I used static analysis to inspect the ELF executable and its embedded data, followed by dynamic analysis with GDB to confirm the first password comparison. Finally, I reversed a byte-wise XOR operation to recover the flag without guessing the dynamically generated second password.

### Tools Used

- Kali Linux
- `file`, `readelf`, `checksec`, `strings`
- `objdump`, GDB
- Python 3

## 2. Identify the Executable

First, I checked the file type:

```bash
file impossible_password.bin
```

The output identifies the target as a dynamically linked, stripped 64-bit ELF executable for x86-64 Linux.

### Screenshot 1 — File identification

![Screenshot 1 — File identification](screenshots/01-file-identification.png)

Next, I inspected the ELF header:

```bash
readelf -h impossible_password.bin
```

Important information:

- Class: ELF64
- Data: Little-endian
- Type: `EXEC`
- Machine: AMD x86-64
- Entry point: `0x4006a0`

### Screenshot 2 — ELF header

![Screenshot 2 — ELF header](screenshots/02-elf-header.png)

## 3. Binary Security Protections

I used `checksec` to inspect the executable's security protections:

```bash
checksec --file=impossible_password.bin
```

The results were:

| Protection | Result |
|---|---|
| RELRO | Partial RELRO |
| Stack Canary | Not found |
| NX | Enabled |
| PIE | Disabled |
| Symbols | No symbols |

The executable is non-PIE, meaning its code addresses are fixed for this binary. The lack of symbols also means function names are not available in the normal disassembly.

### Screenshot 3 — Checksec output

![Screenshot 3 — Checksec output](screenshots/03-checksec.png)

## 4. Search for Interesting Strings

I used `strings` to identify potentially interesting strings embedded in the executable:

```bash
strings impossible_password.bin
```

Some interesting results included:

```text
SuperSeKretKey
time
srand
rand
malloc
strcmp
scanf
printf
```

These strings suggest that the program reads user input, compares strings, and uses a pseudo-random number generator.

### Screenshot 4 — Interesting strings

![Screenshot 4 — Interesting strings](screenshots/04-strings.png)

## 5. Inspect the .rodata Section

To inspect the read-only data section, I executed:

```bash
objdump -s -j .rodata impossible_password.bin
```

At address `0x400a70`, the data contains:

```text
SuperSeKretKey
```

The section also contains the input format string `%20s`, which limits each input read to 20 characters.

### Screenshot 5 — .rodata section

![Screenshot 5 — .rodata section](screenshots/05-rodata.png)

## 6. Analyze the First Password Check

The executable is stripped, so I inspected its disassembly to locate the first call to `strcmp`.

The relevant instructions are:

```asm
mov    rdx,QWORD PTR [rbp-0x8]
lea    rax,[rbp-0x20]
mov    rsi,rdx
mov    rdi,rax
call   strcmp@plt
```

On Linux x86-64, the first two function arguments are passed through `RDI` and `RSI`.

Therefore, the comparison is equivalent to:

```c
strcmp(user_input, "SuperSeKretKey");
```

This confirms that the first password is:

```text
SuperSeKretKey
```

### Confirming with GDB

I started GDB:

```bash
gdb -q ./impossible_password.bin
```

Then I set a breakpoint immediately before the `strcmp` call:

```gdb
break *0x40090d
run
```

When the breakpoint was reached, I inspected the second argument:

```gdb
x/s $rsi
```

Output:

```text
0x400a70: "SuperSeKretKey"
```

This confirms the first password directly from the program's memory.

### Screenshot 6 — GDB password confirmation

![Screenshot 6 — GDB password confirmation](screenshots/06-gdb-first-password.png)

## 7. Analyze the Dynamic Password Generator

After passing the first password check, the program requests a second password.

I disassembled the function around address `0x40078d`:

```bash
objdump -d -M intel \
  --start-address=0x40078d \
  --stop-address=0x40085d \
  impossible_password.bin
```

The disassembly revealed the following operations:

1. The function receives the requested password length.
2. It calls `time(NULL)`.
3. It increments a global counter.
4. It combines the current time, password length, and counter to generate a seed.
5. It initializes the pseudo-random number generator using `srand()`.
6. It allocates memory for the generated password.
7. It uses `rand()` to generate printable characters.
8. It appends a null terminator.

The relevant logic can be reconstructed approximately as:

```c
char *generate(int length)
{
    int max = 126;
    int min = 33;

    time_t t = time(NULL);
    global_counter++;

    srand((unsigned int)(t * length + global_counter));

    char *password = malloc(length + 1);

    if (password == NULL)
        exit(1);

    for (int i = 0; i < length; i++)
        password[i] = min + (rand() % (max + 1 - min));

    password[length] = '\0';

    return password;
}
```

The program calls this generator with a length of 20. Therefore, the second password is generated dynamically and depends on the execution time and global counter.

### Screenshot 7 — Password generator disassembly

![Screenshot 7 — Password generator disassembly](screenshots/07-password-generator-disassembly.png)

## 8. Recover the Flag Through XOR

The main routine contains a hardcoded 20-byte buffer:

```text
A]Kr=9k0=0o0;k1?k81t
```

After the second password comparison succeeds, the program passes this buffer to a function located at `0x400978`.

By inspecting the function's disassembly, I identified a byte-wise XOR operation using the constant `0x09`. Each byte is XORed with `0x09` before being printed.

The decoding operation is:

```c
decoded_byte = encoded_byte ^ 0x09;
```

Instead of guessing the dynamically generated second password, I reproduced the XOR operation using Python:

```bash
python3 -c 's="A]Kr=9k0=0o0;k1?k81t"; print("".join(chr(ord(c)^0x09) for c in s))'
```

The decoded output is:

```text
HTB{40b949f92b86b18}
```

### Screenshot 8 — XOR flag recovery

![Screenshot 8 — XOR flag recovery](screenshots/08-xor-flag-recovery.png)

## 9. Flag

```text
HTB{40b949f92b86b18}
```

## 10. Key Takeaways

This challenge helped me practice several reverse-engineering techniques:

- Identifying ELF executable properties using `file` and `readelf`.
- Inspecting binary security protections using `checksec`.
- Searching for interesting strings using `strings`.
- Analyzing the `.rodata` section using `objdump`.
- Understanding the x86-64 calling convention.
- Using GDB to inspect function arguments and memory.
- Reconstructing a pseudo-random password-generation algorithm.
- Reversing XOR-based encoding to recover hidden data.

### Final Thoughts

This challenge demonstrates the importance of understanding program logic rather than blindly guessing passwords.

By combining static analysis, dynamic analysis, assembly inspection, and Python scripting, I was able to recover the flag directly from the hardcoded encoded buffer.

> **Note:** The flag was recovered through static analysis of the XOR decoding routine. The dynamically generated second password was not recovered, and the complete two-password execution path was not demonstrated.
