# ドット記法の威力

オブジェクト指向言語の機能を Haskell で使うとしたら、という質問に対して、サイモン・ペイトン・ジョーンズは好んで「ドット記法が持つ威力」を引き合いに出しています。Java や Groovy のような言語では、以下のような形式でメソッドを呼び出したいとき、正しいメソッドを見つけるのに便利な IDE のサポート機能が準備されています。

```
receiver.myMethod(arg1);
```

これは、IDE は `receiver` の型を知っていて使用可能なメソッドを検索できるため、それを表示してドットの直後に入力したいものをプログラマに選択させることができるからです。

ほとんどの関数型プログラミング言語おいて、このような状況での便利さは劣ります。というもの、最初に入力するものの典型は関数名であり、IDE は選択肢を提示するために使用出来る文脈をほとんど持っていないからです。

```
myFunction receiver arg1
```

うまいことに Frege には、オブジェクト指向と関数型、両アプローチの利点を組み合わせることのできるドット記法が存在し、それを利用て強力な IDE のサポート機能が利用可能です。

```
receiver.myFunction arg1
```

Frege のドット記法の基本要素は、モジュールシステム、型宣言、そして型クラスです。それでは、これらの仕組みがどのように動作するのかを見てみましょう。

## モジュールシステム

Frege では、名前空間を表す記法として _モジュール_ を使用します。モジュールはスタティックなメソッドとフィールドのみを持つ Java のクラスであると考えることができます。

モジュールは、データ型 (型コンストラクタおよび値コンストラクタ)、型クラスおよび関数をエクスポートすることが可能です。

モジュールを使うためにはインポート文 (修飾付きインポートも可) が必要で、インポートした型と関数はプレフィックス付きで使用することができます。

Caption: 単純なモジュールのインポートと名前空間の使い方

```
import fregefx.javafx.scene.control.TextArea
-- later
TextArea.getCaretPosition inputArea
```

これがドット記法の第一の使い方です。`getCaretPosition` 関数は `TextArea` モジュール由来のものであり、Frege は関数名の検索のため、あるいは曖昧さを排除するためにドットを使用します。

この関数は引数をひとつ (`inputArea`) 取り、その型は `TextArea` です。Frege では、この引数の型が宣言されていたりあるいは推論可能であったりして、すでに判明していることもよくあります。このような場合、ドット記法は `inputArea.getCaretPosition` の形で書くことができます。以下に示したのは、[Frege FregeFX REPL](https://github.com/Dierk/frepl-gui/blob/master/client/src/main/frege/org/frege/Application.fr#L112) のソースコード中に現れる例です。

Caption: オブジェクト指向とごく近い記法

```
insert :: TextArea -> String -> IO ()
insert inputArea text = do
    pos <- inputArea.getCaretPosition
    inputArea.insertText pos text
```

この記法は読みやすく馴染みがあるというだけではなく、強力な IDE のサポート機能が活用することも可能になります。

さらに続きます。
