- def \_\_hash\_\_():
		return int
	
	
	1.use hash() return  a hash code , but results are different between session , not stable
	2.use md5().hexdigest() return unique str type mean 0x format and can convert to int by
```python
import hashlib
hash_code = int(hashlib.md5(input).hexdigest,16)
```