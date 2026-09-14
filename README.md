# GPT-6 Astra のトークン消費を減らす裏技 — 裏で動くサブエージェントを止める＋公式の節約術

📊 [スライド資料はこちら（slides.html）](https://chaaaaarin.github.io/claudecode-channel-20260914/slides.html) | 📋 [1枚まとめ資料はこちら（onepager.html）](https://chaaaaarin.github.io/claudecode-channel-20260914/onepager.html) | 🎁 [プレゼントはこちら](https://chaaaaarin.github.io/gift-library/cc/ep/152/)

GPT-6 Astra（ジーピーティーシックス・アストラ）を Codex（コーデックス）で使い始めた人から、「小さな作業しか頼んでいないのに枠がすぐ減る」という声が出ています。X では、原因を調べたら裏でサブエージェント（親のAIが仕事を分けて渡す、子のAI）が動いていた、止めたら減りが落ち着いた、という個人の検証投稿が28万回以上表示されました（2026年9月13日時点）。本ノートは、この話を OpenAI の公式ドキュメントだけで裏取りし、**サブエージェントの止め方・軽くし方**と、公式がすすめる**節約術4つ**を、初心者向けにまとめたものです。

**確度マーク**: ✅（公式ドキュメント・公式ページで確認済み）／🔶（一次情報あり・解説メディア基準）／⚠️（未確認・推測）。料金・設定名・利用枠は変わるため、実際に使う前に公式で最新を確認してください。

**🎁 この動画にはプレゼントがあります。** ①【今スグ使える】トークン100％節約mdファイル（Codex が必ず読む AGENTS.md に置くだけで、頼んでいないサブエージェント・余計な読み込み・長い前置きや実況にトークンを使わせない約束ごとが毎回効く）②HIROGERU（ヒロゲル）1ヶ月無料クーポン券（AI学習アプリ「HIROGERU」を1ヶ月無料で始められる）。受け取り方は[キットの中身と受け取り方](#kit)、受け皿は[プレゼント図書館](https://chaaaaarin.github.io/gift-library/cc/ep/152/)。

## TL;DR（まず3行で）

1. **今の Codex はサブエージェントが既定でオン。** 子のAIは1体ずつ自分でモデルとツールを使うので、1体で進めるより多くトークンを使う。しかも子は親と同じモデル・同じ強さを引き継ぐ ✅
2. **並列が要らないなら止める（`[agents] enabled = false`）。要るなら子を GPT-5.6 Terra・強さ low・2体までにする。** ✅
3. あわせて、**Light（低い強さ）から始める／Ultra と Max は常用しない／Fast mode を切る（Astra は2.5倍）／読ませる資料・AGENTS.md・MCP を絞る。** 効いたかは `/status`・`/usage`・「設定 → 使用状況」で確かめる ✅

## 目次

1. [なぜ Astra の枠は減るのが速いのか](#s1)
2. [サブエージェントとは何か・いつ動くのか](#s2)
3. [裏技①：サブエージェントを止める](#s3)
4. [裏技②：止めずに「子を軽く・少なく」する](#s4)
5. [裏技③：Light から始める・Ultra と Max は常用しない](#s5)
6. [裏技④：Fast mode を切る](#s6)
7. [裏技⑤：読ませる量を減らす（公式の節約ヒント）](#s7)
8. [効いたかどうかを確かめる](#s8)
9. [API で使っている人は](#s9)
10. [メリット・デメリット総まとめ](#s10)
11. [注意点](#s11)
12. [今日からやること（この順番で）](#s12)
13. [キットの中身と受け取り方](#kit)
14. [まとめ](#s14)
15. [動画内で補足する用語](#s15)

---

<a id="s1"></a>

## 1. なぜ Astra の枠は減るのが速いのか

まず数字から。OpenAI のヘルプと Codex の料金ページには、ChatGPT Plus の5時間あたりの目安がモデル別に載っています。✅

| モデル | Plus・5時間あたりの目安 | 出力100万トークンあたりのクレジット |
|---|---|---|
| GPT-6 Astra | 5〜45通 | 1,250 |
| GPT-5.6 Sol（ソル） | 10〜100通 | 500 |
| GPT-5.6 Terra（テラ） | 25〜200通 | 300 |
| GPT-5.6 Luna（ルナ） | 250〜2,000通 | 30 |

- 目安は「固定の上限ではなく、タスク・モデル・設定で変わる」と明記されています ✅（[OpenAI ヘルプ](https://help.openai.com/en/articles/20001516-managing-usage-with-gpt-6-astra-in-work-and-codex)）
- Codex で使うクレジットの単価は、Astra が Sol の **2.5倍**（入力250／キャッシュ入力25／出力1,250）✅（[Codex 料金](https://learn.chatgpt.com/codex/pricing)）
- ヘルプは、入出力が大きいほど・強さが高いほど・Fast mode を使うほど・手数の多いタスクほど消費が増える、と説明しています ✅

一方で、Astra 自体が「トークンを無駄に食う」わけではありません。API のモデルガイドには「複数の評価で、より少ない出力トークンで強い結果を出し、1トークンの単価は高くても1タスクあたりの推定コストは従来より低い」と書かれています ✅（[Model guidance](https://developers.openai.com/api/docs/guides/latest-model)）。**減りが速い原因は、単価の高さに加えて「どう動かしているか」**——その代表がサブエージェントです。

<a id="s2"></a>

## 2. サブエージェントとは何か・いつ動くのか

**サブエージェント**は、親のAIが作業を分けて渡す子のAIです。たとえば「先週届いた問い合わせ50件を分類して、週次レポートにまとめる」と頼むと、親が「前半25件の分類」「後半25件の分類」「集計」を子に割り振り、結果だけを受け取ってまとめます。

公式の Subagents ページで確認できたこと：

| 公式に書かれていること | 意味 | 確度 |
|---|---|---|
| 今の Codex はサブエージェントの仕組みが**既定でオン** | 何も設定していなければ使える状態 | ✅ |
| 子は1体ずつ自分でモデルとツールを動かすので、**1体で進めるより多くトークンを使う** | 子の数だけ作業が増える（何倍になるかの数字は公式に無い） | ✅ |
| 子のモデル・強さを設定していなければ、**親のモデルと強さを引き継ぐ** | 親が Astra・Extra High なら、子も Astra・Extra High | ✅ |
| Codex は「**頼んだとき**」と「**AGENTS.md やスキルに指示があるとき**」に作業を子に任せる | 自分が頼んだ覚えがなくても、ルールファイルの一文で動くことがある | ✅ |

出典: [Subagents（Codex 公式）](https://learn.chatgpt.com/docs/agent-configuration/subagents)

さらに、Codex のモデルページには「**Ultra** はサブエージェントを使って、複雑な作業の各部分を並列に進める」「**ほとんどの作業に Max も Ultra も要らない**」とあります ✅（[Models](https://learn.chatgpt.com/codex/models)）。API のモデルガイドにも、Astra は作業を分けて子に任せられるよう訓練されている、と書かれています ✅。

**悪者ではない点も公式どおりに押さえておく**: サブエージェントは、探索のメモやテストのログをメインの会話から切り離して、会話が散らからないようにする仕組みです。大量の調査・テスト・要約のように「読む作業を分けられる」場面では、時間短縮にもなります ✅。**損をするのは、並列にする意味のない軽い作業で子が並ぶとき**です。

<a id="s3"></a>

## 3. 裏技①：サブエージェントを止める

並列で進める作業をほとんどしない人は、止めるのがいちばん確実です。ローカルの Codex（デスクトップアプリのローカル作業・CLI・IDE）の設定ファイル `~/.codex/config.toml` に、次の2つを書きます。

```toml
[features]
multi_agent = false

[agents]
enabled = false
```

- `agents.enabled` は既定で `true`。`false` にするとマルチエージェントの機能が止まる ✅（[Subagents](https://learn.chatgpt.com/docs/agent-configuration/subagents)）
- `features.multi_agent` はマルチエージェントの協調ツール（子を作る・指示する・待つ・閉じる）を有効にする設定で、既定でオン ✅（[Configuration Reference](https://learn.chatgpt.com/docs/config-file/config-reference)）
- ファイルを自分で開くのが不安なら、Codex に「バックアップを取ってから、この2行を足して。既存の設定は消さないで。この作業でもサブエージェントは使わないで」と頼めば済みます

**向かない場面・デメリット**:

- 大量のファイル調査やテストを並列で速く進めることはできなくなる。Ultra も子を使うモードなので、合わせて使わない前提になる
- 会社のアカウント（Business・Enterprise）では、管理者がマルチエージェントの可否を固定できるため、手元の設定が効かないことがある ✅（Configuration Reference の requirements 設定）
- 並列が要る日は、`true` に戻してから使う

<a id="s4"></a>

## 4. 裏技②：止めずに「子を軽く・少なく」する

ときどき並列が要る人は、止めずに**子のモデルと強さ、同時に動く数**を決めておきます。

```toml
[agents]
default_subagent_model = "gpt-5.6-terra"
default_subagent_reasoning_effort = "low"
max_concurrent_threads_per_session = 2
```

- 3つとも公式の設定項目 ✅。公式は「軽いサブエージェントの作業で、速く低コストにしたいときは `gpt-5.6-terra` を使う」と案内しています ✅（[Subagents](https://learn.chatgpt.com/docs/agent-configuration/subagents)）
- 単価の差（出力100万トークンあたりのクレジット）：Astra 1,250 → Terra 300。**子1体ぶんの出力単価は約4分の1**（公式の単価表から計算）✅
- 依頼文でモデルや強さを指定すると、依頼文の指定がこの既定より優先されます ✅

**向かない場面**: 子に難しい判断をさせる作業では、Terra・low だと足りないことがある。その作業だけ依頼文でモデルを指定する。

<a id="s5"></a>

## 5. 裏技③：Light から始める・Ultra と Max は常用しない

Codex のモデルページは「必要な結果が出る、いちばん低い強さを使う」と案内しています ✅。デスクトップアプリ・ChatGPT Work（ブラウザ）・IDE では **Light**、CLI では **Low** が、素早く範囲の決まった作業向けです ✅（[Models](https://learn.chatgpt.com/codex/models)）。

| 強さ・モード | 公式の使いどころ |
|---|---|
| Light（CLI は Low） | 素早く、範囲の決まった作業。**まずここから** |
| Medium | もう少し計画が要る作業 |
| High / Extra High | 手順・情報源・判断材料が多い難しい作業 |
| Max | いちばん難しい1件を深く考えさせたいとき |
| Ultra | サブエージェントで作業を分けて並列に進めたいとき |

ヘルプには「**低い強さ＝能力が低い、ではない。Astra の Low は Sol の High を上回ることがある**」「高い強さは枠を多く使い、常に良い結果になるとは限らない」とあります ✅（[OpenAI ヘルプ](https://help.openai.com/en/articles/20001516-managing-usage-with-gpt-6-astra-in-work-and-codex)）。

**向かない場面**: 原因の分からないバグや設計の相談は、Light だと足りないことがある。その1件だけ強さを上げて、終わったら戻す。強さを上げる前に、指示や渡した資料が足りているかを先に見直す（強さを上げても足りない情報は補えない、とヘルプにも明記）✅。

<a id="s6"></a>

## 6. 裏技④：Fast mode を切る

Fast mode は応答を速くする代わりに、クレジットを多く使います。**GPT-6 Astra の Fast mode は通常の2.5倍** ✅（[Speed](https://learn.chatgpt.com/docs/agent-configuration/speed)／[Codex 料金](https://learn.chatgpt.com/codex/pricing)）。

- CLI では `/fast off`・`/fast on`・`/fast status` で切り替え・確認できる ✅
- 速さが仕事の成果に直結する作業のときだけ付けて、終わったら戻す

**向かない場面**: 返事を待つ時間そのものがコストになる作業（その場で人に見せながら直す等）では、あえて付ける判断もある。

<a id="s7"></a>

## 7. 裏技⑤：読ませる量を減らす（公式の節約ヒント）

Codex の料金ページには「利用枠を長持ちさせるには？」という項目があり、次の6つが並んでいます ✅（[Codex 料金](https://learn.chatgpt.com/codex/pricing)）。

| 公式のヒント | かみ砕くと |
|---|---|
| Control the size of your prompts | 指示は正確に、でも要らない前置き・文脈は削る |
| Limit source material | 関連するファイルだけを渡す。期間や範囲も絞る |
| Match the output to the need | 誰向け・どの形・どの長さかを先に決める |
| Reduce the size of your AGENTS.md | 毎回読まれるルールファイルを短くする。大きいリポジトリはフォルダ別に分ける |
| Limit the number of MCP servers you use | MCP サーバーは1つ足すごとにメッセージの文脈が増える。使わないものは止める |
| Switch to a smaller model for routine tasks | 定型の作業は Terra や Luna に振る |

- MCP サーバーは、設定を消さずに `enabled = false` で止められる ✅（[Configuration Reference](https://learn.chatgpt.com/docs/config-file/config-reference)）
- AGENTS.md に「並列で進める」系の指示があると子が動く（2章）ので、短くするついでに確認する
- プレゼントの節約mdファイルは、この考え方（頼んだときだけ子を使う・読む範囲を絞る・前置きや実況を書かない）を、AGENTS.md に置ける20行ほどのルールにまとめたもの

<a id="s8"></a>

## 8. 効いたかどうかを確かめる

| 見る場所 | 分かること | 確度 |
|---|---|---|
| アプリの「設定 → 使用状況」 | 5時間の上限・週の上限の残りと、リセットの時刻 | ✅ |
| CLI の `/status` | 今の会話の設定と、トークンの使用量・残りのコンテキスト | ✅ |
| CLI の `/usage` | アカウントの1日・1週間のトークン使用量 | ✅ |
| CLI の `/subagents`（`/agent`） | 子のAIのスレッドを切り替えて中身を見る | ✅ |
| CLI の `/fast status` | Fast mode がオンかオフか | ✅ |

出典: [Slash commands（Codex CLI）](https://learn.chatgpt.com/docs/cli/slash-commands)／[OpenAI ヘルプ](https://help.openai.com/en/articles/20001516-managing-usage-with-gpt-6-astra-in-work-and-codex)

- ヘルプには「共有の利用枠は、**モデルを切り替えても戻らない**」と書かれています ✅。枠が尽きてから軽いモデルに変えても遅いので、作業を始める前にモデルと強さを決める
- 設定は一度に全部変えず、1つずつ変えて「同じくらいの作業」で減り方を比べる

<a id="s9"></a>

## 9. API で使っている人は

API（自分のキー）で Astra を呼んでいる人は、Codex の設定ファイルではなく、リクエストのパラメータで同じ考え方をします。

| やること | 中身 | 確度 |
|---|---|---|
| 強さを low 基準に | `reasoning.effort` を `low` から。Astra は `none` 非対応で、実質の下限は `low` | ✅ |
| 難しい1問だけ上げる | 会話の途中に `configuration_update` を挟むと、その応答から強さを変えられる（Astra 限定。プロンプトの先頭が変わらないのでキャッシュも効いたまま） | ✅ |
| 答えを短く | `text.verbosity` を `low` に | ✅ |
| キャッシュを効かせる | 変わらない指示・資料を先頭、毎回変わる部分を後ろに。キャッシュ済み入力は通常の0.1倍 | ✅ |
| 入力を 27万2,000 トークン未満に | 超えるとリクエスト全体が入力2倍・出力1.5倍で課金 | ✅ |
| 急がない処理は Batch | 標準の50%。24時間以内に完了 | ✅ |

出典: [Reasoning models](https://developers.openai.com/api/docs/guides/reasoning)／[GPT-6 Astra モデルページ](https://developers.openai.com/api/docs/models/gpt-6-astra)／[Prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching)／[Batch API](https://developers.openai.com/api/docs/guides/batch)

<a id="s10"></a>

## 10. メリット・デメリット総まとめ

| 裏技 | 効くこと | 向かない場面・デメリット |
|---|---|---|
| ① サブエージェントを止める | 軽い作業で子が並ばなくなる | 大量の調査を並列で速く進められない。Ultra も使わない前提 |
| ② 子を軽く・少なく | 並列の良さを残したまま、子の単価と数を抑える | 子に難しい判断をさせる作業では足りないことがある |
| ③ Light から・Ultra と Max は常用しない | 毎回の考える量を最小から始められる | 難しい1件は上げる必要がある。下げすぎのやり直しはかえって高い |
| ④ Fast mode を切る | Astra の2.5倍の消費を避ける | 返事が遅くなる |
| ⑤ 読ませる量を減らす | どの依頼にも上乗せされる読み込みが減る | 資料を絞りすぎると、推測で埋め始める |

<a id="s11"></a>

## 11. 注意点

- X の検証投稿は**個人の体験**です。「止めたら減りが明らかに少ない」は数字として公表されたものではないため、本ノートでは効果の倍率を書いていません。公式も、サブエージェントで何倍になるかの数字は出していません
- ChatGPT Work（ブラウザ）で `config.toml` の設定が効くかは、公式に記載がありません。Work では「Ultra を選ばない」「並列を頼まない」「Light から始める」で同じことをします ⚠️
- 会社のアカウント（Business・Enterprise）では、管理者の設定が優先されることがあります ✅
- Codex のクレジットの倍率（Fast 2.5倍など）は ChatGPT アカウントで使う場合の話です。API キーで Codex を使う場合は API の料金になります ✅
- 設定名・料金・利用枠は変わります。使う前に公式で最新を確認してください

<a id="s12"></a>

## 12. 今日からやること（この順番で）

1. モデルの選択から **Ultra と Max を外し、Light（CLI は Low）** にする
2. 並列が要らなければ **サブエージェントを止める**（`[agents] enabled = false`・`[features] multi_agent = false`）。要るなら **子を Terra・low・2体まで**にする
3. **Fast mode を切る**（`/fast off`）
4. **MCP サーバーと AGENTS.md を見直す**（使わないサーバーを止め、ルールファイルを短くし、「並列で」の指示を確認）
5. 次の同じくらいの作業で、**使用状況の減り方を比べる**（「設定 → 使用状況」・`/status`・`/usage`）

1〜3 は5分で終わります。4 は一度やれば済みます。

<a id="kit"></a>

## 13. キットの中身と受け取り方

配布物は本文に埋め込まず、[プレゼント図書館（ClaudeCodeチャンネル）](https://chaaaaarin.github.io/gift-library/cc/ep/152/)にまとめています。コピーでもダウンロードでも受け取れます。

| キット | 受け取ると何が変わるか |
|---|---|
| 🎁① 【今スグ使える】トークン100％節約mdファイル | Codex が作業の前に必ず読む AGENTS.md に置くだけで、頼んでいないサブエージェント・余計なファイルの読み込み・長い前置きや実況にトークンを使わせない約束ごとが、毎回の作業に効く。設定ファイルを触らず今日から使える。確実に止めたい人向けの設定2行つき |
| 🎁② HIROGERU（ヒロゲル）1ヶ月無料クーポン券 | AI学習アプリ「HIROGERU」を1ヶ月無料で始められる。App Store・Google Play のボタンからそのままダウンロードでき、クーポンコードはワンタップでコピーできる |

受け取り方：概要欄の公式 LINE から[プレゼント図書館](https://chaaaaarin.github.io/gift-library/cc/ep/152/)へ。

<a id="s14"></a>

## 14. まとめ

Astra は「1タスクの効率」は良いモデルです。それでも枠が速く減るのは、単価が Sol の2.5倍（Codex のクレジット）であることに加えて、**裏でサブエージェントが同じ Astra・同じ強さで動いたり、Ultra・Max・Fast mode・大きな資料で1回の依頼がふくらんだりする**からです。並列が要らなければサブエージェントを止め、要るなら子を軽く・少なくする。そのうえで Light から始め、Fast を切り、読ませる量を絞る。最後に使用状況で減り方を比べれば、どの設定が自分に効いたかが分かります。

<a id="s15"></a>

## 15. 動画内で補足する用語

| 用語 | 読み | 意味 |
|---|---|---|
| GPT-6 Astra | ジーピーティーシックス・アストラ | OpenAI のいちばん高性能なモデル（2026年9月公開）。ChatGPT の Work・Codex・API で使える |
| Codex | コーデックス | OpenAI のAIエージェント。デスクトップアプリ・CLI・IDE・ブラウザで使える |
| サブエージェント | — | 親のAIが作業を分けて渡す、子のAI |
| multi_agent | マルチエージェント | 子のAIを作って協調させる機能の設定名 |
| config.toml | コンフィグ・トムル | Codex の設定ファイル（`~/.codex/config.toml`） |
| AGENTS.md | エージェンツ・エムディー | Codex に毎回読ませるルールのファイル |
| MCP | エムシーピー | AIに外部のツールやデータをつなぐ仕組み |
| Light / Medium / Extra High | ライト／ミディアム／エクストラハイ | アプリで選ぶ考える強さ。CLI では low / medium / xhigh など |
| Max / Ultra | マックス／ウルトラ | Max は1件を深く考える、Ultra は子のAIで並列に進めるモード |
| Fast mode | ファストモード | 応答を速くする代わりに多くクレジットを使うモード |
| Sol / Terra / Luna | ソル／テラ／ルナ | GPT-5.6 のモデル。Sol が上位、Terra がふだん使い、Luna が定型作業向け |
| クレジット | — | Codex・Work の使用量を払う単位 |
| /status・/usage | スラッシュ・ステータス／スラッシュ・ユーセージ | CLI で使用量を確かめるコマンド |
| CLI | シーエルアイ | 黒い画面（ターミナル）で使う形 |
