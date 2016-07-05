# リストとパス

リストは Frege でもっともよく使われるデータ構造であり、それゆえに当然、リストを扱うためのうまい手段も多数用意されています。

それでは、実生活で目にするであろうものを思い切り単純化したデータモデルから話を始めましょう。以下、データを問い合わせるための方法をいろいろ見ていきます。

## ドメインモデル

銀行は複数の顧客を持ち、顧客は複数の投資ポートフォリオを持ち、さらに投資ポートフォリオは複数のポジションから成り立っていて、それぞれのポジションはそのポートフォリオ内に証券 (与えられた銘柄の株式だとしましょう) が何株あるかを保持している、と仮定します。

Caption: 単純化された銀行業務のモデル

銘柄は列挙型を使うことで簡単にモデル化できます。

```
data Ticker = GOOG | MSFT | APPL | CANO | NOOB
derive Eq Ticker
```

ドメインの残りの要素は単にレコード構文を用いた通常のデータ構造です。

Caption:  ドメインの核となるデータ構造

```
data Bank       = Bank      { clients       :: [Client]     }
data Client     = Client    { portfolios    :: [Portfolio]  }
data Portfolio  = Portfolio { positions     :: [Position]   }
data Position   = Position  { soMany :: Int, ticker :: Ticker }
```

例として、1000 名のクライアントを持ち、それぞれのクライアントが複数のポジションからなるポートフォリオを 3 つずつ保有するような銀行を作ってみましょう。

```
bank = Bank { clients = replicate 1000
    Client { portfolios = replicate 3
        Portfolio { positions = [
            Position { soMany = 1, ticker = APPL },
            Position { soMany = 2, ticker = MSFT },
            Position { soMany = 8, ticker = CANO }
        ]}
    }
}
```

`replicate` を見たことがない人への補足。この関数はサイズと要素を引数に取り、そのサイズ分だけ要素を並べたリストを生成します。型は以下のとおりです。
 
 ```
 replicate :: Int → a → [a]
 ```
 
 Note: 簡潔かつインクリメンタル開発向き
 
 ドメインの定義は非常に簡潔です。レコード構文を使用する代わりに引数を順番で区別するようにすれば、さらに短くなるでしょう。しかし、後でデータ構造を簡単に拡張できるようにするためここではレコード構文を使用します。この場合でも、値を生成することはすぐにできましたし、サイズを色々と変えてみることも簡単です。
 
 最後に、後で株式銘柄の評価額を算出するために株価情報を用意しておきます。単純な連想リストを使いましょう。
 
 Caption: 連想リストによる株価の定義
 
 ```
 prices = [ (GOOG, 100) , (MSFT, 200) , (APPL, 300) , (CANO, 400) ]
 ```
 
 `NOOB` が株価のリストの中に登場しないことに注意してください。後で、リストに存在しない銘柄の株価を取得するテストの際にこの `NOOB` を使います。
 
  ポジションの評価額を算出する `value` を用いてまずは書いてみましょう。 リストに含まれる各銘柄の価格に、何株保有しているかを掛け合わせます。
 
 Caption: ポジションを評価するテスト
 
 ```
 import Test.QuickCheck

zeroValue     = once     $             0 == value (Position 5 NOOB)
multipleValue = property $ \n -> 100 * n == value (Position n GOOG)
 ```
 
 Note: テストを書く際、`NOOB` のような、つまり銘柄に対して株価が見つからない場合に何を値とすべきかについて考えることになります。ここでは `0` としましたが、これは手作業によるアプリケーションレベルの解決になっています。より広く使用されるライブラリ関数ならば、むしろ `Maybe` を返すなどによってエラーを明示するべきです。
 
 テストファーストで書く場合、コンパイラは `value` 関数が存在しないと言ってきます。`Data.List` にある `lookup` 関数を用いて、株式銘柄の価格を見つけるような `value` の定義を与えましょう。
 
 Caption: value 関数の実装
 
 ```
 import Data.List (lookup)

value :: Position -> Int
value position = calculate $ lookup position.ticker prices where
    calculate Nothing      = 0
    calculate (Just price) = position.soMany * price
 ```
 
 `lookup` が失敗する可能性があるため、`Maybe` を返します。またここでは、`Nothing` と実際の価格を持つ`Just` との場合分けによって `calculate` 関数を局所定義することで、失敗する可能性を表現しています (`maybe` を使って `maybe 0` のようにするやや地味な手もありますが……)。

Caption: なぜ null ではないのか？

Java の開発者であれば、リストへの参照が失敗した際には単に`null` を返せばよいのではないかと疑問に思うかもしれません。しかしご存知のように、後々呼び出し側が null チェックを忘れたとき、 _NullPointerException_ が発生しやすくなります。しかも null がセットされた地点とは遠く離れた場所で。

Frege では _null は存在せず_、したがってもはや _NullPointerException も存在しません_！

## 銀行の規模はどれくらい？

銀行同士はしばしば _保有資産_、すなわちすべての顧客の、すべてのポートフォリオの、すべてのポジションの総評価額で比較されます。

はい、われわれはこの総額を計算できるデータをすでにもっていますね！ データ取得に必要な関数をその型と合わせて以下に示します。

Caption: 関数とその型の組み合わせ

| 番号 | 関数                 | 型                       |
|:----|:----------------------|:-------------------------|
| <1> | `bank.clients`        | `[Client]`               |
| <2> | `Clients.portfolios`  | `Client → [Portfolio]`   |
| <3> | `Portfolio.positions` | `Portfolio → [Position]` |
| <4> | `value position`      | `Position → Int`         |

おや、面白いことが見て取れますね。_それぞれの関数は要素のリストを返し、その要素の一つ一つが次の関数の入力になっています_。ということは、このパターンを一般化し、「[お手軽入出力](easy-io.md)」でやったように _bind_ 関数を使用することができそうです。

<1> と <2> をバインドすると、

```
   <1>             <2>                 return type
[Client] -> (Client -> [Portfolio]) -> [Portfolio]
```
<2> と <3> をバインドすると、

```
    <2>                   <3>               return type
[Portfolio] -> (Portfolio -> [Position]) -> [Position]
```

見ての通り、背後には以下のような型を持つ _bind_ によって一般化されたパターンがあります。

```
[a] → (a → [b]) → [b]
```

嬉しいことに、すでに _bind_ 関数が使える形になっていて、「[お手軽入出力](easy-io.md)」と同じように `>>=` で記述することができます。

<1> と <2> を組み合わせると `bank.clients >>= Client.portfolios`

<2> と <3> を組み合わせると `Client.portfolios >>= Portfolio.positions`

<1> と <2> を組み合わせ、さらにそこに <3> を組み合わせると `bank.clients >>= Client.portfolios >>= Portfolio.positions`

Important: ジャジャーン！ これで銀行が持つすべての顧客の、すべてのポートフォリオの、すべてのポジションを表すことができるシンプルな「パス式」ができあがりました。

最終的に確認しておくと、以下が _bind_ を用いてポジションに対してそれぞれの価格を算出し、すべて加算することで保有資産を算出する仕組みの最初のバージョンです。

Caption: 銀行の保有資産算出、最初のバージョン

```
assetsUnderManagement1 = sum $
    map value $
        bank.clients >>= Client.portfolios >>= Portfolio.positions
```

## 「do」記法と内包表記

これも「[お手軽入出力](easy-io.md)」で見たとおり、_bind_ では 「do」 記法を利用することができます。これを使うと、以下のようなコードになります。

Caption: 「do」 記法を利用した銀行の保有資産算出

```
assetsUnderManagement2 = sum $
    map value do
        client    <- bank.clients
        portfolio <- client.portfolios
        portfolio.positions
```

ここでは、矢印記法 `←` によって計算中の一つ一つの値がリストから _取り出されて_ います。でもちょっと待ってください！ これは完全にどこかで見聞きしたことがある感じですね。リスト内包表記でも同じことができます。

Caption: リスト内包記法を利用した銀行の保有資産算出

```
assetsUnderManagement3 = sum
    [value position |
        client    <- bank.clients,
        portfolio <- client.portfolios,
        position  <- portfolio.positions
    ]
```

実際、両者の記法は等価で、単にスタイルが異なるだけです。

## パスの問い合わせを SQL 風に

_すべての_ 資産ではなく、Canoo 社がこの銀行に保有している資産の総額のみに興味がある場合を考えてみましょう。リスト内包表記を使えばこれは簡単で、また面白いことに SQL と似た部分があることがわかります。

Caption: クエリとしてのリスト内包表記

```
allCanoo3 = sum
    [value position |                       -- SELECT
        client    <- bank.clients,          -- FROM
        portfolio <- client.portfolios,
        position  <- portfolio.positions,
        position.ticker == CANO             -- WHERE
    ]
```

ここでは `value` 関数は SQL でいう射影、`position` は選択、リストは元データであり、ガードが where 節として働きます。

「do」記法が等価になることはすでに述べました。この場合、where 節よる絞り込みは以下のようになります。

Caption: 絞り込みつきの do 記法

```
allCanoo2 = sum $
    map value do
        client    <- bank.clients
        portfolio <- client.portfolios
        filter canoo portfolio.positions
    where
        canoo position = position.ticker == CANO
```

スタイルが微妙に異なることがわかるでしょう。

最後に、パスを用いて絞り込みを表現すると以下のようになります。

```
allCanoo1 = sum $
    map value $
        bank.clients >>= Client.portfolios >>= filter canoo . Portfolio.positions where
            canoo position = position.ticker == CANO
```

このような絞り込みはパス中のどの部分でも書くことができ、また絞り込み以外にもパスを評価する過程でリストに関数をマップしても構いません。

## まとめ

今回は日常のビジネスシーンから始めて、リストの持つ以下のような奥深い性質を見ることができました。

* パスをうまく表現できる
* 「do」記法と組み合わせて使うことができる
* 内包表記はそれほど特別なものではない
* SQL と似た方法で参照によるグラフ構造に対して問い合わせができる

総じて、内包表記が最もつぶしがきく記法で、特に絞り込みと射影には内包表記が向いています。単に値を集計したいのであればパス記法が良いでしょう。

他の言語であっても、パスによる表現が簡潔に書けることがあります。今回で言えば、例えば Groovy の GPath では `bank.clients*.portfolios*.positions.findAll{it.ticker == CANO}*.value().sum()` となります。ただし、コードの見た目のみで比較できるわけではありません。

決め手は遅延評価: Frege が持つ重要な長所として、遅延評価があります。巨大なグラフは決してそのまま具現化されるわけではなく、「(実際には存在しない) 問い合わせ結果のリスト」も具現化されません。パスは巨大なデータ構造ではなく、評価のストリームを組み立てるのです。

## 参考文献

* [Groovy GPath](http://docs.groovy-lang.org/latest/html/documentation/#gpath_expressions)
* [Haskell Wikibook](https://en.wikibooks.org/wiki/Haskell/Understanding_monads/List)
