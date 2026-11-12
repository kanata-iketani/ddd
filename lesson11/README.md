# 第11章 リポジトリとファクトリ — 永続化との付き合い方

戦術的設計の仕上げとして、集約を「どう保存し、どう生み出すか」を学びます。リポジトリとファクトリはドメインモデルを DB や ORM(DB の行とオブジェクトを自動で対応づけるライブラリ)の都合から守る防波堤です。この章を終えると、SQLAlchemy(Python の ORM)や GORM(Go の ORM) とドメインモデルの距離感を自分で判断できるようになります。

## 1. リポジトリ — 集約の出し入れを抽象化する

### 基礎: 永続化・SQL・ORM

| 用語 | 意味 |
|---|---|
| **永続化** | プログラムが終わっても消えないように、データを DB などに保存すること。メモリ上のオブジェクトは、プログラムが止まると消える |
| **SQL** | DB に「この行を読め」「この行を追加しろ」と命令する言語 |
| **ORM** | DB の行とプログラムのオブジェクトを自動で対応づけるライブラリ。Python の SQLAlchemy、Go の GORM など |
| **コレクション** | リストや map(辞書)のように、オブジェクトをまとめて入れておく入れ物 |

### リポジトリ — 集約を「コレクションのように」扱う

リポジトリ (Repository) は、集約(L8)の永続化を抽象化するパターンです。呼び出す側から見ると、あたかも**メモリ上のコレクション**に集約を出し入れしているように見えます。DB・SQL・ORM の存在をドメイン層から隠すのが仕事です。

たとえば注文を使う側のコードは `repo.Save(order)` と `repo.FindByID("o-001")` だけを書きます。その裏で PostgreSQL(広く使われている DB 製品)に SQL を送っているのか、テスト用に map に入れているだけなのかを、使う側は知りません。

### インターフェースとは

**インターフェース (interface)** は「このメソッドを持っていること」という約束(契約)だけを書いた型です。中身の処理は書きません。

- Go では、インターフェースに書かれたメソッドを**同じ名前・引数・戻り値ですべて持つ型**は、自動的にそのインターフェースとして扱えます。Java などのように「implements(実装します)」と宣言する必要はありません
- 使う側をインターフェース型で書いておくと、約束を満たす型なら何でも(本物の DB 版でもテスト用の版でも)差し替えて渡せます

鉄則は2つです。

| 鉄則 | 意味 |
|---|---|
| リポジトリは**集約単位** | `OrderRepository` は作るが、明細だけの `OrderItemRepository` は作らない。明細はルート経由でしか触れないため |
| インターフェースは**ドメイン層**、実装は**インフラ層** | 依存性逆転 (Dependency Inversion)。ドメインが DB に依存するのではなく、DB 実装がドメインの契約に従う |

ドメイン層(業務ルールを置く場所)とインフラ層(DB など技術の詳細を置く場所)の分け方は L12、依存性逆転は L13 で詳しく扱います。ここでは「約束はドメイン側が決め、DB 側はそれに合わせて作る」と押さえてください。

```go
// domain/order/repository.go — ドメイン層(インターフェースだけ)
type OrderRepository interface {
    FindByID(id OrderID) (*Order, error)
    FindUnpaidOrders() ([]*Order, error) // ドメインの言葉で命名
    Save(o *Order) error
}

// infra/postgres/order_repository.go — インフラ層(実装)
type PgOrderRepository struct{ db *sql.DB }

func (r *PgOrderRepository) FindUnpaidOrders() ([]*Order, error) {
    // SQL の詳細はここに閉じ込める
}
```

| 部分 | 意味 |
|---|---|
| `type OrderRepository interface { ... }` | 3つのメソッドを持つことを約束するインターフェース |
| `FindByID(id OrderID) (*Order, error)` | ID で注文を1件探す。見つかった注文(のポインタ)とエラーの2つを返す |
| `FindUnpaidOrders() ([]*Order, error)` | 未入金の注文を全部探す。`[]*Order` は「注文へのポインタのスライス(並び)」 |
| `Save(o *Order) error` | 注文を保存する。失敗したらエラーを返す |
| `type PgOrderRepository struct{ db *sql.DB }` | PostgreSQL 版の実装。`*sql.DB` は Go 標準 `database/sql` パッケージの「DB への接続」 |
| `func (r *PgOrderRepository) FindUnpaidOrders() ...` | 約束のメソッドの1つを実装する部分。中身(SQL を送る処理)は省略しているので、このままではコンパイルできない |

テストでは同じインターフェースを満たすインメモリ実装(DB を使わず、メモリ上の map にデータを持つ実装)に差し替えれば、DB なしで集約とユースケースを検証できます。

### 動かして確かめる

インメモリ実装を作り、インターフェース経由で使う完全版です。

```go
package main

import (
	"errors"
	"fmt"
)

type OrderID string

type Order struct {
	id   OrderID
	paid bool
}

// ドメイン層が決める「契約」: この3つのメソッドを持つものなら何でもよい
type OrderRepository interface {
	FindByID(id OrderID) (*Order, error)
	FindUnpaidOrders() ([]*Order, error)
	Save(o *Order) error
}

// テスト用のインメモリ実装: DB の代わりに map に入れておく
type InMemoryOrderRepository struct {
	data map[OrderID]*Order
}

func (r *InMemoryOrderRepository) FindByID(id OrderID) (*Order, error) {
	o, ok := r.data[id]
	if !ok {
		return nil, errors.New("注文が見つかりません: " + string(id))
	}
	return o, nil
}

func (r *InMemoryOrderRepository) FindUnpaidOrders() ([]*Order, error) {
	var out []*Order
	for _, o := range r.data {
		if !o.paid {
			out = append(out, o)
		}
	}
	return out, nil
}

func (r *InMemoryOrderRepository) Save(o *Order) error {
	r.data[o.id] = o
	return nil
}

// 使う側はインターフェースしか知らない
func CountUnpaid(repo OrderRepository) int {
	orders, _ := repo.FindUnpaidOrders()
	return len(orders)
}

func main() {
	var repo OrderRepository = &InMemoryOrderRepository{data: map[OrderID]*Order{}}
	repo.Save(&Order{id: "o-001", paid: true})
	repo.Save(&Order{id: "o-002", paid: false})

	o, err := repo.FindByID("o-002")
	fmt.Println(o.id, o.paid, err)
	_, err = repo.FindByID("o-999")
	fmt.Println(err)
	fmt.Println("未入金の注文数:", CountUnpaid(repo))
}
```

実行結果:

```text
o-002 false <nil>
注文が見つかりません: o-999
未入金の注文数: 1
```

| 部分 | 意味 |
|---|---|
| `data map[OrderID]*Order` | 注文 ID をキー、注文を値とする map。これが DB の代わり |
| `o, ok := r.data[id]` | map から取り出す。キーがあれば `ok` が true、なければ false |
| `var out []*Order` | 空のスライスを用意し、未入金の注文を `append` で足していく |
| `orders, _ := repo.FindUnpaidOrders()` | `_` は「この戻り値(ここではエラー)は使わない」という印 |
| `var repo OrderRepository = &InMemoryOrderRepository{...}` | インメモリ実装をインターフェース型の変数に入れる。3つのメソッドが1つでも欠けていると、ここでコンパイルエラーになる |
| `CountUnpaid(repo)` | 使う側の関数はインターフェースしか知らない。本番で PostgreSQL 版を渡しても、同じコードのまま動く |

出力の1行目は o-002 が見つかったこと(paid は false、エラーは `<nil>`)、2行目は存在しない ID のエラー、3行目は未入金が o-002 の1件であることを示しています。

Python なら `typing.Protocol` で同じ「約束」を書けます。Go と同じく、メソッドがそろっていれば宣言なしで約束を満たした扱いになります。

```python
from typing import Protocol


class Order:
    def __init__(self, order_id: str, paid: bool):
        self.id = order_id
        self.paid = paid


# Go の interface に相当する「契約」
class OrderRepository(Protocol):
    def find_by_id(self, order_id: str) -> Order: ...
    def find_unpaid_orders(self) -> list[Order]: ...
    def save(self, order: Order) -> None: ...


class InMemoryOrderRepository:
    def __init__(self):
        self.data: dict[str, Order] = {}

    def find_by_id(self, order_id: str) -> Order:
        return self.data[order_id]

    def find_unpaid_orders(self) -> list[Order]:
        return [o for o in self.data.values() if not o.paid]

    def save(self, order: Order) -> None:
        self.data[order.id] = order


def count_unpaid(repo: OrderRepository) -> int:
    return len(repo.find_unpaid_orders())


repo = InMemoryOrderRepository()
repo.save(Order("o-001", True))
repo.save(Order("o-002", False))
print("未入金の注文数:", count_unpaid(repo))
```

実行結果:

```text
未入金の注文数: 1
```

`CountUnpaid(repo)` や `count_unpaid(repo)` のように、使う側が自分で DB 接続を作らず、外から渡してもらう形を**注入**(依存性の注入)と呼びます。テストでは本物の代わりにインメモリ実装を注入します。テスト用の偽物は**モック**や**フェイク**とも呼ばれます。CI(コードを push するたびに自動でテストを回す仕組み)に DB を用意しなくても、この方法でユースケースをテストできます。

💡 コツ: 検索メソッドは `SelectByStatus(1)` のような実装の言葉ではなく、`FindUnpaidOrders`(未入金の注文)のように**ユビキタス言語**(L2)で命名しましょう。呼び出し側のコードがそのまま業務の文章として読めるようになります。

最後に注意点をひとつ。「30日以上未入金なら自動キャンセル」のようなビジネスルールをリポジトリの中に書いてはいけません。リポジトリは出し入れの係であり、判断はドメインオブジェクト(集約やドメインサービス)の仕事です。

### 確認問題

**問1.** ECサイトで「注文明細の数量だけを直接 UPDATE したいので、OrderItemRepository を新設する」という提案が出ました。DDD の観点で最も適切な判断はどれでしょう？

1. 明細は頻繁に更新されるので、専用リポジトリを作って効率化すべきである
2. OrderRepository と OrderItemRepository の両方を作り、用途で使い分けるべきである
3. リポジトリは集約単位に作るものなので、明細の変更も Order(集約ルート)を取得してメソッド経由で行い、OrderRepository で保存すべきである
4. リポジトリを使わず、アプリケーションサービスから直接 SQL を発行すべきである

<details><summary>答えと解説</summary>

正解: **3**

リポジトリは集約単位に作るのが鉄則です。明細 (OrderItem) は Order 集約の内部要素なので、ルートを取得 → `order.ChangeQuantity(...)` → `Save(order)` という流れにすれば「明細合計と注文金額の一致」などの不変条件を集約が守れます。明細専用リポジトリを作ると、ルートを素通りした更新で不変条件が壊れる抜け道ができます(選択肢の前半2つが該当)。直接 SQL はさらに悪く、ルールがコードのあちこちに散らばります。試験的なひっかけとして「効率化」という言葉に釣られないようにしましょう。性能問題は実測してから別の手段(読み取り専用モデルなど)で解決します。

</details>

**問2.** OrderRepository のインターフェース定義は、どの層に置くのが適切でしょう？

1. ドメイン層に置き、実装クラスをインフラ層に置く
2. インフラ層に置き、ドメイン層からそれを import する
3. アプリケーション層に置き、ドメイン層と共有する
4. インターフェースは作らず、実装クラスを直接ドメイン層から使う

<details><summary>答えと解説</summary>

正解: **1**

インターフェースはドメイン層、実装はインフラ層に置きます。これが依存性逆転 (Dependency Inversion) で、「ドメインが DB に合わせる」のではなく「DB 実装がドメインの契約に従う」形になります。インフラ層に置くと、ドメイン層がインフラ層に import 依存してしまい、DB の変更がドメインに波及します。アプリケーション層に置く流儀も存在しますが、リポジトリは集約の出し入れというドメインの関心事なので、ドメイン層に置くのが DDD の標準です。インターフェースなしで実装を直接使うと、テストでインメモリ実装に差し替えることもできなくなります。

</details>

**問3.** 「支払い期限を過ぎた注文の一覧」を取得するリポジトリメソッドの命名として、DDD の考え方に最も合うものはどれでしょう？

1. SelectOrdersWhereStatusEquals2()
2. FindOverdueOrders()
3. GetOrdersByFlag(flag int)
4. QueryOrdersTable(condition string)

<details><summary>答えと解説</summary>

正解: **2**

`FindOverdueOrders`(期限超過の注文を探す)は、ドメインエキスパートが読んでも意味がわかるユビキタス言語による命名です。呼び出し側のコードが「期限超過の注文を探して督促する」という業務の文章のまま読めます。`StatusEquals2` や `ByFlag` は DB の実装都合(マジックナンバー)が漏れており、2 が何を意味するかの知識が呼び出し側に散らばります。`QueryOrdersTable(condition string)` は SQL 文字列を外から渡す設計で、抽象化の意味がなくなる最悪の形です。💡 メソッド名に業務の言葉が使えないときは、そもそもモデルの言葉が整理できていないサインです。

</details>

**問4.** レビュー中に、PostgreSQL 用リポジトリ実装の Save メソッド内で「未入金のまま30日経過していたら注文を自動キャンセルする」処理を見つけました。最も適切な指摘はどれでしょう？

1. 保存のたびにチェックできて効率的なので、このままでよい
2. SQL のトリガーに移すべきである
3. インメモリ実装にも同じ処理をコピーすべきである
4. キャンセル判断はビジネスルールなので、ドメイン層(集約のメソッドやドメインサービス)に移すべきである

<details><summary>答えと解説</summary>

正解: **4**

「30日で自動キャンセル」は立派なビジネスルールであり、リポジトリ(永続化の係)に置いてはいけません。ドメイン層に移せば、ルールがテストしやすく、仕様変更(「60日に延ばす」)の変更箇所も一目瞭然です。リポジトリに置くと、インメモリ実装と挙動がズレてテストが実態と乖離します(選択肢のコピー案はまさにその場しのぎで、二重管理のバグの温床)。SQL トリガーはさらに見えない場所にルールが沈み、ドメインモデルとコードの一致という DDD の土台が崩れます。「保存に紛れ込んだ if 文」はリポジトリへのロジック漏れの典型的なにおいです。

</details>

**問5.** CI 環境に PostgreSQL を用意せずに「注文確定ユースケース」のテストを書きたいです。リポジトリパターンを活かした最も素直な方法はどれでしょう？

1. PostgreSQL をモックするために SQL 文字列の組み立て結果を検証する
2. OrderRepository インターフェースを満たすインメモリ実装(map で保持)を作り、テストではそれを注入する
3. テストのときだけリポジトリを経由せず、集約を直接グローバル変数に保存する
4. ユースケースのテストは諦め、DB が使える結合テストだけで検証する

<details><summary>答えと解説</summary>

正解: **2**

インターフェースの最大の恩恵がこれです。`map[OrderID]*Order` で持つだけのインメモリ実装を注入すれば、DB なしで「確定したら在庫引当イベントが出るか」のようなユースケースの振る舞いを高速にテストできます。SQL 文字列の検証は実装の詳細に結合したもろいテストになり、リファクタリングのたびに壊れます。グローバル変数は並行テストで競合しますし、本番と異なる経路を通るためテストの意味が薄れます。結合テストだけに頼ると、フィードバックが遅く境界条件の網羅も困難です。💡 インメモリ実装は本物のリポジトリと同じ契約(インターフェース)を満たすことがポイントです。

</details>

**問6.** Go で `InMemoryOrderRepository` 型を `OrderRepository` インターフェースとして使えるようにするには、何が必要でしょう？

1. 型の宣言に `implements OrderRepository` と書く
2. OrderRepository と同じパッケージに InMemoryOrderRepository を置く
3. OrderRepository を継承した構造体として定義する
4. インターフェースに書かれたメソッドを、同じ名前・引数・戻り値ですべて持たせる

<details><summary>答えと解説</summary>

正解: **4**

Go のインターフェースは、書かれたメソッドをすべて同じ形で持つ型なら、宣言なしで自動的に満たしたことになります。implements という書き方は Java などの言語のもので、Go にはありません。同じパッケージに置く必要もなく、実際にリポジトリの実装はインフラ層という別パッケージに置きます。Go には構造体の継承もありません。メソッドが1つでも欠けていると、インターフェース型の変数に入れるところでコンパイルエラーになります。

</details>

## 2. ファクトリ — 生成と再構築

### ファクトリ — 「正しい状態でしか生まれない」を保証する

ファクトリ (Factory) は、複雑なオブジェクト生成の手順を隠蔽し、**不変条件(L8)を満たした状態でしか集約が生まれない**ことを保証するパターンです。「オブジェクトを作る専門の係」と考えてください。

とはいえ、すべてにファクトリが必要なわけではありません。

| 状況 | 手段 |
|---|---|
| 引数の検証だけで作れる | コンストラクタ関数(L6 で見た `NewX` 形式の生成関数) `NewOrder(...)` で十分 |
| 生成に他集約の情報が要る(顧客ランクで与信枠が変わる等) | ファクトリ |
| 生成ルールが複雑・条件分岐が多い | ファクトリ |

与信枠は「後払いで買ってよい金額の上限」のことです。

もうひとつ重要なのが、**新規生成 (create) と再構築 (reconstitute) の区別**です。再構築とは、DB に保存済みのデータから集約を復元することです。

```go
// 新規生成: ビジネスルールを検証する(10行超の注文は作れない等)
func NewOrder(customer CustomerID, items []OrderItem) (*Order, error) {
    if len(items) > maxItems {
        return nil, ErrTooManyItems
    }
    return &Order{id: NewOrderID(), items: items}, nil
}

// 再構築: 保存済みデータからの復元。生成時ルールは再適用しない
func ReconstructOrder(id OrderID, items []OrderItem, status Status) *Order {
    return &Order{id: id, items: items, status: status}
}
```

| 部分 | 意味 |
|---|---|
| `if len(items) > maxItems {` | 明細の行数が上限(定数 `maxItems`、定数は L2)を超えていたら |
| `return nil, ErrTooManyItems` | 注文は作らず(`nil`)、エラーを返す |
| `&Order{id: NewOrderID(), items: items}` | 新しい ID を採番(新しい番号を割り当てること)して注文を作り、そのポインタを返す(`&` は L6) |
| `ReconstructOrder(...) *Order` | 戻り値にエラーがない。DB から読んだ値をそのまま詰めるだけで、検査をしないため |
| `id: id, ..., status: status` | ID も状態も、新しく作らず保存されていた値を使う |

なぜ再構築では検証しないのでしょうか。保存されているデータは**当時のルールで妥当だったもの**だからです。「明細は10行まで」というルールを後から追加した場合、昔の12行の注文も読み込めなければ業務が止まってしまいます。

### 動かして確かめる

新規生成・再構築に加えて、顧客のランクで与信枠が変わるファクトリ `OrderFactory.CreateFor` も入れた完全版です(簡単のため、明細は金額の数字だけにしています)。

```go
package main

import (
	"errors"
	"fmt"
)

const maxItems = 10

var ErrTooManyItems = errors.New("明細は10行までです")
var ErrOverCreditLimit = errors.New("与信枠を超えています")

type Order struct {
	id     string
	items  []int // 簡単のため明細は金額だけ
	status string
}

// 新規生成: ビジネスルールを検証する
func NewOrder(id string, items []int) (*Order, error) {
	if len(items) > maxItems {
		return nil, ErrTooManyItems
	}
	return &Order{id: id, items: items, status: "下書き"}, nil
}

// 再構築: 保存済みデータからの復元。生成時ルールは再適用しない
func ReconstructOrder(id string, items []int, status string) *Order {
	return &Order{id: id, items: items, status: status}
}

// ファクトリ: 生成に別集約(顧客)の情報が要る場合
type Customer struct {
	rank string // "gold" または "normal"
}

type OrderFactory struct{}

func (f OrderFactory) CreateFor(c Customer, id string, items []int) (*Order, error) {
	limit := 50000 // 一般会員の与信枠
	if c.rank == "gold" {
		limit = 200000
	}
	total := 0
	for _, p := range items {
		total += p
	}
	if total > limit {
		return nil, ErrOverCreditLimit
	}
	return NewOrder(id, items)
}

func main() {
	twelve := make([]int, 12) // 金額0の明細を12行(行数の確認用)

	_, err := NewOrder("o-001", twelve)
	fmt.Println("新規生成(12行):", err)

	old := ReconstructOrder("o-000", twelve, "確定")
	fmt.Println("再構築(12行):", old.id, len(old.items), "行", old.status)

	f := OrderFactory{}
	_, err = f.CreateFor(Customer{rank: "normal"}, "o-002", []int{80000})
	fmt.Println("一般会員 8万円:", err)
	o, err := f.CreateFor(Customer{rank: "gold"}, "o-003", []int{80000})
	fmt.Println("ゴールド会員 8万円:", o.id, o.status, err)
}
```

実行結果:

```text
新規生成(12行): 明細は10行までです
再構築(12行): o-000 12 行 確定
一般会員 8万円: 与信枠を超えています
ゴールド会員 8万円: o-003 下書き <nil>
```

| 出力 | 何が起きたか |
|---|---|
| 新規生成(12行) | `NewOrder` が「10行まで」のルールで生成を拒んだ |
| 再構築(12行) | 同じ12行でも `ReconstructOrder` は検査せずに復元した(ルール追加前に保存された注文の読み込みを想定) |
| 一般会員 8万円 | ファクトリが顧客(別集約)のランクを見て、与信枠 5 万円を超えるので拒んだ |
| ゴールド会員 8万円 | 与信枠 20 万円以内なので、最後に `NewOrder` を呼んで「下書き」の注文ができた |

ファクトリ `CreateFor` は、顧客という別集約の情報を引数で受け取り、与信のルールを適用してから `NewOrder` に生成を任せています。注文のコンストラクタの中で顧客を DB から読みに行く必要がありません。

### この章の問題に出てくる用語

| 用語 | 意味 |
|---|---|
| new キーワード | Java などでオブジェクトを作るときに書く構文(`new Order()`)。Go にはコンストラクタ専用の構文はなく、`NewOrder` のような関数で代用する |
| 抽象ファクトリ | ファクトリ自体をインターフェースにして、作り方ごと差し替えられるようにする設計パターン。必要になるまで作らなくてよい |

💡 コツ: `ReconstructOrder` はリポジトリ(インフラ層)専用の入口です。アプリケーションサービスが新規作成に使ってしまうと検証をすり抜けるので、パッケージ構成やレビューで誤用を防ぎましょう。

### 確認問題

**問1.** ファクトリパターンを導入する一番の目的として最も適切なものはどれでしょう？

1. オブジェクト生成を高速化するため
2. 不変条件を満たした正しい状態のオブジェクトしか生まれないことを保証するため
3. newキーワードの使用を減らしてコードを短くするため
4. すべての集約の生成方法を統一するため

<details><summary>答えと解説</summary>

正解: **2**

ファクトリの本質は「生成の複雑さを隠蔽し、不正な状態のオブジェクトが世の中に存在できないようにする」ことです。生成直後から不変条件が成立していれば、使う側は常に正しい前提でロジックを書けます。高速化やコードの短さは目的ではありません。「すべての集約で統一」も誤りで、単純な生成はコンストラクタ関数で十分です。道具をそろえること自体が目的化すると、無意味なファクトリクラスが量産されるアンチパターンになります。

</details>

**問2.** メールアドレスの形式チェックだけして会員オブジェクトを作るコードがあります。この生成に専用のファクトリクラスを用意すべきでしょうか？

1. DDDではすべての生成をファクトリにすべきなので用意する
2. リポジトリに生成メソッドを追加する
3. 将来の拡張に備えて抽象ファクトリを先に設計しておく
4. 引数検証だけならコンストラクタ関数(Goなら NewMember)で十分なので、ファクトリは不要である

<details><summary>答えと解説</summary>

正解: **4**

生成の複雑さが「引数の検証だけ」なら、`NewMember(email)` のようなコンストラクタ関数で十分です。ファクトリが正当化されるのは、他集約の情報が必要、生成ルールの条件分岐が複雑、生成する型が状況で変わる(ポリモーフィック)といった場合です。「すべてファクトリ」はパターンの過剰適用で、コードの追跡を無駄に難しくします。リポジトリに生成を持たせるのは責務違反(リポジトリは出し入れの係)。「将来に備えた抽象ファクトリ」は使うかわからない抽象化で、YAGNIに反します。💡 迷ったらまずコンストラクタ関数、複雑になったら昇格、が現実的な順序です。

</details>

**問3.** 新規生成 (create) と再構築 (reconstitute) を区別する理由として最も適切なものはどれでしょう？

1. 保存済みデータは当時のルールで妥当だったものなので、再構築では現在の生成時ルールを再適用せずに復元する必要があるため
2. 再構築のほうが処理が速いため
3. 新規生成はアプリケーション層、再構築はプレゼンテーション層の仕事だから
4. 再構築ではIDの採番が不要になり、DBの負荷が下がるため

<details><summary>答えと解説</summary>

正解: **1**

後からルールを厳しくしたとき(例:「明細は10行まで」を追加)、昔のデータがその検証で弾かれて読み込めなくなったら業務が止まります。だから再構築は「生成時のビジネスルール検証をスキップして忠実に復元する」入口として分けるのです。速度やDB負荷は本質ではありません。再構築はリポジトリ(インフラ層)が使うもので、プレゼンテーション層の仕事でもありません。なお「検証ゼロ」ではなく、型としての整合(enum範囲など)は復元時も守る点に注意しましょう。

</details>

**問4.** 「注文の与信枠は顧客のランク(別集約)によって決まる」ため、注文の新規生成に顧客情報が必要になりました。最も適切な設計はどれでしょう？

1. Order のコンストラクタ内で CustomerRepository を呼び出して顧客を取得する
2. 与信チェックを省略し、あとでバッチで検証する
3. 顧客情報を引数に取るファクトリ(例: OrderFactory.CreateFor(customer, items))に生成を任せ、生成時に与信ルールを適用する
4. アプリケーションサービスに if 文で与信ロジックを直接書く

<details><summary>答えと解説</summary>

正解: **3**

他集約の情報を使う生成は、ファクトリの出番の典型です。顧客を引数で受け取るファクトリにすれば、与信ルールがドメイン層の1箇所に集まり、テストも「ランクBの顧客は50万円まで」のように書けます。コンストラクタ内からリポジトリを呼ぶと、ドメインオブジェクトがインフラに依存し、テストのたびにDBが要る設計になります。アプリケーションサービスに if 文で書くとルールがユースケースごとに重複し始めます(貧血化の入口)。バッチで後追い検証は、不正な注文が一時的に存在することを許す別の設計判断であり、与信のような即時性が求められるルールには不適です。

</details>

**問5.** 同僚が「ReconstructOrder は検証がなくて楽だから」と、ユーザーからの新規注文リクエスト処理に ReconstructOrder を使っています。最も適切な指摘はどれでしょう？

1. 検証ロジックをReconstructOrderの中にも追加して両対応にすべきである
2. 再構築はDBからの復元専用の入口であり、新規作成に使うと不変条件の検証をすり抜けるので、NewOrder(またはファクトリ)に戻すべきである
3. 動いているなら問題ないので、そのままでよい
4. ReconstructOrderを高速な分だけ本番向きなので、むしろ全箇所で統一すべきである

<details><summary>答えと解説</summary>

正解: **2**

ReconstructOrder は「保存済みの(=当時妥当だった)データを忠実に復元する」ための裏口であり、新規作成に使うと「明細10行まで」のような不変条件の検証を素通りした不正な注文が生まれます。これは数日後にデータ不整合として発覚する類のバグです。Reconstruct に検証を足す案は、再構築の目的(古いデータも読めること)を壊すので本末転倒。「動いているからよい」は不正データが作られるまでの時限爆弾です。💡 Go ならReconstruct系をインフラ層のパッケージからしか呼べない場所に置く、Python なら命名とレビュー規約で守る、といった誤用防止が有効です。

</details>

**問6.** 「明細は10行まで」というルールを追加する前に、12行の注文が DB に保存されていました。この注文を DB から読み込むときに使うべき関数と、その結果として正しいものはどれでしょう？

1. NewOrder を使う。ルール違反なのでエラーになり、読み込めないのが正しい
2. NewOrder を使う。12行目以降を切り捨てて10行で読み込む
3. ReconstructOrder を使う。生成時のルールを再適用しないので、12行のまま復元できる
4. ReconstructOrder を使う。12行を検出して自動で2つの注文に分割する

<details><summary>答えと解説</summary>

正解: **3**

保存済みデータの読み込みは再構築なので ReconstructOrder を使います。保存されたデータは当時のルールで妥当だったものなので、今の生成時ルールで弾くと昔の注文が読めなくなり、業務が止まります。NewOrder で読もうとすると「明細は10行までです」のエラーになります。勝手に明細を切り捨てたり分割したりするのは、保存されている事実を書き換えることになり不適切です。

</details>

## 3. ORM・DBスキーマとの付き合い方

### ORM・DB スキーマとの付き合い方

ドメインモデルと DB スキーマ(テーブルの列や型などの設計)は**別物**です。テーブルが正規化(データの重複をなくすようにテーブルを分けること)の都合で分かれていても、ドメインでは1つの集約かもしれません。逆もしかりです。モデルをテーブル構造に引きずられると、「DB に保存しやすい形」が設計を支配し始めます。

| DB のテーブル | ドメインの集約 |
|---|---|
| orders(注文の行) | Order 集約の本体 |
| order_items(明細の行) | Order 集約の中の明細 |
| order_discounts(割引の行) | Order 集約の中の割引 |

テーブルは3つでも、「明細の合計 = 注文の合計」のように一緒に守るルールがあるなら、集約は Order の1つです。

実装上の選択肢は大きく2つです。

| 構成 | 利点 | コスト |
|---|---|---|
| ORM モデルとドメインオブジェクトを**分離**(マッピング層を挟む) | ドメインが純粋に保たれる。不変条件を非公開フィールドで守れる | 変換コードを書く手間 |
| ORM モデルを**そのまま**ドメインに使う | 手軽 | setter 全開放・遅延ロードなどインフラ都合がモデルを侵食 |

マッピング層は「DB の行の形」と「ドメインオブジェクトの形」を相互に変換するコードです。setter は値を書き換えるだけのメソッドです。

SQLAlchemy の宣言的モデル(Python のクラスにテーブルの列を書いて定義する方式。`DeclarativeBase` を継承して作る)や GORM の構造体タグ(L4 で見た、フィールドの後ろに書く `gorm:"..."` のような印)は便利ですが、そのままドメインオブジェクトにすると「全フィールド公開・いつでも書き換え可能」になり、第8章で作った「ルート経由でしか変更できない集約」が崩れます。Go では sqlc(SQL 文から Go のコードを自動生成するツール)で生成した行構造体 → ドメインオブジェクトへの変換をリポジトリ実装内で行うのが相性のよい構成です。

```go
func (r *PgOrderRepository) FindByID(ctx context.Context, id OrderID) (*Order, error) {
    row, err := r.q.GetOrder(ctx, string(id)) // sqlc生成コード
    if err != nil {
        return nil, err
    }
    items := toDomainItems(row.Items)
    return ReconstructOrder(OrderID(row.ID), items, Status(row.Status)), nil
}
```

| 行 | 意味 |
|---|---|
| `ctx context.Context` | 処理の期限や中止の合図を運ぶ Go 標準の引数(L10 ch03) |
| `r.q.GetOrder(ctx, string(id))` | sqlc が生成した「注文を1行読む」関数を呼ぶ。`string(id)` は独自型 OrderID を普通の文字列に変換している |
| `if err != nil { return nil, err }` | 読めなかったら(行がない・DB に接続できない等)、注文なしでエラーを返す |
| `toDomainItems(row.Items)` | 行の形の明細を、ドメインの明細に変換する |
| `ReconstructOrder(OrderID(row.ID), ...)` | ch02 の再構築用の関数で集約を復元する。`OrderID(row.ID)` `Status(row.Status)` は、行の素の値を独自型に変換している |

### 動かして確かめる

DB の代わりに「読み込んだ行」を手で用意し、行 → ドメインへの変換だけを動かします。

```go
package main

import "fmt"

// ---- インフラ側: DB の行の形(sqlc などが生成する構造体のイメージ) ----
type OrderRow struct {
	ID     string
	Status int // DB では数値で保存: 1=下書き, 2=確定
}

type OrderItemRow struct {
	OrderID   string
	ProductID string
	Qty       int
}

// ---- ドメイン側: フィールドは非公開 ----
type OrderID string
type Status int

type OrderItem struct {
	productID string
	qty       int
}

type Order struct {
	id     OrderID
	items  []OrderItem
	status Status
}

func ReconstructOrder(id OrderID, items []OrderItem, status Status) *Order {
	return &Order{id: id, items: items, status: status}
}

// 行の形 → ドメインの形 への変換(リポジトリ実装の中に置く)
func toDomainItems(rows []OrderItemRow) []OrderItem {
	items := make([]OrderItem, 0, len(rows))
	for _, r := range rows {
		items = append(items, OrderItem{productID: r.ProductID, qty: r.Qty})
	}
	return items
}

func main() {
	// orders テーブルの1行と、order_items テーブルの2行を読んだとする
	row := OrderRow{ID: "o-001", Status: 2}
	itemRows := []OrderItemRow{
		{OrderID: "o-001", ProductID: "book", Qty: 1},
		{OrderID: "o-001", ProductID: "pen", Qty: 3},
	}

	o := ReconstructOrder(OrderID(row.ID), toDomainItems(itemRows), Status(row.Status))
	fmt.Printf("%+v\n", *o)
}
```

実行結果:

```text
{id:o-001 items:[{productID:book qty:1} {productID:pen qty:3}] status:2}
```

`%+v` は構造体をフィールド名つきで表示する書式、`*o` はポインタ `o` が指している中身そのものです。orders の1行と order_items の2行が、明細2つを持つ1つの Order 集約になりました。ドメイン側の `Order` のフィールドはすべて小文字で始まる非公開(L8)なので、パッケージの外から勝手に書き換えられません。DB の都合(行が分かれている、状態が数値で保存されている)は変換の中に閉じ込められています。

保存の原則は**集約丸ごと**です。「変更があった明細だけ UPDATE」のような部分更新は、不変条件(合計の一致など)を検証しないまま断片を書き込む危険があります。性能が問題になってから、変更追跡やユニットオブワーク (Unit of Work: 1トランザクション内の変更をまとめてコミットする仕組み) を検討します。

### 遅延ロードと N+1 問題

ORM の**遅延ロード (lazy loading)** は、関連するデータ(注文の明細など)を最初には読まず、プログラムがそれにアクセスした瞬間に初めて DB から読む仕組みです。便利ですが、ループの中で使うと SELECT(DB から読む SQL)が大量に発生します。

| 処理 | 発行される SELECT |
|---|---|
| 注文100件の一覧を読む | 1回 |
| ループで各注文の `order.Items` にアクセスする | 注文ごとに1回 × 100件 = 100回 |
| 合計 | 1 + 100 = 101回 |

最初の1回に、件数分の N 回が加わるので **N+1 問題**と呼びます。

💡 コツ: ORM の遅延ロードは集約の境界を曖昧にします。「ロードしたつもりのない関連がアクセス時に SELECT される」のは、集約の外を勝手に歩き回っているサインです。集約は必要なものを最初から丸ごとロードしましょう。

### この章の問題に出てくる用語

| 用語 | 意味 |
|---|---|
| Alembic | SQLAlchemy 用のマイグレーション(テーブル定義の変更を順番に管理して DB に適用すること)ツール |
| Pydantic | FastAPI でリクエストの形を定義・検査するライブラリ(L12 ch03) |
| キャッシュ | 一度読んだデータを手元に取っておき、次から DB を読まずに使う仕組み |
| コネクションプール | DB への接続をいくつか作っておき、使い回す仕組み |

### 確認問題

**問1.** 注文テーブルが正規化の都合で orders / order_items / order_discounts の3テーブルに分かれています。ドメインモデルの設計として最も適切な考え方はどれでしょう？

1. テーブルごとに1クラスずつ作り、3つのリポジトリを用意する
2. ドメインモデルとDBスキーマは別物なので、ビジネスの不変条件を基準に(例: Order集約1つに明細と割引を内包)モデルを設計し、テーブルとの対応はリポジトリ実装で吸収する
3. テーブルを1つに統合してからモデルを設計する
4. JOINが減るようにモデルを設計する

<details><summary>答えと解説</summary>

正解: **2**

モデルの形を決めるのはビジネスの不変条件であって、テーブル構造ではありません。「割引込みの合計金額が正しいこと」を守る単位が Order 集約なら、3テーブル→1集約の変換はリポジトリ実装(マッピング)の仕事です。テーブルごとにクラスとリポジトリを作る案は、DB都合がモデルを支配する典型で、不変条件を守る場所がなくなります。テーブル統合やJOIN削減を先に考えるのは、パフォーマンスという別の関心事で設計を歪める順序の誤りです(性能は実測してから対処)。

</details>

**問2.** FastAPI + SQLAlchemy のプロジェクトで、宣言的モデル(DeclarativeBase のクラス)をそのままドメインオブジェクトとして使い続けた場合に起こりやすい問題はどれでしょう？

1. SQLの発行回数が必ず増える
2. Alembicのマイグレーションが使えなくなる
3. 全属性が外から自由に書き換えられ、遅延ロードなどインフラ都合がモデルに入り込み、集約の不変条件を守る設計が崩れていく
4. Pydanticとの連携ができなくなる

<details><summary>答えと解説</summary>

正解: **3**

SQLAlchemy の宣言的モデルは「DBの行を映すもの」なので、全属性が公開で、セッションの状態や遅延ロードといったインフラの事情を引きずります。これをドメインオブジェクトと兼用すると、`order.total = 0` のような無検証の書き換えがどこからでもでき、集約で守りたい不変条件が形骸化します。SQL回数やAlembic、Pydantic連携は兼用と直接の関係がありません。なお小規模なCRUDなら兼用で十分なことも多く、「複雑なルールを持つコアドメインでは分離する」という使い分けが現実解です。

</details>

**問3.** 性能改善のため「変更された明細の行だけをUPDATEし、Order全体は保存しない」実装が提案されました。DDDの観点で最も注意すべきリスクはどれでしょう？

1. UPDATE文が長くなり可読性が下がる
2. 集約の不変条件(明細合計と注文金額の一致など)を検証しないまま断片だけがDBに書き込まれ、整合性が壊れる可能性がある
3. インデックスが効かなくなる
4. トランザクションが使えなくなる

<details><summary>答えと解説</summary>

正解: **2**

集約は「丸ごと読み、不変条件を検証し、丸ごと保存する」のが原則です。断片だけの部分更新は、集約がせっかく守っている整合性(合計一致、行数上限など)の検証を素通りする抜け道になります。可読性・インデックス・トランザクションは部分更新の本質的リスクではありません。性能が実測で問題になった場合は、変更追跡付きのユニットオブワーク、あるいは読み取り側を別モデルに逃がす(CQRS的な分離)など、整合性を保ったままの最適化を検討します。💡「速くするために境界を破る」のは最後の手段で、まず測ることが先です。

</details>

**問4.** GORMの遅延ロードを有効にしたところ、画面表示のループ中に order.Items へアクセスするたびSELECTが走っていました(N+1問題)。DDDの集約の考え方に沿った対処はどれでしょう？

1. ループの前にキャッシュを足して SELECT を減らす
2. 画面表示用に全テーブルをJOINしたビューを集約として扱う
3. DBのコネクションプールを増やす
4. 集約は境界内を最初から丸ごとロードする方針に変え、リポジトリの FindByID で明細まで一括取得する(表示専用なら読み取りモデルの分離も検討)

<details><summary>答えと解説</summary>

正解: **4**

遅延ロードは「集約の境界があいまいなままアクセス時に外を歩き回る」動きで、N+1はその症状です。集約は不変条件を守る単位なので、必要なデータ(明細)は最初から丸ごとロードするのが原則で、N+1も自然に消えます。一覧画面のように集約の形が過剰な場面では、読み取り専用のクエリモデルに分ける選択肢もあります(第14章のCQRSにつながる考え方)。キャッシュやコネクションプールは症状への対症療法で、境界のあいまいさという原因が残ります。全テーブルJOINのビューを「集約」と呼ぶのはモデルと表示都合の混同です。

</details>

**問5.** ユニットオブワーク (Unit of Work) の説明として最も適切なものはどれでしょう？

1. 1つのユースケースを1つの関数にまとめる設計手法
2. 1トランザクション内で発生した変更をまとめて追跡し、最後に一括でコミットする仕組み
3. リポジトリを集約ごとに1つずつ作るルール
4. DBの負荷をワーカープロセスに分散する仕組み

<details><summary>答えと解説</summary>

正解: **2**

ユニットオブワークは「この業務処理の中で何が変更されたか」を追跡し、トランザクションの最後にまとめて書き込む仕組みです。SQLAlchemy の Session はまさにこれの実装で、Go では自前のトランザクション管理+変更追跡で近いことをします。複数の書き込みが「全部成功 or 全部失敗」になるので、中途半端な状態が残りません。関数のまとめ方やリポジトリの個数ルール、負荷分散の仕組みではありません。💡「1トランザクション=1集約」の原則と組み合わせると、UoWが扱う変更は基本的に1集約+発行イベントの保存(アウトボックス)に収まります。

</details>

**問6.** ORM の遅延ロードが有効な状態で、注文50件の一覧を1回の SELECT で読み、ループで各注文の明細 (`order.Items`) にアクセスしました。SELECT は合計何回発行されるでしょう？

1. 51回(一覧の1回 + 注文ごとに1回 × 50件)
2. 1回(最初の一覧だけ)
3. 2回(一覧と明細を1回ずつ)
4. 50回(注文ごとに1回だけで、一覧の分は数えない)

<details><summary>答えと解説</summary>

正解: **1**

遅延ロードは、関連データにアクセスした瞬間にその都度 SELECT を発行します。一覧で1回、各注文の明細で50回、合計51回になります。これが N+1 問題です(N=50)。明細まで最初からまとめて読む(集約を丸ごとロードする)と、一覧の1回+明細の1回のように少ない回数にできます。

</details>
