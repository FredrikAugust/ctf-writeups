# [160 points] The Most Secured Crypto-Channel

Using BB84 -- a quantum key distribution system.

We get three files, one which is the qbit positions (-, |, /, \\), one which is the modulation "Bob" used (+ or x), and finally which of the bits he got right (_ = false, v = correct).

After you figure this out, just iterate through the file and find what bits you know and add this to a string;

```python3
correct = list(open('sputnik/transmission3.txt').read())
qbits = list(open('sputnik/transmission1.txt').read())
bob = list(open('sputnik/transmission2.txt').read())

result = ""

for (i, e) in enumerate(correct):
    if e == 'v':
        if bob[i] == '+':
            if qbits[i] == '-':
                result += '0'
            else:
                result += '1'
        else:
            if qbits[i] == '/':
                result += '0'
            else:
                result += '1'

print(''.join(result))
```

After that we get a b64 string (for some reason), and we can decrypt this to get the flag.
