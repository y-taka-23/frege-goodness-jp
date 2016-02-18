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
 
 ドメインの定義は非常に簡潔です。レコード構文を使用する代わりに引数を順番で区別するようにすれば、さらに短くなるでしょう。
 
 最後に、後で株式銘柄の評価額を算出するために株価情報を用意しておきます。単純な連想リストを使いましょう。
 
 Caption: 連想リストによる株価の定義
 
 ```
 prices = [ (GOOG, 100) , (MSFT, 200) , (APPL, 300) , (CANO, 400) ]
 ```
 
 