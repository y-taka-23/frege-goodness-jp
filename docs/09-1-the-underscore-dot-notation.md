# アンダースコア・ドット記法

前回の記事「ドット記法の威力」では、モジュールやデータ型、型クラスのスコープに属する関数をプログラマがより簡単に参照できる方法として、ドット記法の使い方について学びました。

すでに見たとおり、ドット記法には以下の用法があり、ドットの直前には、データ型が参照されるスコープを明示する (`TextArea`) か、もしくは値自体 (`inputArea`) が現れます。

Caption: スコープもしくは値がドットの直前に現れる

```
-- dot notation on scope
TextArea.getCaretPosition inputArea
-- dot notation on reference
inputArea.getCaretPosition
```

アンダースコア・ドット記法を用いることで、さらに簡潔に書くことができるようになります。

## アンダースコア・ドット記法の導入

`inputArea` の型が `TextArea` であることがわかっている、という状況を仮定しましょう。最も単純な例は、明示的にこれを宣言することです。

Caption: 明示的な宣言

```
insertionPoint :: TextArea -> JFX Int
insertionPoint inputArea = inputArea.getCaretPosition
```

これを見ると `inputArea` がやや繰り返しになっていますが、アンダースコア・ドット記法を用いることでこの冗長性を排除することができます。

Caption: アンダースコア・ドット記法による自由変数の参照

```
insertionPoint :: TextArea -> JFX Int
insertionPoint = _.getCaretPosition
```

Caption: 追記

関数定義はラムダ式に名前をつけるだけであるということを忘れないように。関数定義 `f x = x` とラムダ式による宣言 `f = \x → x` とは同じものです。

言い換えれば、アンダースコア・ドット記法は単に

Caption: ラムダ項

```
\inputArea -> inputArea.getCaretPosition
```

を

Caption: アンダースコア・ドット項

```
_.getCaretPosition
```

に置き換えるものであると言えます。

それでは、便利な使い方をもう少し見てみましょう。
