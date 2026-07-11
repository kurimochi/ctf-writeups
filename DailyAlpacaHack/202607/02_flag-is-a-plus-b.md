# Flag is A+B

## 問題
A+B を計算すればフラグがわかるよ! あれ、A と B っていくつだっけ…?

chall.py
```python
from Crypto.Util.number import bytes_to_long
import os
import random

FLAG = bytes_to_long(os.getenv("FLAG", "Alpaca{DUMMY}").encode())

A = random.randint(0, FLAG)
B = FLAG - A

assert A + B == FLAG

print(f"A or B = {A | B}")
print(f"A xor B = {A ^ B}")
```

output.txt
```
A or B = 1708520672692343497693425015709016883325158039728511260268583494549501
A xor B = 1653224853895272618878301831150773792186632776885840473347068872416893
```

## 解説
2 値 $A, B$ について、$A \lor B$ と $A \oplus B$ から $A + B$ を得る問題です。

全加算器を考えてあげると、$A \lor B$ と $A \oplus B$ から $A \land B$ を得られれば解けそうです。  
(もう少し丁寧に解説しますと、XOR は 1 + 0 = 1, 1 + 1 = 0 のように、各ビット「ごとの」加算結果は出してくれますが、1 + 1 だった場合の繰り上がりまでは考慮してくれません。逆に言えば 1 + 1 のパターンの場所、つまり AND さえ分かれば繰り上がりを出して解けるというわけです)

ここで各ビットごとの $A \lor B$ と $A \oplus B$ から考えうる $A, B$ のパターンは、

| $A \lor B$ | $A \oplus B$ | $A, B$         |
| ---------- | ------------ | -------------- |
| 0          | 0            | (0, 0)         |
| 0          | 1            | なし           |
| 1          | 0            | **(1, 1)**     |
| 1          | 1            | (0, 1), (1, 0) |

となります。$A \lor B = 1$ かつ $A \oplus B = 0$ の場合 (1, 1) が唯一表れているので、このパターンを利用します。

こうして得られた $A \land B$ もとい繰り上がりを、それぞれ上位ビットに持っていって加算すれば $A + B$ の完成です。

### Solver
Python で実装しました。なお `carry` の定義にある $(A \lor B) \land \neg(A \oplus B)$ は $A \lor B = 1$ かつ $A \oplus B = 0$ の場合のみ 1 となります。

```python
from Crypto.Util.number import long_to_bytes

a_or_b = 1708520672692343497693425015709016883325158039728511260268583494549501
a_xor_b = 1653224853895272618878301831150773792186632776885840473347068872416893

carry = a_or_b & ~a_xor_b  # A and B = (OR: 1, XOR: 0)
a_plus_b = a_xor_b + (carry << 1)
print(long_to_bytes(a_plus_b).decode())
```
