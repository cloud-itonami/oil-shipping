# oil-shipping

**原油・製品タンカーの航路、STS 移送、海峡エクスポージャ、ダークフリート risk を
扱うと宣言した actor の descriptor と、その書き込みを止める deny-by-default gate。
タンカーの実データも、それを読むグラフも、ここには無い。実装ではない。**

**そして、この actor には後継が居ない。** 兄弟の `oil-refining` は
`MIGRATION-NOTES.md` を持ち `cloud-itonami/kamado`（竈）が
`:actor/supersedes ["oil-refining"]` と機械可読に宣言しているが、**`oil-shipping`
を名指す `supersedes` 行はフリートのどこにも無く、`MIGRATION-NOTES.md` も無い**
（2026-08-09 実測）。**「実装されていない」と「退役した」は違う。ここは前者である**
—— 移り先が無いので、`oil-refining/README.md` の「kamado へ行け」をここに読み替えない。

`oil-*` は 7 本ある。うち 6 本（`upstream` / `midstream` / `refining` / `trading` /
**`shipping`** / `distribution`）が segment の実体を扱う側で、`oil-coverage` だけが
それらを上から測る meta actor。**この repo は 6 本のうち、原油と製品を海上で運ぶ
区間**を担当すると宣言している。

| | ここにあるか |
|---|---|
| actor が**何を名乗り、何を要求し、どの pipeline を持つと宣言しているか** | **ある**（`actor-manifest.jsonld` 8,535 B / `.well-known/did.json` 730 B） |
| **gate**（attestation が 7 つ揃わなければ effect を 1 つも出さない判断） | **ある**（`src/oil_shipping/murakumo.cljc`、233 行 / 9,156 B、17 cell） |
| タンカーを数えるグラフ、cron を撃つ scheduler、XRPC を受ける server | **無い** |
| 船・貨物・ターミナルの実データ | **無い** |
| **risk のデータモデル**（AIS 航跡・制裁リスト・STS 移送・海峡の地理） | **無い**（名前と散文にしか存在しない。後述） |

**ここには動くサービスは無い。** `cell-plan` が返すのは「書くとしたら何をどこに
書くか」という**計画**であって、書き込みそのものではない。`:effects` は
`{:op :mst/put-record ...}` という data であり、それを実行する者はこの repo に居ない。

経緯は [docs/adr/0001-unimplemented-unsuperseded-descriptor.md](docs/adr/0001-unimplemented-unsuperseded-descriptor.md)。
手順は [docs/operator-quickstart.md](docs/operator-quickstart.md)。

## この repo はフリートの hub であり、hub は何も生産しない

`com.etzhayyim.apps.oilShipping.cargo` は **cloud-itonami の 39 manifest 全体で
最も購読者の多い oil 系 collection** である（5 件。次点は
`oilRefining.refinery` の 3 件）。待っているのは `oil-distribution` /
`oil-midstream` / `oil-refining` / `oil-trading` の 4 兄弟と、自分自身。

そして **`OilCargo` を書く者は 39 本のどこにも居ない**:

| ラベル | 書く者 | 読む者 |
|---|---|---|
| `OilCargo` | **0** | `oil-shipping` 5 |
| `Ship` | **0** | `oil-shipping` 1 / `vessel-actor` 10 |
| `OilTerminal` | **0** | `oil-midstream` 4 / `oil-shipping` 1 |
| `flowsTo`（エッジ） | **0** | `oil-shipping` 2 / `oil-midstream` 1 |

**したがって欠けているのは 1 本ぶんの実装ではなく hub の実装である** ——
4 本の兄弟が、誰も書かないノードを唯一読む actor の出力を待っている。

⚠ **この census は末尾を固定した正規表現で数えている**（`\([a-z]+:Ship([^A-Za-z]|$)`）。
末尾を固定しないと `vin-actor` の `MERGE (sc:ShipmentCohort` が「`Ship` を書いて
いる」という偽陽性になる。逆にラベル形だけでは、ラベル名を**文字列として**扱う
`oil-coverage`（`UNWIND [… 'OilCargo' …] AS lbl`）が偽陰性になる。**両方向に
間違えるので、両方出して差を見ること**（`docs/operator-quickstart.md` 手順 4）。

## 「Dark Fleet Risk」を名乗るが、risk 判定は 1 行も無い

manifest は `Dark Fleet Risk` / `AIS anomaly and sanctions-sensitive fleet
monitoring` / `STS transfer tracking` / `OFAC Sanctions` / Hormuz・Suez・
Bab el-Mandeb・Malacca・Bosporus を名乗る。**Cypher に出てくるのは 0 件**である
（すべて `displayName` / `description` / `convoSystemPrompt` / `complianceFrameworks`
という散文と分類ラベルの側にある）。

risk に触る query は 1 本だけで、判定ではなく**判定済みプロパティの読み出し**:

```
MATCH (s:Ship) WHERE s.riskLevel IN ['high', 'critical'] …
MATCH (t:OilTerminal) WHERE t.terminal_type = 'chokepoint' …
```

`riskLevel` を付ける者も `terminal_type = 'chokepoint'` を立てる者もフリートに
居ない（上の表）。**海峡の名前は 1 つもデータになっていない。** gate には
`ststransfer` cell が在るが、STS に触る query は無い。

## 形（兄弟との差）

```
oil-upstream      actors=4 pipelines=8 subs=5 nanoid=01lupstr
oil-midstream     actors=4 pipelines=8 subs=5 nanoid=01lm1dst
oil-refining      actors=4 pipelines=8 subs=5 nanoid=01lr3f1n
oil-trading       actors=4 pipelines=8 subs=4 nanoid=01ltrad3
oil-shipping      actors=4 pipelines=8 subs=6 nanoid=01l5h1p0
oil-distribution  actors=4 pipelines=8 subs=4 nanoid=01ld1str
oil-coverage      actors=6 pipelines=5 subs=12 nanoid=011c0v3r
```

**購読が最多の 6 件で、handler は 1 件**（他 6 本も declared≫handlers=1、
`oil-coverage` は 12 宣言して handler ゼロ）。しかも handler が付いているのは
自分の `cargo` / `route` / `stsTransfer` ではなく、他 actor の
`com.etzhayyim.apps.vessel.portCall` である。

エッジ型を持つのは 7 本中 3 本で、`flowsTo` は `oil-midstream` と共有している
（`oil-upstream` は `feeds`、残り 4 本はエッジ無し）。

## 2 つの DID が食い違い、名乗っている方が解決しない

| | 綴り | 解決 |
|---|---|---|
| `actor-manifest.jsonld` の `@id` と gate の `actor-did` | `did:web:oil-shipping.etzhayyim.com` | **しない**（A レコード無し、`curl` は `000`） |
| `.well-known/did.json` の `id` | `did:web:etzhayyim.com:actor:oil-shipping` | **する**（`200`） |

repo の `did.json` は live の写しでもない（**5 か所ずれる**: 鍵スイート
`ed25519-2020` vs `jws-2020` / `alsoKnownAs` 4 件 vs 空 / `_meta` と
`verificationMethod` の有無 / PDS が `pds.etzhayyim.com` vs `pds.aozora.app` /
2 つ目の service が `AozoraAppView` vs libp2p `AtprotoXrpc`）。**repo が指す PDS は
落ちていて（530）、live が指す方が生きている（200）。**

live の `_meta.primaryLexicon` は `com.etzhayyim.oil-shipping` ——
manifest の `com.etzhayyim.apps.oilShipping.*` ではなく **gate 側の綴り**を支持している。
`live` の PDS に record は 1 件も観測できないが、**存在しない DID でも同じ応答が
返る**ので「未登録」の証拠にはならない（quickstart 手順 8）。

## 検証されていないもの

`actor-manifest.test.ts` の 11 個の `it(` は**一度も実行されていない** ——
vitest を import しているが `package.json` も `node_modules` もこの repo に無い。

## pin が遅れていると、この gate は「無い」ことになる

2026-08-09 時点で `oil-*` 7 本のうち 3 本（`oil-upstream` / `oil-trading` /
**この repo**）の west pin が upstream `main` の 2 commit 手前を指しており、その
2 commit がまさに gate を入れた rescue PR だった（純粋な fast-forward、
`ahead=2 behind=0`）。**遅れた checkout は、既に着地している仕事を「無い」と測る。**
この repo の pin は解消済み。残る 2 本の手順は quickstart 手順 0。

## この repo を動かしたい場合に、外に要るもの

1. `OilCargo` / `Ship` / `OilTerminal` / `flowsTo` を**書く**者
2. cron を撃つ scheduler と XRPC を受ける server（`runtime: k8s-langserver`）
3. 7 つの attestation を発行する主体（無ければ gate は永久に `:blocked`）
4. `com.etzhayyim.apps.oilShipping.*` と `com.etzhayyim.oil-shipping.*` の
   どちらを lexicon の正とするかの決定
5. **risk のデータモデルそのもの**

いずれもこの repo は持っていないし、持っていると主張してもいない。
