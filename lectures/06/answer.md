# 第6回 解答

## 演習

<!-- [tool call] の行は、表示された行をそのまま貼り付けてください。終わらなかった STEP は、空欄のままでかまいません。 -->

- STEP 1 のツールの名前：`glossary_search_glossary`
- STEP 2 の [tool call]：`glossary_search_glossary({'query': '3レベル'})`（`[usage] 入力 1,343 tokens / 出力 151 tokens / モデル応答 2 回`）
- STEP 3 の [tool call]：`fetch_fetch({'url': 'https://modelcontextprotocol.io/'})`（`[usage] 入力 2,230 tokens / 出力 111 tokens / モデル応答 2 回`）
- STEP 4 の [tool call]：
  - `aws_aws___search_documentation({'search_phrase': 'Amazon S3 バージョニング 概要'})`
  - `aws_aws___search_documentation({'topics': ['general', 'reference_documentation'], 'search_phrase': 'What is S3 Versioning Amazon Simple Storage Service'})`
  - `[usage] 入力 11,361 tokens / 出力 154 tokens / モデル応答 3 回`（`--list-tools` で見えたツールは 7 個）
- 気づき・疑問：
  - ツール名は「設定ファイルのキー」「_」「サーバー側のツール名」の順につながる。AWS のツールはサーバー側の名前がもともと `aws___` で始まるので、`aws_aws___search_documentation` のように `aws` が重なって見えた
  - STEP 2 の呼び出しは、第3回（`search_glossary({'query': '3レベル'})`、requests 2回）と比べて、名前に `glossary_` が付いただけで、引数も回数も同じだった。モデルから見ると、関数ツールも MCP のツールも同じ形をしていることを確認できた
  - STEP 3 では、`INSTRUCTIONS` を変えなくても、URL を含む質問でモデルが自分で `fetch_fetch` を選んだ。ログを見ると、fetch サーバーはページを取得する前に robots.txt を確認していた。`https://modelcontextprotocol.io/` は 307 で別のページ（`/docs/2026-07-28/getting-started/intro.md`）に転送されていて、モデルが指定した URL と実際に本文を取得した URL は違っていた。アクセスしたのは自分のコンテナからで、モデルが決めた URL に手元から実際にアクセスしていることも分かった
  - STEP 4 では、1回目に日本語で「Amazon S3 バージョニング 概要」と検索して、バージョニングとは別の結果が返った。2回目は英語に変え、`topics` も指定して検索し直したため、モデル応答が3回になった
  - ツールは 1 → 2 → 7 個と増え、入力トークンは 1,343 → 2,230 → 11,361 と増えた。ただし STEP 3・4 の入力には、取得したページや検索結果、応答の回数の違いも含まれるので、説明文だけでどれだけ増えたかは分けられない
  - `--list-tools` で説明文を読むと、`fetch_fetch` には「もともとインターネットにアクセスできなかったが、このツールでアクセスできるようになった」という、モデルに向けた指示が書かれていた。AWS のツールにも「検索結果をそのまま信頼して答える」といった指示がある。他人が作ったサーバーを足すと、こうした指示もそのままモデルに渡るので、足す前に説明文を読んで確認したほうがよいと感じた

## 宿題

<!-- 基礎・実装・発展のどれか1つに取り組めば完了です。 -->

- 取り組んだレベル：基礎
- やったこと：`app/mcp_server.py` の TODO の下に `server.tool()(tools.get_lecture_summary)` を1行書き足した。エージェント側（`app/mcp_agent.py`）は変えていない。演習 STEP 3・4 で足した fetch と aws は設定に残したまま、`--list-tools`、`--trace "第2回では何を学びますか？"`、`uv run pytest -q` を実行した。
- 結果：
  - `git status`：宿題で変えたのは `app/mcp_server.py` だけ（差分は追加した1行）。ほかの変更は演習で変えた `app/mcp_servers.json` と `answer.md`
  - `--list-tools`：モデルに見えているツールが7個から8個になり、`glossary_get_lecture_summary` が増えた
  - `--trace` の [tool call]：`glossary_get_lecture_summary({'lecture': 2})`
  - `[usage] 入力 5,942 tokens / 出力 231 tokens / モデル応答 2 回`
  - 回答は第2回の要点（事前学習とファインチューニングの違い、SFT、PEFT と LoRA・QLoRA、Colab での LoRA 演習）と一致し、最後に「関係する回：第2回」と示された
  - `uv run pytest -q`：23 passed
- 気づき・疑問：
  - エージェントは起動のたびにサーバーへツールの一覧を問い合わせるので、サーバー側に1行足すだけで、エージェントのコードも設定も変えずにツールが増えた
  - ツールが8個あっても、「第2回」という回の質問には `search_glossary` ではなく `get_lecture_summary` が選ばれ、引数も `lecture: 2` と整数で渡された。docstring の「回の内容や関係を聞かれたときに使う」が、ほかのツールとの呼び分けに効いていると考えられる
  - 入力トークンは 5,942 で、STEP 2（ツール2個、1,343 tokens）より多い。AWS のツールの長い説明文と、返ってきた第2回の要点の両方が入力に入るためと考えられるが、どちらがどれだけ増やしたかはこの結果からは分けられない。fetch と aws を外して同じ質問をすれば確かめられそう
