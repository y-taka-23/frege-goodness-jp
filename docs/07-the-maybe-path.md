# Maybe 付きのパス

「[リストとパス](06-list-and-path.md)」では、リスト構造を利用してパスを表現する方法について扱いました。また `Maybe` 型が、値を取得できる (できない) ことを示すために使えることも確認しました。

今回は、`Maybe` 型が複数現れるような場合でも、パス表現はまったく同じ形式で機能することを確認します。

## ドメインモデル

銀行業務のドメインモデルに戻って、今回は銀行に _星付き_ の顧客 (ウォーレン・バフェットとでもしましょうか) がいる可能性があるとします。その顧客は、彼がお気に入りであるところのトップ運用成績のポートフォリオを保有しているかもしれません。この _星付き_ ポートフォリオには要チェックな、いわば _星付き_ ポジションがオプショナルな値として組み込まれています。

Caption: 単純化された銀行業務ドメインにおける星

このドメインモデルは、レコード構文を用いて、以下のように簡単に表現することができます。「[リストとパス](06-list-and-path.md)」でのデータ構造と似ていることに注意しましょう。異なるのは、型として現れていた _リスト_ が `Maybe` になったことと、それに合わせて名前を変更したことのみです。

Caption: ドメインの核となるデータ構造

```
data Bank       = Bank      { star   :: Maybe Client     }
data Client     = Client    { star   :: Maybe Portfolio  }
data Portfolio  = Portfolio { star   :: Maybe Position   }
data Position   = Position  { soMany :: Int, ticker :: Ticker }
```

Note: 後で取り回しが若干複雑になるのですが、`Position` は常に銘柄を保持しています。これはドメインモデルに気持ち「リアリティ」を残すためです。

それでは例として、オプショナルな値がすべて実際に値を持っている銀行を作ってみましょう。

Caption: すべてに星が付いている銀行

```
starBank = Bank {
    star = Just Client {
        star = Just Portfolio {
            star = Just Position { soMany = 8, ticker = CANO }
        }
    }
}
```

しかし他の銀行には、星付きの顧客がいなかったり、いたとしても優良ポートフォリオを保有していなかったり、保有していても特筆すべきポジションが組み込まれていなかったりするかもしれません。

Caption: 残念な銀行 (仮) 

```
noStarBank      = Bank { star = Nothing }
noPortfolioBank = Bank { star = Just Client { star = Nothing } }
```

星付き顧客のお気に入りの銘柄に投資して一山当てる、なんてことを企んでみましょうか。そのためにはまず、該当する銘柄を見つける必要があります。

## はじめの一歩

星付きの銘柄を見つけるためには、星付き顧客 (もしいれば) の星付きポートフォリオ (もしあれば) の星付きポジション (もしあれば) を見つけなければなりません。

この目的を達成するために、`starTicker` 関数を作りましょう。この関数は `bank` を引数に取り、もし星付きの銘柄が存在すればそれを `Just` の中に入れて返し、存在しなければ `Nothing` を返します。 要するに、以下のような型を持つ関数です。

```
starTicker :: Bank -> Maybe Ticker
```

これを直接実装する方法はいくつも (コンストラクタによる場合分け、case 式、_if_ を入れ子にする、_maybe_ 関数) ありますが、どれを選んだとしても、下に挙げる実装に似たごちゃごちゃした解法になります。

Caption: 動作するが読みにくい実装

```
starTicker bank = starPortfolio bank.star where
    starPortfolio Nothing = Nothing
    starPortfolio (Just client) = starPosition client.star where
        starPosition Nothing = Nothing
        starPosition (Just portfolio) = starTicker portfolio.star where
            starTicker Nothing = Nothing
            starTicker (Just position) = Just position.ticker
```

これが期待通りに動作するかどうかを確認してみましょう。処理中のあちこちに _Nothing_ が現れる可能性があり、その場合にエラーとならないよう保証しておく必要があるので忘れずに。

Caption: テストする

```
mport Test.QuickCheck

star1 = once $ starTicker starBank        == Just CANO
star2 = once $ starTicker noStarBank      == Nothing
star3 = once $ starTicker noPortfolioBank == Nothing
```

このアプローチでも動作はしますが、明らかに改善の余地ありです。

## 表記法の改善

ここは一度戻って、今回何をしているのかについて再考しましょう。

「[リストとパス](06-list-and-path.md)」でやったのと同じように、登場する関数とその型を確認してみます。

Table 1: 関数とその型の組み合わせ

| 番号 | 関数             | 型                           |
|:----|:------------------|:-----------------------------|
| <1> | `bank.star`       | `Maybe Client`               |
| <2> | `Clients.star`    | `Client → Maybe Portfolio`   |
| <3> | `Portfolio.star`  | `Portfolio → Maybe Position` |
| <4> | `Position.ticker` | `Position → Ticker`          |

各関数は、次の関数の引数の型に _Maybe_ をつけたものを返します。したがっておそらく、このパターンを一般化して、関数を _バインド_ することで一行にまとめることができるはずです。

<1> と <2> をバインドすると

```
<1>             <2>                            return type
Maybe Client -> (Client -> Maybe Portfolio) -> Maybe Portfolio
```

<2> と <3> をバインドすると

```
<2>                <3>                              return type
Maybe Portfolio -> (Portfolio -> Maybe Position) -> Maybe Position
```

見ての通り、背後には以下のような型を持つ _bind_ によって一般化されたパターンがあります。

```
Maybe a → (a → Maybe b) → Maybe b
```

嬉しいことに、すでに _bind_ 関数が使える形になっていて、「お手軽入出力」(Todo: 書いたらリンクを貼る) や「[リストとパス](06-list-and-path.md)」と同じように `>>=` で記述することができます。

<1> と <2> を組み合わせると `bank.star >>= Client.star`

<2> と <3> を組み合わせると `Client.star >>= Portfolio.star`

<1> と <2> を組み合わせ、さらにそこに <3> を組み合わせると `bank.star >>= Client.star >>= Portfolio.star`

Important: ジャジャーン！ 今回もシンプルな「パス」表現にたどり着きました。今回は、オプショナルな _星付き_ 顧客の、オプショナルな _星付き_ ポートフォリオの、オプショナルな _星付き_ 　ポジションに対するパスとなっています。

仕上げとして、一つのパスにまとめたものが以下です。

Caption: パス内の Maybe 値の連鎖

```
starTicker bank =
    bank.star >>= Client.star >>= Portfolio.star >>= \position -> Just position.ticker
```

`Position.ticker` も _Maybe_ 型だったらもっとすっきりと書けたでしょう。しかし銘柄がないポジションは存在し得ないため、こちらのほうがリアリティがあります。また、関数に渡す引数をラムダ式のパラメータとして束縛する例としても勉強になります。

型を確認するのは簡単です。「静かなる記号たち」(Todo: 書いたらリンク貼る) ですでに見たように、

```
\position -> Just position.ticker   -- Position -> Maybe Ticker
```

は単なる

```
foo :: Position -> Maybe Ticker
foo position = Just position.ticker
```

の別表記で、いい関数名をわざわざ考える労力を使わずに済ませることができます。

## 「do」記法、ふたたび

一方、_バインド_があるところ、少し足を伸ばせば「do」記法が現れるのはごく自然なことです。

Caption: 「do」記法による星付き銘柄

```
starTicker bank = do
    warrenBuffet  <- bank.star
    starPortfolio <- warrenBuffet.star
    starPosition  <- starPortfolio.star
    Just starPosition.ticker
```

かなりいい感じに読みやすくなりますし、まさに狙ったとおりに動きます。各ステップは `Nothing` に評価され得ること、またその場合にはそれ以上評価を続けることなく _即座に_ `Nothing` が返ることに注意しましょう。

