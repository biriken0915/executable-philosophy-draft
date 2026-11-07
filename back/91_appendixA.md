# 付録A — 開発環境と表記規約

本書のコードは「実行可能な形式」で書かれている（→序章のコードの約束）。本付録は、その実行を可能にする環境の完全な設計書である。三つの節からなる——(1) 依存スタックのセットアップ（docker-compose 一発起動）、(2) 本書のコードの走らせ方（テスト・デモ・スコアカード）、(3) 本書全体の表記規約（コード中の省略・命名・相互参照の約束）。付録A を読む順番は自由だが、**第1章のコードを実行する前に本書き読完ておくことを推奨する**——環境の不一致は、本書の全コードの「動く」の定義を変えるからである。

## A.1 依存スタック

本書が使用する部品の一覧と、本書における役割（役割を書くのは、依存の導入動機を哲学から切断しないため——→4.2 の Why を書け、の適用）。

| 部品 | バージョン | 本書での役割 | 主要節 |
|---|---|---|---|
| Neo4j | 5.x (Community) | 状況グラフ・履歴・監査の本体 | 1.3, 2.4, 5.1 |
| Python | 3.11 | 実装言語全般 | 全章 |
| Pydantic | v2 | Candidate/契約/プロトコルの型 | 3.3, 5.4, 8.4 |
| owlready2 | 0.4x | TBox・記述論理・DL 質問 | 1.3, 3.3, 3.6 |
| z3-solver | 4.12x | 制約充足・飽和判定（整数+論理） | 3.5, 4.5 |
| OPA (Open Policy Agent) | 1.x | Rego ポリシー門 | 6.4 |
| conftest | 0.5x | Rego YAML テスト | 6.4 |
| pySHACL (+ rdflib) | 0.2x / 7.x | 形状検証（→6.5） | 6.5 |
| FastAPI | 0.1xx | 実線の入口 | 7.5, 7.6 |
| LangGraph + checkpointer (Postgres) | 0.2x | 編成・中断・履歴 | 7.4, 7.5 |
| Neo4j testcontainers | 対応版 | 統合テストの使い捨て DB | 7.6 |
| pytest | 8.x | 全検収 | 4.6, 7.6 |

```yaml
# docker-compose.yml — 本書の全体系の一発起動(tsurugi/ に置く)
services:
  neo4j:
    image: neo4j:5
    ports: ["7474:7474", "7687:7687"]
    environment:
      NEO4J_AUTH: neo4j/bookpass        # 本書デモ用。実運用では secret 管理(→4.1)
    volumes: ["./graph/schema.cypher:/data/import/schema.cypher"]
    healthcheck:                        # readiness=防腐の着弾(→7.6 の起動順序設計)
      test: ["CMD", "cypher-shell", "-u", "neo4j", "-p", "bookpass",
             "MATCH (n) RETURN count(n)"]
      interval: 5s
  opa:
    image: openpolicyagent/opa:latest
    command: ["opa", "run", "--server", "--addr", ":8181",
              "-e", "execphil/decision", "/policy"]   # policy/ を read-only mount
    volumes: ["./policy:/policy:ro"]
  postgres:                             # LangGraph checkpointer(→7.4)
    image: postgres:16
    environment: { POSTGRES_DB: cpn, POSTGRES_PASSWORD: bookpass }
  api:
    build: ./flow
    ports: ["8000:8000"]
    depends_on: { neo4j: {condition: service_healthy}, opa: {condition: service_started} }
```

Python 側は `requirements.txt` で固定する（本書の執筆時点での版範囲を上表に示す——未来の版での非互換は→A.3 の表記規約で吸収する設計にしてある）。要点を一つ、哲学の側から注ぐ。**本書の環境構成は、本書の主張の一部分である**——グラフ DB・ポリシーエンジン・形状検証・ステートマシン・LLM ラッパーが別々の部品として配置されていること自体が、「判断の条件を渡す」四つの器（皿・制約・時間・不服従）が**単一の巨大な部品に溶融してはいけない**という設計判断の実装である（→5.3 の司令塔化の防壁は、アーキテクチャの分離として環境に現れる。 opa と Neo4j と LangGraph が一つの進程中に同居する構成は、本書の体系では L2 級の変更として審査される——→2.6 級がアーキテクチャにも効く）。

```python
# graph/schema.cypher — Neo4j スキーマ制約(→4.6 の test_cq.py の前提・本書全域)
CREATE CONSTRAINT term_id      IF NOT EXISTS FOR (t:Term)     REQUIRE t.term_id IS UNIQUE;
CREATE CONSTRAINT goal_id      IF NOT EXISTS FOR (g:Goal)     REQUIRE g.goal_id IS UNIQUE;
CREATE CONSTRAINT action_id    IF NOT EXISTS FOR (a:Action)   REQUIRE a.action_id IS UNIQUE;
CREATE CONSTRAINT cand_id      IF NOT EXISTS FOR (c:Candidate) REQUIRE c.candidate_id IS UNIQUE;
CREATE CONSTRAINT je_id        IF NOT EXISTS FOR (j:JudgmentEvent) REQUIRE j.je_id IS UNIQUE;
CREATE CONSTRAINT exec_id      IF NOT EXISTS FOR (x:ActionExec)    REQUIRE x.exec_id IS UNIQUE;
CREATE INDEX je_time           IF NOT EXISTS FOR (j:JudgmentEvent) REQUIRE j.at IS FOR REQUIREMENT; // placeholder
```

最後の行は**わざと壊している**（構文誤り）。本書の表記規約（A.3）で、壊れたコードは注記つきで明示する慣例を定めてあるが、この一箇所は壊れたまま置いてある——理由を読む練習として。正解は `CREATE INDEX je_time IF NOT EXISTS FOR (j:JudgmentEvent) ON (j.at)` である。Neo4j 5 の索引構文と制約構言の混同は実際の運用で頻出する事故であり、本書の CI（後述の A.2）は schema.cypher に対して `test_schema_loads` でこれを捕まえる——**本書は壊れ方の実例も配Linesする**という方針の、序曲である。

## A.2 本書のコードの走らせ方

三つの実行モードがある。**検収（acceptance）**：本書の主張のテスト群。`pytest tests/` で全章の単体・統合テストが走る（→4.6 の test_cq.py、→6.4 の conftest YAML、→7.6 の test_safety_fall 六障礙、→4.6 の ontology_regression）。testcontainer の Neo4j/Postgres を使い、本書の全 Cypher が同じコンテナで通ることを CI が保証する。本書のコードの「実行可能」の定義は、この CI の緑である——環境依存の「私の環境では動きました」は、→4.4 の「証拠＝データ」の条約により緑として登記されない。

**デモ（demo）**：7.6 の10ステップ通し実演。`make demo` で compose 起動→seed 投入（tsurugi ontology v8・goal 木・障礙集・条約 v8）→テレメトリ1本 POST→ブラウザで共観 UI・why_action 面板・dissent 面板を見る。デモの seed データは本書の各章の例（`metallic-rattle`、stop-line、`vib=51` 等）をそのまま使っており、**読者が本書の例と diff を取れる**ようにしてある——本と環境の DbC（→4.4）である。

**計器（scorecard）**：4.6・7.5・8.1・終章の照会集は `scorecard/` に集めてあり、`make scorecard` で現在値のスナップショットが出る（本書の体系が「毎日回す計器」であることを、make ターゲットとして体現する）。スコアカードの照会はすべて読み取り専用で、グラフを書き換えない——**計器が被観測系に副作用を持つとき、計器は計器でなくなる**（→8.3 の測定される行為の自己適応の、DB 設計版。読み取り専用アカウントで強制する）。

LLM を使うコード（5.4・7.4 の `PureLlm`）は、デフォルトで**スタブ transport**（固定の mock 応答）で走る。本書の全テストは LLM なしで決定論的に再現可能である——この設計は 5.4 の「PureLlm・環境簿記」の直接の帰結であり、付録レベルでも守られている（CI に API key が要らないことが、再現性の物理的条件）。実 LLM を繋ぐ場合（`make demo LLM=local` 等）は、`LlmEnv` の model_ref 簿記が動くので、実測ログ（→5.4 repro_check）が履歴代数に載る。

## A.3 表記規約 — 本書のコードの読み方

本書のコードの全節に共通する約束。**其一、省略の表記**。`...` は本文で説明した内容を省略している（省略箇所の実装有無は、テストの範囲で決まる——省略箇所がテストされていない場合、それは本書が「設計は示した実装は残していない」箇所で、その一覧が A.2 の CI 網羅レポートに出る）。**其二、疑似関数の宣言**。`normalize_telemetry`, `fire_api`, `opa_client` 等の名前は、本書内でシグネチャと振る舞いが本文で定義される境界部品である（境界部品の入出力契約は必ず Pydantic 型または Cypher の返り値形で書かれる——契約なしの名前を本書は使わない）。**其三、Cypher のパラメータ**。`$param` は呼び出し側の dict 由来で、呼び出し側の型は Cypher 直前に Pydantic で宣言される（→4.2）。**其四、命名**。ノードラベルは単数大文字キャメル（`JudgmentEvent`）、関係は全大文字（`JUDGED_ON`）、性質は lower_snake_case、識別子は `book://`・`execphil://`・`amendment://` の名前空間をもつ（→2.4 の ID 設計の反映）。**其五、相互参照**。「→x.y」は本書の節参照で、代码中のコメント `# 5.4 の部品` も同じ参照体系である——コードと哲学の交叉参照は一つの記法で統一される。**其六、壊れたコード**。上記 schema.cypher のような故意的な壊しは、直後または直前の本文で「これは壊れている」と宣言される。宣言なしの壊れは、本書のバグである（→4.4 の不変条件——本書自身も自分の契約の下にある）。

文体・脚注の規約も置いておく。哲学文献の引用は（著者 西暦）の形（→例：（Stiegler 2015））で、原語表記は初出時のみ。各概念の初出箇所に ★ を付け、付録B の用語集エントリに対応させる（★→付録B の語条→該当節、の三連参照が用語の追跡経路になる）。数式は本書では使わない——制約はすべて Cypher・Rego・Pydantic・LTL の形式で書かれる（→3.3・6.4・4.2 の通り、本書の数式はコードである。この規約は、本書の中心命題の表記への浸透である——コードこそが本書の「数式」であり、その可読性はコメントと哲学節の役割である）。

最後に環境の**落日**を宣言する。本書の環境記述（版番号・構文）は E4 級の有効期限つき宣言である——**落日：本書出版から18か月**。期限到達時、付録A の更新か、本書の改版の審議槽に自動で載る（→6.6 の落日条項が、本書自身に適用される。本もまた、落日を持たなければ腐る——落日を持たない文書は、自分の腐敗を隠蔽する機構を欠いた文書である）。

## A.4 起動失敗の目録 — つまりの解読八件

本書の環境（→A.1）とコードを実際に走らせたとき、最初に当たる八件を、症状→原因→処方の順で inscribe する（事故の inscribe、→2.5 の簿記の伦理——本書自らが読者の障礙台帳を持つ）。

1. **neo4j 起動後に schema.cypher が落ちる** → 意図的破損行（→A.3 の規約）を直していない環境で本物の schema と取り違えている。確認：schema.cypher の先頭コメントに「本書用」の表示。本番で同じ構文で書かれた本物の CREATE INDEX は、Neo4j 5 の構文（`ON ... REQUIRE ... IS FOR` は新構文・索引種類指定）——version を 5.x に揃えると通る。
2. ** opa が policy を 0 件で起動し全 deny** → `policies/` の mount path 違い。opa run のログの loaded バイト数で判別（0 件の loaded は「空の条約で門が立った」状態——6.4 の default deny が正しく効いている証拠でもある）。
3. **conftest の rego テストが pass するが本番 gate とズレる** → rego の data 路径（test の input 構造）と API の請求構造が二重定義。処方：rego の schema を openapi と一本の fixtures に統合（→4.4 の宣言と実体の突合）。
4. **pyshacl が Turtle 読み込みで落ちる** → BOM・改行コード。git の `.gitattributes` で `*.ttl text eol=lf` を宣言（環境の簿記、→A.1 の落日の前提として lock 文件と eol 宣言は必須）。
5. **LangGraph の interrupt() が resume しない** → checkpointer の thread_id 不一致（→7.6 の tid の供給元が ingest_api の相関规則と違う）。tid を全段で一本に固定する断言を pipeline 起動時に置く（→5.5 の guard の宣言簿記の pipeline 版）。
6. **PostgresSaver の接続失敗** → docker-compose の postgres 起動順（depends_on の healthcheck 無し）。compose の healthcheck を書く（→A.2 の三モードのうち demo モードのみ postgres 必須——acceptance では postgres 無しで動く設計を崩さないこと）。
7. **z3 が unknown を返す** → quantifier 爆発（→4.5 の穴三）。quantifier を Monormialize するか、unknown を UNKNOWN 登記に落とす（3.5 の三段構え——unknown を UNSAT と違う値として扱う代码を、本書の z3_gate が実施している）。
8. **ダッシュボードの照会が超時** → 計器の Cypher が全スキャン（→A.1 の neo4j 索引が本書の schema に無い）。計器のクエリ全部に時間窓（`at > …P90d`）を強制する——計器が世界を読むのに世界全体を読む必要はない。計器の時間窓の宣言は、計器の落日周期（→6.6 の計器の落日）と同一の設計である。

この八件は本書のコードの「正しさ」の一部である。**環境の失敗を inscribe しない書物の手順は、環境を未知に置く手順**——A 章の落日宣言（18か月）とこの目録は一対で、目録が延びない環境の説明書は既に腐っている（6.6 腐敗の文書版）。

### 実行練習

1. A.1 の compose を自分のマシンで起動し、`schema.cypher` の故意の壊しを CI が捕まえるところまで確認せよ（test_schema_loads を1本書く）。その後、あなたの既存プロジェクトのスキーマ定義に同じ検査を移植し、**今の你的のプロジェクトのスキーマが、起動時に検証されているか**を「されている/されていない」の二択+証拠（テストファイル名）で答えよ。
2. A.3 の規約三（Cypher のパラメータは Pydantic 宣言経由）を、あなたが書いたことのあるクエリ1本に適用して refactor せよ（文字列展開をやめ、型宣言＋パラメータ化する）。refactor 前後で、そのクエリの「契約」（→4.4）が何によって検査されるようになったかを1行で書け。
3. 本書の落日（出版+18か月）に倣い、あなたの主要な設計文書・ランブック・架構ドキュメントの一覧に「落日日」列を持つ表を作れ（落日未定なら未来の日付を必ず入れる）。3か月ごとに落日チェックの審議槽（→6.6）を1つ設定し、期限切れ文書の扱い（更新/廃棄/降格）の級（→2.6）を決めよ。この表が1週間後に空なら、あなたの組織は文書を「持つ」が「生かさない」側である——その診断から始めよ。
