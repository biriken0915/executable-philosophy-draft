# 動く哲学書 — Executable Philosophy (Draft)

> **「形式化とは、判断を機械に委ねることではなく、判断の条件を機械に渡す作業である」**

本リポジトリは、ローカル計算環境（RTX 3060 12GB / ホストRAM 128GB）上で稼働させた推論エンジンとオントロジーパイプラインにより、約4,000万トークン相当の生成を経て出力された「動く哲学書（Executable Philosophy）」の一次ドラフト（約47万字 ≒ 295ページ相当）の記録である。

推敲・校正を経ていない「源泉かけ流し」の未加工データであり、文脈の飛躍、実験的命題、および意図的に壊されたコードを含む。
目的は学術的鑑賞や完成された方法論の提示ではなく、決定論的システム（Neo4j / Z3 / OPA / LangGraph）と哲学概念の接続実験における「たたき台」の物理的固定化（タイムスタンプ）にある。

---

## 本書の構成

- **序章**: [00_preface.md](book/front/00_preface.md) — なぜ「実行可能な」哲学書なのか
- **第一部 構造**
  - 第1章: [ch01.md](book/merged/ch01.md) — 状況と暗黙知
  - 第2章: [ch02.md](book/merged/ch02.md) — 知識の移動と権力
- **第二部 目的**
  - 第3章: [ch03.md](book/merged/ch03.md) — 目的構造と障礙
  - 第4章: [ch04.md](book/merged/ch04.md) — 契約と反証可能性
- **第三部 判断**
  - 第5章: [ch05.md](book/merged/ch05.md) — 勘定（Reckoning）と判断（Judgment）
- **第四部 防腐**
  - 第6章: [ch06.md](book/merged/ch06.md) — 意味の腐敗と境界防御
- **第五部 実行と人間**
  - 第7章: [ch07.md](book/merged/ch07.md) — 実線パイプラインと実践知
  - 第8章: [ch08.md](book/merged/ch08.md) — プロレタリア化の自己批判と知の奪還
- **終章・付録**
  - 終章: [90_epilogue.md](book/back/90_epilogue.md) — 現場の九十日計画と本書の落日
  - 付録: [91_appendixA.md](book/back/91_appendixA.md) / [92_appendixB.md](book/back/92_appendixB.md)

※ 全文を一括で参照する場合は [FULL.md](book/FULL.md) を参照。

---

## コードと検証環境

本書のコードブロックは、以下の環境においてCIおよびローカルコンテナ上で検証されることを前提に設計されている。

- Neo4j 5.x (Cypher)
- Python 3.11 / Pydantic v2
- Z3-Solver (Theorem Prover)
- Open Policy Agent (Rego)
- pySHACL / LangGraph / FastAPI

```cypher
// book-charter — 本書の中心命題を、本書の語彙で登記する
CREATE (t:Treatise {title: '動く哲学書',
        thesis: '形式化は判断の代行ではなく、判断の条件を機械に渡す作業である',
        declared_by: 'author', declared_at: datetime(),
        revalidation_path: 'MATCH (d:DesignDecision)-[:SERVES]->(m:Treatise) '
                         + 'WHERE NOT (d)-[:CONDITIONS]->(:JudgmentSlot) RETURN d'});
## ライセンス

本ドラフトのテキストおよび設計コードは、[MIT License](LICENSE) のもとで公開されている。