# 第7章 値オブジェクト — 不変性とプリミティブ強迫の解消

DDDの戦術的設計でいちばん費用対効果が高いのが値オブジェクトです。この章では「同一性ではなく値で区別する」という考え方を土台に、string や float だらけのコード（プリミティブ強迫）をドメインの言葉を話す型に置き換え、振る舞いまで持たせるところまでを Go のコード例で身につけます。
L6 のエンティティ(ID で区別するオブジェクト)と対になる部品なので、L6 を先に読んでおくと比べながら学べます。

## 1. 値オブジェクトとは — 同一性ではなく値で区別する

### 「どの札か」は気にしない

意味があるのは「1000円という値」であって「どの1000円札か」ではない、という対象があります。注文の合計金額「1000円」は、別の計算で出てきた「1000円」と区別する意味がありません。これが**値オブジェクト (Value Object)** の発想です。一方「会員の田中さん(ID u-001)」は、同姓同名の別人と入れ替わったら大問題です。こちらは値ではなく**同一性 (identity)** で区別する存在で、L6 で扱った**エンティティ**にあたります。

| 対象 | 中身が同じものと入れ替えてよいか | 分類 |
|---|---|---|
| 金額 1000円 | よい(どの 1000円 でも同じ意味) | 値オブジェクト |
| 配送先「東京都港区1-2-3」 | よい | 値オブジェクト |
| 会員 u-001 | だめ(同姓同名でも別人) | エンティティ |

### 値オブジェクトの3つの性質

| 性質 | 意味 | 例 |
|---|---|---|
| 等価性 (equality) | **すべての属性が等しければ「同じ」**。ID は持たない | 1000円(JPY) と 1000円(JPY) は同じ。1000円 と 1000ドル は違う |
| 不変性 (immutability) | 一度作ったら中身を書き換えない | 1000円 の金額を後から 2000 に書き換えることはしない |
| 交換可能性 (replaceability) | 「変更」は中身の書き換えではなく**新しい値での置き換え** | 送料が変わったら、新しい「500円」を作って差し替える |

```go
type Money struct {
    amount   int64    // 最小通貨単位(円なら1円)
    currency Currency
}

func (m Money) Equals(other Money) bool {
    return m.amount == other.amount && m.currency == other.currency
}
```

| 部分 | 意味 |
|---|---|
| `amount int64` | 金額。`int64` は大きな整数まで入る整数型。「最小通貨単位」は、円なら1円、ドルなら1セントを1として数えること(1.50ドルなら 150 と持つ) |
| `currency Currency` | 通貨の種類(`"JPY"` や `"USD"`)。Currency は通貨を表す独自の型(L6 ch02 の `type UserID string` と同じ作り方) |
| `func (m Money) Equals(other Money) bool` | 2つの Money が等しいかを返すメソッド。レシーバが `(m Money)` と `*` なしなのは、値オブジェクトは書き換えないから(違いは ch03 で見ます) |
| `m.amount == other.amount && m.currency == other.currency` | 金額**と**通貨が両方等しければ true。ID は比べない(比べる ID がそもそもない) |

L6 の User.Equals が ID だけを比べたのとは正反対で、値オブジェクトはすべての属性を比べます。動かして確かめます。

```go
package main

import "fmt"

type Currency string

type Money struct {
    amount   int64
    currency Currency
}

func (m Money) Equals(other Money) bool {
    return m.amount == other.amount && m.currency == other.currency
}

func main() {
    a := Money{amount: 1000, currency: "JPY"}
    b := Money{amount: 1000, currency: "JPY"} // 別に作った、同じ中身の値
    c := Money{amount: 1000, currency: "USD"}

    fmt.Println("a と b:", a.Equals(b))
    fmt.Println("a と c:", a.Equals(c))
}
```

```text
a と b: true
a と c: false
```

Python では、`dataclass`(属性を並べるだけでクラスを作れる標準の道具)に `frozen=True` を付けると、等価性と不変性をまとめて持つクラスになります。

```python
from dataclasses import dataclass


@dataclass(frozen=True)  # frozen=True で「作ったら書き換えられない」クラスになる
class Money:
    amount: int
    currency: str


a = Money(1000, "JPY")
b = Money(1000, "JPY")
print("a == b:", a == b)          # 中身が同じなら等しい
print("a is b:", a is b)          # 同じ1個の実体か(別々に作ったので違う)
try:
    a.amount = 2000               # 書き換えようとすると…
except Exception as e:
    print(type(e).__name__, e)
```

```text
a == b: True
a is b: False
FrozenInstanceError cannot assign to field 'amount'
```

`a` と `b` は別々に作った2つの実体ですが、中身が同じなので `==` では等しいと判定されます。書き換えようとすると `FrozenInstanceError`(凍結されたものは変更できない、というエラー)になります。

### 不変だと何がうれしいか

- 複数の場所から**共有しても安全**(誰かに書き換えられる心配がない)
- **並行処理**(複数の処理を同時に動かすこと)で**ロック**(ある処理がデータを使っている間、他の処理を待たせる仕組み)が不要(読むだけなら、同時に読んでも結果が食い違わない)
- 生成時の検証だけで常に妥当な状態を保てるので、**テストが入出力の比較だけで書ける**

たとえば2つの注文が同じ「送料500円」の値を共有していて、片方の処理がそれを 0円 に書き換えられたら、もう片方の注文の送料まで 0円 になってしまいます。不変ならこの事故は起きません。

### 値オブジェクトの候補を見つける

「量」や「記述」を表す概念は値オブジェクトの第一候補です: 金額、数量、期間、住所、メールアドレス、電話番号、座標、温度。判断に迷ったら「**それを追跡したいのか、値が合っていればよいのか**」と自問します。配送先住所は、顧客が引っ越しても過去の注文の配送先が変わってはいけないので「その時点の値」を写し取る値オブジェクトがぴったりです。

| 時点 | 会員の現住所 | 注文 o-001 の配送先(値オブジェクトとして複製) |
|---|---|---|
| 10月 注文時 | 東京都港区1-2-3 | 東京都港区1-2-3 |
| 11月 引っ越し後 | 大阪府大阪市4-5-6 | 東京都港区1-2-3(変わらない) |

💡 コツ: 迷ったら**まず値オブジェクトにできないか**を考えましょう。エンティティは ID の採番・ライフサイクル管理・等価性の扱いなど維持コストが高く、DDD の提唱者エリック・エヴァンスも「可能な限り値オブジェクトを使う」ことを勧めています。

### 確認問題

**問1.** ECサイトで、注文には配送先住所を保存します。顧客が後から自分の住所を変更しても、出荷済みの注文の配送先表示は変わってはいけません。この「配送先住所」のモデリングとして最も適切なのはどれでしょう？

1. 住所エンティティを作り、注文から住所IDで参照する
2. 注文ごとに住所の値オブジェクトを複製して保持する
3. 顧客エンティティの現住所フィールドを注文から直接参照する
4. 住所変更のたびに全注文の住所カラムを更新するバッチを用意する

<details><summary>答えと解説</summary>

正解: **2**

注文に必要なのは「注文時点の配送先という値」であり、どの住所レコードかという同一性ではありません。値オブジェクトとして注文側に写し取れば、顧客の住所変更の影響を自然に受けなくなります。住所IDで参照する案や顧客の現住所を直接参照する案は、顧客が引っ越すと過去の注文の表示まで変わってしまう欠陥があります。全注文を更新するバッチは問題を逆方向に悪化させる発想です。「追跡したいのか、値が合っていればよいのか」が判断のひっかけどころです。

</details>

**問2.** 値オブジェクトの等価性 (equality) の説明として正しいものはどれでしょう？

1. 同じメモリアドレスを指しているときだけ等しい
2. ID フィールドが一致すれば他の属性が違っても等しい
3. すべての属性の値が等しければ等しいとみなす
4. 生成された時刻が同じであれば等しいとみなす

<details><summary>答えと解説</summary>

正解: **3**

値オブジェクトは ID を持たず、保持する属性の値がすべて等しければ「同じ」とみなします。1万円札どうしの関係と同じです。ID で比較するのはエンティティの等価性で、ここを取り違えるのが最もよくある混同です。メモリアドレスでの比較は参照の同一性であってドメイン上の等価性ではなく、生成時刻は値オブジェクトの属性ですらありません。

</details>

**問3.** 値オブジェクトを不変 (immutable) にする利点として、適切で**ない**ものはどれでしょう？

1. 複数の集約から共有しても書き換えによる事故が起きない
2. 並行処理で読み取り時にロックが不要になる
3. 生成時に検証しておけば常に妥当な状態であると信頼できる
4. データベースへの書き込みが常に高速になる

<details><summary>答えと解説</summary>

正解: **4**

不変性の利点は「安全な共有」(複数の集約から共有しても安全。集約は L8)「並行処理での安全性」「生成時検証による常時妥当性」といった、状態変化の心配をなくすことにあります。データベースの書き込み速度は値オブジェクトの不変性とは無関係で、永続化の設計(第11章のリポジトリ)で決まる話です。もっともらしい技術的効能が混ざっていたら、「それは不変性から論理的に導けるか」で切るのがコツです。

</details>

**問4.** 人事給与システムを設計しています。「社員」をエンティティと値オブジェクトのどちらにするか、判断として最も適切なのはどれでしょう？

1. 氏名・生年月日が同じ社員は同一人物とみなしてよいので値オブジェクトにする
2. 昇給や異動の履歴を特定の個人として追跡する必要があるのでエンティティにする
3. 社員データは頻繁に変わるので値オブジェクトにする
4. テーブルの行数が多いのでエンティティにする

<details><summary>答えと解説</summary>

正解: **2**

給与計算では「属性が変わっても同じ人」を追い続ける必要があります。氏名が変わっても、部署が変わっても田中さんは田中さんです。これはまさに同一性 (identity) で区別する必要がある、エンティティの典型です。同姓同名・同じ生年月日の別人は実在し得るため属性での同一視は危険です。「頻繁に変わるから値」「行数が多いからエンティティ」は、どちらもモデリングと無関係な基準を持ち込むひっかけです。

</details>

**問5.** 値オブジェクトである Money の金額を「変更」したいとき、DDD として正しいやり方はどれでしょう？

1. 新しい金額を持つ Money を生成し、持ち主のフィールドを丸ごと置き換える
2. Money に SetAmount メソッドを追加してフィールドを書き換える
3. amount フィールドを公開してプロパティとして直接代入する
4. リフレクションで非公開フィールドを書き換えるユーティリティを作る

<details><summary>答えと解説</summary>

正解: **1**

値オブジェクトの「変更」は交換 (replace) です。新しい値を生成して持ち主側の参照を置き換えれば、不変性を保ったまま状態を進められます。SetAmount や公開フィールドへの代入は不変性を壊し、その値を共有しているすべての場所に影響が波及します。リフレクション(実行中に型やフィールドを調べて操作する仕組み)での書き換えは論外で、生成時検証もすり抜けます。「不変なのにどうやって変えるの?」という疑問には「変えずに取り替える」が答えです。

</details>

**問6.** 値オブジェクト Money の Equals が「金額と通貨がどちらも等しければ true」を返すとき、`Money{1000, "JPY"}.Equals(Money{1000, "USD"})` の結果はどれでしょう？

1. true — 金額が同じだから
2. エラー — ID がないので比較できないから
3. true — どちらも Money 型だから
4. false — 通貨が違うから

<details><summary>答えと解説</summary>

正解: **4**

値オブジェクトの等価性は「すべての属性が等しいか」で判定します。金額は同じ 1000 でも、通貨が JPY と USD で違うので false です。金額だけ、または型が同じというだけで等しいとするのは誤りです。値オブジェクトはそもそも ID を持たず、属性の比較だけで等しさが決まるので、ID がなくても比較できます。

</details>

## 2. プリミティブ強迫 (Primitive Obsession) の解消

### string と float だらけのコードの何が怖いか

**プリミティブ型**は、言語に最初から用意されている基本の型です。Go なら `string`(文字列)、`int`/`int64`(整数)、`float64`(小数)、`bool`(true/false)などです。**プリミティブ強迫 (Primitive Obsession)** とは、ドメインの概念をこうした基本型のままで引き回してしまう癖のことです。

```go
// 悪い例: 全部プリミティブ
func Transfer(fromID string, toID string, amount float64, currency string) error
```

これは関数の**シグネチャ**(関数名・引数の型・戻り値の型だけを書いた、関数の外から見た形)です。「送金元の ID・送金先の ID・金額・通貨を受け取り、失敗したら error を返す」と読めます。このシグネチャには3つの地雷が埋まっています。

1. **引数の取り違えがコンパイルを通る** — `Transfer(toID, fromID, ...)` と逆に渡してもエラーになりません。どちらも string だからです(L6 ch02 で、型を分けると取り違えがコンパイルエラーになるのを見ました)
2. **不正値が混入する** — 負の金額、存在しない通貨コード、空文字の ID を誰も止めません
3. **ルールが散在する** — 「金額は0以上」のチェックが呼び出し側のあちこちにコピペされます

さらに金額を float64 で持つと、**丸め誤差**がお金の計算に紛れ込みます。コンピュータは小数を2進数で持つため、0.1 のような数をぴったり表せず、ごくわずかなずれが出るのです。

```go
package main

import "fmt"

func main() {
    var a float64 = 0.1
    var b float64 = 0.2
    fmt.Println(a + b)
    fmt.Println(a+b == 0.3)

    var yen1 int64 = 1999 // 整数(1円単位)で持てば誤差は出ない
    var yen2 int64 = 1600
    fmt.Println(yen1 + yen2)
}
```

```text
0.30000000000000004
false
3599
```

0.1 と 0.2 を足すと 0.3 ちょうどにならず、`== 0.3` の比較も false になります。整数で持てばこのずれは起きません。これが Money を「最小通貨単位の整数」で持つ理由です。

### Money 型を作る

```go
type Money struct {
    amount   int64    // 最小通貨単位で保持(丸め誤差の排除)
    currency Currency
}

func NewMoney(amount int64, c Currency) (Money, error) {
    if amount < 0 {
        return Money{}, errors.New("金額は0以上にしてください")
    }
    return Money{amount: amount, currency: c}, nil
}

func (m Money) Add(other Money) (Money, error) {
    if m.currency != other.currency {
        return Money{}, ErrCurrencyMismatch // 異なる通貨の加算を禁止
    }
    return Money{m.amount + other.amount, m.currency}, nil
}
```

| 部分 | 意味 |
|---|---|
| `func NewMoney(amount int64, c Currency) (Money, error)` | Money を作るコンストラクタ関数(L6 で見た `New〜` の慣習)。戻り値は Money とエラーの2つ |
| `if amount < 0 { return Money{}, errors.New(...) }` | 負の金額なら、空の Money(`Money{}` = 全フィールドがゼロ値)とエラーを返す |
| `return Money{amount: amount, currency: c}, nil` | 検証を通ったら Money を作り、エラーなし(nil)を返す |
| `func (m Money) Add(other Money) (Money, error)` | 2つの金額を足すメソッド。結果は新しい Money として返す |
| `if m.currency != other.currency` | 通貨が違えば足さずにエラー。ErrCurrencyMismatch は別の場所で `errors.New` で作ってあるエラー |
| `Money{m.amount + other.amount, m.currency}` | 足した金額で新しい Money を作る。`{値, 値}` はフィールド名を省略し、定義の順に値を並べる書き方 |

動かして確かめます。

```go
package main

import (
    "errors"
    "fmt"
)

type Currency string

var ErrCurrencyMismatch = errors.New("通貨が違う金額は足せません")

type Money struct {
    amount   int64
    currency Currency
}

func NewMoney(amount int64, c Currency) (Money, error) {
    if amount < 0 {
        return Money{}, errors.New("金額は0以上にしてください")
    }
    return Money{amount: amount, currency: c}, nil
}

func (m Money) Add(other Money) (Money, error) {
    if m.currency != other.currency {
        return Money{}, ErrCurrencyMismatch
    }
    return Money{m.amount + other.amount, m.currency}, nil
}

func main() {
    _, err := NewMoney(-500, "JPY")
    fmt.Println("マイナス:", err)

    subtotal, _ := NewMoney(3000, "JPY")
    fee, _ := NewMoney(500, "JPY")
    total, err := subtotal.Add(fee)
    fmt.Println("合計:", total.amount, total.currency, err)
    fmt.Println("足す前の小計:", subtotal.amount) // 自分は変わっていない

    usd, _ := NewMoney(10, "USD")
    _, err = subtotal.Add(usd)
    fmt.Println("円+ドル:", err)
}
```

```text
マイナス: 金額は0以上にしてください
合計: 3500 JPY <nil>
足す前の小計: 3000
円+ドル: 通貨が違う金額は足せません
```

| 結果の行 | 分かること |
|---|---|
| `マイナス: ...` | 負の金額の Money は作れない |
| `合計: 3500 JPY <nil>` | 3000円 + 500円 = 3500円。`<nil>` はエラーなし |
| `足す前の小計: 3000` | Add は新しい値を返すだけで、subtotal 自身は書き換えていない(不変) |
| `円+ドル: ...` | 通貨の違う足し算はエラーで止まる |

`_, err = subtotal.Add(usd)` で `:=` ではなく `=` を使っているのは、err がすでに作ってある変数だからです(`:=` は新しい変数を作るとき、`=` は既存の変数に入れ直すとき)。

### 不正な状態を表現できないようにする

```go
type EmailAddress struct {
    value string // 非公開: 検証を通った値しかこの世に存在できない
}

func NewEmailAddress(s string) (EmailAddress, error) {
    if !emailRe.MatchString(s) {
        return EmailAddress{}, fmt.Errorf("不正なメールアドレス: %q", s)
    }
    return EmailAddress{value: s}, nil
}
```

| 部分 | 意味 |
|---|---|
| `value string` | 先頭が小文字なので非公開(パッケージの外から読み書きできない。L6 ch03、詳しくは L8) |
| `emailRe` | メールアドレスの形を表す**正規表現**(文字列のパターンを表す書き方)。断片では定義が省略されている |
| `!emailRe.MatchString(s)` | s がパターンに合うかを調べる。`!` は「ではない」なので、合わなければ |
| `fmt.Errorf("不正なメールアドレス: %q", s)` | 値を埋め込んだエラーを作る。`%q` は s を `"..."` で囲んで埋め込む指定 |

省略されている emailRe を補って動かすと、次のとおりです(練習用のゆるいチェックで、本物のメールアドレスの規則はもっと複雑です)。

```go
package main

import (
    "fmt"
    "regexp"
)

// 練習用のゆるい形式チェック: 「@ の前後に文字があり、後ろに . を含む」
var emailRe = regexp.MustCompile(`^[^@\s]+@[^@\s]+\.[^@\s]+$`)

type EmailAddress struct {
    value string
}

func NewEmailAddress(s string) (EmailAddress, error) {
    if !emailRe.MatchString(s) {
        return EmailAddress{}, fmt.Errorf("不正なメールアドレス: %q", s)
    }
    return EmailAddress{value: s}, nil
}

func main() {
    e1, err := NewEmailAddress("taro@example.com")
    fmt.Println(e1.value, err)

    _, err = NewEmailAddress("taro.example.com")
    fmt.Println(err)
}
```

```text
taro@example.com <nil>
不正なメールアドレス: "taro.example.com"
```

`regexp.MustCompile(...)` は正規表現を使える形に変換する関数です。`@` のない `taro.example.com` は EmailAddress になれず、エラーになります。

フィールドを非公開にしてコンストラクタで検証すれば、**EmailAddress 型の値が存在する = 妥当である**が型レベルで保証されます。英語圏では make illegal states unrepresentable(不正な状態を表現できなくせよ)と呼ばれる原則です。使う側は「このメールアドレスは検証済みだろうか」と心配して再チェックする必要がなくなります。

💡 コツ: 何でもかんでもラップする(型で包む)のはやりすぎです。**ドメインのルール(検証・演算・比較)を持つ概念だけ**を型にしましょう。表示用の自由記述メモまで MemoText 型にしても、得られるものはほぼありません。

### 確認問題

**問1.** ECサイトの合計金額を float64 で計算していたところ、「合計 3,599.9999999 円」のような表示が報告されました。根本対策として最も適切なのはどれでしょう？

1. 表示の直前で四捨五入するユーティリティ関数を全画面に入れる
2. float64 を float32 に変更してメモリ効率も改善する
3. 最小通貨単位の整数と通貨コードを持つ Money 型を導入して計算を寄せる
4. 金額計算のたびに文字列へ変換してから数値に戻す

<details><summary>答えと解説</summary>

正解: **3**

浮動小数点の丸め誤差は2進数で10進小数を正確に表せないことが原因なので、最小単位の整数(円なら1円、ドルならセント)で持つのが根本対策です。Money 型に計算を寄せればルールも一箇所に集まります。表示直前の四捨五入は症状を隠すだけで、誤差は計算のたびに蓄積し続けます。float32 への変更は精度をさらに悪化させる逆走で、文字列経由の変換は遅いうえに問題を解決しません。

</details>

**問2.** `Transfer(fromID string, toID string, amount int64)` で送金元と送金先を逆に渡すバグが本番で発生しました。再発を最も効果的に防ぐ設計はどれでしょう？

1. 引数の順番をコードレビューのチェックリストに追加する
2. AccountID 型と Money 型を導入し、さらに送金を表す構造体や専用メソッドで意味を型に載せる
3. 関数コメントに「第1引数が送金元」と明記する
4. 引数名を fromAccountID / toAccountID にリネームする

<details><summary>答えと解説</summary>

正解: **2**

同じ string どうしの取り違えはコンパイラには見えません。専用の型を導入し、`from.TransferTo(to, money)` のように意味を型とメソッドに載せれば、取り違えの多くがコンパイルエラーか、そもそも書けない形になります。チェックリスト・コメント・リネームはいずれも人間の注意力頼みで、「人は必ず間違える」前提に立つと再発防止の効果は限定的です。レビュー強化がもっともらしく見えるのがひっかけですが、型で防げるものは型で防ぐのが先です。

</details>

**問3.** EmailAddress 型を導入する際、「不正な状態を表現できないようにする」ための実装として最も適切なのはどれでしょう？

1. フィールドを公開し、使う側が必要に応じて検証関数を呼ぶ規約にする
2. フィールドを非公開にし、検証つきコンストラクタだけが生成手段になるようにする
3. 検証は DB の CHECK 制約だけに任せて型はただの別名にする
4. 不正な値でも生成は許し、IsValid メソッドで後から判定できるようにする

<details><summary>答えと解説</summary>

正解: **2**

非公開フィールド+検証つきコンストラクタにすると「EmailAddress が存在する時点で妥当」が保証され、使う側は再検証から解放されます。使う側任せの規約や IsValid 方式は「検証を忘れた不正値」が型の皮をかぶって流通することを許してしまい、型を作った意味が半減します。DB 制約(CHECK 制約など、DB が保存時に値を検査する仕組み)は最後の砦としては有効ですが、エラーがアプリの奥深くで遅れて発覚するため、生成時点で止める設計の代わりにはなりません。

</details>

**問4.** Money 型で通貨の異なる加算 `jpy.Add(usd)` が呼ばれたときの振る舞いとして、最も適切な設計はどれでしょう？

1. その日の為替レートを自動取得して換算してから加算する
2. 通貨不一致のエラーを返し、換算は明示的な別のユースケースとして扱う
3. 金額部分だけを加算し、通貨は先に呼んだ側のものを採用する
4. パニック(クラッシュ)させて開発中に気づけるようにする

<details><summary>答えと解説</summary>

正解: **2**

異なる通貨の加算はドメインのルール違反なので、エラーとして明示的に扱うのが基本です。換算が必要なら「レートを指定して換算する」という別の明示的な操作にします。為替レートの自動取得は加算という純粋な計算に外部依存と暗黙の仕様を持ち込み、金額だけの加算は静かに間違ったお金を作る最悪の選択です。パニック(Go でプログラムを異常終了させる仕組み)は回復の機会を奪うため、入力起因で起こり得るエラーには不適切です。

</details>

**問5.** プリミティブ強迫の解消を進めるとき、値オブジェクト化の優先度が最も**低い**のはどれでしょう？

1. 検証ルールと等価判定が必要なメールアドレス
2. 通貨と丸めのルールを持つ金額
3. 管理画面にそのまま表示するだけの自由記述の備考欄
4. 開始が終了より前という制約を持つ予約期間

<details><summary>答えと解説</summary>

正解: **3**

値オブジェクト化の対価は「ドメインのルールを型に閉じ込められること」です。メールアドレス(形式検証)、金額(通貨・丸め)、期間(開始<終了の不変条件)はどれもルールを持つので型にする価値があります。自由記述の備考欄には守るべきルールがほぼなく、ラップしても得るものがありません。「すべてをラップするのが正しいDDD」と考えてしまうのがやりすぎ側のひっかけです。

</details>

**問6.** Go で `var a float64 = 0.1` と `var b float64 = 0.2` を足し、`fmt.Println(a + b)` を実行しました。表示されるものとして正しいのはどれでしょう？

1. 0.3
2. コンパイルエラー
3. 0.2
4. 0.30000000000000004

<details><summary>答えと解説</summary>

正解: **4**

float64 は小数を2進数で持つため、0.1 や 0.2 をぴったり表せず、足すと 0.30000000000000004 になります。そのため `a+b == 0.3` も false です。お金をこの型で計算すると、合計が 1円 ずれるような誤差が紛れ込みます。だから Money は「最小通貨単位(円なら1円)の整数」で持ちます。0.3 ちょうどになると思うのが典型的な落とし穴で、このコードはコンパイルエラーにもなりません。

</details>

## 3. 値オブジェクトに振る舞いを持たせる

### 値は「データの袋」ではない

プリミティブを型で包んだだけでは、まだ半分です。値オブジェクトの真価は**ドメインの計算や判断をメソッドとして持たせた**ときに出ます。

```go
type Period struct {
    start, end time.Time // start < end をコンストラクタで保証
}

// 判断: その日時はこの期間に含まれるか
func (p Period) Contains(t time.Time) bool {
    return !t.Before(p.start) && t.Before(p.end)
}

// 計算: 期間を延長した「新しい期間」を返す(自分は変えない)
func (p Period) ExtendBy(d time.Duration) Period {
    return Period{p.start, p.end.Add(d)}
}
```

| 部分 | 意味 |
|---|---|
| `start, end time.Time` | 開始と終了の日時。`time.Time` は Go 標準の「日時」の型。同じ型のフィールドはカンマで並べて書ける |
| `t.Before(p.start)` | t が開始より前なら true |
| `!t.Before(p.start)` | 「開始より前ではない」=開始と同じかそれより後 |
| `t.Before(p.end)` | 終了より前 |
| `Contains` の全体 | 開始 ≦ t < 終了 なら true。開始の瞬間は含み、終了の瞬間は含まない |
| `time.Duration` | 「時間の長さ」の型。`48 * time.Hour` で48時間 |
| `p.end.Add(d)` | 終了日時に d を足した新しい日時 |
| `Period{p.start, p.end.Add(d)}` | 開始はそのまま、終了だけ延ばした**新しい** Period を作って返す |

動かして確かめます。11月1日 0時 から 11月8日 0時 の直前までのセール期間を作ります。

```go
package main

import (
    "fmt"
    "time"
)

type Period struct {
    start, end time.Time
}

func (p Period) Contains(t time.Time) bool {
    return !t.Before(p.start) && t.Before(p.end)
}

func (p Period) ExtendBy(d time.Duration) Period {
    return Period{p.start, p.end.Add(d)}
}

// day は「2026年11月の d 日 0時0分」を作る練習用の関数
func day(d int) time.Time {
    return time.Date(2026, 11, d, 0, 0, 0, 0, time.UTC)
}

func main() {
    sale := Period{start: day(1), end: day(8)} // 11/1 0:00 〜 11/8 0:00 の直前まで

    fmt.Println("11/1 は含まれる?", sale.Contains(day(1)))
    fmt.Println("11/5 は含まれる?", sale.Contains(day(5)))
    fmt.Println("11/8 は含まれる?", sale.Contains(day(8)))

    longer := sale.ExtendBy(48 * time.Hour) // 2日延長
    fmt.Println("延長後の終わり:", longer.end.Format("01/02"))
    fmt.Println("元の終わり:", sale.end.Format("01/02"))
    fmt.Println("延長後なら 11/8 は?", longer.Contains(day(8)))
}
```

```text
11/1 は含まれる? true
11/5 は含まれる? true
11/8 は含まれる? false
延長後の終わり: 11/10
元の終わり: 11/08
延長後なら 11/8 は? true
```

`time.Date(年, 月, 日, 時, 分, 秒, ナノ秒, 時間帯)` は日時を作る関数、`Format("01/02")` は「月/日」の形の文字列にする指定です。ExtendBy の後も `sale` の終わりは 11/08 のままで、延長された期間は `longer` という別の値として返ってきています。

ポイントは2つです。

1. **副作用のない関数 (side-effect-free function)** — **副作用**とは、戻り値を返す以外に、何かを書き換えてしまうことです。副作用のない関数は、自分自身も引数も変更せず、結果を新しい値として返します。何度呼んでも同じ入力なら同じ出力なので、テストは「入力と期待する出力」の表を書くだけです
2. **値レシーバ+新しい値を返す** — Go ではポインタレシーバで書き換える誘惑がありますが、値オブジェクトでは値レシーバにして「取り替え」のスタイルを貫きます

### 値レシーバとポインタレシーバの違い

メソッドのレシーバには2種類あります。

| 書き方 | 名前 | メソッドに渡るもの | 中で書き換えると |
|---|---|---|---|
| `func (q Quantity) ...` | 値レシーバ | 呼び出し元の**コピー** | コピーが変わるだけ。呼び出し元は変わらない |
| `func (q *Quantity) ...` | ポインタレシーバ | 呼び出し元の**置き場所** | 呼び出し元そのものが変わる |

L6 の `Rename` や `Cancel` は、エンティティの状態を変えるためにポインタレシーバでした。値オブジェクトは書き換えないので値レシーバにします。違いを動かして確かめます。

```go
package main

import "fmt"

type Quantity struct {
    n int
}

// ポインタレシーバ: 呼び出し元の値そのものを書き換える
func (q *Quantity) IncrementInPlace() {
    q.n++
}

// 値レシーバ: q はコピー。新しい値を作って返す
func (q Quantity) Increment() Quantity {
    return Quantity{q.n + 1}
}

func main() {
    a := Quantity{n: 1}
    b := a.Increment()
    fmt.Println("値レシーバ:   a =", a.n, " b =", b.n)

    c := Quantity{n: 1}
    c.IncrementInPlace()
    fmt.Println("ポインタレシーバ: c =", c.n)
}
```

```text
値レシーバ:   a = 1  b = 2
ポインタレシーバ: c = 2
```

`q.n++` は q.n を1増やす書き方です。値レシーバの Increment では `a` は 1 のまま、増えた結果は新しい値 `b` として返ります。ポインタレシーバの IncrementInPlace では `c` 自身が 2 に変わりました。

### ユビキタス言語がコードに現れる

```go
// ドメインエキスパートの発言:「注文金額は小計に送料を加えたもの」
total, err := subtotal.Add(shippingFee)

// 「キャンペーンは注文日が適用期間に含まれるときだけ有効」
if campaign.Period().Contains(order.OrderedAt()) { ... }
```

| 行 | 意味 |
|---|---|
| `total, err := subtotal.Add(shippingFee)` | 小計(Money)に送料(Money)を足した合計と、エラーを受け取る(ch02 の Add) |
| `campaign.Period()` | キャンペーンの適用期間(Period)を取り出す getter |
| `order.OrderedAt()` | 注文日時(time.Time)を取り出す getter |
| `.Contains(...)` | 注文日時が期間に含まれるか。`{ ... }` は含まれたときの処理(省略) |

`if order.Subtotal + fee.Amount > 0` のようなプリミティブ演算の羅列と比べてください。**業務の文がそのままコードの文になる** — これが値オブジェクトに振る舞いを持たせる最大の理由で、L2 で学んだユビキタス言語の実装手段そのものです。

### 自己検証との合わせ技

生成時にコンストラクタで検証(ch02)してあるので、メソッドの中では「start と end はすでに妥当」と信頼して計算に集中できます。検証・計算・判断がひとつの型に凝集し(関係するものが1か所に集まり)、散らばった if 文が消えていきます。

💡 コツ: アプリケーション側(画面やAPIから受けた処理の流れを組み立てる部分。詳しくは L12)に `if` でドメインの判断を書きそうになったら、「この判断は値オブジェクトのメソッドにならないか?」と一度疑ってください。`Contains` や `CanShip` のような**動詞のメソッド名がユビキタス言語の語彙**になります。

### 確認問題

**問1.** 「会員ランクがゴールド以上で、かつ注文日がキャンペーン期間内なら送料無料」というロジックの『期間内か』の判定が、アプリケーションサービス3箇所に `t.After(start) && t.Before(end)` としてコピペされています。最も適切な改善はどれでしょう？

1. Period 値オブジェクトに Contains メソッドを持たせ、各所から使う
2. 共通の静的ユーティリティ関数 DateUtil.Between にまとめる
3. 3箇所のコメントに計算式の意味を丁寧に書く
4. 判定を SQL の WHERE 句に移してアプリから消す

<details><summary>答えと解説</summary>

正解: **1**

「期間に含まれるか」はキャンペーン期間というドメイン概念の判断なので、Period.Contains として概念のそばに置くのが適切です。境界を含むか等のルールも1箇所に定まります。DateUtil のような汎用ユーティリティは動きはしますが、ドメインの語彙にならず「どの境界ルールが正か」も分散したままです。コメントはコピペの増殖を止められません。SQL への移動は画面やバッチなど複数経路で同じ判断が必要になった瞬間に破綻します。

</details>

**問2.** 副作用のない関数 (side-effect-free function) として実装された Money.Add の説明として正しいものはどれでしょう？

1. レシーバ自身の amount を増やし、成功を bool で返す
2. 合計を表す新しい Money を返し、レシーバも引数も変更しない
3. 計算結果をデータベースに保存してから返す
4. グローバルな合計カウンタを更新しつつ結果を返す

<details><summary>答えと解説</summary>

正解: **2**

副作用のない関数は「既存の状態を一切変えず、結果を新しい値として返す」ものです。同じ入力なら必ず同じ出力なので、テストも並行処理も単純になります。レシーバの書き換えは不変性を破壊し、その Money を共有している箇所すべてに影響します。データベース保存やグローバル更新はまさに副作用で、純粋な計算に I/O を混ぜると再利用もテストも難しくなります。

</details>

**問3.** Go で値オブジェクト Quantity に「1個増やす」操作を実装します。値オブジェクトの流儀に合うのはどれでしょう？

1. func (q *Quantity) Increment() { q.n++ }
2. 公開フィールドにして q.N++ を呼び出し側に書いてもらう
3. func (q Quantity) Increment() Quantity { return Quantity{q.n + 1} }
4. sync.Mutex を内蔵してロックを取りながら q.n++ する

<details><summary>答えと解説</summary>

正解: **3**

値レシーバで受けて「増えた後の新しい Quantity」を返すのが、不変・交換可能という値オブジェクトの性質に合った実装です。ポインタレシーバでの書き換えは共有している他の持ち主にまで変更が波及し、公開フィールドは検証もルールも素通りになります。Mutex(sync.Mutex。同時に1つの処理だけが触れるようにするロックの部品)内蔵は「書き換える前提」の発想で、そもそも不変にすればロック自体が不要という値オブジェクトの利点を捨てています。

</details>

**問4.** NewPeriod コンストラクタで「開始は終了より前」を検証済みの Period があります。Contains メソッドの実装方針として適切なのはどれでしょう？

1. メソッド冒頭で毎回 start < end を再検証し、違反ならパニックする
2. 万一に備えて Contains の戻り値を (bool, error) にして検証エラーも返す
3. 呼び出し側に検証用の IsValid を先に呼ぶことを規約で義務づける
4. 生成時の検証を信頼し、含まれるかどうかの計算だけに集中する

<details><summary>答えと解説</summary>

正解: **4**

非公開フィールド+検証つきコンストラクタの構えができていれば、「Period が存在する = 妥当」なので、メソッド内の再検証は不要な防御的コードです。これが自己検証 (self-validation) の配当で、計算ロジックが素直になります。毎回の再検証やエラー戻り値は呼び出し側を複雑にするだけで、IsValid 規約方式は検証忘れを再び人間の注意力に委ねる後退です。再検証したくなったら「生成経路に穴がないか」を先に疑いましょう。

</details>

**問5.** ドメインエキスパートが「注文金額は小計に送料を加えたもの」と言っています。この知識の表現として最もユビキタス言語に忠実なコードはどれでしょう？

1. total := subtotal.Add(shippingFee)
2. total := CalcUtil.Sum(o.SubtotalYen, o.ShipFeeYen)
3. total := o.SubtotalYen + o.ShipFeeYen
4. total := priceService.Compute(o, "TOTAL_WITH_SHIP")

<details><summary>答えと解説</summary>

正解: **1**

「小計に送料を加える」という業務の文が `subtotal.Add(shippingFee)` とほぼ一対一で対応しており、通貨不一致の検出などのルールも Money の中で守られます。int の直接加算やユーティリティ関数は動きますが型の守りも語彙も失われ、文字列キーでモードを渡すサービスは意味をコードの外(マジックストリング)に追い出してしまう最も遠い形です。業務の文とコードの文の距離が、モデルとコードが一致しているかの手軽なリトマス試験紙になります。

</details>

**問6.** 値レシーバのメソッド `func (q Quantity) Increment() Quantity { return Quantity{q.n + 1} }` について、`a := Quantity{n: 1}` のあと `b := a.Increment()` を実行しました。a.n と b.n の値はどれでしょう？

1. a.n = 2、b.n = 2
2. a.n = 2、b.n = 1
3. a.n = 1、b.n = 2
4. a.n = 1、b.n = 1

<details><summary>答えと解説</summary>

正解: **3**

値レシーバ `(q Quantity)` のメソッドには a のコピーが渡されるので、a 自身は変わらず a.n = 1 のままです。Increment は q.n + 1 = 2 を持つ新しい Quantity を返すので、b.n = 2 になります。a まで 2 になるのは、ポインタレシーバ `(q *Quantity)` で中身を書き換えた場合です。値オブジェクトはこのように「自分は変えずに新しい値を返す」形で作ります。

</details>
