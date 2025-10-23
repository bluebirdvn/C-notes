# Review knowledge about C

## Group Bitwise

1. How to set a bit in a variable?
- Use shift-left operator and OR operator.
- `variable |= (1 << bit_need_set)`

- for example:
	```
	int x = 0b00000001; // x = 1
	x |= (1 << 5);// set bit 5, x = 0b00100001

	```

2. How to clear a bit in a variable?

- Use shift-left operator and AND operator.
- `variable &= ~(1 << bit_need_clear)`

- For example: 
	```
	int x = 0b00100001; // x = 33
	x &= ~(1 << 5);// clear bit 5, x = 0b00000001

	```
3. How to 