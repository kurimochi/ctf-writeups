# zen.zip

## 問題
醜いより美しいほうがいい。

make_zip.sh
```shell
#!/bin/sh
set -eu

cd /tmp
python -m this >zen.txt
printf '%s\n' "$FLAG" >flag.txt
zip -0 -X -q -P "$PASSWORD" zen.zip zen.txt flag.txt
cp zen.zip /output/
```

## 解説
ひとまず、`zen.zip` に含まれているであろう `zen.txt` は固定らしいです。

```
$ python -m this
The Zen of Python, by Tim Peters

Beautiful is better than ugly.
Explicit is better than implicit.
Simple is better than complex.
Complex is better than complicated.
Flat is better than nested.
Sparse is better than dense.
Readability counts.
Special cases aren't special enough to break the rules.
Although practicality beats purity.
Errors should never pass silently.
Unless explicitly silenced.
In the face of ambiguity, refuse the temptation to guess.
There should be one-- and preferably only one --obvious way to do it.
Although that way may not be obvious at first unless you're Dutch.
Now is better than never.
Although never is often better than *right* now.
If the implementation is hard to explain, it's a bad idea.
If the implementation is easy to explain, it may be a good idea.
Namespaces are one honking great idea -- let's do more of those!
```

Zip ファイルの暗号化について調査すると、Wikipedia[^1] 曰く ZipCrypto というレガシーが存在するそうです。この問題で Zip の生成に用いられている Zip 3.0 もこれを利用しています[^2]。

さて、ZipCrypto は既知平文攻撃に脆弱であることが知られています[^3]。そのため十分に長くかつ既知の `zen.txt` を足がかりに `flag.txt` を復元できないか考えます。

## Solver
ZipCrypto の既知平文攻撃には `bkcrack` というツールが提供されており、非常に簡単に再現できます[^4]。

1. 既知の平文を用意
2. 同じ圧縮オプションで Zip 化
3. `bkcrack` で復号済 Zip を取得
4. 展開 → flag 取得

```shell
python -m this > zen.txt
zip -0 -X zen-plain.zip zen.txt # Same options
bkcrack -C zen.zip -c zen.txt -P zen-plain.zip -p zen.txt -D decrypted.zip  # Attack!!
unzip decrypted.zip -d decrypted
cat decrypted/flag.txt
```


[^1]: 英語版 : https://en.wikipedia.org/wiki/ZIP_(file_format)#Encryption
[^2]: 参考 : https://unix.stackexchange.com/questions/713439/what-is-the-kind-of-encryption-used-in-zip-3-0
[^3]: https://math.ucr.edu/~mike/zipattacks.pdf
[^4]: GitHub : https://github.com/kimci86/bkcrack
