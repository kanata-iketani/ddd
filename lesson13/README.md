# 第13章 アーキテクチャパターン（ヘキサゴナル・オニオン・クリーン）

戦術パターンで作ったドメインモデルを、DB やフレームワークの都合から守るのがアーキテクチャの仕事です。この章ではヘキサゴナル・オニオン・クリーンという 3 つの有名パターンの「共通する本質」をつかみ、Go / Python の現実的なプロジェクト構成に落とします。L11 のリポジトリと interface、L12 のアプリケーションサービスとレイヤーを使うので、あやふやなら先に見直してください。

## 1. 依存性逆転とヘキサゴナルアーキテクチャ

### 基礎の基礎: 「依存する」とは

コードの世界で「A が B に**依存する**」とは、A が B を import したり呼び出したりしていて、**B が変わると A も直す必要が出る**関係のことです。

| 書き方 | 依存の向き |
|---|---|
| 注文のコードが `database/sql`(Go 標準の DB 接続パッケージ)を import する | 注文 → DB |
| 注文のコードが HTTP クライアント(外部の Web サービスを呼ぶ部品)を使う | 注文 → 外部サービス |

この講座でいう**外界**は、DB・外部サービス・Web フレームワーク・画面など、ドメインの外側にあるものすべてです。

### 依存の向きがドメインを壊す

素朴に書くと、ドメインロジックが `database/sql` や HTTP クライアントを直接 import しがちです。こうなると DB の都合(スキーマ=テーブルの列の構成の変更、接続方法)がビジネスルールのコードに波及し、テストにも実 DB が必要になります。

これを断ち切るのが**依存性逆転の原則 (Dependency Inversion Principle, DIP)** です。「具象(DB 実装)ではなく抽象(interface)に依存せよ」。

- **具象**: 実際に動く中身を持つ実装。例: PostgreSQL(よく使われる DB 製品)に SQL を投げて保存するコード
- **抽象**: 中身を持たず「何ができるか」だけを決めたもの。Go では **interface**(L11 で見た、メソッドの一覧だけを決めた型)

Go では **interface を使う側(ドメイン・ユースケース側)で定義する**のが慣例で、DIP と相性抜群です。

```go
// domain/order.go — ドメイン側が「欲しい能力」を interface で宣言する(=ポート)
type OrderRepository interface {
    Save(ctx context.Context, o *Order) error
    FindByID(ctx context.Context, id OrderID) (*Order, error)
}

// adapter/postgres/order_repo.go — 外側が実装する(=アダプタ)
type OrderRepo struct{ db *sql.DB }
func (r *OrderRepo) Save(ctx context.Context, o *domain.Order) error { /* SQL */ }
```

| 部分 | 意味 |
|---|---|
| `type OrderRepository interface { ... }` | 「Save と FindByID を持つもの」という型を決める。中身(どう保存するか)は書かない |
| `ctx context.Context` | 処理の中断や時間切れを伝えるための、Go で慣例的に第1引数に置く値。いまは読み飛ばしてかまいません |
| `Save(...) error` | 保存して、失敗したら error を返す(成功なら nil) |
| `FindByID(...) (*Order, error)` | ID で探して、見つかった注文へのポインタと error の2つを返す(L4 で見た多値返し) |
| `type OrderRepo struct{ db *sql.DB }` | 外側の実装。中に DB 接続 `*sql.DB` を持つ |
| `func (r *OrderRepo) Save(...)` | OrderRepo に Save メソッドを付ける。`domain.Order` は「domain パッケージの Order」 |
| `/* SQL */` | 本当はここに INSERT 文などを書く(省略) |

Go では「この型は OrderRepository を実装します」と宣言する必要がありません。**interface に書かれたメソッドを全部持っている型は、自動的にその interface として使えます**。OrderRepo が Save と FindByID を持てば、OrderRepository として渡せます。

### なぜ「逆転」と呼ぶのか

| | ドメインのコードが import するもの | 依存の向き |
|---|---|---|
| 素朴な書き方 | `database/sql` | ドメイン → DB |
| DIP を使う | 何も import しない(interface は自分で定義) | ドメイン ← DB 実装(アダプタがドメインの interface に合わせる) |

DB の実装側が、ドメインの決めた interface に合わせにいく形になります。矢印の向きが素朴な書き方と逆になるので「逆転」です。

### 動かして確かめる

DB を用意しなくても動くように、通知(メール/チャット)を例にした完全版です。

```go
package main

import "fmt"

// ===== 内側: ユースケースが「欲しい能力」を宣言する(ポート) =====
type Notifier interface {
	Notify(to string, msg string) error
}

// 内側の処理は Notifier という interface しか知らない
func ConfirmOrder(n Notifier, orderID string) error {
	// (ここで注文を確定する処理をしたとして)
	return n.Notify("buyer@example.com", "注文 "+orderID+" を確定しました")
}

// ===== 外側: ポートを実装するアダプタ =====
type EmailNotifier struct{}

func (e EmailNotifier) Notify(to string, msg string) error {
	fmt.Println("[メール]", to, "宛:", msg)
	return nil
}

type ChatNotifier struct {
	Channel string
}

func (c ChatNotifier) Notify(to string, msg string) error {
	fmt.Println("[チャット]", c.Channel, ":", msg)
	return nil
}

func main() {
	ConfirmOrder(EmailNotifier{}, "o-001")
	ConfirmOrder(ChatNotifier{Channel: "#orders"}, "o-002")
}
```

実行結果:

```text
[メール] buyer@example.com 宛: 注文 o-001 を確定しました
[チャット] #orders : 注文 o-002 を確定しました
```

| 部分 | 意味 |
|---|---|
| `type Notifier interface { Notify(...) error }` | ポート。「Notify できるもの」という型 |
| `func ConfirmOrder(n Notifier, ...)` | 引数は interface。メールかチャットかを知らない |
| `type EmailNotifier struct{}` | 中身(フィールド)を持たない struct。メソッドだけを持たせるために使う |
| `func (e EmailNotifier) Notify(...)` | Notify を持つので、EmailNotifier は自動的に Notifier として使える |
| `ConfirmOrder(EmailNotifier{}, "o-001")` | メールのアダプタを渡す。ConfirmOrder の中身は1行も変えずに… |
| `ConfirmOrder(ChatNotifier{...}, "o-002")` | …チャットのアダプタに差し替えられる |

メソッドが足りない型を渡すと、実行する前にコンパイル(Go がプログラムを機械語に変換する段階)で止まります。`Notify` ではなく `Send` しか持たない `FaxNotifier` を渡したときのエラーです。

```text
./main.go:17:15: cannot use FaxNotifier{} (value of type FaxNotifier) as Notifier value in argument to ConfirmOrder: FaxNotifier does not implement Notifier (missing method Notify)
```

「FaxNotifier は Notifier を実装していない(Notify メソッドがない)」という意味です。

Python では `typing.Protocol` で同じ形を書けます。

```python
from typing import Protocol


# 内側: 欲しい能力を宣言する(ポート)
class Notifier(Protocol):
    def notify(self, to: str, msg: str) -> None: ...


def confirm_order(n: Notifier, order_id: str) -> None:
    n.notify("buyer@example.com", f"注文 {order_id} を確定しました")


# 外側: アダプタ
class EmailNotifier:
    def notify(self, to: str, msg: str) -> None:
        print("[メール]", to, "宛:", msg)


class ChatNotifier:
    def __init__(self, channel: str):
        self.channel = channel

    def notify(self, to: str, msg: str) -> None:
        print("[チャット]", self.channel, ":", msg)


confirm_order(EmailNotifier(), "o-001")
confirm_order(ChatNotifier("#orders"), "o-002")
```

```text
[メール] buyer@example.com 宛: 注文 o-001 を確定しました
[チャット] #orders : 注文 o-002 を確定しました
```

### ヘキサゴナル(ポート&アダプタ)

**ヘキサゴナルアーキテクチャ (Hexagonal Architecture)** は、この考えをシステム全体に広げたものです。中心(ドメイン+ユースケース)を六角形で描き、その周りに外界とつなぐ部品を並べた図から、この名前が付きました。

| 用語 | 意味 | 例 |
|---|---|---|
| ポート (Port) | 内側が定義する interface | `OrderRepository`、`NotificationPort` |
| アダプタ (Adapter) | 外界とポートを変換する実装 | Postgres 実装、HTTP ハンドラ |
| プライマリアダプタ | アプリを**駆動する**側 | HTTP ハンドラ、CLI、cron |
| セカンダリアダプタ | アプリに**駆動される**側 | DB、メール送信、外部 API |

- **HTTP ハンドラ**: ブラウザなどから届いたリクエストを受け取り、処理を呼ぶ関数
- **CLI**: ターミナルで打つコマンドとして動かす入口
- **cron**: 「毎日3時に実行」のように決まった時刻に処理を起動する仕組み
- **駆動する側**: 外から処理を呼び出して動かし始める側。**駆動される側**: 中の処理から呼ばれて仕事をする側

ECサイトで「注文する」ボタンが押されたときの流れに当てはめると、次のようになります。

| 順 | 部品 | 種類 |
|---|---|---|
| 1 | HTTP ハンドラがリクエストを受ける | プライマリアダプタ |
| 2 | 注文のユースケースと Order がルールを確認する | 内側 |
| 3 | `OrderRepository` 経由で DB に保存する | ポート → セカンダリアダプタ |
| 4 | `Notifier` 経由で確認メールを送る | ポート → セカンダリアダプタ |

内側(ドメイン+ユースケース)は外界を一切知らず、外界の交換(Postgres→DynamoDB(AWS の DB サービス)、メール→Slack(チャットツール))はアダプタの差し替えだけで済みます。

なお、SQL を DB の中に保存しておく**ストアドプロシージャ**に移す方法もありますが、ビジネスルールが DB の中に入るだけで、ドメインから外界への依存は消えません。

💡 コツ: 「import の矢印が常に内側(ドメイン)へ向いているか」だけをレビューで見れば、このアーキテクチャは守れます。

### 確認問題

**問1.** ECサイトのドメイン層にある Order 構造体のメソッドが database/sql を直接 import して UPDATE 文を発行しています。DDD の観点で最も適切な改善はどれでしょう?

1. UPDATE 文を文字列定数に切り出して SQL の重複をなくす
2. Order のメソッドから SQL を削除し、ドメイン層に OrderRepository interface を定義して DB 実装は外側のアダプタに移す
3. database/sql をラップした共通 DB ユーティリティパッケージをドメイン層に作る
4. UPDATE をストアドプロシージャに移動して SQL をドメインコードから見えなくする

<details><summary>答えと解説</summary>

正解: **2**

依存性逆転 (DIP) の典型的な適用場面です。ドメイン側に「欲しい能力」を interface(ポート)として宣言し、SQL を書く実装(アダプタ)は外側に置けば、ドメインは DB を知らなくなり、スキーマ変更の影響が遮断され、モックでテストもできます。SQL の定数化やストアドへの移動は置き場所が変わるだけで「ドメインが永続化の都合に依存している」構造は残ります。共通 DB ユーティリティをドメイン層に置くのは、むしろ依存を固定化する逆方向の一手です。

</details>

**問2.** ヘキサゴナルアーキテクチャにおける「ポート (Port)」の説明として正しいものはどれでしょう?

1. ドメインやユースケースなど内側が定義する interface で、外界に求める能力や外界への入口を抽象化したもの
2. HTTP サーバーが待ち受ける TCP ポート番号の設計規約
3. DB ドライバが提供する接続プールの実装
4. 外部 SaaS が公開している REST API の仕様書

<details><summary>答えと解説</summary>

正解: **1**

ポートは内側(ドメイン・アプリケーション)が定義する抽象 = interface です。外界がそれを実装(または呼び出し)する側がアダプタです。TCP ポートとは無関係なので名前に引っ張られないよう注意しましょう。DB ドライバの接続プールや外部 API 仕様は「外界側」の話で、ポートはあくまで内側が主導権を持って定義するのがポイントです。試験的なひっかけとして「外部が定義する」と書かれたら誤りです。

</details>

**問3.** 次のうち「プライマリ(駆動する側)アダプタ」に分類されるものはどれでしょう?

1. 注文データを保存する PostgreSQL リポジトリ実装
2. 領収書メールを送る SMTP クライアント
3. 決済代行サービスを呼び出す HTTP クライアント
4. ユーザーのリクエストを受けてユースケースを呼び出す HTTP ハンドラ

<details><summary>答えと解説</summary>

正解: **4**

プライマリアダプタは「アプリケーションを**駆動する**側」= 入口です。HTTP ハンドラ、CLI、メッセージコンシューマ、cron ジョブなどが該当します。残りの 3 つ(DB 保存、メール送信、外部 API 呼び出し)はアプリケーションから**駆動される**セカンダリアダプタです。「呼び出しの矢印がアプリに向かって入るか、アプリから出ていくか」で見分けるのがコツです。

</details>

**問4.** 通知機能をメール送信から Slack 通知に切り替える計画があります。将来の切り替え影響を最小にする設計として最も適切なものはどれでしょう?

1. メール送信コードに if 文を足して Slack 分岐を追加できるようにしておく
2. ユースケース層から SMTP ライブラリを直接呼び、設定ファイルで送信先を切り替える
3. ユースケース側に Notifier interface(ポート)を定義し、メール実装と Slack 実装をアダプタとして差し替え可能にする
4. 通知処理を共通ユーティリティ関数 SendNotification にまとめて全箇所から呼ぶ

<details><summary>答えと解説</summary>

正解: **3**

外界(通知手段)の交換はヘキサゴナルの得意技です。内側に Notifier interface を定義しておけば、メール実装を Slack 実装に差し替えるだけで、ユースケースのコードは 1 行も変わりません。if 分岐や共通関数は呼び出し側に実装詳細の知識が残り、SMTP 直呼びは依存の向きが外向きのままです。「内側は抽象だけを知る」構造にしてこそ、テスト時もフェイク実装を注入できます。

</details>

**問5.** Go で DDD を実践するとき、リポジトリ interface を定義する場所として最も適切なのはどれでしょう?

1. DB ドライバと同じ infrastructure パッケージ(実装の近くにまとめる)
2. interface 専用の共通パッケージ interfaces/ を作って全部そこに置く
3. それを使うドメイン・ユースケース側のパッケージ(利用側が欲しい能力を宣言する)
4. main パッケージ(依存の組み立てと同じ場所)

<details><summary>答えと解説</summary>

正解: **3**

Go の慣例「interface は使う側で定義する」は DIP そのものです。ドメイン側に置けば import の矢印が内向きになり、ドメインはインフラを知りません。実装の隣に interface を置くと、利用側が実装パッケージを import する羽目になり依存が逆転しません(ヘキサゴナルが崩れる典型)。interfaces/ 共通パッケージは一見整理されて見えますが、文脈から切り離された抽象置き場になりがちで、どの層が主導権を持つかも曖昧になります。main は組み立て(DI)の場所であって宣言の場所ではありません。

</details>

**問6.** この章の完全版で、`Notify` メソッドを持たず `Send` メソッドだけを持つ `FaxNotifier` を `ConfirmOrder(FaxNotifier{}, "o-003")` のように渡しました。Go ではどうなるでしょう?

1. 実行はされるが、Notify の部分だけ何もせずに終わる
2. コンパイルの段階でエラーになり、プログラムは実行されない
3. Go が名前の近い Send メソッドを自動で代わりに呼ぶ
4. 実行時に初めてエラーになり、途中まで処理が進んでから止まる

<details><summary>答えと解説</summary>

正解: **2**

Go の interface は「書かれたメソッドを全部持っている型だけが、その interface として使える」というルールです。FaxNotifier には Notify がないので、`FaxNotifier does not implement Notifier (missing method Notify)` というコンパイルエラーになり、実行まで進みません。何もせずに終わることはなく、名前が近いメソッドを代わりに呼ぶ仕組みもありません。実行時ではなくコンパイル時に見つかるので、アダプタを差し替えたときの書き漏れにすぐ気づけます。

</details>

## 2. オニオン・クリーンアーキテクチャ — 共通する本質

### 同心円の仲間たち

ヘキサゴナル(ch01)の後に提案された **オニオンアーキテクチャ (Onion Architecture)** と **クリーンアーキテクチャ (Clean Architecture)** は、どちらも同心円(中心が同じ複数の円)で層を描きます。**層(レイヤー)** は L12 で見たとおり、役割ごとにコードを分けたまとまりです。

| | 中心 | その外 | さらに外 | 最外殻 |
|---|---|---|---|---|
| オニオン | ドメインモデル | ドメインサービス | アプリケーションサービス | UI・DB・テスト |
| クリーン | エンティティ | ユースケース | インターフェースアダプタ | フレームワーク・ドライバ |

表の言葉を整理します。

- **ドメインモデル/エンティティ**: L6〜L9 で見たエンティティ・値オブジェクト・集約。クリーンアーキテクチャの「エンティティ」は、DDD のエンティティより広く「業務ルールを持つオブジェクト全般」を指します
- **アプリケーションサービス/ユースケース**: L12 で見た、「注文する」のような利用者の操作1つ分の処理の流れ
- **インターフェースアダプタ**: ch01 のアダプタとほぼ同じ。HTTP の入力をユースケースの引数に変える**コントローラ**、結果を画面向けの形に整える**プレゼンター**、DB とのやり取りを包む**ゲートウェイ**(リポジトリの実装)など
- **フレームワーク・ドライバ**: Web フレームワーク(gin など、Web アプリを作るための部品一式)や DB ドライバ(DB と通信するライブラリ)

名前も円の枚数も違いますが、**本質は 3 つとも同じ**です。

1. **ドメインが中心** — ビジネスルールが一番内側に住む
2. **依存は常に内向き** — 内側は外側を import しない
3. **外界は交換可能** — DB・フレームワーク・UI は「詳細」であり、アダプタ(境界の interface)越しに差し替えられる

ここでいう**詳細**とは「ビジネスルールの正しさに関係しない、取り替えのきく技術の選択」のことです。ECサイトなら次のとおりです。

| 円 | ECサイトでの例 |
|---|---|
| 中心 | `Order`、`Money`、「確定後は明細を変えられない」というルール |
| ユースケース | `PlaceOrder`(注文する) |
| アダプタ | 注文 API の HTTP ハンドラ、`OrderRepository` の PostgreSQL 実装 |
| 最外殻(詳細) | gin、PostgreSQL そのもの |

たとえばユースケースが `gin.Context`(gin がリクエストごとに渡してくる値)を引数に取ると、内側が最外殻の gin を import することになり、「依存は内向き」が破れます。

### 最大の実利: テスタビリティ

**テスタビリティ**はテストのしやすさです。依存が内向きなら、ドメインとユースケースは **DB もフレームワークもなしで単体テスト**(関数やメソッドを1つずつ確かめる自動テスト。L8 で見た `testing`)できます。

本物の DB の代わりに、テスト用の簡易な実装を渡します。これを**フェイク (fake)** と呼び、外から部品を渡すことを**注入**と呼びます。

```go
// リポジトリのフェイクを注入してユースケースをテスト
repo := &fakeOrderRepo{}            // map で持つだけの実装
uc := usecase.NewPlaceOrder(repo)
err := uc.Execute(ctx, cmd)         // DB 不要・ミリ秒で完走
```

| 行 | 意味 |
|---|---|
| `repo := &fakeOrderRepo{}` | フェイクを作る。`&` で作った値へのポインタを得る |
| `uc := usecase.NewPlaceOrder(repo)` | ユースケースを作り、フェイクを渡す(注入)。ユースケースは interface しか知らないので本物かフェイクかを区別しない |
| `err := uc.Execute(ctx, cmd)` | ユースケースを実行する。`cmd` は「注文の中身」をまとめた入力の値 |

### 動かして確かめる

ユースケースとテストを、そのまま `go test` で動く形にしたものです。まずユースケース側(`usecase.go`)です。

```go
package placeorder

import "errors"

type Order struct {
	ID       string
	Quantity int
}

// ポート: 保存の能力だけを宣言する
type OrderRepository interface {
	Save(o *Order) error
}

// ユースケース: 中に OrderRepository(interface)を持つ
type PlaceOrder struct {
	repo OrderRepository
}

func NewPlaceOrder(repo OrderRepository) *PlaceOrder {
	return &PlaceOrder{repo: repo}
}

func (uc *PlaceOrder) Execute(id string, qty int) error {
	if qty < 1 {
		return errors.New("数量は1以上にしてください")
	}
	return uc.repo.Save(&Order{ID: id, Quantity: qty})
}
```

テスト側(`usecase_test.go`)です。Go ではファイル名が `_test.go` で終わるファイルにテストを書き、`func TestXxx(t *testing.T)` の形の関数が1つのテストになります。

```go
package placeorder

import "testing"

// フェイク: DB の代わりに map に入れるだけの実装
type fakeOrderRepo struct {
	saved map[string]*Order
}

func (f *fakeOrderRepo) Save(o *Order) error {
	f.saved[o.ID] = o
	return nil
}

func TestExecute_Saves(t *testing.T) {
	repo := &fakeOrderRepo{saved: map[string]*Order{}}
	uc := NewPlaceOrder(repo)
	if err := uc.Execute("o-1", 2); err != nil {
		t.Fatal(err)
	}
	if repo.saved["o-1"].Quantity != 2 {
		t.Fatal("保存された数量が違う")
	}
}

func TestExecute_RejectsZero(t *testing.T) {
	repo := &fakeOrderRepo{saved: map[string]*Order{}}
	uc := NewPlaceOrder(repo)
	if err := uc.Execute("o-2", 0); err == nil {
		t.Fatal("数量0が通ってしまった")
	}
	if len(repo.saved) != 0 {
		t.Fatal("保存されてはいけない")
	}
}
```

| 部分 | 意味 |
|---|---|
| `type fakeOrderRepo struct { saved map[string]*Order }` | 保存先は DB ではなく map(Python の dict に当たる、キーと値の組の入れ物) |
| `func (f *fakeOrderRepo) Save(...)` | Save を持つので OrderRepository として渡せる |
| `map[string]*Order{}` | 空の map を作る |
| `t.Fatal(...)` | テストを失敗させ、メッセージを表示する |
| `repo.saved["o-1"].Quantity` | フェイクの中身を直接見て、本当に保存されたかを確かめる |
| `len(repo.saved) != 0` | 数量0のときは何も保存されていないことを確かめる |

`go test -v` で実行した結果です(`-v` は各テストの結果を1つずつ表示するオプション)。

```text
=== RUN   TestExecute_Saves
--- PASS: TestExecute_Saves (0.00s)
=== RUN   TestExecute_RejectsZero
--- PASS: TestExecute_RejectsZero (0.00s)
PASS
ok  	placeorder	0.006s
```

`PASS` は合格、`(0.00s)` はかかった秒数です(最後の行の秒数は実行のたびに少し変わります)。DB を起動していないので、一瞬で終わります。

本物の DB でテストする方法もあります(SQLite=ファイル1つで動く軽量な DB、docker compose=DB などのサーバーを1コマンドでまとめて起動する道具)。ただし起動と後片付けに時間がかかるので、ビジネスルールの確認は上のように DB なしで行い、DB の動作確認は少数のテストに分けるのが一般的です。

テストが速いとリファクタリング(外から見た動きを変えずにコードの中身を整理すること)が怖くなくなり、モデルを深める(第1章の知識の噛み砕き)サイクルが回り続けます。

💡 コツ: 「ヘキサゴナル派 vs クリーン派」の論争に実りはありません。チームで合意すべきは図の名前ではなく**「依存は内向き」というルールただ1つ**です。

### 確認問題

**問1.** ヘキサゴナル・オニオン・クリーンの 3 つのアーキテクチャに共通する本質として最も適切なものはどれでしょう?

1. レイヤーを必ず 4 枚に分割し、層を飛ばした呼び出しを禁止すること
2. ドメインを中心に置き、依存の向きを常に内向きに保ち、外界を交換可能な詳細として扱うこと
3. マイクロサービスへの分割を前提とした API 設計を行うこと
4. すべての層の間に DTO を挟んでデータを変換すること

<details><summary>答えと解説</summary>

正解: **2**

3 つのパターンは図の描き方(六角形か同心円か)と層の名前が違うだけで、「ドメイン中心・依存は内向き・外界は詳細」という原則は共通です。層の枚数は 4 枚と決まっているわけではなく(オニオンとクリーンでも違います)、マイクロサービス分割とは独立の話です。DTO 変換は手段の一つであって本質ではありません。名前の違いに惑わされず原則を見る、が実務での正解です。

</details>

**問2.** クリーンアーキテクチャを採用したプロジェクトで、ユースケース層のコードが Web フレームワーク(gin)の gin.Context を引数に取っています。この設計の最大の問題はどれでしょう?

1. gin.Context は構造体が大きく、引数に渡すとメモリ効率が悪い
2. gin のバージョンアップ時に再コンパイルが必要になる
3. HTTP の都合という「詳細」に内側の層が依存しており、フレームワーク交換やフレームワーク抜きのテストができなくなる
4. gin.Context はスレッドセーフでないため並行処理でバグが出る

<details><summary>答えと解説</summary>

正解: **3**

依存の向きの違反が本質的な問題です。ユースケースはフレームワーク(最外殻の詳細)を知ってはならず、gin.Context を受け取った時点で「内向きの依存」ルールが崩れています。HTTP から必要な値はプライマリアダプタ(ハンドラ)で取り出し、ユースケースにはプレーンな入力 DTO を渡すのが正しい形です。メモリ効率やスレッドセーフ性は副次的な話で、仮に問題なくても構造上の欠陥(テスト不能・交換不能)は残ります。

</details>

**問3.** オニオンアーキテクチャの最も内側(中心)に置かれるものはどれでしょう?

1. エンティティや値オブジェクトからなるドメインモデル
2. リポジトリの DB 実装
3. アプリケーションサービス(ユースケース)
4. コントローラとプレゼンター

<details><summary>答えと解説</summary>

正解: **1**

オニオンの中心はドメインモデル(エンティティ・値オブジェクト)です。その外側にドメインサービス、アプリケーションサービスと続き、DB 実装・UI・コントローラは最外殻に属します。「ビジネスルールが一番内側、技術詳細が一番外側」という配置はクリーンアーキテクチャ(中心=エンティティ)でも同じです。リポジトリは interface が内側・実装が外側に分かれる点もひっかけどころです。

</details>

**問4.** 「注文金額の割引ルールが正しいか」を検証するテストを、DB や Web サーバーを一切起動せずミリ秒単位で大量に実行したいです。これを可能にする設計上の前提として最も適切なものはどれでしょう?

1. テスト用の軽量インメモリ DB (SQLite) を CI に用意しておく
2. ドメイン層とユースケース層が外界に依存せず、リポジトリ等はフェイク実装を注入できるようになっている
3. docker compose でテスト環境を 1 コマンド起動できるようにしておく
4. 割引ルールを SQL のストアドプロシージャに集約しておく

<details><summary>答えと解説</summary>

正解: **2**

依存が内向きであれば、割引ルール(ドメイン)のテストは純粋な Go/Python のユニットテストになり、フェイクのリポジトリを注入すればユースケースまで外界なしで検証できます。SQLite や docker compose は「DB が必要な構造のまま」高速化する緩和策で、根本の依存は切れていません。ストアドに集約すると、むしろテストに DB が必須になり逆効果です。アーキテクチャの実利はこのテスタビリティに最も表れます。

</details>

**問5.** 設計会議で「ヘキサゴナルにすべきか、クリーンアーキテクチャにすべきか」で議論が 1 時間紛糾しています。テックリードとして最も適切な進め方はどれでしょう?

1. 知名度が高いクリーンアーキテクチャに多数決で決める
2. 両方採用し、サービスごとに好きな方を選べるようにする
3. 外部の有名企業の採用事例が多い方に合わせる
4. 3 パターンの本質は同じだと整理し、「依存は内向き」のルールと層の責務だけをチームで合意して名前の議論は打ち切る

<details><summary>答えと解説</summary>

正解: **4**

図の名前の選択はドメインを守るという目的に対してほぼ無差別で、実際に守られるかはレビューで依存の向きを徹底できるかで決まります。合意すべきは原則(依存は内向き・ドメインにフレームワークを入れない)と各層の責務です。多数決や知名度・他社事例は根拠になりませんし、サービスごとにバラバラの用語を許すと会話のコスト(ユビキタス言語の崩壊)が増えるだけです。

</details>

**問6.** この章の完全版のテスト `TestExecute_RejectsZero` で、`uc.Execute("o-2", 0)` を呼んだ直後の `len(repo.saved)` はいくつになるでしょう?

1. 1。数量0でも注文はいったん保存される
2. 実行するたびに変わる
3. 0。数量チェックで error が返り、Save が呼ばれない
4. DB に接続できないためテストが実行できない

<details><summary>答えと解説</summary>

正解: **3**

Execute は最初に `if qty < 1` を確認し、0 なら error を返してその場で終わります。`uc.repo.Save(...)` の行まで進まないので、フェイクの map には何も入らず 0 件のままです。保存は数量チェックを通った後にだけ行われます。結果は毎回同じで、フェイクは map を使うだけなので DB への接続もそもそも必要ありません。

</details>

## 3. 現実のプロジェクト構成 — 小さく始めてやりすぎない

### Go での構成例

教科書の同心円を、Go では次のようなパッケージ構成(フォルダ構成)に落とすのが定番です。

```text
internal/
  domain/          # エンティティ・値オブジェクト・リポジトリinterface
    order/         #   境界づけられたコンテキスト or 集約ごとに分割
  usecase/         # アプリケーションサービス(ユースケース)
  adapter/
    handler/       # プライマリ: HTTP ハンドラ (gin/echo はここだけ)
    postgres/      # セカンダリ: リポジトリ実装
    slack/         # セカンダリ: 通知実装
cmd/api/main.go    # 依存の組み立て(DI)と起動
```

| フォルダ | 意味 |
|---|---|
| `internal/` | Go が特別扱いするフォルダ名。この中のパッケージは、同じプロジェクトの外からは import できない |
| `domain/` | 中心の円。外のパッケージを import しない |
| `usecase/` | ユースケース。domain だけを import する |
| `adapter/` | 外側。gin や echo(どちらも Go の Web フレームワーク)、DB ライブラリを import してよいのはここだけ |
| `cmd/api/main.go` | プログラムの入口(`func main` がある場所) |

**DI (Dependency Injection、依存性の注入)** は、部品を使う側の中で作らず、外で作って渡すことです。main.go で部品を内側から順に組み立てて渡すのが定番です。形だけを示すと次のとおりです(本物の DB が必要なので、このままでは動きません)。

```go
func main() {
    db := openDB()                     // 外界: DB に接続する
    repo := postgres.NewOrderRepo(db)  // セカンダリアダプタを作る
    uc := usecase.NewPlaceOrder(repo)  // ユースケースにポートとして渡す
    h := handler.NewOrderHandler(uc)   // プライマリアダプタに渡す
    http.ListenAndServe(":8080", h)    // ポート番号 8080 で HTTP を待ち受ける
}
```

ch01 の完全版の `main` で `ConfirmOrder(EmailNotifier{}, ...)` と渡していたのと同じことを、部品を増やして行っています。どの実装を使うかを知っているのは main.go だけです。

Python / FastAPI でも考え方は同じです: `app/domain/`(Pydantic=FastAPI でデータの形の定義と検証に使うライブラリ、に依存しない純粋なクラス)、`app/usecase/`、`app/adapter/`(routers=URL ごとの処理を書く場所・repositories)。ルーターは薄く、ビジネスルールは domain へ。

「薄いルーター」の具体例です。割引ルールを FastAPI を import しない関数にしておけば、サーバーを起動せずに呼んで確かめられます。

```python
# domain 側: FastAPI も DB も import しない、ただの関数
def apply_discount(total: int, is_member: bool) -> int:
    if is_member and total >= 5000:  # 会員で5000円以上なら
        return total - 500           # 500円引き
    return total


print(apply_discount(6000, True))
print(apply_discount(6000, False))
print(apply_discount(3000, True))
```

```text
5500
6000
3000
```

ルーター側は `total = apply_discount(req.total, req.is_member)` のように、受け取った値を渡して結果を返すだけにします。

L11 で見た ORM の SQLAlchemy(Python で DB のテーブルをクラスとして扱うライブラリ)のモデルクラスも、DB の都合を持つ外側の部品です。ビジネスルールはそこではなく domain に置きます。また、機能ごとに別々のサーバーへ分ける**マイクロサービス**(小さなサービスをネットワーク越しに連携させる構成)は、この章の話とは別の判断です。まず1つのプログラムの中で domain を分けるだけで十分な効果があります。

### 段階的に適用する

最初から全部やる必要はありません。効果が大きい順に:

1. **domain パッケージを分離**し、ビジネスルールをそこへ寄せる(これだけで価値の 7 割)
2. リポジトリを interface 化してテストを外界から切る(ch02 のフェイクが使えるようになる)
3. ユースケース層を立ててハンドラを薄くする

### やりすぎのサイン

| 症状 | 診断 |
|---|---|
| 1 行の変更に 5 ファイルの修正が必要 | 抽象の層が多すぎる |
| 実装が 1 つしかない interface が大量にある | 「いつか差し替えるかも」の先回りしすぎ |
| 新人がどこに何を書くか 1 週間迷う | チームの理解度と乖離 |
| どの層にも同じ形の受け渡しコードが並ぶ | **ボイラープレート**(中身のない定型コード)が増えすぎている |

interface を残す価値があるのは、ch02 のようにテストでフェイクに差し替えたい所(リポジトリ・外部サービス)や、実際に実装が複数ある所です。そうでない所は、具体的な型をそのまま使ってかまいません。

💡 コツ: アーキテクチャは**ドメインを守るための手段であって目的ではありません**。守るほどのドメインの複雑さがない画面(単純 CRUD=作成・読み取り・更新・削除だけの処理)まで同じ儀式を強いるのはコストです(第16章で詳述)。

### 確認問題

**問1.** 社内の小さな備品予約アプリ(実質 CRUD が 3 画面)に、エンティティ/ユースケース/ゲートウェイ/プレゼンターの 4 層+全リポジトリの interface 化を導入したところ、開発速度が大きく落ちました。最も適切な見直しはどれでしょう?

1. 開発速度は品質への投資と割り切り、構成を維持して教育を強化する
2. 層を増やして責務をさらに細分化し、1 ファイルあたりの行数を減らす
3. コード生成ツールを導入して 4 層のボイラープレートを自動生成する
4. ドメインの複雑さに対して構造が過剰と判断し、薄い構成(シンプルな CRUD 実装)に縮退する

<details><summary>答えと解説</summary>

正解: **4**

アーキテクチャは複雑なドメインを守るための手段です。守るべきビジネスルールがほぼない CRUD アプリでは、層の維持費だけが残り、リターンがありません。構造の複雑さはドメインの複雑さに比例させるのが原則です。教育強化・さらなる分割・コード生成はいずれも「過剰な構造を維持するためのコスト追加」であり、原因(過剰適用)を治していません。DDD を使わない判断も DDD の一部です。

</details>

**問2.** Go プロジェクトの internal/domain パッケージに置くべきものとして最も適切な組み合わせはどれでしょう?

1. エンティティ・値オブジェクト・リポジトリの interface
2. HTTP ハンドラと gin のミドルウェア
3. リポジトリの PostgreSQL 実装と SQL マイグレーション
4. 環境変数の読み込みと DI コンテナの設定

<details><summary>答えと解説</summary>

正解: **1**

domain には「ビジネスの語彙と規則」= エンティティ・値オブジェクト・ドメインサービス、そして欲しい能力の宣言であるリポジトリ interface を置きます。実装(PostgreSQL・SQL)は adapter 側、HTTP ハンドラはプライマリアダプタ、環境変数や DI の組み立ては cmd/main の仕事です。「interface は domain、実装は adapter」という分離が依存を内向きに保つ鍵で、ここを混ぜると円が崩れ始めます。

</details>

**問3.** FastAPI のルーター関数の中に、在庫チェック・割引計算・ポイント付与のロジックが 200 行書かれています。段階的に改善するとき最初の一手として最も適切なものはどれでしょう?

1. まずルーターを別ファイルに移動してファイルを分割する
2. 200 行にユニットテストを書いてからマイクロサービスに切り出す
3. ビジネスルール(在庫・割引・ポイント)をフレームワーク非依存の domain / usecase に抽出し、ルーターは入出力変換だけの薄い層にする
4. SQLAlchemy のモデルクラスにロジックを移動する

<details><summary>答えと解説</summary>

正解: **3**

問題の本質は「ビジネスルールがプライマリアダプタ(ルーター)に住んでいる」ことです。ルールを FastAPI 非依存の関数・クラスへ抽出すれば、テスト可能になり再利用もできます。ファイル分割は見た目が変わるだけで依存構造は同じ、マイクロサービス化は構造問題を分散システム問題に拡大するだけです。SQLAlchemy モデルへの移動は ORM(外側の詳細)へルールを寄せる逆方向で、DB 非依存のテストができなくなります。

</details>

**問4.** 「実装クラスが 1 つしかない interface がプロジェクトに 40 個ある」というレビュー指摘がありました。この状態の最も適切な評価はどれでしょう?

1. interface は多いほど疎結合なので、むしろ健全な状態である
2. テスト注入や依存逆転に必要な境界(リポジトリ・外部サービス)以外は先回りの抽象化であり、不要な interface は削って具象を直接使ってよい
3. Go では interface の数がコンパイル時間を悪化させるので性能問題として対処する
4. すべての interface にモック生成ツールを適用して活用度を上げる

<details><summary>答えと解説</summary>

正解: **2**

interface が価値を持つのは「外界との境界(DB・外部 API・通知)」と「テストで差し替えたい場所」です。ドメイン内部の具象クラスまで一律に interface を被せるのは『いつか差し替えるかも』という投機的一般化で、読み手のジャンプ回数と保守コストだけが増えます。疎結合は目的の複雑さに見合ってこそ健全です。モック生成の全面適用は過剰抽象を固定化します。コンパイル時間は論点ではありません。

</details>

**問5.** アーキテクチャ導入の方針として、この章の考え方に最も合致するものはどれでしょう?

1. 最初の 1 行を書く前に全層・全規約を完備し、以後の変更を認めない
2. 著名な OSS のディレクトリ構成を丸ごとコピーし、チームの実情は考慮しない
3. アーキテクチャ論は生産性を下げるので、構成を決めずに各自自由に書く
4. まず domain の分離だけから始め、ドメインの複雑さとチームの理解度に合わせて段階的に層を足す

<details><summary>答えと解説</summary>

正解: **4**

効果の大きい一手(ビジネスルールを domain に寄せる)から始め、必要が実証されてから interface 化・ユースケース層と足していくのが、リターンに対してコストを最小に保つ進め方です。最初から全部決めて固定するのはドメイン理解の進化(モデルの深化)と矛盾し、他社構成の丸コピーはドメインもチームも違う前提を無視します。逆に無秩序も、依存の向きという最低限のルールすら守れず論外です。

</details>
