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

