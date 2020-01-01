# XOR is friend not food

We're given this string:

```
\t\x1b\x11\x00\x16\x0b\x1d\x19\x17\x0b\x05\x1d(\x05\x005\x1b\x1f\t,\r\x00\x18\x1c\x0e
```

And a crib.

From there, create a python script to bruteforce the key;

```python3
file_content = ""

with open("./input") as f:
    file_content = f.read().split(' ')

print(file_content)

content = [int(x, 16) for x in file_content]

print(content)

crib = 'ctflearn{'
solution = ''

for i in range(len(crib)):
    for c in range(0x20, 0x7f):  # printable ASCII
        if content[i] ^ c == ord(crib[i]):
            solution += chr(c)
            break

print(solution)
```

We now get a key, which we can use on [cyberchef](https://gchq.github.io/CyberChef/) to perform the XOR.
