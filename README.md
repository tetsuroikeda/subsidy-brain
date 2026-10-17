# subsidy-brain

[![ci](https://github.com/tetsuroikeda/subsidy-brain/actions/workflows/demo.yml/badge.svg)](https://github.com/tetsuroikeda/subsidy-brain/actions)

**補助金の名前を1行渡すと、公募要領に沿った申請書の下書き（Word）を作るプロトタイプです。**

> 2026年5月に作った試作品で、それ以降は更新していません。同梱のサンプルは、すべて架空の事業者データです。

---

## できること

たとえば「持続化補助金 第19回」と渡すと、1つのコマンドで次の流れを最後まで進めます。

1. **公募要領を探す**：補助金の名前から公式サイトを Web 検索し、公募要領（PDF）と様式ファイル（Word・Excel）を取得する。
2. **申請書の形を決める**：公募要領を読み、何節に分けるか、各節の字数の上限、必要なグラフや表、加点項目を整理する。
3. **本文を書く**：事業者のデータ（`demo/sample_company.yaml`）をもとに、Claude が各節の本文と図表を作る。
4. **様式に記入する**：公式の様式ファイルが手に入れば、罫線や書式を崩さずに、決められた欄へ本文を入れる。手に入らなければ「DRAFT」と明記した下書きを作る。
5. **見直す**：6つの観点で自己採点し、目標点に届かなければ、点数の低い節から書き直して採点し直す。

できあがるもの：申請書の下書き（.docx）、経費明細（.xlsx、ある場合）、実行の記録（manifest.json）。

---

## 試し方

Python 3.11 以上と [uv](https://docs.astral.sh/uv/) が必要です。

```bash
git clone https://github.com/tetsuroikeda/subsidy-brain && cd subsidy-brain
uv sync --extra dev
uv run pytest -ra                                       # テスト（23件）
uv run python demo/run_natural_demo.py "持続化補助金 第19回"
```

最後のコマンドで `demo/output/` に申請書の下書きができます。

### 動かし方は2通り

| | オフライン（API キーなし） | 実モデル（`--live`） |
|---|---|---|
| 公募要領の取得 | しない（汎用の構成を使う） | Web 検索で探して取得する |
| 本文 | あらかじめ用意した固定の文章 | Claude が書く |
| 用途 | 流れを確かめる、テスト・CI | 実際の下書きを作る |
| 必要なもの | なし | `ANTHROPIC_API_KEY`（費用がかかる） |

```bash
ANTHROPIC_API_KEY=sk-ant-... \
  uv run python demo/run_natural_demo.py --live "持続化補助金 第19回"
```

オフラインで実行した例（2026-10-17）：14節、4,718字（目標 10,326字）、自己採点 84/100（目標 85 に未達。書き直し4回で 79→84）、docx 約88KB。

---

## 仕組み

| 段階 | すること | 主なファイル |
|---|---|---|
| 探す | 補助金名から公式サイトと様式ファイルを探し、取得して保存する | `agents/subsidy_discoverer.py`、`agents/guideline_fetcher.py` |
| 読む | 公募要領から、節の構成・字数・図表・加点項目を取り出す | `agents/profile_synthesizer.py` |
| 書く | 事業者データと公募要領から本文を作り、様式に記入する | `agents/story_builder.py`、`tools/official_form_filler.py`、`tools/xlsx_filler.py` |
| 直す | 自己採点し、弱い節を書き直す | `tools/adoption_estimator.py`、`tools/refinement_loop.py` |

### 設計で気をつけたこと

- **補助金ごとのデータを持たない**：字数や必須の節を手で書いたデータは持たず、毎回その場で公募要領を読む。公募要領にない数字を推測で埋めないためです。
- **公式様式を壊さない**：Word は見出しを目印に、決められた欄だけに本文を入れる。Excel はファイルの中の文字の部分だけを書き換え、グラフや書式、名前付きの範囲はそのまま残す。
- **「公式」と「DRAFT」を混同しない**：公式の様式が手に入らないときは、DRAFT と明記した下書きにする。
- **決まった形で受け取る**：Claude からはスキーマに沿った JSON で受け取り（tool use）、Pydantic で型を確かめる。
- **記録はファイルで残す**：実行の記録や採点の履歴は、ローカルの JSONL ファイルに残す。データベースは使わない。

### 自己採点について

次の6つの観点で、0〜100点の目安を出します。採択されるかどうかの予測ではありません。

自社固有のデータが入っているか／課題と施策が対応しているか／加点項目を使っているか／具体的な数字があるか／字数が範囲に収まっているか／図表がそろっているか

---

## できないこと・注意

- 出てくるのは**下書き**です。内容と数字は、提出前に必ず人が確認してください。
- 公募要領や様式の取得は Web 検索に頼っているので、補助金によっては見つからないことがあります。
- 既定のモデル名など、設定は2026年5月時点のものです。

---

## フォルダ構成

```
subsidy-brain/
├── agents/     # 探す・読む・書く・確かめる各段階の処理
├── tools/      # 様式への記入、自己採点、書き直し、Web 検索、記録など
├── schemas/    # Pydantic のデータ定義
├── config/     # 設定とログ
├── demo/       # 実行用のスクリプトと架空の事業者データ
├── docs/       # 設計の説明（architecture.md）
├── presets/    # 空（必要なときだけ、補助金ごとの設定を置く）
├── templates/  # 空（取得した様式や下書きのひな形が実行時に入る）
└── tests/      # テスト（23件）
```

## テスト

`uv run pytest -ra` で23件のテストが動きます。CI（GitHub Actions）では、lint（ruff）・テスト・オフラインの実行を確かめています。

| ファイル | 件数 | 確かめること |
|---|---|---|
| `test_natural_pipeline.py` | 10 | 公募要領の整理、キャッシュ、名前から下書きまでの一連の流れ、補助金ごとに違う構成になること |
| `test_bonus_items.py` | 6 | 補助金ごとの加点項目と、その判定・記載 |
| `test_official_form_filler.py` | 3 | 様式の全部の欄が埋まること、元の文言が残ること、見出しの表記ゆれへの対応 |
| `test_operations.py` | 4 | API キーがないときの動き、費用の集計、3つの補助金の並行処理、ログ |

---

## English summary

subsidy-brain is a prototype (May 2026) that drafts a Japanese SME grant application from a single line: the name of the grant. It looks up the call guidelines, works out the required sections and character limits, drafts each section with Claude, fills the official Word/Excel form without breaking its formatting, then self-scores the draft and rewrites the weakest sections. It ships with fictional company data only. Without an API key it runs end to end with a fixed mock; `--live` uses Claude and web search. The self-score is a heuristic, not a prediction of selection. Not updated since May 2026.

## ライセンス

MIT（[LICENSE](LICENSE)）
