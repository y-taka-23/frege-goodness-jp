# 火星探査機コマンドと状態付き計算

「[型クラスを利用したミニ DSL](12-a-mini-dsl-with-type-classes.md)」で扱った内容をさらに発展させて、今回は DSL にふたつの新機能を導入しましょう。

* `move rover 20 meters forward` のような、英文として読めるコマンド式
* 同じ探査機に対するコマンドの逐次実行

## 第一段階：コマンド式

コマンド式を導入するとコードが英文のように読める形になるため、種々の DSL において便利 (あるいは単純にカッコいい) です。ただしこれを実現するためには、すべての記号類を使用しないようにする必要があり、たいていの言語では困難を極めます。しかし幸いにして Frege にとってはごく簡単。次のような例がどのようにして実装されるのかを見てみましょう。

Caption: 単独のコマンド

```
move rover 20 meters forward
```

単に `move` 関数に引数を 4 つ渡しただけです！ 以上。

ほとんどこれだけですが、`rover` の型が何であるべきか、またそれ以外の引数が何なのかは確認しておく必要があります。次のようにしましょう。`rover` だけはレコード構文を使用して _Position_ をよろしく定義し、残りの引数には _Double_ を使用します。戻り値は新しい位置です。

Caption: コマンド式が機能するようにする

```
data Position = Pos { x, y :: Double }
derive Show Position

rover   = Pos 0 0
meters  = 1.0
forward = 1.0

move :: Position -> Double -> Double -> Double -> Position
move position distance unit direction =
    position.{x <- (+ distance * unit * direction) }  -- (1)
```

おなじみですが、`(1)` で 「The Power of the Dot」(Todo: 書いたらリンクを貼る) で登場した形式を用いて x 座標を変更しています。

Homework I: やる気のある人は、「[型安全な DSL を目指して](13-enhancing-the-dsl-for-type-safety.md)」で定義した型安全な長さの単位系を再利用して、上のコードを改良してみましょう。

現時点ではうまく機能するように見えますが、複数の動作を順番に実行させようとした途端、このコードではむしろ使い勝手が悪くなります。最後の動作の結果が次の動作の引数になるよう、更新された位置情報を取り回す必要があるのです。

```
pos1 = move rover ...
pos2 = move pos1  ...
```

単に位置を変更することができれば楽なのですが……。

## 第二段階：状態の導入

幸いにして、Frege には今回の目的にぴったりの _State_ 型が存在します。

*  `State Position` 型は「_Position_ 型で表される状態」として使うことができます。
* `State` には高階関数を引数として取る `modify` 関数があり、この `modify` がどのように位置を更新するかを決めます。はい、これを新たに `move` として使えばよいのです！
* また `State` には `execState` 関数があり、初期位置から始まる状態付き計算の実行 (ここでは連続した移動) を行うことができます。完璧です。

"bind" again: 状態付き計算は、表面的には _move_ 関数を複数つなげることに相当します。これら複数の関数は必ず順番通りに呼び出され、また `execState` は移動の結果として得られる位置を次の移動に _バインド_ します。

"bind" が (`>>=` 演算子の形で) 現れる時、これを do 記法を用いて書くことができます。今回の最終形でもこれを使うことにします。以下がコードです。

Caption: コマンド式で探査機を移動させる

```
module CommandExpressionRover where

import frege.control.monad.State

data Position = Pos { x, y :: Double }
derive Show Position

rover   = Pos 0 0
feet    = 0.305
meters  = 1.0
forward = 1.0
back    = -1.0

move ∷ Double -> Double -> Double -> State Position ()  -- (1)
move distance unit direction = State.modify update where
    update pos = pos.{x <- (+ distance * unit * direction) }

with start f = State.execState f start   -- (2)

main = do
    endPosition = with rover do
        move 20 meters forward           -- (3)
        move 10 feet back
    println $ "moved the rover around, ending at " ++ show endPosition
```

このコードは以下を出力します。

```
moved the rover around, ending at Pos 16.95 0.0
```

わかりにくい部分についていくつか補足しましょう。

(1) を見ると、型シグネチャは `State Position ()` になっています。なぜ単位型 `()` が使われているのでしょうか？  一般的に `State` は、`State s a` のようにひとつでなくふたつの型引数をとります。ここで `s` は管理されている状態の型、今回であれば `Position` であり、`a` は状態付き計算が _評価された_ 結果の型です。今回は状態の変化だけを見ているため、`a` には特に用がありません。`runState` や `evalState` を使ってみると違いがわかるでしょう。

(2) ではちょっとした便利関数 `with` を導入していますが、これは単に `execState` の引数の順番を入れ替えて do 記法を使いやすくするためです。また文章として読んだ感じも悪くないと思います。同じことを直接行う `flip` 関数がすでに存在しますが、これを使うと目障りな感じになります。

そして (3) ではついに探査機が動きました！ この部分が状態付き計算です。しかも DSL としてコンパイル可能！ さらに (言語マニア向け情報ですが) 暗黙的なデリゲートを使うことなく 100 パーセント型安全に動作します。

Homework II: 例えば `turn 90 degrees` のように、次の動作の方向を変える関数を追加してみましょう。凝りたい人は、[FregeFX](https://github.com/Frege/FregeFX) をチェックアウトして Canvas で移動の様子を描画してみましょう。

## 状態を持たない「状態」

純粋関数型言語で状態を扱うのは困難である、と主張する人はたくさんいます。確かに、単に `position.x += distance * unit * direction` と書くことに比べると多少頭は使うでしょう。しかし、上にあげたコードは充分に読みやすいはずです。

逆に、状態を扱うことが純粋性という目標を損なうことはないのでしょうか？ 知らない間に暗黒面に落ちているのでは？ いえ、そんなことはありません！

Staying pure: 上で挙げたコードはあたかもミュータブルな状態を扱っているように見えますが、実際はそうではありません。`State` はある関数から次の関数へとイミュータブルな状態を受け渡すだけであって、__状態を変更することはありません__。`State` を使ったとしてもやはり純粋関数的なのです。

また、戻り値を取得する際に、`State` の文脈なしで裸の `endPosition`が得られたことに注意しましょう。これは `State` が純粋だからこそ可能なことです。

## 参考文献

* [Haskell Wikibook](https://en.wikibooks.org/wiki/Haskell/Understanding_monads/State)
* [Frege Language Reference](http://www.frege-lang.org/doc/Language.pdf), section 3.2 "Primary Expression"
* [Groovy Mars Rover DSL](http://www.infoq.com/presentations/groovy-dsl-mars)