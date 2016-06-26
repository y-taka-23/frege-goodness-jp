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
