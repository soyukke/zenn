---
title: "Apex言語の爆速LSP＆静的解析ツールをZigで作った"
emoji: "⚡"
type: "tech"
topics: ["apex", "salesforce", "zig", "lsp", "静的解析"]
published: true
---

Salesforce Apex向けのLanguage Server（LSP）、静的解析、テスト実行を1つのバイナリに統合したツール「apexgov」をZigで作りました。エディタ統合（VS Code / Neovim）だけでなく、CLIとしてCI/CDパイプラインにも組み込めます。

https://github.com/soyukke/apexgov

https://marketplace.visualstudio.com/items?itemName=soyukke.apexgov

Neovimにも対応しています。

まだ不十分なところは多々あります。

## Apexとは

ApexはSalesforceプラットフォーム上で動くJava風のプログラミング言語です。Salesforceの業務ロジックやトリガー処理を書くために使います。マルチテナント環境で動くため、「ガバナ制限」というリソース制約（SOQLクエリ100回まで、DML操作150回まで等）が厳しく、開発者はこれを常に意識する必要があります。

## なぜ作ったか

公式のApex Language ServerはJava（Jorje）で動きます。

**会社PCのメモリが貧弱だときつい。** プロジェクトが大きくなるほど顕著で、数百ファイルのプロジェクトだとLSPの起動だけで数GBのメモリを持っていかれます。他のアプリと併用するとまともに動かない。

もう一つの不満は、ガバナ制限の静的検知が欲しかったこと。ヘルパーメソッド経由の間接的なループ内SOQL/DMLの検出、ループ回数の推定、CPU消費の見積もりなど、もっと深い解析が欲しかった。

そうだ、Zigで作ろう！

## デモ

### LSP

コード補完、定義ジャンプ、ホバー、リネーム、参照検索などに対応しています。

### 静的解析

Salesforce公式サンプルの [dreamhouse-lwc](https://github.com/trailheadapps/dreamhouse-lwc) に対して実行してみます。

```bash
apexgov check force-app --format text
```

```
GeocodingService.cls:38
  AG010 [warning] Callout executed inside loop
  - Loop upper bound is dynamic/unknown.
  - Callout in loop cannot be proven safe (estimated 1 per iteration).

GeocodingService.cls:40
  AG004 [warning] JSON processing inside loop
  - Serialize/deserialize outside loops where possible.
```

該当コードはこれです：

```apex
for (GeocodingAddress address : addresses) {
    // ... URL構築 ...
    HttpResponse response = http.send(request);  // ← AG010: Callout in loop
    List<Coordinates> deserializedCoords =
        (List<Coordinates>) JSON.deserialize(     // ← AG004: JSON in loop
            response.getBody(), List<Coordinates>.class);
}
```

`addresses`のサイズが不明なので、Callout制限（100回）を超える可能性があります。公式サンプルにもこうしたリスクが潜んでいます。

ヘルパーメソッドの呼び出しチェーンをファイルを跨いで追跡するので、間接的にループ内で呼ばれるSOQL/DMLも検知します。

## ベンチマーク

### 比較対象

- **公式 Apex Language Server** (Java/Jorje) — VS Code Salesforce Extension Pack同梱
- **apexgov** (Zig) — シングルバイナリ、外部依存ゼロ

計測環境: macOS, Apple Silicon, テストリポジトリは [NPSP](https://github.com/SalesforceFoundation/NPSP) (1,044 .clsファイル) を使用。LSPはstdioでJSON-RPCリクエストを送信し、`ps`でRSSを計測。

### バイナリサイズ・起動時間

![バイナリサイズと起動時間の比較](/images/apexgov-lsp-size-startup.png)

| | apexgov | 公式 Apex LSP |
|---|---|---|
| バイナリサイズ | **7.0 MB** (シングルバイナリ) | 24 MB + JVM |
| 起動時間 | **4.6s** | 9.1s |

### LSP起動 + ファイルopen時のメモリ

![LSPメモリ使用量の比較](/images/apexgov-lsp-memory.png)

| リポジトリ | ファイル数 | apexgov | 公式LSP |
|---|---|---|---|
| dreamhouse-lwc | 9 | **3.5 MB** | 288 MB |
| apex-recipes | 139 | **3.9 MB** | 264 MB |
| NebulaLogger | 226 | **3.9 MB** | 289 MB |
| NPSP | 1,044 | **4.3 MB** | 270 MB |

公式LSPはJVMの起動コストが支配的で、ファイル数に関係なく約270MB前後。apexgovは3〜4MBで完了します。**約70倍の差**です。

### メモリ推移（全ファイルopen後に放置）

NPSPの全1,044ファイルをdidOpenで開き、60秒間メモリ推移を記録しました。

![LSPメモリ推移](/images/apexgov-lsp-memory-soak.png)

| 時点 | apexgov | 公式LSP (Jorje) |
|---|---|---|
| 起動直後 | 2 MB | 4 MB |
| 全ファイルopen後 | **183 MB** | **448 MB** |
| 60秒後 | **183 MB** (変化なし) | **448 MB** (微増) |

### 実際の使用時のメモリについて

上記の計測はLSPにJSON-RPCで直接リクエストを送る方式のため、VS Codeで実際に使用する場合とは条件が異なります。公式LSPはワークスペース全体のインデックス構築、sObject定義の解決、バックグラウンドコンパイルなどを行うため、実プロジェクトではさらにメモリを消費します。

実際、公式リポジトリのissueでも以下の報告があります：

- [#6153](https://github.com/forcedotcom/salesforcedx-vscode/issues/6153): Apexクラス1,300個のプロジェクトで**Apex Language Serverだけで2〜3GB**消費。4プロジェクト開くと合計10GB超。workaroundとして外部メモリ圧縮ツールの使用で対応（根本修正なし）
- [#2121](https://github.com/forcedotcom/salesforcedx-vscode/issues/2121): 3,000行以上のApexクラスでLSPがハング。「メモリ割り当てを増やせ」という回答のみ（根本修正なし）
- [#2410](https://github.com/forcedotcom/salesforcedx-vscode/issues/2410): SObject定義の更新時にCPU 100%。同様に「`salesforcedx-vscode-apex.java.memory`設定を増やせ」で対応

メモリの少ない会社PCでブラウザ・Slack・VS Codeを併用していると、公式LSPのメモリ消費は実際に厳しいです。apexgovはこの問題を根本的に解消します。

## ガバナ制限の静的検知

### ガバナ制限とは

Salesforceはマルチテナント環境なので、1つのトランザクションで使えるリソースに上限があります。代表的なものだと：

- SOQLクエリ: 同期100回 / 非同期200回
- DML操作: 同期150回 / 非同期300回
- CPU時間: 同期10,000ms / 非同期60,000ms

これを超えると即座に `System.LimitException` で処理が止まります。開発環境では少量データで動いていたのに、本番で大量データが流れた途端に落ちる。Apex開発者なら一度は経験するやつです。

### 検出ルール

| ID | 検出対象 |
|---|---|
| AG001 | ネストされたループ |
| AG002 | ループ内 SOQL |
| AG003 | ループ内 DML |
| AG004 | ループ内 JSON シリアライズ/デシリアライズ |
| AG005 | ループ内 clone/deepClone |
| AG006 | ループ内コレクション確保 |
| AG007 | ループ内文字列連結 |
| AG008 | ループ内 SOSL |
| AG009 | ヒューリスティック CPU 見積もり |
| AG010 | ループ内 HTTP callout |
| AG011 | ループ内 Messaging.sendEmail |

### Call Graph解析

単純にループ内を見るだけではなく、メソッドの呼び出しチェーンを辿ります。

```apex
// AccountService.cls
public void processAccounts(List<Account> accounts) {
    for (Account acc : accounts) {
        updateRelated(acc);  // ← ここでAG003を検出
    }
}

// AccountHelper.cls
public void updateRelated(Account acc) {
    update acc.Contacts;  // DML操作
}
```

ファイルをまたいだ呼び出しも追跡し、`implements`/`extends` による動的ディスパッチも解決します。

### CI/CDへの組み込み

SARIF形式で出力できるので、GitHub ActionsなどのCIに組み込めます。

```bash
apexgov check force-app --format sarif --out reports/apexgov.sarif
```

### 静的解析の実行速度

`apexgov check`はZigネイティブで動作し、JVMを起動しません。

| リポジトリ | ファイル数 | 実行時間 |
|---|---|---|
| dreamhouse-lwc | 9 | **0.01s** |
| apex-recipes | 139 | **0.04s** |
| NebulaLogger | 226 | **0.14s** |
| NPSP | 1,044 | **0.69s** |

1,044ファイルのプロジェクトでも1秒以内で完了します。CIに組み込んでも待ち時間はほぼありません。

## テスト実行（開発中）

Apexでは本番環境へのデプロイに[75%以上のコードカバレッジ](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_code_coverage_intro.htm)が必要です。プロジェクトが大きくなるとテスト実行に時間がかかり、`sf apex run test`で数十分待たされることも珍しくありません（[#2475](https://github.com/forcedotcom/cli/issues/2475): 2,000テストで50分、[#5589](https://github.com/forcedotcom/salesforcedx-vscode/issues/5589): 2,300+テストでheap out of memory）。

apexgovではApexテストをZigネイティブのインタプリタでローカル実行できます。Salesforce環境へのデプロイもJVMも不要です。

```bash
apexgov interpret test force-app/main/default/classes
```

[salesforce-test-factory](https://github.com/dhoechst/Salesforce-Test-Factory)で実行した例：

```
interpret: loaded 3 Apex source file(s)
interpret: registered 15 class(es), 0 trigger(s), 0 parse error(s)
[PASS] TestFactoryTest#when_objectIsCreated_expect_defaultFieldsArePopulated
[PASS] TestFactoryTest#when_objectIsInserted_expect_defaultFieldsArePopulated
[PASS] TestFactoryTest#when_objectIsCreatedWithSpecificDefaultsSet_expect_defaultFieldsArePopulated
[PASS] TestFactoryTest#when_objectIsInsertedWithSpecificDefaultsSet_expect_defaultFieldsArePopulated
[PASS] TestFactoryTest#when_ListOfObjectsIsCreated_expect_defaultFieldsArePopulated
[PASS] TestFactoryTest#when_ListOfObjectIsInserted_expect_defaultFieldsArePopulated
[PASS] TestFactoryTest#when_ListOfObjectsIsCreatedWithSpecificDefaultsSet_expect_defaultFieldsArePopulated
[PASS] TestFactoryTest#when_ListOfObjectsIsInsertedWithSpecificDefaultsSet_expect_defaultFieldsArePopulated

--- Results: 8 total, 8 passed, 0 failed ---
```

LSPのCodeLensから`@IsTest`メソッドを直接実行することもできます。

### sf cliとの比較

| | sf apex run test | apexgov interpret test |
|---|---|---|
| 実行時間 | **15.2秒** | **0.015秒** |
| 結果 | 8/8 Pass | 8/8 Pass |

![sf cli vs apexgov テスト実行比較](/images/apexgov-test-bench.gif)
*左: sf apex run test（Salesforceサーバー上で実行） / 右: apexgov interpret test（ローカル実行）*

sf cliはSalesforce環境にデプロイしたコードをサーバー上で実行するため、ネットワーク往復とサーバー側の処理時間がかかります。apexgovはローカルで完結するため、テスト数が増えるほど差が大きくなります。

テスト結果はPASS / FAIL / ERRORの3パターンで表示されます：

```
[PASS] CalculatorTest#testAdd
[FAIL] CalculatorTest#testAddFail: 2 + 3 should not be 10 | Expected: 10, Actual: 5
[ERROR] CalculatorTest#testDivideByZero: ApexException: Cannot divide by zero
```

FAILはアサーション失敗時にExpected/Actualを表示、ERRORは未捕捉の例外をメッセージ付きで表示します。Salesforceのセキュリティモデル（`PermissionSet`、`ObjectPermissions`、フィールドレベルセキュリティ）もインタプリタ上でエミュレートしています。

まだ開発中で対応していないテストパターンもありますが、一部のリポジトリではフルパスしています。

## 開発の話

### 最初はApex→Javaトランスパイラだった

元々はApexのテストコードをJavaにトランスパイルして、ローカルでテスト実行するツールとして作り始めました。Salesforce環境にデプロイしなくてもテストを回せるようにしたかった。

ただ、Javaへのトランスパイル＆実行に時間がかかる。JVMの起動、トランスパイルの複雑さ、Apex標準ライブラリのエミュレーション...どんどん重くなっていく。

結局、**Zigで完結する方向に転換**しました。Lexer、Parser、ASTを切り出して、そこからLanguage Serverを作り始めた。Zigならシングルバイナリで配布できるし、起動も速い。

### 実リポジトリでテスト

GitHubで公開されているApexリポジトリをテストケースとして使い、改善を重ねました。

- [dreamhouse-lwc](https://github.com/trailheadapps/dreamhouse-lwc) (9ファイル)
- [apex-recipes](https://github.com/trailheadapps/apex-recipes) (139ファイル)
- [NebulaLogger](https://github.com/jongpie/NebulaLogger) (226ファイル)
- [NPSP](https://github.com/SalesforceFoundation/NPSP) (1,044ファイル)
- [fflib-apex-mocks](https://github.com/apex-enterprise-patterns/fflib-apex-mocks) (38ファイル)
- [fflib-apex-common](https://github.com/apex-enterprise-patterns/fflib-apex-common) (32ファイル)

実際のプロジェクトで誤検知ゼロを目指してチューニングしています。
