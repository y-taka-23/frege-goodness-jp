# 型安全な DSL を目指して

「[型クラスを利用したミニ DSL](docs/12-a-mini-dsl-with-type-classes.md)」では、距離を扱う DSL においてミリメートル単位の長さを表現するために `Int` 型を使用しました。この方法はとっつきやすいのですが限界もあります。`Int` を他の目的のために再利用することができませんし、また型安全性も欲しいところです。

以下、どのようにしてこの問題を改善できるのかを見ていきましょう。

## 新しい目標

今回の DSL では、以前と同様に距離の単位が扱えるだけでなく、さらに時間の単位も扱えるようにします。両者を組み合わせることで、そのまま速度の単位も扱うことができます。

Caption: 今回の単位系 DSL でやりたいこと

```
10.m   -- distance in meters
10.sec -- duration in seconds
1000.m `per` 60.min == 1.kmh  -- velocity
```

時間と距離を足すような不正な演算は、型システムによって防止します。

実装の戦略としては、まず前回のコードに直接手を入れてリファクタリングすることで、動作は前回とまったく変わらないようにしつつ、時間を扱う部分を既存コードの変更なしに追加できる状態にします。

## 第一段階：リファクタリング

前回のコードでは `Int` 型を `Millimeter` 型クラスのインスタンスにしていました。今回はこれを一般化し、`Millimeter` を `Num` 型クラスのインスタンスにして演算ができるようにしましょう。

Caption: Num の制約を満たすために実装が必要な関数

```
data Millimeter = MM Int

instance Num Millimeter where
    zero = MM 0
    one  = MM 1
    (MM a) + (MM b) = MM (a+b)
    (MM a) - (MM b) = MM (a-b)
    (MM a) * (MM b) = MM (a*b)
    MM a  <=> MM b  = a <=> b
    hashCode (MM a) = hashCode a
    fromInt     a   = MM a
    fromInteger a   = MM a.fromIntegral
```

これでミリメートル単位での計算が可能になりました。

Caption: ミリメートル単位での計算

```
MM 10 + MM 10 == MM 20
```

ここで、`10.mm` が値 `MM 10` になるようにするために、`Millimeter` を用いて新しく `Metric` 型クラスを作成し、それを `Int` の代わりに戻り値として使います。

Caption: 戻り値を Int から Millimeter に変更

```
class (Integral a) => Metric a  where
    mm :: a -> Millimeter
    cm :: a -> Millimeter
    m  :: a -> Millimeter

instance Metric Int where
    mm i = MM i
    cm i = i.mm * 10
    m  i = i.cm * 100
```

さて、以上のリファクタリングの後でも、前回のコードはリファクタリング前と同じように動作します。

Caption: 前回とまったく同じ

```
main args = do
    println $ 10.m - 20.cm + 10.mm - 3.cm  == 9780.mm
```

それではなぜ、わざわざリファクタリングしたのでしょうか？

* 前回よりも型安全になった
* 新しい単位を追加できるようになった

## 第二段階：時間の追加

次のステップとして、既存のコードにはまったく触れないようにしつつ、時間を使った計算ができるように _非侵入的_ な差分を適用します。考え方は上と同じです。

Caption: 時間に関する部分をまとめて追加

```
data Seconds = SEC Int

instance Num Seconds where
    zero = SEC 0
    one  = SEC 1
    (SEC a) + (SEC b) = SEC (a+b)
    (SEC a) - (SEC b) = SEC (a-b)
    (SEC a) * (SEC b) = SEC (a*b)
    SEC a  <=> SEC b  = a <=> b
    hashCode (SEC a)  = hashCode a
    fromInt     a     = SEC a
    fromInteger a     = SEC a.fromIntegral

class (Integral a) => Time a where
    sec     :: a -> Seconds
    minutes :: a -> Seconds
    h       :: a -> Seconds

instance Time Int where
    sec     i = SEC i
    minutes i = i.sec * 60
    h       i = i.minutes * 60

main args = do
    println $ 3.sec + 1.h - 30.minutes == 1803.sec
```

何が起こるのかを確認してみましょう。

## 全体をまとめる

構造を変更したことでいくつか顕著な利点が得られます。

* 間違えて `10.m + 10.sec` と書いてしまっても、コンパイラが `Type error in expression "sec 10".  Type is Seconds used as Millimeter` のようにエラーを出してくれます。わかりやすいエラーメッセージですね！ Frege はこちらの心の中が読めるのでしょうか？
* `Int` 型の値を計算の中に混ぜることも _依然として_ 可能で、その場合 _安全_ に _正しい_ 型に変換されます！ `10.mm + 3 == 13.mm` も `10.sec + 3 == 13.sec` も正しく、かつ型安全です。この変換は `Num` 型クラスにおける `fromInt` の実装よって規定されています。

さらに一歩進めて、距離と時間を混ぜることで速度を表現することもできます。

Caption: 距離 / 時間による速度の表現

```
data Velocity = KMH Double
derive Eq Velocity

class (Real a)  => Unit a where
    kmh :: a -> Velocity

instance Unit Double where
    kmh val = KMH val

per (MM mm) (SEC sec) =
    KMH (( mm.fromIntegral / sec.fromIntegral) * 0.0036)

main args = do
    println $ (500.m * 3) `per` (3000.sec + 600)  == 1.5.kmh
```

かなりいい感じの DSL に仕上がりました。以下のように便利な特徴があります。

* 完全な型安全性
* 完全な型推論
* 素晴らしいエラーメッセージ

この DSL を読んだり書いたりするのも難しくありません。

今回の DSL を作成する上での立役者は、ドット記法と組み合わせた型クラスの使い方でした。侵入的なリファクタリングを間に一回挟むことで、ほとんどの部分を非侵入的な追加として実装することができました。

また、今回行った変更はすべて _頑健_ です。

* 非侵入的な追加はそもそも定義から頑健である
* 必要な時には、型システムによってリファクタリングの _頑健性_ が保証される