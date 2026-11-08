# 第3回 解答

## 演習

- 第1回（素のLLM）とエージェントの回答の違い：
  - 同じモデル（gemini-3.5-flash-lite）で「宿題の3レベル」を質問した。第1回の `app.ask` は3回とも「おそらく」「一般的に」と前置きして、レベル1～3の難易度別という一般論を答え、正解とは一致しなかった。
  - `app.agent` は search_glossary で用語集を検索してから、「基礎（講義内容の確認）・実装（自分でコードを書く）・発展（調査や新たな試行）、どれか1つで完了、原則として採点しない」と答え、`data/glossary.json` の `homework-levels` と一致した。最後に「第1回」と関係する回も示した（`logs/ex2_levels.txt`）。
- `--trace` で見えたツール呼び出し（ツール名と引数）：
  - 「3レベル」の質問：`search_glossary({'query': '3レベル'})`
  - 「LoRAとQLoRAの違いは？」：`search_glossary({'query': 'LoRA'})` の1回だけ。資料の例（LoRA と QLoRA を別々に検索）とは違った（`logs/ex3_lora.txt`）。宿題で同じ質問をもう1回実行すると、今度は LoRA と QLoRA を別々に検索した。
  - 「第2回の内容から確認問題を1問出して」：`make_quiz({'lecture': 2})` が呼ばれ、「「過学習」の意味は？」が出題された。答えは回答本文には書かれなかった（`logs/ex5_quiz.txt`）。
- 1つの質問での LLM 呼び出し回数（`requests`）：
  - 上の3つの質問とも `requests=2`（`tool_calls=1`）。入力トークンは 985 / 1149 / 918。
- 追加したツールと、docstring に書いた説明：
  - `make_quiz(lecture: int) -> str`。用語集から `lecture` が一致する用語を `random.choice` で1つ選び、`{"question": "「○○」の意味は？", "answer": 定義}` を返す。用語がない回は、用語がある回の一覧を付けたメッセージを返す（`make_quiz(0)` →「第0回の用語は用語集にありません。用語がある回: 1, 2, 3, 4, 6」）。
  - docstring：「指定した回（1始まりの整数）の用語から1つを選び、確認問題と答えを返す。「第2回の確認問題を出して」「第3回の復習クイズがほしい」のように、復習用の問題を求められたときに使う。lecture には回の番号を入れる。」
  - `--list-tools` で make_quiz と docstring が表示され、`uv run pytest -q` は 15 passed（`logs/list_tools.txt`、`logs/pytest.txt`）。

## 宿題

- 取り組んだレベル：基礎
- やったこと：
  - 質問「LoRAとQLoRAの違いは？」を `--trace` 付きで、次の3条件で各2回実行した。変えるのは一度に1か所だけで、条件ごとに `git checkout -- app/` で元に戻した。
    - A（元のまま）：run1 は演習の `ex3_lora.txt`、run2 は `hw_A_baseline_run2.txt`
    - B（INSTRUCTIONS から「用語や講座に固有のことを答える前に、必ず search_glossary で用語集を確認する。」の行を削除）：`hw_B_diff.txt`、`hw_B_no_must_search_run1/2.txt`
    - C（search_glossary の docstring の「1つだけ入れる」を「まとめて入れる」に変更）：`hw_C_diff.txt`、`hw_C_query_bundle_run1/2.txt`
  - モデルは gemini-3.5-flash-lite（既定値）。どの条件も make_quiz を登録した状態で実行した。レート制限を避けるため実行の間に20秒待ち、429 / 503 は出なかった。
- 結果：

| 条件 | run | `[tool call]` | tool_calls | requests | 入力トークン |
|---|---|---|---|---|---|
| A 元のまま | 1 | `search_glossary({'query': 'LoRA'})` | 1 | 2 | 1149 |
| A 元のまま | 2 | `search_glossary({'query': 'LoRA'})`, `search_glossary({'query': 'QLoRA'})` | 2 | 2 | 1398 |
| B 「必ず検索」を削除 | 1 | `search_glossary({'query': 'LoRA'})`, `search_glossary({'query': 'QLoRA'})` | 2 | 2 | 1350 |
| B 「必ず検索」を削除 | 2 | `search_glossary({'query': 'LoRA'})`, `search_glossary({'query': 'QLoRA'})` | 2 | 2 | 1350 |
| C 「まとめて入れる」 | 1 | `search_glossary({'query': 'LoRA QLoRA'})` | 1 | 2 | 1087 |
| C 「まとめて入れる」 | 2 | `search_glossary({'query': 'LoRA QLoRA'})` | 1 | 2 | 1087 |

  - 変わったこと：C では2回とも、用語をまとめた `'LoRA QLoRA'` で1回だけ検索した。入力トークンは 1087 で、6回の中で最も少なかった。
  - 変わらなかったこと：B で「必ず search_glossary で確認する」を消しても、2回とも答える前に検索した。入力トークンは A run2（同じ呼び方）の 1398 から 1350 に減った。すべての実行で `requests=2` で、回答の最後に「第2回」が付いた。回答の要点（LoRA は重みを固定して低ランク行列だけ学習、QLoRA はベースモデルを4ビットなどに量子化して LoRA で学習）は6回とも同じで、書き方だけが変わった。
  - 揺れたこと：元のままの A でも、run1 は LoRA の1回だけ、run2 は LoRA と QLoRA の2回と、検索の回数が変わった。
- 気づき・疑問：
  - B で検索が消えなかったのは、search_glossary の docstring にも「答える前に使うこと」と書かれていて、同じ指示が2か所にあるためだと考える（推測。両方消す条件は試していない）。モデルのふるまいは INSTRUCTIONS だけでなくツールの説明文にも左右されると実感した。
  - C のクエリ `'LoRA QLoRA'` は、`_normalize` で空白が消えて `loraqlora` になる。コードを読むと、名前がクエリに含まれる用語（lora、qlora）も部分一致で拾うので、今回は偶然うまくいったと考える。ただし `--trace` の `[tool return]` は120字で切れるので、QLoRA が結果に含まれていたかはログからは確かめられていない。別の組み合わせでは片方しか当たらないこともあり得る。
  - A run1 が LoRA の1回で済んだのも、「LoRA」が「QLoRA」に部分一致するため、最初の検索で両方の定義が返った可能性がある（これも推測）。
  - tool_calls が2でも requests は2のままだった。LLM が1回の応答で2つの呼び出しをまとめて依頼しているように見える。一方で、ツールの結果は次の入力に載るので、入力トークンは検索が増えるほど増えた（1087 → 1149 → 1398）。
  - 限界：各条件2回だけで、元のままの条件でも呼び方が揺れたので、B や C の違いが変更のせいだとは言い切れない。回答の評価は目視による。
  - AIツール：実行の操作と下書きに Claude を使用し、内容はログと照らして確認した。
