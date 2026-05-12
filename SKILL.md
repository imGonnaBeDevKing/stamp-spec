---
name: somo-stamp-spec
description: LINE スタンプの設計書を AskUserQuestion で要件を徹底的に詰めながら量産するスキル。出力は ~/stamp-specs/<スタンプ名>/ ディレクトリ配下の3ファイル構成（character.md / spec.md / pose-library.md）で、character.md は画像生成AIに毎回コピペで先頭投入される「契約書」として扱う。色（HEX・面積比）・パーツ（形状名詞・相対位置・サイズ比）・比率（頭身・顔比率・占有率）・ポーズ（カメラ距離・体の向き°・視線°・表情ID・口眉腕）を数値+固有名詞で完全言語化し、曖昧語（cute/nice/soft/a bit）を NG ワードとして明示禁止する。プロンプトは GPT-4o image（GPT Image 1）／ Gemini Nano Banana ／ Stable Diffusion (NovelAI) の3系統を併記。生成後はフェーズ7の検証ループで差分を character.md に反映して進化させる。ユーザーが「LINEスタンプの設計書を作る」「somo-stamp-spec」「LINEスタンプの仕様を書きたい」「思い通りの画像が出ない」などと言ったときに起動する。
tools: AskUserQuestion, Write, Read, Edit, Bash
---

# somo-stamp-spec — LINE スタンプ設計書 量産システム（強化版）

このスキルは、LINE スタンプ（LINE Creators Market 審査対象）の設計書を、AskUserQuestion による対話ヒアリングだけで一つずつ確定させながら `~/stamp-specs/<スタンプ名>/` 配下の **3ファイル**として出力する。

最終成果物は、画像生成AI（**GPT-4o image / GPT Image 1**、**Gemini Nano Banana (2.5 Flash Image)**、**Stable Diffusion / NovelAI**）に**1回入力するだけで審査通過品質の画像が出力される**ことを目標とする。曖昧さを残さない。

---

## 出力ファイル構造（最重要）

```
~/stamp-specs/<スタンプ名>/
├── character.md      # キャラ定義のみ。AI への入力時に丸ごと先頭にコピペする「契約書」
├── spec.md           # スタンプ構成・審査チェック・画風統一・各枚プロンプト・申請手順
└── pose-library.md   # 全N枚分のポーズ・表情・カメラを数値+名詞で固定した一覧
```

**運用原則**：各スタンプの個別プロンプトは「`character.md` の全文 + `pose-library.md` の該当行 + 規格・ネガティブ」の3層を AI に渡す前提で書く。`character.md` を独立ファイルにすることで、AI への入力時に「キャラ定義部分だけ抜粋して貼る」操作の抜けを防ぐ。

---

## 起動時の原則

1. **必ず AskUserQuestion で完結する。** 憶測で埋めない。詰め切れなかった項目は「未確定（推測しない）」と明示し、フェーズ7の検証ループで埋める。
2. **フェーズ固定・順序固定。** 下記の8フェーズを上から順に実行。各フェーズ内で AskUserQuestion を最大4問単位で呼ぶ。
3. **LINE 審査ガイドライン完全網羅。** サイズ・余白・著作権・禁止表現を全項目チェックリスト化して `spec.md` に明記する。
4. **画像生成AIは3系統並列。** `character.md` と各プロンプトに **GPT-4o image 用 / Nano Banana 用 / SD・NovelAI 用** の3系統を必ず併記する。
5. **デフォルト40枚。** ユーザーが他枚数を希望しない限り、40枚分のプロンプトを生成する。
6. **キャラ確定方式は両対応。** 画像が提供されたら Read してビジュアル特徴を抽出。提供なしならフェーズ1で AskUserQuestion で詳細ヒアリング。
7. **曖昧語禁止。** 「かわいい」「やや」「明るい」「いい感じ」「a bit」「kind of」「soft」「nice」「cute」を最終成果物（character.md / プロンプト）に**1個も残さない**。レビュー段階で grep で検出して全削除する。
8. **再生成要求の判定。** ユーザーが「思い通りでない」「違う」「もう一度」と言ったら、新規ヒアリングではなく**フェーズ7（検証ループ）に直行**する。
9. **十四式マッピング必須。** ユーザーの CLAUDE.md の「十四式」構造でスタンプ制作プロセスを写像する。比喩・感情語は使わない。**spec.md 本文の最先頭セクションは「## 十四式マッピング」、その次が「## 対象価値一致サマリ」で固定**（somo 共通ルール）。フェーズ0.7 で確定する。

---

## フェーズ0：スタンプ名・キャラ画像の有無確定

最初の発話：「LINE スタンプ設計書を対話で作ります。スタンプ名と、キャラクター画像があるかどうかから確認します」とだけ述べ、即 AskUserQuestion を呼ぶ。

聞く内容：

1. **スタンプ名（Other で自由入力前提）**：日本語名・英文ローマ字名どちらでも可。これがディレクトリ名になる。
2. **キャラクター画像の有無**：「これからアップロード/パスを伝える」／「画像なし、ゼロから設計する」／「テキストスタンプでキャラなし」
3. **総枚数**：8 / 16 / 24 / 32 / **40（推奨・デフォルト）**
4. **タッチ（画風）の方向性**：ゆるかわ手描き / シンプルベクター / リアル系 / アニメ調 / ピクセルアート / 写真風 / その他

スタンプ名確定後、即座に：

```bash
mkdir -p ~/stamp-specs/<スタンプ名>
```

以降、フェーズ1〜6が完了するたびに該当ファイル（`character.md` / `spec.md` / `pose-library.md`）を**逐次追記・更新**する。

**画像が提供される場合**：ユーザーから画像パスを受け取り Read で読み込む。読み込んだ内容（色・体型・特徴的パーツ・線質・塗り方）をフェーズ1で character.md に書き込む。

---

## フェーズ0.5：対象価値一致 ヒアリング（必須・マネタイズ軸確定）

**somo ルートコンセプト「対象価値一致（ピンポイント一致）」に基づく。** LINE スタンプの「誰が・どんな場面で・何を伝えたくて・なぜ120円を払うか」を全項目具体的に埋める。曖昧語禁止。

AskUserQuestion を2ラウンド（各4問）で8項目を確定する。

### ラウンド0.5-A（4問）

1. **①対象**：このスタンプを買う人を1文で。年齢帯・性別・どんなトーク相手がいるか（Other 自由入力）
2. **②タイミング**：どんな会話場面で送るか（例：「返事が思いつかないとき」「感謝を軽く伝えたいとき」）。Other 自由入力
3. **③状態・心理**：送り手の感情状態。選択肢例＝笑いを誘いたい・共感を示したい・照れて言えない・面倒を省きたい・Other
4. **④課題**：既存スタンプでは何が足りないか。「今まで使っていたスタンプの不満」（Other 自由入力）

### ラウンド0.5-B（4問）

5. **⑤提供価値**：このスタンプで何が伝わるか。「○○な気持ちが○○なキャラで伝わる」の形式（Other 自由入力）
6. **⑥差分**：他の似たようなスタンプと比べてこれを選ぶ理由（Other 自由入力）
7. **⑦許容価格**：購入決断ポイント。選択肢例＝120円（最小セット）・250円・370円・610円・Other
8. **⑧継続理由**：「トークルームに入れ続ける理由」または「削除せずに使い続ける理由」（Other 自由入力）

2ラウンド完了後、Claude は以下の1文を生成してユーザーに提示する。承認されるまで修正する：

> 「〇〇な人が、〇〇の会話場面で、〇〇の気持ちを伝えたいとき、既存スタンプでは〇〇が足りず、このスタンプで〇〇が伝わるため、〇〇円で購入し使い続ける」

**この1文が承認されるまでフェーズ0.7 / フェーズ1 には進まない。**

確定後、spec.md の **本文最先頭** に以下の順で置く（somo 共通ルール）：

0. **読み方ガイド（AI向けノート・blockquote）** — somo/SKILL.md の「読み方ガイドの固定テンプレート」をそのままコピペ。スタンプ用の追記（character.md / pose-library.md との関係）があれば末尾に1〜2行追加可（骨格は変えない）。
1. **## 十四式マッピング**（プレースホルダ）— 中身は「※フェーズ0.7 で確定する」と注記し、空のテーブル枠（① 認 〜 ⑭ 継）だけ先置き
2. **## 対象価値一致サマリ** — 承認された1文 + ①〜⑧の確定値テーブル

その後、続くセクション（用途配分・審査ガイドライン・プロンプト集等）は下に書く。フェーズ0.7 完了時に Edit でプレースホルダを実マッピングに置換する。

---

## フェーズ0.7：十四式マッピング = 入力→出力フィードバックループ定義（必須・spec.md 最先頭セクション）

**難しく考えない。** 十四式は構造が維持するための条件を観測した式であり、マッピングとは「**入力から出力までちゃんと通るための定義**」である。LINE スタンプの場合、1スタンプ生成1サイクルの入出力は**ほぼ機械的に固定**できる。AskUserQuestion は呼ばず、Claude が以下の固定テーブルを spec.md 本文最先頭に書き込み、書き終えたあとユーザーに「OK／補正」を1回確認する。比喩・感情語は使わない。

**1サイクルの入力（固定）**：
- character.md §1.5-G の [CHARACTER LOCK] 該当系統ブロック（GPT-4o image / Nano Banana / SD のどれか）
- pose-library.md の該当行（カメラ・体向き・視線・表情ID・口・眉・腕・小道具・台詞・台詞処理）
- spec.md §5-1 の共通プロンプト（規格・著作権・ネガティブ）
- 個別指示（カテゴリ・台詞文字列）

**1サイクルの出力（固定）**：
- LINE 審査通過する透過 PNG 1枚（370×320、12% inset、HEX 一致、形状名詞一致、表情 ID 一致、半透明ピクセルなし）

spec.md 本文最先頭の「## 十四式マッピング」セクションに以下を書く：

```markdown
## 十四式マッピング — 入力→出力フィードバックループ定義

**1サイクルの入力**：character.md §1.5-G + pose-library.md 該当行 + 共通プロンプト + 個別指示
**1サイクルの出力**：LINE 審査通過する透過 PNG 1枚（370×320、12% inset、HEX 一致、形状名詞一致、表情 ID 一致）

| 式 | 1サイクル内の役割 | 本スタンプでの具体 |
|---|---|---|
| ① 認 | 入力プロンプトが観測された | AI に [CHARACTER LOCK + pose 行 + 共通] が渡された瞬間 |
| ② 思 | 入力が内部処理に入った（中央未成形） | AI が画像生成を開始（latent 上の未成形） |
| ③ 鳴 | 内部処理が確定した（中央成形） | AI が画像を確定（生成完了） |
| ④ 示 | 出力先への指向が決まった | PNG として書き出される指向 |
| ⑤ 応 | 出力が実体化した | 透過 PNG ファイル 1枚 |
| ⑥ 衡 | 入→出 1サイクルが定常成立 | 同じ入力で何度生成しても審査通過品質が出る状態 |
| ⑦ 縁 | 複数サイクルの相互参照 | スタンプ間の連続使用・組み合わせ・シリーズ感 |
| ⑧ 家 | 並列サイクルの持続単位 | 1セット N 枚として持続する制作単位 |
| ⑨ 幅 | 1サイクルが崩壊しない入力範囲 | LINE 審査が通る規格・著作権・表現の範囲 |
| ⑩ 許 | 正常とみなす入力範囲（バリデーション） | 曖昧語ゼロ・HEX 一致・形状名詞一致・表情ID 一致・12% inset |
| ⑪ 循 | 許内＝再現可能・正常系 | 同じプロンプトで何度でも審査通過品質が出る |
| ⑫ 乱 | 許外（要エラーハンドリング） | 色ドリフト・形状ずれ・キャラ非同一・余白不足（フェーズ7 検証ループ対象） |
| ⑬ 絶 | 幅の外（不成立） | リジェクト・著作権侵害発覚・キャラ崩壊（設計やり直し） |
| ⑭ 継 | 1サイクル終了→次サイクル開始 | 次スタンプ番号への遷移、または同番号の再生成（フェーズ7 経由） |

### 通り道（フィードバックループ）

入力（CHARACTER LOCK + pose 行 + 共通）→ ① 認 → ② 思 → ③ 鳴 → ④ 示 → ⑤ 応（透過 PNG）→ 出力 → ⑭ 継 → 次スタンプの入力

- ⑩ 許（曖昧語ゼロ・HEX 一致・形状一致）の範囲内なら ⑥ 衡 として安定
- 許の外（幅の内）に出たら ⑫ 乱 → フェーズ7 検証ループで character.md を強化して再投入
- 幅の外に出たら ⑬ 絶 → 設計やり直し
- 全 N 枚が安定持続すれば ⑧ 家 = 1セット完成
```

書き出したあと AskUserQuestion で確認：

- question：「この十四式マッピングで spec.md 最先頭に書きました。補正しますか？」
- header：「十四式マッピング確認」
- multiSelect：false
- options：
  - `OK、このまま進む` — description: "提示テーブルをそのまま採用してフェーズ1 へ進む"
  - `補正したい行がある` — description: "Other で補正したい式番号と内容を伝える"

OK が出るまでフェーズ1 には進まない。補正があれば Edit で該当行を更新してから再度この AskUserQuestion を呼ぶ。

---

## フェーズ1：キャラクター設定

### 画像なしルート

AskUserQuestion を最大2回（合計最大8問）呼ぶ。

**1回目（4問）**：

1. **種族・モチーフ**：人間／犬／猫／うさぎ／鳥／架空生物／無生物（食べ物・物体）／その他
2. **体型プロポーション**：2〜3頭身（デフォルメ強）／4〜5頭身（中間）／6頭身以上（リアル寄り）／不定形
3. **ベースカラー**：白／黒／茶／パステル系／ビビッド系／モノクロ／カラフル混在
4. **性格イメージ**：ゆる／元気／クール／毒舌／天然／真面目／その他

**2回目（4問）**：

5. **顔の特徴（multiSelect）**：丸目／点目／ジト目／キラキラ目／半目／涙目／眉なし／頬染め固定／口大きめ／口小さめ／牙／ヒゲ
6. **特徴的パーツ（multiSelect）**：帽子／リボン／メガネ／首輪／しっぽ／羽／角／模様（ぶち・しま）／なし
7. **線画スタイル**：太線でクッキリ（推奨：LINE標準）／細線で繊細／線なし（フラット）／鉛筆風ラフ
8. **塗りスタイル**：ベタ塗り（推奨：LINE標準）／グラデーション／水彩風／陰影あり（セルシェード）／テクスチャあり

### 画像ありルート

提供画像を Read した結果から自動抽出した特徴を提示し、不明点のみ AskUserQuestion。最低限聞く項目：

1. **抽出特徴の確認**：「画像から○○・△△・□□を抽出しました。これで character.md に確定してよいか？」（Yes/補正/別画像参照）
2. **画像で表現されていない要素の補完**：性格／台詞トーン／追加ポーズの自由度

---

## フェーズ1.5：再現性仕様（色・パーツ・比率の完全言語化）— 最重要

このフェーズの確定内容はすべて **`character.md`** に書き込む。AskUserQuestion を最大3回（合計最大12問）呼ぶ。

### 1.5-A：色の指定（カラーロール × HEX 必須）

AskUserQuestion で**全項目を HEX で**確定する。「Other 自由入力」で `#RRGGBB` を受け付ける。AI 任せ・「明るい色」等の曖昧語は不可。

| カラーロール | 用途 | 例 |
|---|---|---|
| `body_main` | 体の主色 | `#F5E6D3` |
| `body_sub` | お腹・差し色等の副色 | `#FFFFFF` |
| `outline` | 線画の色 | `#2B1F18` |
| `shadow_1` | 影色1段目 | `#D9C4AE` |
| `shadow_2` | 影色2段目（使う場合のみ） | `#B89C82` |
| `highlight` | ハイライト色 | `#FFFAF0` |
| `eye_white` | 白目 | `#FFFFFF` |
| `eye_pupil` | 黒目・瞳 | `#1A1A1A` |
| `eye_highlight` | 瞳のハイライト | `#FFFFFF` |
| `cheek` | 頬染め | `#FFB6B0` |
| `mouth_inside` | 口内 | `#C04848` |
| `accessory_1` | 特徴的パーツ1の色 | `#E63946` |
| `accessory_2` | 特徴的パーツ2の色 | （該当時のみ） |
| `text_main` | スタンプテキスト色 | `#222222` |
| `text_outline` | テキスト縁取り色 | `#FFFFFF` |

**配色比率（必須・推奨止まりではない）**：色は単に並べず、**画面占有比を必ず指定**する。例 `body_main 60% / body_sub 25% / accessory 10% / outline 5%`。AskUserQuestion で「均等推奨：60-25-10-5 でよいか／自由指定」を聞く。

### 1.5-B：パーツの指定（部位 × 形状名詞 × 相対位置 × サイズ比）

各パーツを以下フォーマットで character.md に書く。**「相対位置」と「サイズ比」は顔幅・頭高を 1.0 とした倍率**で記述（絶対 px は使わない）。

| パーツ | 形状名詞 | 相対位置 | サイズ比 | 色ロール |
|---|---|---|---|---|
| 顔輪郭 | 円／丸み四角／涙滴 | — | 顔幅 1.0 / 顔高 1.0 | outline + body_main |
| 目（左） | 楕円ベタ／丸点／半月／キラキラ星 | 顔の上から **0.40**、左端から **0.30** | 顔幅の **0.18** | eye_white + eye_pupil |
| 目（右） | （左と対称） | 顔の上から **0.40**、左端から **0.70** | 顔幅の **0.18** | 同上 |
| 鼻 | 三角／点／無し | 顔の上から **0.55** | 顔幅の **0.06** | outline |
| 口 | 線一本／三日月／逆三角／オープン | 顔の上から **0.70** | 顔幅の **0.25** | outline (+ mouth_inside) |
| 頬 | 楕円ピンク／斜線2本／なし | 顔の上から **0.60**、左右両端寄り | 顔幅の **0.15** | cheek |
| 耳・角・羽等 | （該当時） | 顔上端より **+0.20** 上 | 顔幅の **0.30** | body_main / accessory_1 |
| 体 | 卵型／矩形丸／無し（顔のみ） | 顔下端から下方向 | 顔幅の **1.10** × 顔高の **0.90** | body_main |
| 手 | ミトン丸／指あり／無し | 体の左右 **0.50** 高さ | 顔幅の **0.30** | body_main |
| 足 | 楕円／ちょこん／無し | 体の下端 | 顔幅の **0.25** | body_main |
| アクセサリー | 用途固定の名詞 | 個別 | 個別 | accessory_1/2 |

AskUserQuestion で**形状名詞**と**サイズ比の調整希望**を確認。デフォルト値を提示し、ユーザーが「Other」で別倍率を入れた場合のみ上書き。

### 1.5-C：比率の指定（全身比 × 顔比 × 画面占有）

固定3項目を AskUserQuestion で確定。

1. **頭身**：1.5頭身（顔がほぼ全身）／2頭身／2.5頭身／3頭身
2. **顔の縦横比**：1.00：1.00（真円）／1.00：0.95（やや縦長）／1.00：1.05（やや横長）
3. **画面占有率**：キャラの最大寸が画像領域の **70%／80%／90%**（**80% 推奨**）

これらは**全枚数共通**で character.md 冒頭に固定し、各プロンプトでも毎回繰り返す（再現性のため冗長を許容）。

### 1.5-D：線質・シェーディング段数（必須）

AskUserQuestion で確定：

1. **線質名詞**：均一クリーンライン（推奨）／粗いラフライン／途切れあり手描き／線なしフラット
2. **線太さ相当**：2px相当／3px相当（推奨）／4px相当／6px相当
3. **シェーディング段数**：フラット（影なし）／セルシェード1段（shadow_1のみ）（推奨）／セルシェード2段（shadow_1+shadow_2）／グラデーション（非推奨・LINE審査で目立つ場合あり）
4. **グラデーション禁止フラグ**：true（推奨：色が指定通り出やすい）／false

### 1.5-E：表情コード（E01〜E10）の事前定義

40枚分のスタンプで使う表情を**事前に ID 化**して character.md に持つ。pose-library.md からは ID 参照だけで済む。

AskUserQuestion で「使用する表情を multiSelect で選択」：

| ID | 名称 | 目の形 | 眉 | 口 | 頬 |
|---|---|---|---|---|---|
| E01 | 喜び | 半月（笑い目） | 平行 | 三日月オープン | あり |
| E02 | 驚き | 真円大 | 釣り上げ | 真円オープン | なし |
| E03 | 怒り | 釣り目 | への字釣り上げ | 逆三角オープン | なし |
| E04 | 悲しみ | 涙目 | ハの字 | への字 | なし |
| E05 | 無表情 | 楕円ベタ | 平行 | 線一本 | なし |
| E06 | 困惑 | ジト目 | ハの字 | 三日月逆 | なし |
| E07 | 眠気 | 半目 | 平行 | 小三角オープン | あり |
| E08 | キラキラ | キラキラ星 | 釣り上げ | 三日月オープン | あり |
| E09 | 照れ | 半月伏し目 | ハの字 | 小三日月 | あり強 |
| E10 | クール | 半目 | 釣り上げ | 線一本 | なし |

各 ID に対応する形状名詞は character.md 内で**完全に固定**する。pose-library.md は ID だけ書く。

### 1.5-F：曖昧語禁止規約（自動適用・ヒアリング不要）

character.md と全プロンプトに以下の固定セクションを埋め込む：

```
[NG VOCABULARY — never appear in this spec or in generated prompts]
日本語：かわいい / やや / 大きめ / 明るい / 暗い / いい感じ / 適度に / ほどよく / なんとなく / それっぽい
English: cute, nice, soft, a bit, kind of, somewhat, lovely, beautiful, pretty, charming, slight, gentle, lightly, kinda
[Resolution rule] If any of these would naturally appear, replace with: a HEX color, a numeric ratio (e.g. 0.18 of face width), or a noun shape (e.g. "ellipse solid", "crescent line").
```

→ 設計書を書き終えた後、Bash で `grep -inE '(かわいい|やや|大きめ|明るい|cute|nice|soft|a bit|kind of)' ~/stamp-specs/<name>/` を実行して**1件もヒットしないこと**を確認する。ヒットがあれば character.md を Edit で全置換する。

### 1.5-G：[CHARACTER LOCK] ブロックの3系統文字列化

確定した色・パーツ・比率・線質・表情を、**3系統の固定文字列**に展開し、character.md の「§1.5-G」として保存する。フェーズ5の各プロンプトはこの3ブロックを毎回先頭に貼る。

#### GPT-4o image / GPT Image 1 用（自然言語+構造化箇条書き）

```
[CHARACTER LOCK — Strictly follow every line. Do not paraphrase, do not infer, do not soften.]
- Head-to-body ratio: <頭身>. Face aspect ratio: <顔縦:横>. Character occupies <70/80/90>% of canvas longest side, centered, with at least 12% inset from every edge.
- Body main color #<body_main>, sub color #<body_sub>, outline color #<outline> at <2/3/4/6>px-equivalent uniform clean lineart (no double lines, no sketch lines, no broken edges).
- Shading: <flat / cel-shaded 1-tone using shadow_1 #<shadow_1> / cel-shaded 2-tone using shadow_1 #<shadow_1> and shadow_2 #<shadow_2>>. No gradient. No painterly blending.
- Color area ratio: body_main <60>%, body_sub <25>%, accessory <10>%, outline <5>%.
- Face: <円/丸み四角/涙滴> shape.
- Eyes: <形状名詞>, positioned at 0.40 from face top, 0.30 and 0.70 from face left, size 0.18 of face width. White #<eye_white>, pupil #<eye_pupil>, highlight dot #<eye_highlight>.
- Mouth: <形状名詞>, at 0.70 from face top, width 0.25 of face width, inside #<mouth_inside>.
- Cheeks: <形状名詞>, color #<cheek>, size 0.15 of face width.
- Accessory: <名詞> at <位置>, size <比率>, color #<accessory_1>.
- Expression IDs available: E01..E10 as defined in this spec; the per-sticker prompt names exactly one ID and you render that exact shape combination.
- Forbidden vocabulary in interpretation: cute, nice, soft, a bit, kind of, lovely, pretty. Do not infer beyond what is written. If a property is missing, leave it neutral; do not invent.
[/CHARACTER LOCK]
```

#### Gemini Nano Banana (2.5 Flash Image) 用（簡潔自然言語+一貫性強調）

```
[CHARACTER LOCK — keep exact same character across all images]
Same character every time. <頭身> head-to-body, face aspect <顔縦:横>, fills <70/80/90>% of canvas centered with 12% margin on all sides.
Colors (use exactly these HEX, no shifting): body #<body_main>, sub #<body_sub>, outline #<outline>, shadow_1 #<shadow_1>, cheek #<cheek>, eye pupil #<eye_pupil>.
Lineart: <2/3/4/6>px uniform clean. Shading: <flat / cel 1-tone / cel 2-tone>. No gradient.
Eye shape: <形状名詞>, size 0.18 face width, at 0.40 top / 0.30 & 0.70 left.
Mouth: <形状名詞> at 0.70 top, width 0.25.
Cheeks: <形状名詞>, #<cheek>, 0.15 face width.
Color area ratio body:sub:accessory:outline = 60:25:10:5.
Use expression ID exactly as specified per sticker (E01..E10 defined below).
[/CHARACTER LOCK]
```

#### Stable Diffusion / NovelAI 用（タグ列+重み記法）

```
[CHARACTER LOCK]
(consistent character:1.4), (same character design:1.3), <chibi/2heads/2.5heads/3heads>,
body color #<body_main>, sub color #<body_sub>, lineart color #<outline>, lineart (uniform thick:1.2 / medium / thin),
eyes:(<形状タグ>:1.1) color #<eye_pupil>, eye position upper-mid face, eye size (0.18 of face width:1.1),
mouth:(<形状タグ>:1.0) at 0.70 face top, mouth inside #<mouth_inside>,
cheek:(<形状タグ>:0.9) color #<cheek>,
accessory:(<名詞タグ>:1.1) color #<accessory_1>,
flat color, (cel shading 1-tone / 2-tone / no shading), no gradient,
(character occupies 80% of canvas:1.2), centered, (12% margin on all sides:1.1),
expression: <E01..E10 のタグ展開>,
[/CHARACTER LOCK]
```

→ プレースホルダ `<…>` はフェーズ1.5-A〜E の確定値で埋めて文字列化し、character.md §1.5-G に3ブロックすべて保存する。

---

## フェーズ1.6：ポーズ・表情・構図の数値化規約（pose-library.md）

このフェーズの目的は **40枚分のポーズ・表情・カメラ・台詞を pose-library.md に固定**して揺れをなくすこと。

### 1.6-A：固定フィールド定義

`pose-library.md` 冒頭に以下の規約セクションを書く（character.md と重複可）：

| 項目 | 記述形式 | 必須 |
|---|---|---|
| カメラ距離 | フェイス／バスト／ニーアップ／フル の4択 | 必須 |
| 体の向き | 正面0° / 斜め右30° / 斜め右45° / 横右90° / 後ろ向き180°（左側ミラー可） | 必須 |
| 視線方向 | カメラ目線／右上15°／左下20°／伏し目／流し目 | 必須 |
| 表情コード | E01〜E10（character.md §1.5-E 定義のID） | 必須 |
| 口の状態 | 閉じ／開け小（高さ0.10）／開け大（高さ0.20）／三日月／逆三角 | 必須 |
| 眉の状態 | 平行／ハの字／への字／釣り上げ | 必須 |
| 腕ポーズ | 数値+名詞：「両腕頭上V字、肘伸展」等 | 必須 |
| 小道具 | 名詞+位置（例：「右手にハート、顔の右斜め上」）／なし | 必須 |
| 台詞 | 4文字以内は直描き想定、5文字以上は後合成想定（フラグ明記） | 任意 |

### 1.6-B：40枚分のテーブル

`pose-library.md` 本体は1行=1スタンプの表：

```markdown
| # | カテゴリ | カメラ | 体向き | 視線 | 表情 | 口 | 眉 | 腕 | 小道具 | 台詞 | 台詞処理 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 01 | 挨拶 | バスト | 正面0° | カメラ目線 | E01 | 三日月 | 平行 | 右手挙手、左手腰 | なし | おはよう | 直描き |
| 02 | 返事 | バスト | 斜め右30° | カメラ目線 | E01 | 三日月 | 平行 | 右手親指立て、左手腰 | なし | OK | 直描き |
| ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... |
```

AskUserQuestion で確認する内容：

1. **生成方針**：全40枚自動生成（ユーザーは後で個別修正）／主要10枚はユーザーと一緒に詰める／全件1枚ずつ確認
2. **必須カテゴリ**（multiSelect）：挨拶／返事／感情／謝罪・お礼／応援／日常行動／季節／ビジネス／ネタ／無言

「全件自動生成」を選んだ場合：上記の固定フィールドをすべて埋めた40行を機械的に生成。曖昧表現は使わない。

---

## フェーズ2：用途カテゴリ配分（spec.md へ）

AskUserQuestion で（4問）：

1. **配分方針**：定番網羅型（挨拶・返事・感情・行動を均等）／感情特化型／ビジネス特化／推し活・ファン向け／ネタ・ギャグ特化／ニッチ特化
2. **テキスト有無の比率**：全枚数テキスト入り／約半分テキスト入り／テキスト最小限／テキストなし
3. **言語**：日本語のみ／英語のみ／日本語＋英語ミックス／無言主体
4. **台詞長さ方針**：全4文字以内（直描き優先）／4文字超も許容（後合成前提）／混在

→ 結果は spec.md に書き、pose-library.md の「台詞」「台詞処理」列の埋め方の指針になる。

---

## フェーズ3：LINE 審査ガイドライン完全網羅チェックリスト（spec.md へ・固定）

### 3-1. 規格仕様（生成時保証を強化）

| 項目 | 要件 |
|---|---|
| メイン画像 | 240 × 240 px / PNG / 透明背景 |
| スタンプ画像 | 370 × 320 px（最大） / PNG / 透明背景 |
| トークルームタブ画像 | 96 × 74 px / PNG / 透明背景 |
| 余白 | 画像端から最低 **10 px**。生成プロンプトには「**12% inset on all sides**」を明記（10px = 約3% だが AI 制御余裕のため大きめ指定） |
| 解像度 | 72 dpi 以上 |
| カラーモード | RGB |
| ファイル形式 | PNG（透過情報含む） |
| 背景 | 完全透明。半透明ピクセル混入禁止 |
| 1枚あたり最大ファイルサイズ | 1 MB 以内 |
| 総枚数選択肢 | 8 / 16 / 24 / 32 / 40 |

**生成時保証の文言（毎枚プロンプトに必須）**：

- GPT-4o image：`Output as PNG with fully transparent background, alpha channel preserved. No semi-transparent pixels at edges. No drop shadow extending into background. Character bounding box must be inset by at least 12% of canvas on all four sides; no element touches any edge.`
- Nano Banana：`Transparent PNG output, no white or solid background, no shadow on background. 12% margin all sides.`
- SD/NovelAI：ネガティブに `white background, opaque background, drop shadow background, edge touching, cropped, semi-transparent pixels` を含める。透過困難なら**フォールバック**：マゼンタ単色背景 `#FF00FF` で出力 → 後処理で除去（後処理は別スキル領域だが、生成側の指示文として spec.md に明記）。

### 3-2. 著作権・肖像権チェック（全件NG）

- [ ] 既存キャラクターの引用・模倣・パロディ
- [ ] 実在の有名人の写真・似顔絵・名前
- [ ] 企業ロゴ・ブランドマーク
- [ ] 第三者の写真
- [ ] 既存の商業作品の構図・名シーンの流用
- [ ] トレース疑義を生む既存画像参照

→ プロンプト末尾に必ず：`original character only, no copyrighted material, no real persons, no brand logos, no LINE official marks`

### 3-3. ネガティブ指定（破綻パターン拡充）

**禁止表現（審査NG）**：暴力・流血・武器／性的・露骨／差別表現／政治的主張／宗教シンボル／違法行為／自殺・自傷／侮辱語／個人情報／QRコード・バーコード／他社サービス名／LINE公式マーク模倣。

**破綻パターン（品質NG）**：

- 体型崩れ：`extra limbs, missing limbs, fused fingers, six fingers, four fingers (unless designed), asymmetric eyes, twisted neck, wrong body proportions`
- 色ドリフト：`color shift, hue change, gradient leak, palette mismatch, off-color`
- 線の崩れ：`broken lineart, double lines, sketch lines, unclean edges, inconsistent line weight`
- 規格違反：`cropped, out of frame, no padding, edge touching, semi-transparent pixels, white background, opaque background, drop shadow background`
- キャラ非同一：`off-model, inconsistent design, different character, character drift, alternate outfit (unless specified), face shape change`
- テキスト破綻：`misspelled text, garbled text, fake characters, broken kana, illegible text`

GPT-4o image / Nano Banana 用には自然言語版（`Do not produce: extra limbs, fused fingers, asymmetric eyes, ...`）として書き下す。

### 3-4. テキスト要件

- [ ] 文字スタンプは読みやすさ優先
- [ ] 誤字脱字なし
- [ ] 日本語スタンプ内に外国語混在は最小限
- [ ] スラング・若者言葉は使ってよいが侮辱語は不可

---

## フェーズ4：画風・カラーパレット・統一仕様の確定（spec.md へ）

AskUserQuestion で（4問）：

1. **カラーパレット指定**：character.md の HEX に準拠（推奨）／系統指定（パステル等）／キャラ画像に準拠／AI 任せ（非推奨）
2. **線画の太さ統一**：太線（4〜6px相当・推奨）／中線（2〜3px）／細線（1px）／線なし
3. **影・ハイライト方針**：完全フラット／セルシェード1段（推奨）／セルシェード2段／グラデーション（非推奨）
4. **共通背景処理**：完全透明（必須）／キャラ周囲の発光許容するか／白フチの有無

→ ここで決まった画風が**全N枚で完全統一**されることを spec.md に明記。

---

## フェーズ5：画像生成AI 用プロンプト群の生成（最重要セクション）

`spec.md` に以下を**全枚数分**機械的に展開する。

### 5-1. 共通プロンプト（全枚数共通の前置き）

**重要**：以下の各プロンプトの先頭に、**character.md の §1.5-G 該当系統ブロックをそのままコピペ**する。1文字も改変しない。

#### GPT-4o image / GPT Image 1 用 — 自然言語+構造化

```
[CHARACTER LOCK ← character.md §1.5-G の GPT-4o image 用ブロックをそのまま貼る]

Create a LINE sticker illustration. Strictly follow:
- Size: 370x320 pixels, PNG with fully transparent background, alpha channel preserved
- Character bounding box inset by at least 12% from every canvas edge; no element touches any edge
- No semi-transparent pixels at edges; no drop shadow extending into background
- Original character only — no copyrighted IP, no real persons, no brand logos, no LINE official marks
- Strictly use the colors (HEX), part shapes (named), sizes (ratios), and expression ID from CHARACTER LOCK. Do not invent, infer, or substitute.
- Consistent character design across all stickers in the set
- No violence, no sexual content, no discrimination, no political/religious statements, no QR/barcodes, no third-party brand names

Do not produce: extra limbs, missing limbs, fused fingers, six fingers, asymmetric eyes, twisted neck, color shift, gradient leak, broken lineart, double lines, off-model, character drift, misspelled text, garbled text, white background, opaque background, drop shadow background, cropped, edge touching, semi-transparent pixels.
```

#### Gemini Nano Banana 用 — 簡潔自然言語

```
[CHARACTER LOCK ← character.md §1.5-G の Nano Banana 用ブロックをそのまま貼る]

LINE sticker. 370x320 PNG, transparent background. 12% margin all sides. Original character only. Use exact HEX colors and shapes from CHARACTER LOCK. Same character every time. No violence, no NSFW, no logos, no real persons.

Avoid: extra limbs, fused fingers, asymmetric eyes, color shift, gradient leak, broken lineart, off-model, white background, edge touching, garbled text.
```

#### Stable Diffusion / NovelAI 用 — タグ列

```
Positive: [CHARACTER LOCK ← character.md §1.5-G の SD 用ブロック], original character, line sticker, transparent background, isolated on transparent, 370x320, (12% padding all sides:1.2), simple flat colors, clean uniform lineart, centered composition, expressive pose, masterpiece, best quality
Negative: copyrighted character, real person, brand logo, watermark, signature, text artifact, blurry, low quality, jpeg artifacts, cropped, out of frame, extra limbs, missing limbs, fused fingers, six fingers, asymmetric eyes, twisted neck, bad anatomy, color shift, hue change, gradient leak, palette mismatch, off-model, inconsistent character, character drift, broken lineart, double lines, sketch lines, unclean edges, NSFW, gore, weapons, blood, political symbols, religious symbols, qr code, barcode, third-party logo, LINE official mark, white background, opaque background, drop shadow background, edge touching, semi-transparent pixels, misspelled text, garbled text
Recommended: SDXL or Pony/Animagine/Illustrious base, sampler DPM++ 2M Karras, steps 28, CFG 6-7, seed fixed per set (e.g. 1234567)
```

### 5-2. 各スタンプ個別プロンプト（N枚分）

`spec.md` にテーブル形式で展開：

```markdown
### スタンプ #01 — <カテゴリ>：<台詞 or 無言>

**pose-library.md #01 参照**：カメラ=バスト / 体向き=正面0° / 視線=カメラ目線 / 表情=E01 / 口=三日月 / 眉=平行 / 腕=右手挙手・左手腰 / 小道具=なし / 台詞=おはよう / 台詞処理=直描き

**GPT-4o image 用**：
[共通プロンプト 5-1 GPT-4o image] + The character is in bust shot, body facing 0° (front), looking at camera, expression E01 (joy: half-moon eyes, parallel brows, crescent open mouth, cheeks visible), right arm raised waving, left arm at hip. Render the text "おはよう" in the upper-right area in <フォント方針> at #<text_main> with #<text_outline> outline, clearly readable, no spelling errors. If uncertain about glyph, leave the area blank rather than producing fake characters.

**Nano Banana 用**：
[共通プロンプト 5-1 Nano Banana] + Bust shot, front 0°, camera-eye contact, E01 (joy), right hand waving, left at hip. Text "おはよう" upper-right, color #<text_main> with #<text_outline> outline.

**SD/NovelAI 用**：
Positive: [共通Positive 5-1 SD] + (bust shot:1.1), (front view:1.1), looking at viewer, (E01 joy expression:1.2), half-moon eyes, parallel brows, crescent open mouth, right arm raised waving, left hand on hip, text "おはよう" upper-right, gothic bold font
Negative: [共通Negative 5-1 SD]
```

N枚分これを繰り返す。フェーズ1.6-B で「全枚数自動生成」を選んだ場合は機械的に埋め、「主要のみ確認」を選んだ場合は確認した分は具体的に、残りはテンプレ枠で出して後で埋められるようにする。

### 5-2.5. テキスト（台詞）描画ルール

「思い通り出ない」に台詞崩れが含まれるため、以下を厳守：

- **日本語短文（4文字以内）**：直描き OK。フォント方針は character.md §1.5-A の `text_main` / `text_outline` に従う。フォント名詞を ID 化（F01ゴシック太字 / F02丸ゴシック / F03手書き風）。
- **5文字以上 or 細かい記号**：プロンプトで「テキスト領域を空白として確保」指示。`Reserve a clear blank rectangular area at <位置> for text overlay (added later); do not render any text in that area.` を入れる。台詞処理列に「後合成」と明記。
- **文字色・縁取り**：character.md の `text_main` / `text_outline` HEX を毎回繰り返す。
- **fake glyph 防止**：`If uncertain about a glyph, leave the area blank rather than producing fake characters or wrong kana.` をプロンプト末尾に入れる。

### 5-3. メイン画像・トークタブ画像 用プロンプト

| 画像 | サイズ | プロンプト方針 |
|---|---|---|
| メイン画像 | 240×240 | フルカラー・キャラの最も特徴的なポーズ。E01（喜び）、バスト、正面0°、カメラ目線が無難。 |
| トークタブ画像 | 96×74 | 単色シルエット推奨。`silhouette only, single color #<outline> on transparent` の指示。 |

それぞれに 3系統（GPT-4o image / Nano Banana / SD）のプロンプトを書く。

### 5-4. キャラシート（検収用）プロンプト — 設計書完成直後の必須ステップ

character.md を書き終えたら、**最初に必ずキャラシート1枚を生成してユーザーに見せる**。これで character.md の言語化が足りているかを早期に検証する（出力検証であり、入力リファレンスとしては使わない）。

```
[CHARACTER LOCK ← GPT-4o image 用ブロック全文]

Render a character reference sheet on a single canvas, 1024x1024, white background:
- Top-left panel: front view (0°), full body, neutral pose, expression E05 (neutral)
- Top-right panel: 3/4 view (right 30°), full body, expression E01 (joy)
- Bottom-left panel: side view (right 90°), full body, expression E05 (neutral)
- Bottom-right panel: 6 expression close-ups in 2x3 grid (E01, E02, E03, E04, E07, E09 from CHARACTER LOCK)
Label each panel with its name in small text at the bottom of the panel.
Strictly follow CHARACTER LOCK for all colors (HEX), shapes, ratios.
```

ユーザーがキャラシートを見て「OK」を出すまで、フェーズ7（検証ループ）で character.md を更新し続ける。OK が出てから初めて全N枚の生成プロンプトに進む。

---

## フェーズ6：書き出し制作・申請チェックリスト（spec.md 末尾固定）

### 画像生成後の処理

- [ ] 全画像の透過確認
- [ ] 全画像のサイズ確認（370×320 内、12% インセット）
- [ ] 全画像のファイルサイズ確認（1MB 以内）
- [ ] 画風統一性の目視確認（全枚並べて違和感なし）
- [ ] テキストの誤字脱字確認
- [ ] メイン画像（240×240）作成
- [ ] トークタブ画像（96×74）作成

### LINE Creators Market 申請

- [ ] 販売情報入力（タイトル・説明文・タグ）
- [ ] スタンプ説明文に著作権侵害なきことを明記
- [ ] コピーライト表記（© <作者名> <年>）
- [ ] テストデバイスでトーク表示プレビュー確認
- [ ] 価格設定（120円〜 / クリエイター取り分 50%）
- [ ] 申請送信 → 審査（通常 1〜7 営業日）

### 過去事故防止メモ

- 既存IPに似てしまった場合 → リジェクト確定。デザインから作り直し
- 余白不足（10px未満）→ 自動リジェクト。生成時に12%指定で実測クリア
- 半透明ピクセル混入 → リジェクト要因。生成プロンプトに `crisp edges, no semi-transparent pixels` を必ず追加

---

## フェーズ7：検証ループ（生成後の差分反映）

ユーザーが「思い通りでない」「違う」「もう1回」「色が違う」「キャラが違う」等と言ったら、**新規ヒアリングではなくここに直行**する。

### 7-1. 入力受け取り

ユーザーから生成画像のパス or アップロードを受け取る。複数枚なら順に Read。

### 7-2. 差分テーブル作成

character.md と pose-library.md を Read し、生成画像と項目ごとに照合：

| 項目 | 仕様（character.md） | 生成画像で観測 | 差分 |
|---|---|---|---|
| body_main 色 | #F5E6D3 | #E8D9C0 寄り | 色ドリフト：3% 暗化 |
| 目の形状 | 楕円ベタ | 半月になっている | 形状名詞ミス |
| 目のサイズ比 | 顔幅の 0.18 | 約0.25 で大きい | サイズ比ずれ |
| 線の太さ | 3px相当・均一クリーン | 場所により2-5px・不均一 | 線質ずれ |
| 余白 | 12% インセット | 上端接触 | 規格違反 |
| キャラ同一性 | （前回と一致すべき） | 顔輪郭が涙滴→円に変化 | character drift |

差分はユーザーに表で見せる。

### 7-3. character.md / pose-library.md の更新

差分の根本原因が「言語化不足」「曖昧語残存」「形状名詞抜け」「数値抜け」のどれかを判定し、character.md を Edit で**強化**する：

- 曖昧語が残っていた → 数値・HEX・形状名詞に置換
- NG語リストに追加すべき語 → §1.5-F に追記
- 形状名詞が「楕円」のように不明瞭 → 「楕円ベタ（アスペクト比1.5:1、垂直軸基準）」のように補強
- サイズ比の単位が曖昧 → 「顔幅の 0.18」のように基準を明記

更新が完了したら character.md の §1.5-G の3系統 [CHARACTER LOCK] ブロックも全文再生成する。

### 7-4. 再生成プロンプトを返す

更新後の character.md を先頭に貼った再生成プロンプト（該当スタンプ番号分）をユーザーに返す。「次の生成でこれをそのまま入力してください」と指示。

### 7-5. 再検証

ユーザーが再生成画像を返してきたら 7-1 に戻る。差分ゼロになるまでループ。

→ これにより設計書は「1発勝負」ではなく**生成失敗を吸収して進化する契約書**になる。

---

## フェーズ8：章ラベル整合性確認（必須・完成宣言の前）

フェーズ1〜7 までで設計書3ファイルを書き終えたら、各ファイル内の全 ## と ### セクション見出しに **十四式・対象価値一致のラベル**が付いているかを確認する。

### 8-A：機械的チェック（Bash）

```bash
# spec.md / character.md / pose-library.md でラベル無し見出しを抽出
for f in ~/stamp-specs/<スタンプ名>/*.md; do
  echo "=== $f ==="
  grep -nE '^##+ ' "$f" \
    | grep -vE '十四式マッピング|対象価値一致サマリ|読み方ガイド' \
    | grep -vE '— 十四式[①-⑭−]+ ?/ ?対象価値一致[①-⑧−]+' \
    || echo "OK: 全見出しにラベル付与済み"
done
```

ヒットがあれば、そのセクションにラベルが付いていない。Edit で追記する。

### 8-B：推奨ラベル候補（somo 共通ルール準拠）

**spec.md：**
| 章 | 推奨ラベル |
|---|---|
| 2. スタンプ用途配分 | `十四式④ / 対象価値一致②③` |
| 3. LINE 審査ガイドライン | `十四式⑨⑩ / 対象価値一致⑥` |
| 4. 画風・カラー・統一仕様 | `十四式③⑩ / 対象価値一致⑥` |
| 5. プロンプト集 | `十四式①②③④⑤ / 対象価値一致⑤⑥` |
| 6. 制作・申請チェックリスト | `十四式⑤⑥ / 対象価値一致−` |
| 7. 検証ループ運用ルール | `十四式⑫⑭ / 対象価値一致−` |

**character.md：**
| 章 | 推奨ラベル |
|---|---|
| 1. キャラクター基本設定 | `十四式③ / 対象価値一致⑤⑥` |
| 1.5-A 色 | `十四式③⑩ / 対象価値一致⑥` |
| 1.5-B パーツ | `十四式③⑩ / 対象価値一致⑥` |
| 1.5-C 比率 | `十四式③⑩ / 対象価値一致⑥` |
| 1.5-D 線質・シェーディング | `十四式③⑩ / 対象価値一致⑥` |
| 1.5-E 表情コード | `十四式③⑩ / 対象価値一致③⑤` |
| 1.5-F NG VOCABULARY | `十四式⑩ / 対象価値一致−` |
| 1.5-G CHARACTER LOCK 3系統 | `十四式①② / 対象価値一致−` |

**pose-library.md：**
| 章 | 推奨ラベル |
|---|---|
| 規約（フェーズ1.6-A） | `十四式④⑩ / 対象価値一致−` |
| 全N枚テーブル | `十四式④ / 対象価値一致②③` |

### 8-C：ユーザー確認（AskUserQuestion）

機械チェックが OK になったら、推定ラベル一覧をテーブルでユーザーに見せる：

- question：「章ラベルはこれでよいですか？」
- header：「ラベル整合性」
- multiSelect：false
- options：
  - `OK、このまま完成` — description: "全章ラベルを承認して設計書完成宣言（曖昧語チェック）へ進む"
  - `補正したい章がある` — description: "Other で補正したい章名と新ラベルを伝える"

OK が出るまで完成宣言できない。補正があれば Edit で該当行のラベルを更新し、再度 8-A 機械チェックを通す。

### 8-D：「対象価値一致−」過剰検出

`対象価値一致−` のラベルが付いた章が **3つ以上** あれば、Claude は以下を AskUserQuestion で確認：

- question：「対象価値一致に貢献しない章が3つ以上あります。実装すべきか再検討しますか？」
- header：「過剰実装の警告」
- multiSelect：false
- options：
  - `削れる章がある、削除する` — description: "ユーザーが章名を Other で指定して削除"
  - `すべて必要、このまま進む` — description: "全章を維持して完成宣言へ"

これは過剰実装の自動検出機能（somo 共通ルール）。

---

## 最終出力テンプレート

すべてのフェーズ完了後の `~/stamp-specs/<スタンプ名>/` 構造：

### character.md

```markdown
# <スタンプ名> — キャラクター定義（character.md）

> このファイルは画像生成AIに **毎回プロンプト先頭にコピペ**される契約書である。
> 曖昧語禁止。数値+固有名詞でのみ記述する。

## 1. キャラクター基本設定
（フェーズ1の確定内容）

## 1.5-A 色（HEX + 配色比率）
（フェーズ1.5-A の確定テーブル）

## 1.5-B パーツ（形状名詞・相対位置・サイズ比）
（フェーズ1.5-B の確定テーブル）

## 1.5-C 比率（頭身・顔縦横比・画面占有率）
（フェーズ1.5-C の確定値）

## 1.5-D 線質・シェーディング段数
（フェーズ1.5-D の確定値）

## 1.5-E 表情コード定義（E01〜E10）
（フェーズ1.5-E の確定テーブル：各IDの目・眉・口・頬の形状名詞）

## 1.5-F NG VOCABULARY（曖昧語禁止リスト）
（固定セクション）

## 1.5-G [CHARACTER LOCK] 3系統ブロック
### GPT-4o image / GPT Image 1 用
### Gemini Nano Banana 用
### Stable Diffusion / NovelAI 用
```

### spec.md

```markdown
# <スタンプ名> — LINE スタンプ設計書（spec.md）

生成日：<YYYY-MM-DD>
総枚数：<N>枚
画風：<ひとこと>
character.md と pose-library.md を併せて参照。

> **このファイルを読むAIへ（読み方ガイド）**
>
> 本設計書は CLAUDE.md の十四式（固定構造）に基づく。冒頭2セクション（## 0.7 十四式マッピング → ## 0.5 対象価値一致サマリ）で本プロジェクトの「入力→出力ループ」と「マネタイズ枠」が確定するので、以降の章を読む前に必ず読むこと。
>
> **十四式マッピング = 入力→出力フィードバックループの通り道定義。** 十四式は構造が維持するための条件を観測した式であり、マッピングとは「入力から出力までちゃんと通るための定義」である。難しく考えない。各式の役割：
> - ① 認=入力観測 / ② 思=内部処理開始（中央未成形）/ ③ 鳴=内部処理確定（中央成形）/ ④ 示=出力指向 / ⑤ 応=出力実体化
> - ⑥ 衡=入→出1サイクル定常成立 / ⑦ 縁=複数サイクル相互参照 / ⑧ 家=並列サイクル持続単位
> - ⑨ 幅=崩壊しない入力範囲 / ⑩ 許=正常とみなす入力範囲（バリデーション）
> - ⑪ 循=許内・正常系 / ⑫ 乱=許外・要エラーハンドリング / ⑬ 絶=幅外・クラッシュ／不成立
> - ⑭ 継=1サイクル終了→次サイクル開始
> - 通り道：`入力 → ① → ② → ③ → ④ → ⑤ → 出力 → ⑭ → 次サイクルの ①`。⑩ 許の範囲内で ⑥ 衡 が安定。
>
> **対象価値一致サマリ = この出力に「誰がいくら払うか」を 8 軸＋必須1文で確定。** 生成・レビューでは、この必須1文に貢献しない要素を新規追加しない。
>
> **成果物タイプ → 入出力**（本設計書は **画像生成（LINE スタンプ）**）：
> - 画像生成: プロンプト一式（CHARACTER LOCK + pose 行 + 共通） → 規格適合の透過 PNG
> - 1サイクル単位 = 1スタンプ生成
>
> **本設計書の章ラベル形式**（重要・全AI共通の読み方）：
> spec.md / character.md / pose-library.md のすべての ## と ### セクション見出し末尾に `— 十四式<番号> / 対象価値一致<番号>` のラベルが付いている。
> - 例：`## 5. 画像生成AI プロンプト集 — 十四式①②③④⑤ / 対象価値一致⑤⑥`
> - 読み方：この章は十四式の 1サイクル全体（① 認 から ⑤ 応 まで）を確定し、対象価値一致の ⑤提供価値・⑥差分 を実現する
> - `十四式−` は「該当なし」、`対象価値一致−` も同様
> - 機械パース：`/— 十四式([①-⑭−]+) \/ 対象価値一致([①-⑧−]+)/`
>
> **十四式の番号**：①認 ②思 ③鳴 ④示 ⑤応 ⑥衡 ⑦縁 ⑧家 ⑨幅 ⑩許 ⑪循 ⑫乱 ⑬絶 ⑭継
> **対象価値一致の軸番号**：①対象 ②タイミング ③状態・心理 ④課題 ⑤提供価値 ⑥差分 ⑦コスト ⑧継続理由
>
> **規範（全章共通）**：
> - 比喩・感情語・解釈語は使わない（CLAUDE.md 固定ルール）
> - 曖昧語禁止（「いい感じ」「やや」「明るい」「cute」「nice」「soft」等は HEX・数値・形状名詞に置換。NG リストは character.md §1.5-F 参照）
> - 設計書（character.md / spec.md / pose-library.md）と生成画像が矛盾した場合、**設計書を正**として character.md を強化し再生成（フェーズ7 検証ループ）
> - **画像生成 AI に渡すプロンプトは必ず character.md §1.5-G の [CHARACTER LOCK] 該当系統ブロックを先頭に貼る**
> - **章のラベルが付いていない要素は設計書として不完全**。読む AI はラベル無しの章を見つけたら作者に通知する

## 0.7 十四式マッピング — 入力→出力フィードバックループ定義

**1サイクルの入力**：character.md §1.5-G + pose-library.md 該当行 + 共通プロンプト + 個別指示
**1サイクルの出力**：LINE 審査通過する透過 PNG 1枚

| 式 | 1サイクル内の役割 | 本スタンプでの具体 |
|---|---|---|
| ① 認 | 入力プロンプトが観測された | ... |
| ② 思 | 入力が内部処理に入った（中央未成形） | ... |
| ③ 鳴 | 内部処理が確定した（中央成形） | ... |
| ④ 示 | 出力先への指向が決まった | ... |
| ⑤ 応 | 出力が実体化した | ... |
| ⑥ 衡 | 入→出 1サイクルが定常成立 | ... |
| ⑦ 縁 | 複数サイクルの相互参照 | ... |
| ⑧ 家 | 並列サイクルの持続単位 | ... |
| ⑨ 幅 | 1サイクルが崩壊しない入力範囲 | ... |
| ⑩ 許 | 正常とみなす入力範囲（バリデーション） | ... |
| ⑪ 循 | 許内＝再現可能・正常系 | ... |
| ⑫ 乱 | 許外（要エラーハンドリング） | ... |
| ⑬ 絶 | 幅の外（不成立） | ... |
| ⑭ 継 | 1サイクル終了→次サイクル開始 | ... |

### 通り道

入力 → ① → ② → ③ → ④ → ⑤ → 出力 → ⑭ → 次サイクルの ①

- ⑩ 許の範囲内で ⑥ 衡 が成立
- 許外（幅内）→ ⑫ 乱（フェーズ7 検証ループ）
- 幅外 → ⑬ 絶（設計やり直し）

## 0.5 対象価値一致サマリ

> <フェーズ0.5 で承認された1文>

| 軸 | 内容 |
|---|---|
| ① 対象 | ... |
| ② タイミング | ... |
| ③ 状態・心理 | ... |
| ④ 課題 | ... |
| ⑤ 提供価値 | ... |
| ⑥ 差分 | ... |
| ⑦ 許容価格 | ... |
| ⑧ 継続理由 | ... |

## 2. スタンプ用途配分 — 十四式④ / 対象価値一致②③
（フェーズ2の確定内容）

## 3. LINE 審査ガイドライン適合チェックリスト — 十四式⑨⑩ / 対象価値一致⑥
### 3-1. 規格仕様（生成時保証文言含む） — 十四式⑨ / 対象価値一致−
### 3-2. 著作権・肖像権 — 十四式⑨ / 対象価値一致−
### 3-3. ネガティブ指定（破綻パターン拡充版） — 十四式⑫ / 対象価値一致−
### 3-4. テキスト要件 — 十四式④⑩ / 対象価値一致⑤

## 4. 画風・カラー・統一仕様 — 十四式③⑩ / 対象価値一致⑥
（フェーズ4の確定内容）

## 5. 画像生成AI プロンプト集 — 十四式①②③④⑤ / 対象価値一致⑤⑥
### 5-1. 共通プロンプト（GPT-4o image / Nano Banana / SD の3系統） — 十四式①② / 対象価値一致−
### 5-2. 各スタンプ個別プロンプト（#01 〜 #N、3系統） — 十四式④⑤ / 対象価値一致②⑤
### 5-2.5. テキスト描画ルール — 十四式④⑩ / 対象価値一致⑤
### 5-3. メイン画像・トークタブ画像 — 十四式④⑤ / 対象価値一致⑥
### 5-4. キャラシート（検収用）プロンプト — 十四式⑥⑦ / 対象価値一致−

## 6. 制作・申請チェックリスト — 十四式⑤⑥ / 対象価値一致−
（フェーズ6の固定チェックリスト）

## 7. 検証ループ運用ルール — 十四式⑫⑭ / 対象価値一致−
（フェーズ7の概要：再生成要求時の挙動）
```

### pose-library.md

```markdown
# <スタンプ名> — ポーズライブラリ（pose-library.md）

## 規約（フェーズ1.6-A）
（固定フィールド定義表）

## 全N枚テーブル（フェーズ1.6-B）
| # | カテゴリ | カメラ | 体向き | 視線 | 表情 | 口 | 眉 | 腕 | 小道具 | 台詞 | 台詞処理 |
| ... |
```

---

## 量産モードのための追加ルール

- **連続生成要求**：「もう1セット作って」「次のスタンプ」等。前回の `~/stamp-specs/<前スタンプ名>/` を提示した上で、新しいスタンプ名を聞いてフェーズ0からやり直す。
- **同名スタンプの再生成要求**：既存ディレクトリがあれば上書き可否を確認。
- **テンプレ再利用**：「前の○○と同じキャラで別シリーズ」と言われたら、既存 character.md を Read して流用し、spec.md と pose-library.md のみ新規ヒアリング。
- **思い通り出ない要求**：フェーズ7に直行（新規ヒアリングしない）。

---

## 曖昧語チェックの自動実行

設計書を書き終えるたびに以下を Bash で実行：

```bash
grep -inE '(かわいい|やや|大きめ|明るい|暗い|いい感じ|適度|ほどよく|なんとなく|cute|nice|soft|a bit|kind of|somewhat|lovely|pretty|charming|slight|gentle)' ~/stamp-specs/<スタンプ名>/*.md
```

**ヒットがあれば Edit で全置換**するまで完成扱いにしない（CHARACTER LOCK 内の NG リスト記述自体を除く）。

---

## 完成後フック：レビュー確認（必須・スキップ禁止）

`~/stamp-specs/<スタンプ名>/` 配下の3ファイル（character.md / spec.md / pose-library.md）を Write で書き出し、曖昧語チェックも通過した**直後**、AskUserQuestion を1回呼ぶ。

- question：「設計書をレビューしますか？」
- header：「レビュー」
- multiSelect：false
- options：
  - `はい、いまレビューする` — description: "出力された3ファイルを一緒に見直し、章ごとに修正点を差し戻す"
  - `いいえ、自分で読む` — description: "ディレクトリパスだけ提示して終了。後で自分で開く"
  - `要点だけ要約して` — description: "全文ではなく対象価値一致の必須1文と CHARACTER LOCK 核項目（色HEX・パーツ・比率）だけを抜粋して読み上げる"

応答に応じて：
- **はい、いまレビューする** → character.md → spec.md → pose-library.md の順に Read で読み直し、章ごと（**spec.md 冒頭の十四式マッピング／対象価値一致サマリ**／CHARACTER LOCK／用途配分／審査ガイドライン／カラー・統一仕様／プロンプト集／pose-library 規約・全Nテーブル）に「OK か / 差し戻すか」を AskUserQuestion で確認。差し戻しが出たら該当フェーズに戻して修正し、Write 後に再びこのフックを呼ぶ。
- **いいえ、自分で読む** → `~/stamp-specs/<スタンプ名>/` の絶対パスと3ファイル名だけ提示して終了。
- **要点だけ要約して** → spec.md 本文最先頭2セクション（十四式マッピング／対象価値一致サマリ）と CHARACTER LOCK の色HEX・パーツ・比率を抜粋出力して終了。

前置き禁止。書き出し直後（曖昧語チェック通過後）に即フックを呼ぶ。

---

## 呼び出し時の最初の発話

このスキルが起動したら、まず一文で「LINE スタンプ設計書を対話で作ります。まずスタンプ名と、キャラ画像の有無から確認します」と述べ、即 AskUserQuestion を呼ぶ。前置き長文は不要。

ただし、ユーザーが「思い通りに出ない」「違う」「色が違う」「キャラが違う」「もう1回」等と言って起動した場合は、「検証ループに入ります。生成画像のパスと、どこが想定と違うかを教えてください」と述べてフェーズ7に直行する。

---

**somoMCP system version: v2.0.0 (2026-05-12)** — Apple Docs MCP 統合 / Foundation Models 採用判定マトリクス / App Store 審査ゲート 14 番号別チェックリスト / 設計書完成後の 12 段階実装ロードマップ / API リファレンス スナップショット必須記録
