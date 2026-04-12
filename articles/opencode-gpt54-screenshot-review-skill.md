---
title: "GitHub Copilot CLIの画像レビューをスキル化したらUI改善が捗った"
emoji: "👁"
type: "tech"
topics: ["githubcopilot", "githubcopilotcli", "claudecode", "gamedev"]
published: true
---

## はじめに

GitHub Copilot CLI で画像を送れるの知ってますか？

```bash
copilot -p "@screenshot.png このUIをレビューして" --allow-all --model gpt-5.4 -s
```

`@` にファイルパスを付けるだけで、GPT-5.4 にスクリーンショットを送って画像レビューしてもらえます。これが思った以上に実用的だったので、**AIコーディングエージェントから呼べるスキルにして、UI改善のワークフローを回せるようになった話**を書きます。

https://github.com/soyukke/screenshot-review

## なぜ GPT-5.4 の画像レビューなのか

GPT-5.4 は画像理解に強いモデルです。マルチモーダルベンチマーク MMMU-Pro で 81.2%（Claude Opus 4.6 は 73.9%、[出典](https://portkey.ai/blog/gpt-5-4-vs-claude-opus-4-6/)）、UI操作ベンチマーク OSWorld で 75.0% と人間超え（[出典](https://openai.com/index/introducing-gpt-5-4/)）。

体感の差も大きいです。同じゲーム画面を Claude Opus 4.6 と GPT-5.4 にレビューさせた比較がこちら。

![レビュー対象の画面](/images/reward-before.png)

**Claude Opus 4.6 の指摘：**
> - 4枚のカードが左寄せで、右側に大きな空白がある
> - カード背景のレアリティ色が強い
> - Meta Points +42 がカード群から離れていて孤立感がある

**GPT-5.4 の指摘：**
> - 右側の大きな空白が強く目立ち、レイアウトバランスが崩れている。カード数が固定でないなら中央寄せ、固定なら**カード幅や全体コンテナ幅を詰めて「埋まって見える」構成**にすべき
> - カード下部の効果説明はかなり小さく、特に**白〜灰文字が背景色にやや沈む。効果文は1段階大きくするか、行間を広げる**と読みやすくなる
> - レアリティ色は分かりやすいが、**カード背景色まで強く色づけされているため、名前・本文・レアリティ表示が競合**している。背景は抑えめ、レアリティ名や枠・アイコン発光で差を出す方が整理される

GPT-5.4 は**改善の具体的な方向性まで提示**しています。「色が強い」ではなく「名前・本文・レアリティ表示が競合している」、「テキストが小さい」ではなく「白〜灰文字が背景色に沈む、行間を広げるべき」。この差が、実際の改善作業の精度に直結します。

この画像レビュー力を、AIコーディングエージェントから自動で呼べるようにスキル化しました。

```
AIコーディングエージェント (コード生成・修正)
    ↓ スクリーンショット撮影
    ↓ GitHub Copilot CLI
GPT-5.4 (画像認識・画像レビュー)
    ↓ フィードバック
AIコーディングエージェント (修正を反映)
```

## GitHub Copilot CLI の画像送信の詳細

冒頭で紹介した `@` 構文の詳細です（v1.0.24+）。

- `@` + ファイルパスで画像を添付
- `--model gpt-5.4` でモデルを指定
- `-s` で統計情報を省略（スキルから呼ぶときに便利）
- `--allow-all` で権限を許可

複数画像の Before/After 比較もできます。

```bash
copilot -p "@before.png @after.png 2つを比較してレビューして" --allow-all --model gpt-5.4 -s
```

## 作ったスキル

私は Claude Code のスキル（`.claude/skills/`）として使っていますが、他のエージェントでも同様に使えます。

### screenshot-review スキル：GPT-5.4 によるUIレビュー

AIエージェントが画像を確認したいとき、シェルから `copilot` コマンドを呼んで GPT-5.4 にレビューを委託します。スキルの中核部分はこれだけです。

```markdown
### Step 1: スクリーンショットの確認

レビュー対象の画像ファイルを確認する。

### Step 2: プロンプトの組み立て

ユーザーの依頼内容に応じてレビュープロンプトを組み立てる。
デフォルト観点: レイアウトバランス、色コントラスト、可読性、視覚的階層、改善優先度

### Step 3: Copilot CLI 実行

copilot -p "@<画像パス> <プロンプト>" --allow-all --model gpt-5.4 -s

### Step 4: 結果の報告

指摘事項を要約し、「改善に取り組みますか？」と提案する
```

スキルの全文は [GitHub リポジトリ](https://github.com/soyukke/screenshot-review) を参照してください。

### スクリーンショットの撮影はプロジェクト次第

レビュー対象のスクリーンショットをどう撮るかはプロジェクトによって異なります。

| プロジェクト | スクリーンショット取得方法 |
|---|---|
| Webアプリ | Playwright / Puppeteer |
| モバイルアプリ | iOS Simulator / adb |
| ゲーム（Bevy等） | ヘッドレス撮影バイナリ |
| CLI ツール | `script` コマンド |

私のプロジェクト（Bevy製ゲーム）では、専用の screenshot スキルで撮影を自動化しています。「スクショ撮って、GPTにレビューしてもらって」の一言で撮影からレビューまで回せます。

## 使い方

### セットアップ

```bash
# GitHub Copilot CLI をインストール
brew install --cask copilot-cli

# 認証
copilot login
```

### スキルの導入（Claude Code の例）

Claude Code の場合、`.claude/skills/` にスキルファイルを配置します。

```bash
mkdir -p .claude/skills/screenshot-review
curl -o .claude/skills/screenshot-review/SKILL.md \
  https://raw.githubusercontent.com/soyukke/screenshot-review/main/SKILL.md
```

あとは Claude Code に「UIレビューして」「GPTに見せて」と言うだけです。他のエージェントでも、同様にスキル/エージェント指示として配置すれば使えます。

## 実例：報酬画面を GPT-5.4 レビューで改善する

実際にこのワークフローで改善した例を紹介します。対象は自作のインディーゲーム **Capsulitas**（ローグライク × カプセルパズル、Rust + Bevy）の報酬画面です。

https://store.steampowered.com/app/4222380/Capsulitas/

GPT-5.4 のレビュー、Claude Code による実装、そして人間の微調整を繰り返したリアルな記録です。

### Step 0: Before

![改善前の報酬画面](/images/reward-before.png)

この画面を GPT-5.4 に送ります。

```bash
copilot -p "@./screenshots/reward-before.png \
You are a game UI reviewer for a roguelike capsule puzzle game with dark theme. \
Review the attached screenshot of the Reward screen. \
Give feedback in Japanese on: layout balance, color contrast, text readability, \
visual hierarchy, and usability. Prioritize improvements as high/medium/low." \
  --allow-all --model gpt-5.4 -s
```

GPT-5.4 の主な指摘：

> - **高**: カード背景のレアリティ色が強すぎて、名前・本文・レアリティ表示が競合している。背景は抑えめにして、枠やアイコン発光で差を出すべき
> - **高**: `Meta Points +42` が孤立している。結果サマリー枠にまとめると理解しやすい
> - **中**: `Retry` と `Back` のボタンが報酬エリアから離れすぎている。距離を詰めると操作の流れが良くなる

具体的で、すぐにコードに落とし込める指摘です。Claude Code にそのまま渡して実装してもらいます。

### Step 1: カード背景の統一 + レアリティ枠線

GPT-5.4 の「背景色を抑えて枠線でレアリティを表現すべき」という指摘を反映。

![Step1: カード背景統一](/images/reward-step1-card-border.png)

カード背景をダークに統一し、レアリティはボーダー色（青=Rare、紫=Epic、金=Legendary）で表現するように変更しました。

### Step 2: サマリーボックスの追加

GPT-5.4 の「情報が散っている」という指摘を受けて、`Reached Level` と `Meta Points` を上部のサマリーボックスに統合。

![Step2: サマリーボックス](/images/reward-step2-summary-box.png)

### Step 3: ボタン間隔の調整（GPT vs 人間）

GPT-5.4 は「ボタンが離れすぎ、詰めるべき」と指摘。Claude Code が間隔を詰めた結果：

![Step3: ボタン詰めすぎ](/images/reward-step3-button-close.png)

**ここで人間が介入。** 実際に見ると詰めすぎて窮屈に感じる。GPT-5.4 に before/after の2枚を送って比較してもらいつつ、人間の感覚で中間値に調整しました。

![Step3 修正: ボタン間隔調整](/images/reward-step3-button-revised.png)

> **人間のレビューメモ**: AIレビューの指摘は方向性として正しいが、数値の落とし所は人間の感覚で微調整が必要。「詰める」と言われてゼロにするのではなく、元の値との中間あたりが丁度良い。

GPT-5.4 に修正後の2枚を送って確認してもらうと「2枚目の余白調整の方が自然。採用するなら2枚目」とのこと。

### Step 4: 背景にシェーダーを追加

ここも人間判断。GPT-5.4 は指摘していなかったけど、「単色背景が味気ない」と感じたのでプラズマシェーダーの背景を追加しました。

![Step4: シェーダー背景](/images/reward-step4-shader.png)

GPT-5.4 に最終比較をさせると：

> 背景演出は画面の格を上げています。暗めのオーバーレイも効いていて、UI自体はちゃんと読めます。致命的な blocker はありません。出荷してよい出来です。

### 最終結果

![最終版](/images/reward-final.png)

さらに人間レビューで余白の微調整（パディング統一、カードマージン追加）を入れて完成。

### Before / After

| Before | After |
|---|---|
| ![Before](/images/reward-before.png) | ![After](/images/reward-final.png) |

### この改善で見えたこと

**GPT-5.4 が得意なこと：**
- レイアウトの問題点の特定（「右側の空白」「情報の散在」）
- 改善の方向性の提示（「枠線で表現すべき」「サマリーにまとめるべき」）
- Before/After の比較評価

**人間が必要なこと：**
- 数値の落とし所（「詰める」→ どれだけ詰めるか）
- 感覚的な判断（「味気ない」→ シェーダー追加）
- 最終的な品質基準の決定

GPT-5.4 は**方向性を示すコンパス**、Claude Code は**実装の手**、人間は**最終判断者**。この三者の分業が、このワークフローの本質です。

## おわりに

AIコーディングエージェントは万能じゃない。画像理解が弱いなら、得意なモデルに任せればいい。

GitHub Copilot CLI の `@` 構文で画像を渡せるので、スキルから `copilot` コマンドを呼ぶだけでマルチモーダルな画像レビューが手に入ります。

「コード生成はいつものエージェント、画像レビューは GitHub Copilot の GPT-5.4」—— この分業体制は、UIを持つプロジェクトならどこでも使えるパターンだと思います。

スキルはこちらで公開しています。

https://github.com/soyukke/screenshot-review
