# QuickCheck で性質テスト

Frege には性質テスト (propery-based test) を可能にする機能が備わっています。この機能で何ができるのかを少し見ていきましょう。

「種も仕掛けもあるんです」(Todo: 書いたらリンク貼る) では、以下のような関数 _f_ および _g_ を使用しました。

```
f :: [a] -> [a]  -- we should think of any such function
f  = reverse     -- that was our pick

g :: Int -> String
g x = show x ++ show x ++ show x
```

そして記事の結論として、

```
map g (f [1, 2, 3])
```

および

```
f (map g [1, 2, 3])
```

のいずれを選んでも同じ結果になるということがわかりました。

言い換えれば、テスト可能な性質 (個人的には _不変量_ という用語のほうが好きですが) を発見したというわけです。
