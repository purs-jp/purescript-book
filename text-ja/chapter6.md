# 型クラス

## この章の目標

この章では、PureScriptの型システムにより可能になっている強力な抽象化の形式を導入します。
そう、型クラスです。

データ構造をハッシュ化するためのライブラリを本章の動機付けの例とします。
データ自体の構造について直接考えることなく複雑な構造のデータのハッシュ値を求める上で、型クラスの仕組みがどのように働くかを見ていきます。

また、PureScriptのPreludeや標準ライブラリに含まれる、標準的な型クラスも見ていきます。PureScriptのコードは概念を簡潔に表現するために型クラスの強力さに大きく依存しているので、これらのクラスに慣れておくと役に立つでしょう。

オブジェクト指向の方面から入って来た方は、「クラス」という単語がそれまで馴染みのあるものとこの文脈とでは _かなり_
異なるものを意味していることに注意してください。

## プロジェクトの準備

この章のソースコードは、ファイル `src/data/Hashable.purs`で定義されています。

このプロジェクトには以下の依存関係があります。

- `maybe`: 省略可能な値を表す `Maybe`データ型が定義されています。
- `tuples`: 値の組を表す `Tuple`データ型が定義されています。
- `either`: 非交和を表す `Either`データ型が定義されています。
- `strings`: 文字列を操作する関数が定義されています。
- `functions`: PureScriptの関数を定義するための補助関数が定義されています。

モジュール `Data.Hashable`では、これらのパッケージによって提供されるモジュールの幾つかをインポートしています。

## 見せて！

型クラスの最初の簡単な例は、既に何回か見たことがある関数で提供されています。
`show`は何らかの値を取り、文字列として表示する関数です。

`show`は `Prelude`モジュールの `Show`と呼ばれる型クラスで次のように定義されています。

```haskell
class Show a where
  show :: a -> String
```

このコードでは、型変数`a`を引数に取る`Show`という新しい*型クラス*を宣言しています。

型クラス _インスタンス_ には、型クラスで定義された関数の、その型に特殊化された実装が含まれています。

例えば、Preludeにある `Boolean`値に対する `Show`型クラスインスタンスの定義は次の通りです。

```haskell
instance Show Boolean where
  show true = "true"
  show false = "false"
```

このコードは型クラスのインスタンスを宣言します。
`Boolean`型は *`Show`型クラスに属すもの* としています。

> ピンとこなければ、生成されるJSのコードは以下のようになります。
>
> ```javascript
> var showBoolean = {
>     show: function (v) {
>         if (v) {
>             return "true";
>         };
>        if (!v) {
>             return "false";
>         };
>         throw new Error("Failed pattern match at ...");
>     }
> };
> ```
>
> 生成される名前が気に入らなければ、型クラスインスタンスに名前を与えられます。
> 例えば次のようにします。
>
> ```haskell
> instance myShowBoolean :: Show Boolean where
>   show true = "true"
>   show false = "false"
> ```
>
> ```javascript
> var myShowBoolean = {
>     show: function (v) {
>         if (v) {
>             return "true";
>         };
>        if (!v) {
>             return "false";
>         };
>         throw new Error("Failed pattern match at ...");
>     }
> };
> ```

PSCiでいろいろな型の値を`Show`型クラスを使って表示してみましょう。

```text
> import Prelude

> show true
"true"

> show 1.0
"1.0"

> show "Hello World"
"\"Hello World\""
```

この例では様々な原始型の値を `show`しましたが、もっと複雑な型を持つ値も`show`できます。

```text
> import Data.Tuple

> show (Tuple 1 true)
"(Tuple 1 true)"

> import Data.Maybe

> show (Just "testing")
"(Just \"testing\")"
```

`show`の出力は、REPLに（あるいは`.purs`ファイルに）貼り戻せば、表示されたものを再作成できるような文字列であるべきです。
以下では`logShow`を使っていますが、これは単に`show`と`log`を順に呼び出すものであり、引用符なしに文字列が表示されます。
`unit`の表示は無視してください。
第8章で`log`のような`Effect`を調べるときに押さえます。

```text
> import Effect.Console

> logShow (Tuple 1 true)
(Tuple 1 true)
unit

> logShow (Just "testing")
(Just "testing")
unit
```

型 `Data.Either`の値を表示しようとすると、興味深いエラー文言が表示されます。

```text
> import Data.Either
> show (Left 10)

The inferred type

    forall a. Show a => String

has type variables which are not mentioned in the body of the type. Consider adding a type annotation.
```

ここでの問題は `show`しようとしている型に対する
`Show`インスタンスが存在しないということではなく、PSCiがこの型を推論できなかったということです。
これは推論された型で*未知の型*`a`とされていることが示しています。

`::`演算子を使って式に註釈を付けてPSCiが正しい型クラスインスタンスを選べるようにできます。

```text
> show (Left 10 :: Either Int String)
"(Left 10)"
```

`Show`インスタンスを全く持っていない型もあります。
関数の型 `->`がその一例です。
`Int`から `Int`への関数を `show`しようとすると、型検証器によってその旨のエラー文言が表示されます。

```text
> import Prelude
> show $ \n -> n + 1

No type class instance was found for

  Data.Show.Show (Int -> Int)
```

型クラスインスタンスは次の2つのうち何れかの形で定義されます。
型クラスが定義されている同じモジュールで定義するか、型クラスに「属して」いる型と同じモジュールで定義するかです。
これらとは別の場所で定義されるインスタンスは[「孤立インスタンス」](https://github.com/purescript/documentation/blob/master/language/Type-Classes.md#orphan-instances)と呼ばれ、PureScriptコンパイラでは許されていません。
この章の演習の幾つかでは、その型の型クラスインスタンスを定義できるように、型の定義を自分の`MySolutions`モジュールに複製する必要があります。

## 演習

1. （簡単）`Show`インスタンスを`Point`に定義してください。
   第4章の`showPoint`関数と同じ出力に一致するようにしてください。
   *補足*：`Point`はここでは（`type`同義語ではなく）`newtype`です。
   そのため`show`の仕方を変えられます。
   こうでもしないとレコードへの既定の`Show`インスタンスから逃れられません。

    ```haskell
    {{#include ../exercises/chapter6/test/no-peeking/Solutions.purs:Point}}
    ```

## よく見かける型クラス

この節では、Preludeや標準ライブラリで定義されている標準的な型クラスを幾つか見ていきましょう。
これらの型クラスはPureScript特有の抽象化をする上で多くのよくあるパターンの基礎を形成しています。
そのため、これらの関数の基本についてよく理解しておくことを強くお勧めします。

### Eq

`Eq`型クラスは`eq`関数を定義しています。
この関数は2つの値について等値性を調べます。
実は`==`演算子は`eq`の別名です。

```haskell
class Eq a where
  eq :: a -> a -> Boolean
```

何れにせよ、2つの引数は同じ型を持つ必要があります。
異なる型の2つの値を等値性に関して比較しても意味がありません。

PSCiで `Eq`型クラスを試してみましょう。

```text
> 1 == 2
false

> "Test" == "Test"
true
```

### Ord

`Ord`型クラスは`compare`関数を定義します。
この関数は2つの値を比較するのに使えるもので、その値の型は順序付けに対応したものです。
比較演算子`<`、`>`と厳密な大小比較ではない`<=`、`>=`は`compare`を用いて定義されます。

*補足*：
以下の例ではクラスシグネチャに`<=`が含まれています。
この文脈での`<=`の使われ方は、`Eq`が`Ord`の上位クラスであり、比較演算子としての`<=`の用途を表す意図はありません。
後述の[上位クラス](#上位クラス)の節を参照してください。

```haskell
data Ordering = LT | EQ | GT

class Eq a <= Ord a where
  compare :: a -> a -> Ordering
```

`compare`関数は2つの値を比較して`Ordering`を返します。
これには3つ選択肢があります。

- `LT`- 最初の引数が2番目の値より小さいとき。
- `EQ`- 最初の引数が2番目の値と等しいとき。
- `GT`- 最初の引数が2番目の値より大きいとき。

ここでも`compare`関数についてPSCiで試してみましょう。

```text
> compare 1 2
LT

> compare "A" "Z"
LT
```

### Field

`Field`型クラスは加算、減算、乗算、除算などの数値演算子に対応した型を示します。
これらの演算子を抽象化して提供されているので、適切な場合に再利用できるのです。

> *補足*：型クラス`Eq`ないし`Ord`とちょうど同じように、`Field`型クラスはPureScriptコンパイラで特別に扱われ、`1 + 2 * 3`のような単純な式は単純なJavaScriptへと変換されます。
> 型クラスの実装に基いて呼び出される関数呼び出しとは対照的です。

```haskell
class EuclideanRing a <= Field a
```

`Field`型クラスは、幾つかのより抽象的な*上位クラス*が組み合わさってできています。
このため、`Field`の操作の全てを提供しているわけではないが、その一部を提供する型について抽象的に説明できます。
例えば、自然数の型は加算及び乗算については閉じていますが、減算については必ずしも閉じていません。
そのため、この型は`Semiring`クラス（これは`Num`の上位クラスです）のインスタンスですが、`Ring`や`Field`のインスタンスではありません。

上位クラスについては、この章の後半で詳しく説明します。
しかし、全ての[数値型クラスの階層](https://a-guide-to-the-purescript-numeric-hierarchy.readthedocs.io/en/latest/introduction.html)（[チートシート](https://harry.garrood.me/numeric-hierarchy-overview/)）について述べるのはこの章の目的から外れています。
この内容に興味のある読者は`prelude`内の `Field`に関するドキュメントを参照してください。

### 半群とモノイド

`Semigroup`（半群）型クラスは、2つの値を連結する演算子 `append`を提供する型を示します。

```haskell
class Semigroup a where
  append :: a -> a -> a
```

文字列は普通の文字列連結について半群をなし、配列も同様です。
その他の標準的なインスタンスは`prelude`パッケージで提供されています。

以前に見た `<>`連結演算子は、 `append`の別名として提供されています。

（`prelude`パッケージで提供されている）`Monoid`型クラスには`mempty`という名前の空の値の概念があり、`Semigroup`型クラスを拡張します。

```haskell
class Semigroup m <= Monoid m where
  mempty :: m
```

ここでも文字列や配列はモノイドの簡単な例になっています。

ある型にとっての`Monoid`型クラスインスタンスとは、「空」の値から始めて新たな結果に組み合わせ、その型を持つ結果を*累算*する方法を記述するものです。
例えば、畳み込みを使って何らかのモノイドの値の配列を連結する関数を書けます。
PSCiで以下の通りです。

```haskell
> import Prelude
> import Data.Monoid
> import Data.Foldable

> foldl append mempty ["Hello", " ", "World"]
"Hello World"

> foldl append mempty [[1, 2, 3], [4, 5], [6]]
[1,2,3,4,5,6]
```

`prelude`パッケージにはモノイドと半群の多くの例を提供しており、以降もこれらを本書で扱っていきます。

### Foldable

`Monoid`型クラスが畳み込みの結果になるような型を示す一方、`Foldable`型クラスは畳み込みの元のデータとして使えるような型構築子を示しています。

また、 `Foldable`型クラスは配列や
`Maybe`などの幾つかの標準的なコンテナのインスタンスを含む`foldable-traversable`パッケージで提供されています。

`Foldable`クラスに属する関数の型シグネチャは、これまで見てきたものよりも少し複雑です。

```haskell
class Foldable f where
  foldr :: forall a b. (a -> b -> b) -> b -> f a -> b
  foldl :: forall a b. (b -> a -> b) -> b -> f a -> b
  foldMap :: forall a m. Monoid m => (a -> m) -> f a -> m
```

`f`を配列の型構築子として特殊化すると分かりやすいです。
この場合、任意の`a`について`f a`を`Array
a`に置き換えられますが、そうすると`foldl`と`foldr`の型が、最初に配列に対する畳み込みで見た型になると気付きます。

`foldMap`についてはどうでしょうか。
これは `forall a m. Monoid m => (a -> m) -> Array a -> m`になります。
この型シグネチャでは、型`m`が`Monoid`型クラスのインスタンスであれば、返り値の型として任意に選べると書かれています。
配列の要素をそのモノイドの値へと変える関数を与えられれば、そのモノイドの構造を利用して配列上で累算し、1つの値にして返せます。

それではPSCiで `foldMap`を試してみましょう。

```text
> import Data.Foldable

> foldMap show [1, 2, 3, 4, 5]
"12345"
```

ここでは文字列用のモノイドと`show`関数を選んでいます。
前者は文字列を連結するもので、後者は`Int`を文字列として書き出すものです。
そうして数の配列を渡すと、それぞれの数を`show`して1つの文字列へと連結した結果を得ました。

しかし畳み込み可能な型は配列だけではありません。
`foldable-traversable`では`Maybe`や`Tuple`のような型にも`Foldable`インスタンスが定義されており、`lists`のような他のライブラリでは、各々のデータ型に対して`Foldable`インスタンスが定義されています。
`Foldable`は*順序付きコンテナ*の概念を見据えたものなのです。

### 関手と型クラス則

PureScriptでは、副作用を伴う関数型プログラミングのスタイルを可能にするための型クラスの集まりも定義されています。
`Functor`や`Applicative`、`Monad`といったものです。
これらの抽象化については後ほど本書で扱いますが、ここでは`Functor`型クラスの定義を見てみましょう。
既に`map`関数の形で見たものです。

```haskell
class Functor f where
  map :: forall a b. (a -> b) -> f a -> f b
```

`map`関数（別名`<$>`）は関数をそのデータ構造まで「持ち上げる」(lift) ことができます。
ここで「持ち上げ」という言葉の具体的な定義は問題のデータ構造に依りますが、既に幾つかの単純な型についてその動作を見てきました。

```text
> import Prelude

> map (\n -> n < 3) [1, 2, 3, 4, 5]
[true, true, false, false, false]

> import Data.Maybe
> import Data.String (length)

> map length (Just "testing")
(Just 7)
```

`map`演算子は様々な構造の上でそれぞれ異なる挙動をしますが、 `map`演算子の意味はどのように理解すればいいのでしょうか。

直感的には、 `map`演算子はコンテナのそれぞれの要素へ関数を適用し、その結果から元のデータと同じ形状を持った新しいコンテナを構築するものとできます。
しかし、この着想を精密にするにはどうしたらいいでしょうか。

`Functor`の型クラスのインスタンスは、*関手則*と呼ばれる法則を順守するものと期待されています。

- `map identity xs = xs`
- `map g (map f xs) = map (g <<< f) xs`

最初の法則は _恒等射律_ (identity law)
です。これは、恒等関数（引数を変えずに返す関数）をその構造まで持ち上げると、元の構造をそのまま返すという意味です。恒等関数は入力を変更しませんから、これは理にかなっています。

第2の法則は*合成律*です。
構造を1つの関数で写してから2つめの関数で写すのは、2つの関数の合成で構造を写すのと同じだ、と言っています。

「持ち上げ」の一般的な意味が何であれ、データ構造に対する持ち上げ関数の正しい定義はこれらの法則に従っていなければなりません。

標準の型クラスの多くには、このような法則が付随しています。
一般に、型クラスに与えられた法則は、型クラスの関数に構造を与え、普遍的にインスタンスについて調べられるようにします。
興味のある読者は、既に見てきた標準の型クラスに属する法則について調べてみてもよいでしょう。

### インスタンスの導出

インスタンスを手作業で描く代わりに、ほとんどの作業をコンパイラにさせることができます。
この[型クラス導出手引き](https://github.com/purescript/documentation/blob/master/guides/Type-Class-Deriving.md)を見てください。
そちらの情報が以下の演習を解く手助けになることでしょう。

## 演習

（簡単）次のnewtypeは複素数を表します。

```haskell
{{#include ../exercises/chapter6/test/no-peeking/Solutions.purs:Complex}}
```

1. （簡単）`Complex`に`Show`インスタンスを定義してください。
   出力の形式はテストで期待される形式と一致させてください（例：`1.2+3.4i`、`5.6-7.7i`など）。

2. （簡単）`Eq`インスタンスを`Complex`に導出してください。
   *補足*：代わりにこのインスタンスを手作業で書いてもよいですが、しなくていいのになぜすることがありましょう。

3. （普通）`Semiring`インタンスを`Complex`に定義してください。
   *補足*：[`Data.Newtype`](https://pursuit.purescript.org/packages/purescript-newtype/docs/Data.Newtype)の`wrap`と`over2`を使ってより簡潔な解答を作れます。
   もしそうするのでしたら、`Data.Newtype`から`class
   Newtype`をインポートしたり、`Newtype`インスタンスを`Complex`に導出したりする必要も出てくるでしょう。

4. （簡単）（`newtype`を介して）`Ring`インスタンスを`Complex`に導出してください。
   *補足*：代わりにこのインスタンスを手作業で書くこともできますが、そう手軽にはできません。

    以下は第4章からの`Shape`のADTです。

    ```haskell
    {{#include ../exercises/chapter6/test/no-peeking/Solutions.purs:Shape}}
    ```

5. （普通）（`Generic`を介して）`Show`インスタンスを`Shape`に導出してください。
   コードの量はどのくらいになりましたか。
   また、第4章の`showShape`と比較して`String`の出力はどうなりましたか。
   *手掛かり*：[型クラス導出](https://github.com/purescript/documentation/blob/master/guides/Type-Class-Deriving.md)手引きの[`Generic`から導出する](https://github.com/purescript/documentation/blob/master/guides/Type-Class-Deriving.md#deriving-from-generic)節を見てください。

## 型クラス制約

型クラスを使うと、関数の型に制約を加えられます。
例を示しましょう。
`Eq`型クラスインスタンスを使って定義された等値性を使って、3つの値が等しいかどうかを調べる関数を書きたいとします。

```haskell
threeAreEqual :: forall a. Eq a => a -> a -> a -> Boolean
threeAreEqual a1 a2 a3 = a1 == a2 && a2 == a3
```

この型宣言は `forall`を使って定義された通常の多相型のようにも見えます。
しかし、二重線矢印 `=>`で型の残りの部分から区切られた、型クラス制約 (type class constraint) `Eq a`があります。

インポートされたモジュールのどれかに `a`に対する `Eq`インスタンスが存在するなら、どんな型 `a`を選んでも
`threeAsEqual`を呼び出すことができる、とこの型は言っています。

制約された型には複数の型クラスインスタンスを含めることができますし、インスタンスの型は単純な型変数に限定されません。 `Ord`と
`Show`のインスタンスを使って2つの値を比較する例を次に示します。

```haskell
showCompare :: forall a. Ord a => Show a => a -> a -> String
showCompare a1 a2 | a1 < a2 =
  show a1 <> " is less than " <> show a2
showCompare a1 a2 | a1 > a2 =
  show a1 <> " is greater than " <> show a2
showCompare a1 a2 =
  show a1 <> " is equal to " <> show a2
```

`=>`シンボルを複数回使って複数の制約を指定できることに注意してください。
複数の引数のカリー化された関数を定義するのと同様です。
しかし、2つの記号を混同しないように注意してください。

- `a -> b`は _型_ `a`から _型_ `b`への関数の型を表します。
- 一方で、`a => b`は _制約_ `a`を型`b`に適用します。

PureScriptコンパイラは、型の注釈が提供されていない場合、制約付きの型を推論しようとします。これは、関数に対してできる限り最も一般的な型を使用したい場合に便利です。

PSCiで `Semiring`のような標準の型クラスの何れかを使って、このことを試してみましょう。

```text
> import Prelude

> :type \x -> x + x
forall (a :: Type). Semiring a => a -> a
```

ここで、この関数に`Int -> Int`または`Number -> Number`と註釈を付けることはできます。
しかし、PSCiでは最も一般的な型が`Semiring`で動作することが示されています。
こうすると`Int`と`Number`の両方で関数を使えます。

## インスタンスの依存関係

制約された型を使うと関数の実装が型クラスインスタンスに依存できるように、型クラスインスタンスの実装は他の型クラスインスタンスに依存できます。これにより、型を使ってプログラムの実装を推論するという、プログラム推論の強力な形式を提供します。

`Show`型クラスを例に考えてみましょう。
それぞれの要素を `show`する方法がある限り、その要素の配列を `show`する型クラスインスタンスを書くことができます。

```haskell
instance Show a => Show (Array a) where
  ...
```

型クラスインスタンスが複数の他のインスタンスに依存する場合、括弧で囲んでそれらのインスタンスをコンマで区切り、それを`=>`シンボルの左側に置くことになります。

```haskell
instance (Show a, Show b) => Show (Either a b) where
  ...
```

これらの2つの型クラスインスタンスは `prelude`ライブラリにあります。

プログラムがコンパイルされると、`Show`の正しい型クラスのインスタンスは `show`の引数の推論された型に基づいて選ばれます。
選択されたインスタンスが沢山のそうしたインスタンスの関係に依存しているかもしれませんが、このあたりの複雑さに開発者が関与することはありません。

## 演習

1. （簡単）以下の宣言では型 `a`の要素の空でない配列の型を定義しています。

    ```haskell
    {{#include ../exercises/chapter6/test/no-peeking/Solutions.purs:NonEmpty}}
    ```

    `Eq a`と`Eq (Array a)`のインスタンスを再利用し、型`NonEmpty`に`Eq`インスタンスを書いてください。
    *補足*：代わりに`Eq`インスタンスを導出できます。

1. （普通）`Array`の`Semigroup`インスタンスを再利用して、`NonEmpty`への`Semigroup`インスタンスを書いてください。

1. （普通）`NonEmpty`に`Functor`インスタンスを書いてください。

1. （普通）`Ord`のインスタンス付きの任意の型`a`が与えられているとすると、新しくそれ以外のどんな値よりも大きい「無限の」値を付け加えられます。

    ```haskell
    {{#include ../exercises/chapter6/test/no-peeking/Solutions.purs:Extended}}
    ```

    `a`への`Ord`インスタンスを再利用して、`Extended a`に`Ord`インスタンスを書いてください。

1. （難しい）`NonEmpty`に`Foldable`インスタンスを書いてください。
   *手掛かり*：配列への`Foldable`インスタンスを再利用してください。

1. （難しい）順序付きコンテナを定義する（そして `Foldable`のインスタンスを持っている）ような型構築子
   `f`が与えられたとき、追加の要素を先頭に含める新たなコンテナ型を作れます。

    ```haskell
    {{#include ../exercises/chapter6/test/no-peeking/Solutions.purs:OneMore}}
    ```

   このコンテナ `OneMore f`もまた順序を持っています。
   ここで、新しい要素は任意の `f`の要素よりも前にきます。
   この `OneMore f`の `Foldable`インスタンスを書いてみましょう。

    ```haskell
    {{#include ../exercises/chapter6/test/no-peeking/Solutions.purs:OneMore_Foldable}}
      ...
    ```

1. （普通）`nubEq`関数を使い、配列から重複する`Shape`を削除する`dedupShapes :: Array Shape -> Array
   Shape`関数を書いてください。

1. （普通）`dedupShapesFast`関数を書いてください。
   `dedupShapes`とほぼ同じですが、より効率の良い`nub`関数を使います。

## 多変数型クラス

型クラスが引数として1つの型だけを取れるのかというと、そうではありません。
その場合がほとんどですが、型クラスは*ゼロ個以上の*型変数を持てます。

それでは2つの型引数を持つ型クラスの例を見てみましょう。

```haskell
module Stream where

import Data.Array as Array
import Data.Maybe (Maybe)
import Data.String.CodeUnits as String

class Stream stream element where
  uncons :: stream -> Maybe { head :: element, tail :: stream }

instance Stream (Array a) a where
  uncons = Array.uncons

instance Stream String Char where
  uncons = String.uncons
```

この `Stream`モジュールでは、要素のストリームのような型を示すクラス `Stream`が定義されています。
`uncons`関数を使ってストリームの先頭から要素を取り出すことができます。

`Stream`型クラスは、ストリーム自身の型だけでなくその要素の型も型変数として持っていることに注意してください。これによって、ストリームの型が同じでも要素の型について異なる型クラスインスタンスを定義できます。

このモジュールでは2つの型クラスインスタンスが定義されています。
`uncons`がパターン照合で配列の先頭の要素を取り除くような配列のインスタンスと、文字列から最初の文字を取り除くような文字列のインスタンスです。

任意のストリーム上で動作する関数を記述できます。
例えば、ストリームの要素に基づいて `Monoid`に結果を累算する関数は次のようになります。

```haskell
import Prelude
import Data.Monoid (class Monoid, mempty)

foldStream :: forall l e m. Stream l e => Monoid m => (e -> m) -> l -> m
foldStream f list =
  case uncons list of
    Nothing -> mempty
    Just cons -> f cons.head <> foldStream f cons.tail
```

PSCiで使って、異なる `Stream`の型や異なる `Monoid`の型について `foldStream`を呼び出してみましょう。

## 関数従属性

多変数型クラスは非常に便利ですが、紛らわしい型や型推論の問題にも繋がります。
単純な例として、上記で与えられた`Stream`クラスを使い、ストリームに対して汎用的な`tail`関数を書くことを考えてみましょう。

```haskell
genericTail xs = map _.tail (uncons xs)
```

これはやや複雑なエラー文言を出力します。

```text
The inferred type

  forall stream a. Stream stream a => stream -> Maybe stream

has type variables which are not mentioned in the body of the type. Consider adding a type annotation.
```

エラーは、 `genericTail`関数が `Stream`型クラスの定義で言及された
`element`型を使用しないので、その型は未解決のままであることを指しています。

更に残念なことに、特定の型のストリームに`genericTail`を適用できません。

```text
> map _.tail (uncons "testing")

The inferred type

  forall a. Stream String a => Maybe String

has type variables which are not mentioned in the body of the type. Consider adding a type annotation.
```

ここでは、コンパイラが `streamString`インスタンスを選択することを期待しています。
結局のところ、 `String`は `Char`のストリームであり、他の型のストリームであってはなりません。

コンパイラは自動的にそう推論できず、`streamString`インスタンスに目が向きません。
しかし、型クラス定義に手掛かりを追加すると、コンパイラを補助できます。

```haskell
class Stream stream element | stream -> element where
  uncons :: stream -> Maybe { head :: element, tail :: stream }
```

ここで、 `stream -> element`は _関数従属性_ (functional dependency) と呼ばれます。関数従属性は、多変数型クラスの型引数間の関数関係を宣言します。この関数の依存関係は、ストリーム型から（一意の）要素型への関数があることをコンパイラに伝えるので、コンパイラがストリーム型を知っていれば要素の型を割り当てられます。

この手掛かりがあれば、コンパイラが上記の汎用的な尾鰭関数の正しい型を推論するのに充分です。

```text
> :type genericTail
forall (stream :: Type) (element :: Type). Stream stream element => stream -> Maybe stream

> genericTail "testing"
(Just "esting")
```

多変数の型クラスを使用して何らかのAPIを設計する場合、関数従属性が便利なことがあります。

## 型変数のない型クラス

ゼロ個の型変数を持つ型クラスさえも定義できます。
これらは関数に対するコンパイル時の表明に対応しており、型システム内においてそのコードの大域的な性質を把握できます。

重要な一例として、前に部分関数についてお話しした際に見た`Partial`クラスがあります。
`Data.Array.Partial`に定義されている関数`head`と`tail`を例に取りましょう。
この関数は配列の先頭と尾鰭を`Maybe`に包むことなく取得できます。
そのため配列が空のときに失敗する可能性があります。

```haskell
head :: forall a. Partial => Array a -> a

tail :: forall a. Partial => Array a -> Array a
```

`Partial`モジュールの `Partial`型クラスのインスタンスを定義していないことに注意してください。
こうすると目的を達成できます。
このままの定義では `head`関数を使用しようとすると型エラーになるのです。

```text
> head [1, 2, 3]

No type class instance was found for

  Prim.Partial
```

代わりに、これらの部分関数を利用する全ての関数で `Partial`制約を再発行できます。

```haskell
secondElement :: forall a. Partial => Array a -> a
secondElement xs = head (tail xs)
```

前章で見た`unsafePartial`関数を使用し、部分関数を通常の関数として（不用心に）扱うことができます。この関数は
`Partial.Unsafe`モジュールで定義されています。

```haskell
unsafePartial :: forall a. (Partial => a) -> a
```

`Partial`制約は関数の矢印の左側の括弧の中に現れますが、外側の `forall`では現れません。
つまり、 `unsafePartial`は部分的な値から通常の値への関数です。

```text
> unsafePartial head [1, 2, 3]
1

> unsafePartial secondElement [1, 2, 3]
2
```

## 上位クラス

インスタンスを別のインスタンスに依存させることによって型クラスのインスタンス間の関係を表現できるように、いわゆる*上位クラス*を使って型クラス間の関係を表現できます。

あるクラスのどんなインスタンスも、別のクラスのインスタンスである必要があるとき、後者の型クラスは前者の型クラスの上位クラスであるといいます。
そしてクラス定義で逆向きの太い矢印 (`<=`) を使って上位クラス関係を示します。

[既に上位クラスの関係の例を目にしました](#ord)。
`Eq`クラスは`Ord`の上位クラスですし、`Semigroup`クラスは`Monoid`の上位クラスです。
`Ord`クラスの全ての型クラスインスタンスについて、その同じ型に対応する `Eq`インスタンスが存在しなければなりません。
これは理に適っています。
多くの場合、`compare`関数が2つの値の大小を付けられないと報告した時は、同値であるかを判定するために`Eq`クラスを使うことでしょう。

一般に、下位クラスの法則が上位クラスの構成要素に言及しているとき、上位クラス関係を定義するのは筋が通っています。
例えば、任意の`Ord`と`Eq`のインスタンスの対について、もし2つの値が`Eq`インスタンスの下で同値であるなら、`compare`関数は`EQ`を返すはずだと推定するのは理に適っています。
言い換えれば、`a == b`が真であるのは`compare a b`が厳密に`EQ`に評価されるときなのです。
法則のレベルでのこの関係は、`Eq`と`Ord`の間の上位クラス関係の正当性を示します。

上位クラス関係を定義する別の理由となるのは、この2つのクラスの間に明白な "is-a" の関係があることです。
下位クラスの全ての構成要素は、上位クラスの構成要素でもあるということです。

## 演習

1. （普通）部分関数`unsafeMaximum :: Partial => Array Int -> Int`を定義してください。
   この関数は空でない整数の配列の最大値を求めます。
   `unsafePartial`を使ってPSCiで関数を試してください。
   *手掛かり*：`Data.Foldable`の`maximum`関数を使います。

1. （普通）次の `Action`クラスは、ある型の別の型での動作を定義する、多変数型クラスです。

    ```haskell
    {{#include ../exercises/chapter6/test/no-peeking/Solutions.purs:Action}}
    ```

   *動作*とは、他の型の値を変更する方法を決めるために使われるモノイドな値を記述する関数です。
   `Action`型クラスには2つの法則があります。

    - `act mempty a = a`
    - `act (m1 <> m2) a = act m1 (act m2 a)`

    空の動作を提供しても何も起こりません。
    そして2つの動作を連続で適用することは結合した動作を適用することと同じです。
    つまり、動作は`Monoid`クラスで定義される操作に倣っています。

   例えば自然数は乗算のもとでモノイドを形成します。

    ```haskell
    {{#include ../exercises/chapter6/test/no-peeking/Solutions.purs:Multiply}}

    {{#include ../exercises/chapter6/test/no-peeking/Solutions.purs:semigroupMultiply}}

    {{#include ../exercises/chapter6/test/no-peeking/Solutions.purs:monoidMultiply}}
    ```

    この動作を実装するインスタンスを書いてください。

    ```haskell
    {{#include ../exercises/chapter6/test/no-peeking/Solutions.purs:Multiply_Action}}
      ...
    ```

    インスタンスが上で挙げた法則を見たさなくてはならないことを思い出してください。

1. （難しい）`Action Multiply Int`のインスタンスを実装するには複数の方法があります。
   どれだけ思い付きますか。
   PureScriptは同じインスタンスの複数の実装を許さないため、元の実装を置き換える必要があるでしょう。
   *補足*：テストでは4つの実装を押さえています。

1. （普通）入力の文字列を何回か繰り返す`Action`インスタンスを書いてください。

    ```haskell
    {{#include ../exercises/chapter6/test/no-peeking/Solutions.purs:actionMultiplyString}}
      ...
    ```

    *手掛かり*：Pursuitでシグネチャが[`String -> Int -> String`](https://pursuit.purescript.org/search?q=String%20-%3E%20Int%20-%3E%20String)の補助関数を検索してください。
    なお、`String`は（`Monoid`のような）より汎用的な型として現れます。

    このインスタンスは上に挙げた法則を満たすでしょうか。

1. （普通）インスタンス `Action m a => Action m (Array a)`を書いてみましょう。
   ここで、 配列上の動作はそれぞれの要素を独立に実行するものとして定義されます。

1. （難しい）以下のnewtypeが与えらえているとき、`Action m (Self m)`のインスタンスを書いてください。
   ここでモノイド`m`はそれ自体が持つ`append`を用いて動作します。

    ```haskell
    {{#include ../exercises/chapter6/test/no-peeking/Solutions.purs:Self}}
    ```

    *補足*：テストフレームワークでは`Self`と`Multiply`型に`Show`と`Eq`インスタンスが必要になります。
    手作業でこれらのインスタンスを書いてもよいですし、[`derive newtype instance`](https://github.com/purescript/documentation/blob/master/language/Type-Classes.md#derive-from-newtype)と書くだけでコンパイラに取り仕切ってもらうこともできます。

1. （難しい）多変数型のクラス `Action`の引数は、何らかの関数従属性によって関連づけられるべきですか。
   なぜそうすべき、あるいはそうすべきでないでしょうか。
   *補足*：この演習にはテストがありません。

## ハッシュの型クラス

この最後の節では、章の残りを使ってデータ構造をハッシュ化するライブラリを作ります。

> なお、このライブラリは説明だけを目的としており、堅牢なハッシュ化の仕組みの提供は意図していません。

ハッシュ関数に期待される性質とはどのようなものでしょうか。

- ハッシュ関数は決定的でなくてはなりません。
  つまり、同じ値は同じハッシュコードに写さなければなりません。
- ハッシュ関数はいろいろなハッシュ値の集合で結果が一様に分布しなければなりません。

最初の性質はちゃんとした型クラスの法則のように見えます。
その一方で、2番目の性質はよりくだけた規約の条項のようなもので、PureScriptの型システムによって確実に強制できるようなものではなさそうです。
しかし、これは型クラスから次のような直感が得られるでしょう。

```haskell
{{#include ../exercises/chapter6/src/Data/Hashable.purs:Hashable}}
```

これに、 `a == b`ならば `hash a == hash b`を示唆するという関係性の法則が付随しています。

この節の残りの部分を費やして、`Hashable`型クラスに関連付けられているインスタンスと関数のライブラリを構築していきます。

決定的な方法でハッシュ値を結合する方法が必要になります。

```haskell
{{#include ../exercises/chapter6/src/Data/Hashable.purs:combineHashes}}
```

`combineHashes`関数は2つのハッシュ値を混ぜて結果を0-65535の間に分布します。

それでは、`Hashable`制約を使って入力の種類を制限する関数を書いてみましょう。
ハッシュ関数を必要とするよくある目的としては、2つの値が同じハッシュコードにハッシュ化されるかどうかを判定することです。
`hashEqual`関係はそのような機能を提供します。

```haskell
{{#include ../exercises/chapter6/src/Data/Hashable.purs:hashEqual}}
```

この関数はハッシュコードの等値性を利用したハッシュ同値性を定義するために`Data.Function`の
`on`関数を使っていますが、これはハッシュ同値性の宣言的な定義として読めるはずです。
つまり、それぞれの値が `hash`関数に渡されたあとで2つの値が等しいなら、それらの値は「ハッシュ同値」です。

原始型の `Hashable`インスタンスを幾つか書いてみましょう。
まずは整数のインスタンスです。
`HashCode`は実際には単なる梱包された整数なので、単純です。
`hashCode`補助関数を使えます。

```haskell
{{#include ../exercises/chapter6/src/Data/Hashable.purs:hashInt}}
```

パターン照合を使うと、`Boolean`値の単純なインスタンスも定義できます。

```haskell
{{#include ../exercises/chapter6/src/Data/Hashable.purs:hashBoolean}}
```

整数のインスタンスでは、`Data.Char`の `toCharCode`関数を使うと`Char`をハッシュ化するインスタンスを作成できます。

```haskell
{{#include ../exercises/chapter6/src/Data/Hashable.purs:hashChar}}
```

（要素型が `Hashable`のインスタンスでもあるならば）配列の要素に `hash`関数を
`map`してから、`combineHashes`関数を使って結果のハッシュを左側に畳み込むことで、配列のインスタンスを定義します。

```haskell
{{#include ../exercises/chapter6/src/Data/Hashable.purs:hashArray}}
```

既に書いたものより単純なインスタンスを使用して、新たなインスタンスを構築しているやり方に注目してください。
`String`を`Char`の配列に変換し、この新たな`Array`インスタンスを使って`String`のインスタンスを定義しましょう。

```haskell
{{#include ../exercises/chapter6/src/Data/Hashable.purs:hashString}}
```

これらの `Hashable`インスタンスが先ほどの型クラスの法則を満たしていることを証明するにはどうしたらいいでしょうか。
同じ値が等しいハッシュコードを持っていることを確認する必要があります。
`Int`、`Char`、`String`、`Boolean`のような場合は単純です。
`Eq`の意味では同じ値でも厳密には同じではない、というような型の値は存在しないからです。

もっと面白い型についてはどうでしょうか。
`Array`インスタンスの型クラスの法則を証明するにあたっては、配列の長さに関する帰納を使えます。
長さゼロの唯一の配列は `[]`です。
配列の `Eq`の定義により、任意の2つの空でない配列は、それらの先頭の要素が同じで配列の残りの部分が等しいとき、またその時に限り等しくなります。
この帰納的な仮定により、配列の残りの部分は同じハッシュ値を持ちますし、もし `Hashable
a`インスタンスがこの法則を満たすなら、先頭の要素も同じハッシュ値を持つことがわかります。
したがって、2つの配列は同じハッシュ値を持ち、`Hashable (Array a)`も同様に型クラス法則に従います。

この章のソースコードには、 `Maybe`と `Tuple`型のインスタンスなど、他にも `Hashable`インスタンスの例が含まれています。

## 演習

 1. （簡単）PSCiを使って、定義した各インスタンスのハッシュ関数をテストしてください。
    *補足*：この演習には単体試験がありません。
 1. （普通）関数`arrayHasDuplicates`を書いてください。
    この関数はハッシュと値の同値性に基づいて配列が重複する要素を持っているかどうかを調べます。
    まずハッシュ同値性を`hashEqual`関数で確認し、それからもし重複するハッシュの対が見付かったら`==`で値の同値性を確認してください。
    *手掛かり*：`Data.Array`の `nubByEq`関数はこの問題をずっと簡単にしてくれるでしょう。
 1. （普通）型クラスの法則を満たす、次のnewtypeの `Hashable`インスタンスを書いてください。

    ```haskell
    {{#include ../exercises/chapter6/test/no-peeking/Solutions.purs:Hour}}

    {{#include ../exercises/chapter6/test/no-peeking/Solutions.purs:eqHour}}
    ```

   newtypeの `Hour`とその `Eq`インスタンスは、12を法とする整数の型を表します。
   したがって、例えば1と13は等しいと見なされます。
   そのインスタンスが型クラスの法則を満たしていることを証明してください。
 1. （難しい）`Maybe`、`Either`そして`Tuple`への`Hashable`インスタンスについて型クラスの法則を証明してください。
    *補足*：この演習にテストはありません。

## まとめ

この章では*型クラス*を導入しました。
型クラスは型に基づく抽象化で、コードの再利用のために強力な形式化ができます。
PureScriptの標準ライブラリから標準の型クラスを幾つか見てきました。
また、ハッシュ値を計算するための型クラスに基づく独自のライブラリを定義しました。

この章では型クラス法則も導入しましたが、これは抽象化に型クラスを使うコードについての性質を証明する手法でした。
型クラス法則は*等式推論*と呼ばれる、より大きな分野の一部です。
そちらではプログラミング言語の性質と型システムがプログラムを論理的に追究するために使われています。
これは重要な考え方で、本書では今後あらゆる箇所で立ち返る話題となるでしょう。
