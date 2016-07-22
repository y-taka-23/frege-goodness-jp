# お手軽入出力

従来、純粋関数型言語においては、入出力は不可能とまでは言わないまでも難しく扱いにくいものだと考えられてきました。

簡単な例として、コンソールに 1、2、3 を出力するためのメソッド (Java および Groovy) および関数 (Frege) を挙げ、比較してみましょう。

この例は単純化しすぎているように見えますが、実は思い切って普段の常識を疑ってみることで面白いことが学べます。

## コード比較：Java 対 Groovy 対 Frege

今回、比較する対象は以下のコードです。

Caption: Java のメソッド

```
// Java
void print123() {
    System.out.println("1");
    System.out.println("2");
    System.out.println("3");
}
```

Caption: Groovy のメソッド

```
// Groovy
def print123() {
    println "1"
    println "2"
    println "3"
}
```

Groovy 版は書き方がいくつも考えられます。Java と全く同じ書き方をすることも可能で、それもまた正しい Groovy プログラムです。今回はもっとコンパクトなバージョンを採用しました。

Caption: Frege の関数

```
-- Frege
print123 = do
    println "1"
    println "2"
    println "3"
```

驚くべきことに、3 種類の中で _最も短いのは Frege_ で、4 文字分ですが Groovy 版よりも若干短くなっています！ さらに、記号類は Frege 版が一番少ないことがわかります。

_上の例には、明らかに同じコードの繰り返しが含まれています。Groovy でも Frege でも 3 行分をまとめて 1 行にすることは簡単にできるのですが、この件は後の記事で扱います。_

迷信破れたり: というわけで、純粋関数型プログラミングでは入出力は難しいという迷信には、簡単に反論できます。

しかし、ここで驚嘆すべきポイントがあります。Frege 版は最も短いというだけではなく、何が起こるかについて最も明示的なのです！

## 最も明示的なのは？

各言語の型シグネチャからわかることをまとめてみましょう。

### Java

このメソッドは `void` を返します。これを見てもほとんど何もわかりませんが、しかし少なくとも、Java コンパイラが誤りを検出するため、戻り値をうっかり何らかの参照に代入してしまうことはありません。しかしながら、このシグネチャからは `stdout` に出力されることは全く見て取れません。 実はこれはかなり驚くべきことです。catch するかシグネチャに記述しなくてはならない例外である __IOException を `println` がスローしないのはなぜ__ なのでしょう？ さらに、`System.out` は _グローバルな可変フィールド_ であり、アクセスに対して _全く保護されていません_。少なくとも getter メソッドは存在するのが通例です。

### Groovy

Groovy は Java にとてもよく似た特徴を持っていますが、例外に関してはもっと誠実です。例外は必ず記述されているなどと騙ったりしません。ここで使用している戻り値の型 `def` は、メソッドが戻り値を持つことを意味します。戻り値の値は最後に評価された式の値、今回の場合で言えば `println` が返す値である `null` です。しかし繰り返しになりますが、これが Groovy のスタイルであり、Groovy はプログラマへの信頼に基づいています。

### Frege

今回のコードは型について何も語っていませんが、型が存在しないわけではありません。型は _推論_ されており、REPL で `:type print123` と入力すれば `IO ()` (「アイオーユニット」と読む) であることがわかります。これが知るべき情報のすべてです。この型情報を見れば、関数が入出力とそれに関連するすべての操作を行う (可能性がある) ことがわかります。型に IO が現れないような他の関数から呼び出すことはできません。

## 根本的な違い

根本的な違いとして、Java や Groovy のような命令型言語では、表面上、3 回の `println` 文の間に関連性がないことが挙げられます。文を並べ替えたり、完全に無関係な計算を間に挟むことが可能で、型シグネチャは変化しません。

Frege では状況は全く異なります。Frege には文が存在せず、あるのは式だけです。

`println "2"` と `println "3"` は、単に 2 行が順番に並んでいるというわけではなく、極めて強い関連を持っています。この 2 行は両方とも式であり、_IO_ 型が指定する方法で両者を _バインド_ する関数の引数になっています。_バインド_ された結果の戻り値の型は二つ目の引数と同じで、ここでは `IO ()` です。

Caption: (println "3") が IO () を返すため、全体も IO () を返す

```
-- pseudocode
bind (println "2") (println "3")
```

そして当然、`println "1"` についても同じことが言えます。

Caption: 全体が IO () を返すのはわかりますね？

```
-- pseudocode
bind (println "1")  (bind (println "2") (println "3"))
```

Note: _バインド_ は、関数名としては IO 型クラスには現れません。演算子 `>>=` の形でのみ定義され、この演算子を「バインド」と読みます。すなわちここで挙げた疑似コード `bind a b` は実際のコードでは `a >>= b` となります。

命令型プログラムのように見えますが、関数全体で　1 つの式になっているため、型推論は非常に簡単です。

以上のように、様々なものをキーワード `do` によって 1 つの式に押しつぶすことができ、このキーワードは結果となる式の _バインド_ 関数と呼ばれています。今回の式は `IO` 型であるため、`IO.bind` が使用されます。

例えば「[静かなる記号たち](silent-notation.md)」ですでに見たとおり、関数型プログラミングの意味論ではプログラムは最後から遡って読むことになります。これは「do」記法でも同様です。最終的な結果の型を確認するために、今回は右から左ではなく、下から上に向かって読みます。

Note: 個人的な経験から: 自分の場合、この仕組みがどうやって動いているのか、特に _do_ が使用する _bind_ を一体どのように判断するのか、理解するのにしばらく時間がかかりました。このあたりの説明は世の中に山ほどありますが、自分が知りたい点だけは見つけられませんでした。もしかすると他の人にとっては自明すぎるのかもしれません。

ちなみに、_bind_ を中置演算子のように使ってみると面白い類似が見て取れます。

```
-- pseudocode
a `bind`
b `bind`
c
```

命令型言語のスタイルで以下のように書いた場合とよく似ています。

```
a ;
b ;
c
```

これが、_bind_ が冗談半分に _プログラマブル・セミコロン_ などと呼ばれる理由です。bind は二つの関数を与えられたコンテクストにおいてどのように合成するか、また後で見るように、最初の関数の結果をどのように次の関数の引数として _バインド_ するかを規定します。

## まとめ

* IO 処理は、Frege のような純粋関数型言語であっても、実に簡単に書くことができる
* 純粋関数型言語では、IO が存在しないわけではなく、IO について厳密に明示しなくてはならない
* 関数型のコードは、その本質を損なうことなく命令型のコードであるかのように見せることができる
* 副作用が存在するとき、do 記法が役に立つ