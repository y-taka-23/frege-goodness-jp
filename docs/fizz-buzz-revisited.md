# FizzBuzz 問題ふたたび

前回実装した FizzBuzz の解法はそれ自体すでに、関数型設計の利点について面白い視点が得られたという意味で、非常に有意義なものでした。

しかし、まだ改善の余地もあります。実装に現れている部品のいくつかは「場当たり的な何か」のように見えますし、望ましい結果を得る手段を「慎重に選択する」ことができるかもしれません。そこで、もっと型主導のアプローチを用いて FizzBuzz 問題にもう一度立ち戻り、以下のようなポイントを見てみましょう。

* 解法の背後にある本質的な考え方
* 抽象化することの価値
* 目的にあった手段の選定

書き直すことで解法が劇的に変わるわけではありませんが、設計の意図がいくらか明確になり、応用の際にもっと使い回しが効くようになります。

## ポイントその 1：FizzBuzz パターン

前回のバージョンではビジネスルールの記述において、何もしないことを表現するために空字列 `""` を採用しました。そしてこれがうまく機能するのは、空文字列を他の文字列と (左右問わず) 連結しても元の文字列と同じものになるからです。

Caption: 空文字列による連結

```
"fizz" ++ ""     == "fizz"
"" ++ "buzz"     == "buzz"
"fizz" ++ "buzz" == "fizzbuzz"
"" ++ ""         == ""
```

前回の方法が使えるのは、空文字列がビジネルルール内に決して現れない場合のみです。空文字列はいわば例外値であり、この解法は「全域的」なものではないのですが、その事実は明示されていません。

これは望ましいことではありません。

* この方法が適用可能な状況が制限される。もし明日、上司が「7 個ごとの数字は (空文字列を出力して) 無視する」のようなルールを追加してきたら問題が発生する
* 空文字列は、値が存在しないことを _伝えては_ くれない

しかし心配は無用、この目的にぴったりの型 `Maybe String` が存在し、`Nothing` もしくは `Just "fizz"` のどちらかの値をとることができます。

問題は、Maybe 型の値が連結できるのかどうかです。確認してみましょう。

`Maybe String` の値を String と同じように連結してみます。そのために、以下のような関数 `mappend` の存在を仮定します。

Caption: `Maybe String` 値の連結はどうあるべきか

```
Just "fizz" `mappend` Nothing     == Just "fizz"
Nothing     `mappend` Just "buzz" == Just "buzz"
Just "fizz" `mappend` Just "buzz" == Just "fizzbuzz"
Nothing     `mappend` Nothing     == Nothing
```

うまいことに、このような機能は標準ライブラリにあらかじめ存在しており、`Data.Monoid` モジュールに定義されています。_モノイド_ は _連結_ する機能 (`mappend` あるいは `<>` 演算子で表される) を備えた型を数学的に抽象化したもので、次のようなルールに従います。

* 単位元 `mempty` が存在し、`""` や `Nothing` のように振る舞う
* 連結は結合法則を満たす、すなわち `(a <> b) <> c == a <> (b <> c)`

文字列型は `mappend = (++)` と `mempty = ""` に対してモノイドになります。

`Maybe String` 型は上で見た `mapend` と `mempty = Nothing` に対してモノイドになります。

Caption: マニア向け

実は、`a` がモノイドである時、`Just x` と `Just y` の結合を `Just (x <> y)` と定めることで `Maybe a` もモノイドになります。このとき、`a` の単位元は決して必要とはならないため、実際には
`a` はいわゆる _半群_、すなわちモノイドから単位元の要求を外したもので充分です。

以上の情報を組み合わせることで、FizzBuzz のためのビジネスルールとその組み合わせは以下のように定義することができます。

Caption: モノイドによる FizzBuzz ルール

```
import Data.Monoid

fizzes   = cycle [Nothing, Nothing, Just "fizz"]
buzzes   = cycle [Nothing, Nothing, Nothing, Nothing, Just "buzz"]
pattern  = zipWith mappend fizzes buzzes
```

一見して明らかにわかりやすくなりました。それでは、数列をマスクするためにこのパターンをどうやって使うのかを見ていきましょう。

## ポイントその 2：条件の組み合わせ

次にポイントになるのは、数列とFizzBuzz のパターンとを組み合わせて、`Nothing` に評価される数字を「素通り」させる部分です。

言い方を変えれば、_Maybe_ 型から、もし値が存在すれば _Just_ 値に、そして _Nothing_ の場合はデフォルト値に評価されるような関数が必要だということです。

Caption: fromMaybe の動作

```
fromMaybe number (Just x) = x
fromMaybe number Nothing = number
```

さて、これで `x` や `number` の型とロジックが完全に独立しました。確かに _Maybe_ 型は、まさにこのような関数において便利に使用できます。すでに存在する機能を再利用することで、ここでも抽象化の利益を享受することができるのです。

Caption: 最終的なリファクタリング結果

```
module FizzBuzz where

import Data.Monoid

nums     = map show [1..]
fizzes   = cycle [Nothing, Nothing, Just "fizz"]
buzzes   = cycle [Nothing, Nothing, Nothing, Nothing, Just "buzz"]
pattern  = zipWith mappend fizzes buzzes

fizzbuzz = zipWith fromMaybe nums pattern

main _ = do
    println $ take 100 fizzbuzz
```

この回答をさらに改良するのもありです。例えば、`[Nothing, Nothing]` を `repeat 2 Nothing` で置き換えることができるでしょう。

いずれにしても、下の参考文献を当たれば、モノイドとなるデータ型は _本当に多数_ 存在することがわかるでしょう。これらを調べ、また実際に使ってみることは、今回の解法の本質を掴む上でしばしば助けになります。

本記事では、ビジネスルール上「起こるかもしれない」という性質を、型として抽象化することで捉えました。

_今回のリファクタリングを提案してくれた Ingo Wechsung 氏に感謝します！_

## 参考文献

* Monoid in Frege doc: [http://www.frege-lang.org/doc/frege/data/Monoid.html](http://www.frege-lang.org/doc/frege/data/Monoid.html)
