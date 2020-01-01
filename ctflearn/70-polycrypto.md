# [70 points] Polycrypto

We're given a link to a wikipedia article about "Finite field arithmetic" and a polynomial.

We can deduce that we're supposed to represent the polynomial as a binary string. To do that, perform these steps;

- remove `f = `
- `s/x\^//g` (remove "x^")
- replace the final "1" with "0" because that's how it's actually represented (makes more sense if you think about 1 as x^0)

Create a binary string with this script

```python3
# open file
# remove f =
# s/x\^//g
# replace the final 1 with 0

with open('./field.txt') as f:
    content = map(int, f.read().strip().split(' + '))

solution = []
print(content)

for i in range(208):
    solution += "0"

for i in content:
    solution[i] = "1"

print(''.join(solution))
```

Put it into cyberchef with the recipe "Reverse" and then "From binary".
