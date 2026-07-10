# Let's Poly1305!

## 問題
Poly1305 の鍵の半分は分かっています。

server.py
```python
import os
from secrets import token_bytes
from Crypto.Hash.Poly1305 import Poly1305_MAC

FLAG = os.getenv("FLAG", "Alpaca{dummy}")
TARGET = b"admin=true"

# Poly1305_MAC clamps r internally
r, s = token_bytes(16), token_bytes(16)
print(f"HINT! r: {r.hex()}")

message = input("message: ").encode()
if not message or message == TARGET:
    raise SystemExit("invalid message")
print(Poly1305_MAC(r, s, message).hexdigest())

if input("tag: ") == Poly1305_MAC(r, s, TARGET).hexdigest():
    print(FLAG)
else:
    print("invalid")
```

## 解説
Poly1305 の鍵である $r,s$ のうち $r$ だけ明かされ、その後任意の平文に対する MAC 生成 1 回を経て `admin=true` に対する MAC を当てる、という流れです。

Poly1305 の[仕様](https://tex2e.github.io/rfc-translater/html/rfc8439.html#2-5--The-Poly1305-Algorithm)や[実装](https://github.com/tex2e/chacha20-poly1305/blob/master/poly1305.py#L32)を見るに、明かされていない方の $s$ は最後に足されるだけで、そこまで複雑な処理はされていないようです。つまり、 $r$ を用いて $s$ が足される前までを再現し実際の MAC と差分を取るだけで $s$ が得られます。

### Solver

Python と `pycryptodome`, `pwntools` で実装しました。なお 16 行目の `tag_dash` の定義では `s` にヌル終端文字列を指定していますが、これは数値表現での 0 に等しく、結果として `tag_dash` は $s$ を足す前までの再現となっています。

```python
from Crypto.Hash.Poly1305 import Poly1305_MAC
from pwn import *

conn = process(["python", "server.py"])
# HOST = "34.170.146.252"
# PORT = 12581
# conn = remote(HOST, PORT)

conn.recvuntil(b"r: ")
r = bytes.fromhex(conn.recvline()[:-1].decode())

MESSAGE = b"hogehoge"
conn.recvuntil(b"message: ")
conn.sendline(MESSAGE)
tag = bytes.fromhex(conn.recvline()[:-1].decode())
tag_dash = Poly1305_MAC(r, b"\x00" * 16, MESSAGE).digest()
s = (int.from_bytes(tag, "little") - int.from_bytes(tag_dash, "little")).to_bytes(
    16, "little"
)

TARGET = b"admin=true"
conn.recvuntil(b"tag: ")
conn.sendline(Poly1305_MAC(r, s, TARGET).hexdigest().encode())

flag = conn.recvline()[:-1].decode()
print(flag)
```
