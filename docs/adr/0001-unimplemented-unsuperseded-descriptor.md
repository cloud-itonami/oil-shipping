# ADR-0001 — oil-shipping は未実装かつ未 supersede の descriptor である

- **Status**: accepted
- **Date**: 2026-08-09
- **Scope**: `cloud-itonami/oil-shipping`（superproject `com-junkawasaki/root` の west project）

## Context

この repo は 2026-05-21 に `etzhayyimcojp/20-actors` から切り出された actor
snapshot で、2026-07-18 の rescue PR #1 で cljc の gate が加わった。それ以降
README が無く、**何が在って何が無いかを述べる場所が repo 内に存在しなかった。**

隣の `oil-refining` が同時期に同じ形の文書を得ているが、**そちらの結論をここに
転写すると誤る**。実測した差は次のとおり。

## Decision

**この repo を「superseded な descriptor snapshot」ではなく「未実装かつ未 supersede
の descriptor + gate」として記述する。** 根拠（すべて 2026-08-09 実測、手順は
`docs/operator-quickstart.md`）:

1. **後継が居ない。** `cloud-itonami` 配下の `manifest.edn` で `:actor/supersedes` を
   宣言しているのは `kamado`（`["oil-refining"]`）と `business-manager`（自分の
   legacy manifest を指す散文）の 2 件だけで、**`oil-shipping` を名指す行は無い**。
   `MIGRATION-NOTES.md` も無い（7 本中 `oil-refining` のみが持つ）。
2. **gate は在り、deny-by-default は実際に効く。** 17 cell / 7 gate。attestation を
   6 つ与えても `:blocked` / `effects=0`、7 つで全 17 cell が `:ready`。
3. **実装・実データ・グラフ・server は無い。** `cell-plan` が返すのは
   `{:op :mst/put-record …}` という data であり、実行する者は repo 内に居ない。
4. **この repo はフリートの hub である。**
   `com.etzhayyim.apps.oilShipping.cargo` は 39 manifest 中で最も購読者が多い
   oil 系 collection（5 件）だが、**`OilCargo` / `Ship` / `OilTerminal` / `flowsTo` を
   書く者は 1 本も存在しない**。欠落は 1 actor 分ではなく hub 分である。
5. **名乗っている risk 機能に実体が無い。** `Dark Fleet Risk` / AIS / 制裁 /
   STS / 海峡名は Cypher に 0 件。risk query は `s.riskLevel` と
   `t.terminal_type='chokepoint'` の読み出しのみで、それを立てる者も居ない。
6. **DID が 2 通りあり、manifest と gate が名乗る方が解決しない。**
   live の `_meta.primaryLexicon` は gate 側の綴りを支持している。

## 併せて記録する — pin の遅れが gate を隠していた

計測時、`oil-*` 7 本のうち **3 本**（`oil-upstream` / `oil-trading` /
`oil-shipping`）の west pin が upstream `main` の 2 commit 手前を指しており、
その 2 commit が gate を入れた rescue PR だった（`ahead=2 behind=0` の純粋な
fast-forward）。**checkout を読む計測は、この 3 本の substrate を実際より低く
測っていた。** 「repo に X が無い」と書く前に pin の鮮度を確かめる必要がある、
という一般則の実例。この repo の pin は本 ADR と同時に解消した。残る 2 本は
未処理（手順は `docs/operator-quickstart.md` 手順 0）。

## Consequences

- **`oil-refining` の「後継 kamado へ行け」をこの repo に読み替えてはならない。**
  移り先が無いので、ここでの選択肢は「実装する」か「明示的に retire する」で
  あって「移る」ではない。
- **retire するなら hub であることを先に解く。** 4 兄弟がこの repo の `cargo`
  collection を購読しており、黙って畳むと 4 本の購読が宙に浮く。
- **実装するなら、必要なものの大半は repo の外にある。** 書き手・scheduler・
  server・attestation 発行者・lexicon の決定・risk のデータモデル（README 末尾）。
- gate は `do not extend` のような制約を知らない。7 つ揃えば 17 cell すべてが
  `:ready` になる —— **宣言を強制する仕組みはコードのどこにも無い。**

## Alternatives considered

- **`oil-refining` と同じ「superseded」記述にする** —— 却下。`supersedes` を
  宣言する者が実在しない。片側の類推で退役を宣言すると、hub を待っている 4 本に
  対して誤った信号を出す。
- **7 本ぶんまとめて 1 つの文書にする** —— 却下。実測すると nanoid・購読数・
  エッジ型・gate の有無・後継の有無がそれぞれ違い、共通文書は必ずどれかについて
  嘘になる（この ADR 自体が、転写しかけて実測で止めた記録である）。
- **`vitest` を入れて `.ts` テストを走るようにする** —— 却下（別スコープ）。
  この ADR は「何が在るか」を述べるもので、無いものを足す変更はしていない。
