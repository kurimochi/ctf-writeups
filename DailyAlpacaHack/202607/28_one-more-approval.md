# One More Approval

## 問題
この承認は既に使用済みです。

server.py
```python
import os
from Crypto.Hash import SHA256
from Crypto.PublicKey import ECC
from Crypto.Signature import DSS


MESSAGE = b"action=read_flag"

key = ECC.generate(curve="P-256")
signature = DSS.new(key, "fips-186-3").sign(SHA256.new(MESSAGE))
used = {signature}

print(signature.hex())

try:
    candidate = bytes.fromhex(input("signature: "))
    if candidate in used:
        raise ValueError
    DSS.new(key.public_key(), "fips-186-3").verify(SHA256.new(MESSAGE), candidate)
    print(os.getenv("FLAG", "Alpaca{dummy}"))
except ValueError:
    print("invalid signature")
```

## 解説
ECDSA ( 楕円曲線署名 ) について、署名 1 つから異なる有効な署名を生成しなさいという問題です。

ECDSA は「可塑性」という、署名 $(r,s)$ が有効な場合 $(r, -s \bmod n)$ ( ただし $n$ は ECDSA のパラメータ ) も有効になってしまうという特性を持ちます。これを利用すれば公開鍵も秘密鍵もなしに別の値の署名を生成できます。

### Solver

**出力を Hex にするのをお忘れずに！！** ( 1 敗 )

```python
from pwn import *

N = 0xFFFFFFFF00000000FFFFFFFFFFFFFFFFBCE6FAADA7179E84F3B9CAC2FC632551

HOST = "34.170.146.252"
PORT = 26931
conn = remote(HOST, PORT)

sig = bytearray.fromhex(conn.recvline()[:-1])

s = int.from_bytes(sig[32:])
s = N - s

sig[32:] = int.to_bytes(s, 32)
conn.recvuntil(b"signature: ")
conn.sendline(sig.hex().encode())

flag = conn.recvline().decode()[:-1]
print(flag)
```

---

## おまけ

### 数学的な解説
以下、ECDSA が可塑性を満たす、つまり有効な署名 $(r,s)$ に対し $(r,-s \bmod n)$ も有効であることを証明します。( 便宜上 $\mod n$ を省略 )  
なお、$Q_A$ を公開鍵とします。

ECDSA の署名検証では
$$
R = u_1 \ast G + u_2 \ast Q_A
$$
ただし
$$
\begin{align*}
u_1 &= zs^{-1} \\
u_2 &= rs^{-1}
\end{align*}
$$
を計算します。

ここで $s' = -s$ とすると、
$$
(s')^{-1} = -s^{-1}
$$
したがって
$$
\begin{align*}
u_1' &= -u_1 \\
u_2' &= -u_2 \\
R' &= - u_1 \ast G - u_2 \ast Q_A = -(u_1 \ast G + u_2 \ast Q_A) = -R
\end{align*}
$$

逆元の定義より $-R = (x_R, -y_R)$ なので、
$$
x_R = x_{R'}
$$

ECDSA の検証では $r \equiv x_R \pmod n$ のみを確認するため、$(r,s)$ が正当な署名なら $(r,-s)$ も有効となります。

よって、ECDSA は可塑性を満たします。

### 影響・対策
ECDSA の可塑性はときにブロックチェーンにも影響を与えます。
* リプレイ攻撃 : 他人の署名から可塑性によって別の署名を生成し、他人になりすまして不正に取引を成立させる攻撃
* トランザクション可塑性攻撃 : 署名を可塑性で異なるものにすり替え、相手の想定しないトランザクション ID に変化させる攻撃[^1][^2]

対策として、有効な $s$ を低い方に絞る Low-S ルールや、可塑性を考慮したシステム設計[^3]、あるいは元から可塑性のない EdDSA や Schnorr 署名などへの移行が挙げられます。

[^1]: Mt.Gox 事件など、過去にはビットコインもその影響を受けている
[^2]: 参考: https://note.com/standenglish/n/nb6829de5ccd7
[^3]: トランザクション可塑性攻撃に対して、ビットコインでは署名をトランザクションデータから省く SegWit という改良が施されている