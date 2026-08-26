---
title: "BunのZig→Rustリライトに触発され、プログラミング言語kizuを作り始めた ── 明示的な制御・メモリ安全・高速コンパイルを目指して"
emoji: "🩹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["自作言語", "コンパイラ", "rust", "zig", "kizu"]
published: true
---


## はじめに

BunのZig→Rustリライトに触発されて、プログラミング言語kizuの開発を始めました。
現在はkizu自身で書いたセルフホストコンパイラを実装し、GitHubでソースコードとリリースを公開しています。

https://github.com/kizu-lang/kizu

## なぜkizuを作り始めたか

AIを使って色々と開発する中で以下を感じていました。
- Rustは好きですが、私の開発環境ではCIに時間がかかり、ビルド成果物でローカルのストレージが圧迫されることを負担に感じていました。
- Zig言語も思想が好きだが、メモリ安全性はコンパイラでチェックしたい。
- bunのリライト事件でメモリ安全はやっぱり欲しいと思った

上記のように、私が欲しい言語像が明確になってきたので作り始めました
開発する上で、以下のことを主眼においています。

- メモリ安全。
    - 理由: 仕組みでカバーできるものは仕組みで対策したい
- 隠れた制御を嫌う。明示的である。
    - 理由: 人間も目で追ってレビューしやすく、LLMも読んだまま動作を解釈できるものにしたい
- 高速CIを回せるコンパイル
    - 理由: CIを開発のボトルネックにしたくない
- ストレージを圧迫しない
    - 理由: 並列開発、大量開発に耐えたい
- なるべく依存なしで実装できるように標準ライブラリを厚くしたい

この考え方は、Google Developers Blogの[GoがAI支援開発に適している理由](https://developers.googleblog.com/why-go-is-an-ideal-language-for-ai-assisted-software-engineering/)が説く、AI時代にはコードを書く速さよりレビュー・検証・保守のしやすさが重要になるという主張にも通じます。

## kizuの特徴

今の段階のkizuが持っている特徴をいくつか紹介します

### 所有権

kizuでは、メモリ安全を実現する仕組みの一つとして、所有権を採用しています。

```kizu
import std::mem;
import std::string;

fn consume(text: string::String) {
    let bytes = text.as_bytes();
    print(bytes);
    text.deinit();
}

fn main() -> !void {
    let allocator = mem::page_allocator();
    let text = try string::from_bytes(allocator, "owned by main");
    consume(move text);

    // move 後の text を使うとコンパイルエラーになる。
    // let bytes = text.as_bytes();
    // print(bytes);
    return;
}
```

### deferによる明示的な後処理

Rustのように、スコープを抜けたときに自動で`Drop`が呼ばれる仕組みはありません。スコープ終了時に処理を実行したい場合は、Zigのように`defer`で明示します。
ただし、`defer`自体が必須なのではありません。所有する値を直接`deinit`することも、`move`を明示して別の関数や戻り値へ所有権を移すこともできます。所有する値を未処理のままスコープを抜けると、コンパイルエラーになります。

```kizu
struct Resource {
    name: []u8,
}

fn (self: Resource) deinit() {
    print(self.name);
}

fn use_resources() {
    let first = Resource { name: "first" };
    defer first.deinit();

    let second = Resource { name: "second" };
    defer second.deinit();
}

fn main() {
    use_resources();
}

// 出力:
// second
// first
```

### errdeferによる失敗時の後処理とmoveによる退役

メモリなどのリソースを確保した後、後続の処理がerrorになった場合に備えて、errdeferでcleanupを登録します。
errdeferは、現在のスコープがその値を所有したままerrorで抜けるときだけ実行されます。
一方、成功時にその値をreturnしたり、別の関数へ渡したりすると、所有権と解放責任は移動先へ移ります。
kizuでは、その所有権の移動をmoveで明示します。moveした時点で対応するerrdeferは退役します。
ここでいう退役とは、以降のerror経路では実行されなくなることです。

```kizu
import std::mem;
import std::string;

fn make_message(allocator: Allocator) -> !string::String {
    var message = string::new(allocator);
    errdefer message.deinit();

    try message.append_bytes("hello, kizu");
    return move message;
}

fn main() -> !void {
    let allocator = mem::page_allocator();
    let message = try make_message(allocator);
    defer message.deinit();

    let bytes = message.as_bytes();
    print(bytes);
    return;
}
```

(これはわかりにくいかも)


### 借用と長寿命のHandle

kizuには、所有権を移動せずに値を一時的に借りる借用（borrow）があります。
借用するときは、読み取り専用の参照`&T`と、値を変更できる可変参照`&var T`を使います。

現在のkizuは、明示的なライフタイムパラメーターを採用していません。
関数から参照を返す場合は、戻り値を引数に結び付くものとしてコンパイラが判断します。

```kizu
// 戻り値の参照は引数valueに結び付く。
// ライフタイムパラメーターは書かない。
fn borrow_value(value: &i64) -> &i64 {
    return value;
}

fn main() {
    let value = 42;
    let borrowed = borrow_value(value);

    // .*で参照先の値を読む。
    print(borrowed.*);
}
```

ライフタイムを明示せずに書ける一方で、借用関係が暗黙的になるため、よりよい表現方法についてはまだ設計を詰めきれていません。

グラフの辺のように長期間保持したい関係は参照として保存せず、Arenaが値をまとめて所有し、その中の値を示すHandleで表現します。
Handleは所有権を持たないコピー可能なIDで、対応するArenaより長生きできません。
値へアクセスするときだけ、`arena.at(handle)`からArenaに結び付いた短い参照`&T`を取得します。

```kizu
import std::arena;
import std::mem;

struct Node {
    value: i64,
    // HandleはArena内のNodeを示す、コピー可能なID。
    next: ?arena::Handle<Node>,
}

fn main() {
    let allocator = mem::page_allocator();

    // Nodeを所有するのはArenaだけ。HandleはNodeを所有しない。
    let nodes = arena::new<Node>(allocator);

    // スコープを抜けるとき、Arenaの全要素と保存領域をまとめて解放する。
    defer nodes.deinit();

    // addはNodeをArenaへ移動し、そのNodeを示すHandleを返す。
    let tail = nodes.add(Node { value: 2, next: null });
    let head = nodes.add(Node { value: 1, next: tail });

    // atはHandleを使ってNodeへアクセスし、Arenaに結び付いた短い参照を返す。
    let node = nodes.at(head);
    print(node.value);

    // Node間の長期間保持する関係はHandleで保存し、
    // 実際に値を読むときだけatで短い参照を取得する。
    if node.next |next| {
        print(nodes.at(next).value);
    }
}
```

## 今できていること

### セルフホストコンパイラ

kizuのコンパイラはkizu自身で実装しており、v0.2.1から配布バイナリにもこの実装を使用しています。
配布されたコンパイラで、同じコンパイラのソースを再びビルドできることも確認しています。
Go版は最初のビルドと、両実装の出力を比較するために残しています。

### jsonライブラリ

jsonライブラリの`encode<T>`と`decode<T>`は、comptimeで型`T`の構造を調べ、JSONとkizuの値を相互変換します。呼び出す側では型引数を明示します。

jsonライブラリでは、次のようにcomptimeで型`T`の公開フィールドを列挙しています。

```kizu
// 実際の実装から、optionalの処理を省いて要点だけを抜粋
comptime for std::meta::public_fields<T>() |field| {
    try encode_field<std::meta::field_type<T, field>>(
        encoder,
        std::meta::field_name<T, field>(),
        std::meta::field<T, field>(value),
    );
}
```

`public_fields<T>()`でフィールドを列挙し、`field_type`・`field_name`・`field`から型、名前、値を取得しています。
([v0.2.1の実装](https://github.com/kizu-lang/kizu/blob/v0.2.1/lib/kizu/std/src/json/json.kizu#L313-L339))

```kizu
import std::json;
import std::mem;
import std::string;

pub struct User {
    pub name: string::String,
    pub age: i64,
    pub active: bool,
}

fn main() -> !void {
    let allocator = mem::page_allocator();
    let document =
        \\{"age": 20, "active": true, "name": "alice"}
    ;

    let user = try json::decode<User>(allocator, document);
    defer user.deinit();

    var out = string::new(allocator);
    defer out.deinit();
    try json::encode<User>(allocator, &user, &var out);

    let bytes = out.as_bytes();
    print(bytes);
    return;
}

// 出力:
// {"name":"alice","age":20,"active":true}
```

## AIを使った開発の反省

当初はセルフホストを急ぐあまり、コンパイラ開発の知識が十分にないままAIに実装を指示し続けました。生成されたコードが正しいのか、設計として妥当なのかを私自身が判断できないまま実装を重ねた結果、旧セルフホスト実装は16万行を超えました。最終的には、数ヶ月かけた実装をいったん削除し、作り直すことになりました。

## 課題

### セルフホストコンパイラのパフォーマンス

Goで書かれたコンパイラのほうがセルフホストコンパイラより速度が速く、メモリ消費も少ないです。
調査と改善を進めています。

### コード量が多い

v0.2.1時点で、テストコードを除いた単純な行数では、kizuで記述したコンパイラはGo版の約1.6倍です。
std apiの改善でどうにかできるのか、この長さを許容できるうまみを主張できるか。


## おわり

以上、欲望駆動開発をしているkizu言語の紹介でした。
AIのおかげでプログラミング言語作成の障壁は大きく下がりました。
自分の欲しいものや理想を追い求められるいい時代ですね！！！
よかったら、kizuを触って遊んでみてください！
みんなも「僕の考えた最強の言語」を作ろう！
