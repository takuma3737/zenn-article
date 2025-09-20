---
title: "[go1.25]json/v1→v2で何が変わったん？〜気になった変更点をお届け〜"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Go,JSON]
published: true
---

# はじめに
Go1.25リリースに伴い、GoのJSONパッケージもv2が実験的にリリースされました。私的に気になったv1→v2への変更点をお届けします。

# 1. GoでのJSON利用のユースケース

1. **API通信**
    1. Webサーバーやクライアントでリクエスト/レスポンスのやり取り
    2. 例:APIのリクエストボディをstructにUnmarshal、レスポンスをMarshalして返す(oapiの生成コードを見るとそうなってた)
2. **設定ファイルの読み書き**
    1. JSON形式でアプリケーションの設定を管理する
    2. 例：vscodeのsetting.json
3. **ログやデータ保存**
    1. 構造化ログをJSON形式で出力 → 集約基盤（Cloud Logging, [Datadog](https://qiita.com/kurakura0916/items/c7901f95e34da7c66ceb)など）に流す

# 2. なぜJSON周りの改善が重要なのか

1. **利用頻度が高い**
    1. APIやログは毎日触るレベル。ユーザーが何か操作するたびにAPIが実行されるのでユーザー体験に直結する.
    
    → JSONはAPI・設定・ログなど、どのサービスでも使う中核技術
    

# 3. v1の抱えていた課題

1. **omitemptyが柔軟ではない**

:::details 詳細

v1 ではフィールドタグ `json:"FieldName,omitempty"` を付けたとき、 そのフィールド値が次のいずれかに該当するときだけ「省略」されます
    

- `false`（bool）
- `0`（整数 / 浮動小数）
- `""`（空文字列）
- `len == 0` の配列 / スライス / マップ
- `nil` のポインタ
- `nil` のインタフェース値
- `nil` のチャンネル / 関数（ほぼ使わない）
- `nil` の `struct` など
    
逆に言ったら以下のようなゼロっぽく見えても消えない
- `time.Time` のゼロ（`time.Time{}`）
- 中身がゼロ値の struct

**典型的な困るケース**
1. ゼロ時刻を消したい！

    ```go
    type Event struct {
        Start time.Time `json:"start,omitempty"` // ゼロでも出力される
    }
    ```

→ポインタ化することで値が入らなかったらnullとして扱われるためフィールドが削除される。
    
2. 空structを「フィールドごと」消したい

    ```go
    type Filter struct {
        // すべてのフィールドが omitempty で落ちた結果 {} になる
        // でも Filter 自体は struct 値なので親からは消えない
    }
    type Request struct {
        Filter Filter `json:"filter,omitempty"` // {} が必ず残る
    }
    ```

→ 構造体は値型で実体が必ず生成されるため、API 仕様で「空ならフィールド自体を省略したい」ができない。回避策としてはポインタ化する必要がある。
    
**なぜ「柔軟でない」と感じるか**
- goとjsonでnullとemptyの定義が異なるが、それが考慮されていない
  - goは値型と参照型があり、それぞれ初期値がemptyとnull扱いになる。
  - JSONはキーがあってnull（{ "foo": null }）かキーがあってempty(””や[])という区別がある
  - omiteemptyのルールとして、フィールドがゼロ値ならJSON出力時に省略されるが、strcut{}はゼロ値だがnullが入ってしまうので消えない、みたいな事象が起きる。[参考](https://qiita.com/c6tower/items/aa9adf05ba54229b27fa)
    
結果として、不必要なポインタ化やラッパーを作ることになる
    
:::

2. **Unmarshalで大文字小文字の区別が出来ない**
   1. Unmarshal時に、JSONオブジェクト名とGo構造体フィールド名が大文字と小文字を区別されないため、セキュリティ脆弱性とパフォーマンス制限をもたらす可能性がある。
   2. **`"FirstName"`**と**`"firstname"`**のいずれも**`FirstName`**フィールドにマッピングされる
3. **MarshalJsonとUnmarshalJsonでパフォーマンス改善の余地あり**
   1. 内部実装の詳細はさておき、MarshalJsonでは解析結果を再フォーマットしたり、UnmarshalJsonでは呼び出す前と呼び出しで二重解析したりと無駄が多い
   2. こちらを[参照](https://arc.net/l/quote/xmqhojoa)

# 4. v2への変更の概要

1. **omitzeroの導入**

v2 では `omitempty` の再定義と、新しい `omitzero` の導入で柔軟性を高めた

| **タグ** | **意味（v2）** |
| --- | --- |
| `omitempty` | 「そのフィールドを JSON にエンコードした結果が 空JSON値(`null` / `""` / `[]` / `{}`) なら省略」 |
| `omitzero` | フィールドがGo値のゼロ値であるか、trueを返す「IsZero() bool」メソッドを実装している場合に省略 |

 ※v2でv1と同一の動作を望む場合、**`json.OmitEmptyWithLegacyDefinition(true)`**オプションを使用

2. **デフォルトで大文字・小文字を区別されるようになった**
   1. `json:"...,case:ignore"`タグオプションを使用してフィールドごとに大文字小文字の無視を明示的に有効にしたり、`json.MatchCaseInsensitiveNames(true)`オプションで全体に適用したりできるらしい
3. **特にUnmarshalで性能が向上した**
   1. [こちら](https://arc.net/l/quote/enxbzqum)を参照

# 5. コード例

1. **omitzeroとomitemptyの比較**
:::details 詳細

```go
type MyStruct struct {
	Foo string `json:",omitzero"`
	Bar []int  `json:",omitempty"`
	Baz *MyStruct `json:",omitzero,omitempty"`
}

// Demonstrate behavior of "omitzero".
b, err := json.Marshal(struct {
	Bool         bool        `json:",omitzero"`
	Int          int         `json:",omitzero"`
	String       string      `json:",omitzero"`
	Time         time.Time   `json:",omitzero"`
	Addr         netip.Addr  `json:",omitzero"`
	Struct       MyStruct    `json:",omitzero"`
	SliceNil     []int       `json:",omitzero"`
	Slice        []int       `json:",omitzero"`
	MapNil       map[int]int `json:",omitzero"`
	Map          map[int]int `json:",omitzero"`
	PointerNil   *string     `json:",omitzero"`
	Pointer      *string     `json:",omitzero"`
	InterfaceNil any         `json:",omitzero"`
	Interface    any         `json:",omitzero"`
}{
	Bool:         false,
	Int:          0,
	String:       "",
	Time:         time.Date(1, 1, 1, 0, 0, 0, 0, time.UTC),
	Addr:         netip.Addr{},
	Struct:       MyStruct{Bar: []int{}, Baz: new(MyStruct)},
	SliceNil:     nil,
	Slice:        []int{},
	MapNil:       nil,
	Map:          map[int]int{},
	PointerNil:   nil,
	Pointer:      new(string),
	InterfaceNil: nil,
	Interface:    (*string)(nil),
})
if err != nil {
	log.Fatal(err)
}
(*jsontext.Value)(&b).Indent()
fmt.Println("OmitZero:", string(b))

// Demonstrate behavior of "omitempty".
b, err = json.Marshal(struct {
	Bool         bool        `json:",omitempty"`
	Int          int         `json:",omitempty"`
	String       string      `json:",omitempty"`
	Time         time.Time   `json:",omitempty"`
	Addr         netip.Addr  `json:",omitempty"`
	Struct       MyStruct    `json:",omitempty"`
	Slice        []int       `json:",omitempty"`
	Map          map[int]int `json:",omitempty"`
	PointerNil   *string     `json:",omitempty"`
	Pointer      *string     `json:",omitempty"`
	InterfaceNil any         `json:",omitempty"`
	Interface    any         `json:",omitempty"`
}{
	Bool:         false,
	Int:          0,
	String:       "",
	Time:         time.Date(1, 1, 1, 0, 0, 0, 0, time.UTC),
	Addr:         netip.Addr{},
	Struct:       MyStruct{Bar: []int{}, Baz: new(MyStruct)},
	Slice:        []int{},
	Map:          map[int]int{},
	PointerNil:   nil,
	Pointer:      new(string),
	InterfaceNil: nil,
	Interface:    (*string)(nil),
})
if err != nil {
	log.Fatal(err)
}
(*jsontext.Value)(&b).Indent()
fmt.Println("OmitEmpty:", string(b))

---
output:

OmitZero: {
	"Struct": {},
	"Slice": [],
	"Map": {},
	"Pointer": "",
	"Interface": null
}
OmitEmpty: {
	"Bool": false,
	"Int": 0,
	"Time": "0001-01-01T00:00:00Z"
}
```

**ざっくりと、、、**
    
omitzero: **Go言語のゼロ値基準**
    
omitempty: **JSONの空値基準（null、""、[]、{}）**
    
**【注意】omitzeroとomitemptyどちらでも省略されるケース**
    
1. String : ””
   1. Goのゼロ値だが、空文字はJSONの空値だから
2. PointerNil、InterfaceNil
   1. Goのゼロ値だが、nilはJSONの空値だから
::: 

2. **大文字・小文字の比較**
:::details 詳細
    
**※v2での動作**

```go
// JSON入力（さまざまなキーの表記揺れを含む）
const input = `[
		{"firstname": true},
		{"firstName": true},
		{"FirstName": true},
		{"FIRSTNAME": true},
		{"first_name": true},
		{"FIRST_NAME": true},
		{"first-name": true},
		{"FIRST-NAME": true},
		{"unknown": true}
	]`

// "case:ignore" を付けない場合、Unmarshal は完全一致のみを探す
var caseStrict []struct {
	X bool `json:"firstName"`
}
if err := json.Unmarshal([]byte(input), &caseStrict); err != nil {
	log.Fatal(err)
}
fmt.Println(caseStrict) // → 完全一致した1件のみマッチする

// "case:ignore" を付けた場合、まず完全一致を探し、
// 見つからなければ大文字小文字を無視してマッチングする
var caseIgnore []struct {
	X bool `json:"firstName,case:ignore"`
}
if err := json.Unmarshal([]byte(input), &caseIgnore); err != nil {
	log.Fatal(err)
}
fmt.Println(caseIgnore) // → 8件すべてマッチする

---
output

[{false} {true} {false} {false} {false} {false} {false} {false} {false}]
[{true} {true} {true} {true} {true} {true} {true} {true} {false}]
    
 ```

:::

# 6. まとめ

- JSONの扱いがより柔軟になった
- エンコード/デコードの性能が上がったので多少API処理が速くなるかも？
- まだ実験的な段階で、v1との互換性が保証されていないので確立されるまでは現状維持で良いかも(公式がそう言ってる)

他にも色々変更点があるので気になったらぜひ！！

- 新しいAPI
- UTF-8の処理
- time.Durationの処理
- nilスライス・マップのマーシャリング

などなど、、

# 参考

https://future-architect.github.io/articles/20250806a/

https://gosuda.org/ja/blog/posts/go-1-25-encoding-json-v1-vs-v2-z74a36b08

https://pkg.go.dev/encoding/json/v2

https://github.com/golang/go/discussions/63397