# operator quickstart

**この repo で「動かせる」ものは 1 つだけ**（手順 5 の gate）。残りは、宣言と現実が
どれだけずれているかを **自分の端末で確かめる**ための手順である。

**先に読むこと: この actor は superseded ではない**（手順 1）。兄弟の `oil-refining`
には後継 `kamado` が居るが、**この repo には移り先が無い**。「実装されていない」と
「退役した」は違う。ここは前者である。

所要 5 分。必要なのは `git` / `jq` / `curl` / `dig` / `nbb`。
下に貼ってある出力はすべて 2026-08-09 に実際に実行した結果の写しで、手打ちではない。
**手順 0〜4 は `..` / `../../..` を引く** ので、`oil-*` 7 本と **`kamado`** が
superproject の `orgs/cloud-itonami/` に checkout されている前提。

---

## 手順 0 — checkout が gate を持っているか確かめる（ここで 3 本が落ちていた）

```bash
ls src/oil_shipping/murakumo.kotoba && git log --oneline -1
```

```
src/oil_shipping/murakumo.kotoba
ddeb3a7 Merge pull request #1 from etzhayyim/rescue/murakumo-wip-20260718
```

（この写しは `ddeb3a7` 時点のもの。**`ddeb3a7` 以降ならどの commit でもよい** ——
この文書自体が入った commit を含む。1 行目の `src/…` が出ることだけが条件。）

**`src/` が無い場合、それは「gate が無い」ではなく「pin が遅れている」である。**
2026-08-09 の実測では、`oil-*` 7 本のうち **3 本**（`oil-upstream` / `oil-trading` /
**`oil-shipping`**）の west pin が upstream `main` の 2 commit 手前を指しており、
その 2 commit がまさに gate を入れた rescue PR だった:

```bash
(cd ../../.. && for n in oil-upstream oil-midstream oil-refining oil-trading \
                         oil-shipping oil-distribution oil-coverage; do
  printf "%-18s " $n
  pin=$(grep -A2 "name: $n\$" manifest/west.yml | grep revision | awk '{print $2}')
  head=$(gh api repos/cloud-itonami/$n/commits/main --jq '.sha')
  if [ "$pin" = "$head" ]; then echo "pin==main ${pin:0:7}"
  else gh api "repos/cloud-itonami/$n/compare/$pin...$head" \
         --jq '"pin='"${pin:0:7}"' main='"${head:0:7}"'  \(.status) ahead=\(.ahead_by) behind=\(.behind_by)"'
  fi
done)
```

```
oil-upstream       pin=7b2867d main=bed612b  ahead ahead=2 behind=0
oil-midstream      pin==main 46c3a08
oil-refining       pin==main 3e8a96e
oil-trading        pin=f072718 main=e6d246e  ahead ahead=2 behind=0
oil-shipping       pin=69945b7 main=ddeb3a7  ahead ahead=2 behind=0
oil-distribution   pin==main fb067ef
oil-coverage       pin==main 07ab2f5
```

`status: ahead` かつ `behind=0` は**純粋な fast-forward** である（force-push ではない）。
遅れた 3 本には upstream 側に確かに gate が在る:

```bash
for n in oil-upstream oil-trading oil-shipping; do
  printf "%-16s " $n
  gh api "repos/cloud-itonami/$n/git/trees/main?recursive=1" \
    --jq '[.tree[]|select(.type=="blob" and (.path|startswith("src/")))|.path+" ("+(.size|tostring)+"B)"]|join(", ")'
done
```

```
oil-upstream     src/oil_upstream/murakumo.cljc (8813B)
oil-trading      src/oil_trading/murakumo.cljc (8500B)
oil-shipping     src/oil_shipping/murakumo.kotoba (9156B)
```

**遅れた checkout は、既に着地している仕事を「無い」と測る。** 成熟度計測は
checkout を読むので、この 3 本は substrate 軸を実際より低く測られていた。

⚠ **上の表は、この文書が入る前の状態である。** この repo（`oil-shipping`）の pin は
本 commit と同時に前進させたので、今日同じコマンドを打つと `oil-shipping` は
`pin==main` に変わり、**`oil-upstream` と `oil-trading` の 2 本だけが遅れたまま**に
なる。2 本は別途直す（この文書の担当範囲外）。
pin の前進は west.yml を手で編集せずサーバ側 single-entry commit で行う
（skill `west-pin-advance`）:

```bash
kbb --backend sci --classpath ".:scripts/nbb_compat" scripts/west-pin-put.cljs oil-shipping <sha> --dry-run
kbb --backend sci --classpath ".:scripts/nbb_compat" scripts/west-pin-put.cljs oil-shipping <sha>
printf '%s\n' oil-shipping | xargs west update --fetch smart
```

（`xargs` は必須。`printf … | west update` は**引数ゼロ = 全 project 更新**になる。）

---

## 手順 1 — 後継が居ないことを確かめる（この repo で最初に確かめるべきこと）

兄弟の `oil-refining` は `MIGRATION-NOTES.md` を持ち、`kamado` が
`:actor/supersedes ["oil-refining"]` と機械可読に宣言している。**同じ問いを
`oil-shipping` に投げると、何も返らない。**

```bash
(cd .. && grep -rn 'actor/supersedes' --include='manifest.edn' . )
```

```
business-manager/manifest.edn:13: :actor/supersedes "actor-manifest.jsonld (legacy T1 MCP-Compose / RisingWave-Cypher; deprecated, removed after 1 R-cycle)"
kamado/manifest.edn:11: :actor/supersedes ["oil-refining"]
```

**`oil-shipping` を名指す行は無い。** `MIGRATION-NOTES.md` も無い:

```bash
(cd .. && for n in oil-upstream oil-midstream oil-refining oil-trading \
                   oil-shipping oil-distribution oil-coverage; do
  printf "%-18s MIGRATION-NOTES=%s\n" $n "$([ -f $n/MIGRATION-NOTES.md ] && echo yes || echo no)"
done)
```

```
oil-upstream       MIGRATION-NOTES=no
oil-midstream      MIGRATION-NOTES=no
oil-refining       MIGRATION-NOTES=yes
oil-trading        MIGRATION-NOTES=no
oil-shipping       MIGRATION-NOTES=no
oil-distribution   MIGRATION-NOTES=no
oil-coverage       MIGRATION-NOTES=no
```

**したがって `oil-refining/README.md` の「後継 kamado へ行け」をこの repo に
読み替えてはいけない。** ここは退役していない。実装されていないだけである。

---

## 手順 2 — descriptor の形を数える

README の表の数は、すべてこの 1 コマンドから出ている。

```bash
jq -r '{pipelines:(.pipelines|length), actors:(.actors|length),
        capabilities:(.capabilities|length),
        subscribeRepos:(.triggers.subscribeRepos.collections|length),
        nanoid:.nanoid}' actor-manifest.jsonld
```

```json
{
  "pipelines": 8,
  "actors": 4,
  "capabilities": 5,
  "subscribeRepos": 6,
  "nanoid": "01l5h1p0"
}
```

兄弟 6 本と比べる。**`actors` / `pipelines` は同じで、購読数が最も多いのがこの repo**
（`oil-coverage` だけ形からして別物）:

```bash
(cd .. && for n in oil-upstream oil-midstream oil-refining oil-trading \
                   oil-shipping oil-distribution oil-coverage; do
  printf "%-17s " $n
  jq -r '"actors=" + ((.actors|length)|tostring) +
         " pipelines=" + ((.pipelines|length)|tostring) +
         " subs=" + ((.triggers.subscribeRepos.collections|length)|tostring) +
         " nanoid=" + .nanoid' $n/actor-manifest.jsonld
done)
```

```
oil-upstream      actors=4 pipelines=8 subs=5 nanoid=01lupstr
oil-midstream     actors=4 pipelines=8 subs=5 nanoid=01lm1dst
oil-refining      actors=4 pipelines=8 subs=5 nanoid=01lr3f1n
oil-trading       actors=4 pipelines=8 subs=4 nanoid=01ltrad3
oil-shipping      actors=4 pipelines=8 subs=6 nanoid=01l5h1p0
oil-distribution  actors=4 pipelines=8 subs=4 nanoid=01ld1str
oil-coverage      actors=6 pipelines=5 subs=12 nanoid=011c0v3r
```

**6 件宣言して handler は 1 件**（7 本すべて同じ形で、`oil-coverage` だけ handler ゼロ）:

```bash
jq -r '.triggers.subscribeRepos.collections[]' actor-manifest.jsonld
jq -r '.pipelines[] | select(.trigger.type=="subscribeRepos")
       | "handler: " + ((.trigger.collections//[])|join(","))' actor-manifest.jsonld
```

```
com.etzhayyim.apps.oilShipping.cargo
com.etzhayyim.apps.oilShipping.route
com.etzhayyim.apps.oilShipping.stsTransfer
com.etzhayyim.apps.vessel.voyage
com.etzhayyim.apps.vessel.portCall
com.etzhayyim.apps.port.portCallEvent
handler: com.etzhayyim.apps.vessel.portCall
```

**自分の `cargo` / `route` / `stsTransfer` を購読しておきながら、handler が付いて
いるのは他 actor の `vessel.portCall` だけ**である。

---

## 手順 3 — この repo はフリートの hub である（そして hub は何も生産しない）

`com.etzhayyim.apps.oilShipping.cargo` は **フリート 39 本のなかで最も購読者が
多い oil 系 collection** である:

```bash
(cd .. && for f in */actor-manifest.jsonld; do
  jq -r '(.triggers.subscribeRepos.collections//[])[]' "$f" 2>/dev/null
done | grep -iE 'oil(Upstream|Midstream|Refining|Trading|Shipping|Distribution|Coverage)\.' \
  | sort | uniq -c | sort -rn | head -5)
```

```
   5 com.etzhayyim.apps.oilShipping.cargo
   3 com.etzhayyim.apps.oilRefining.refinery
   2 com.etzhayyim.apps.oilRefining.yieldSnapshot
   2 com.etzhayyim.apps.oilMidstream.flow
   1 com.etzhayyim.apps.oilUpstream.production
```

誰が待っているのか:

```bash
(cd .. && for f in */actor-manifest.jsonld; do
  n="${f%/actor-manifest.jsonld}"
  jq -e '(.triggers.subscribeRepos.collections//[])
         | index("com.etzhayyim.apps.oilShipping.cargo")' "$f" >/dev/null 2>&1 && echo "$n"
done)
```

```
oil-distribution
oil-midstream
oil-refining
oil-shipping
oil-trading
```

**4 本の兄弟がこの repo の cargo を待っている。** そして次の手順が示すとおり、
それを書く者はフリートのどこにも居ない。

---

## 手順 4 — 触るラベルと、それを書く者（居ない）

```bash
jq -r '.pipelines[] | .trigger.type + " " +
       (.trigger.cron // .trigger.nsid // ((.trigger.collections//[])|join(","))) +
       "  steps=" + ((.steps|length)|tostring)' actor-manifest.jsonld
```

```
cron 0 */8 * * *  steps=5
subscribeRepos com.etzhayyim.apps.vessel.portCall  steps=1
xrpc com.etzhayyim.apps.oilShipping.routing.getCargo  steps=1
xrpc com.etzhayyim.apps.oilShipping.routing.listCargoes  steps=1
xrpc com.etzhayyim.apps.oilShipping.routing.getRouteExposure  steps=1
xrpc com.etzhayyim.apps.oilShipping.routing.listChokepoints  steps=1
xrpc com.etzhayyim.apps.oilShipping.health  steps=1
cron 0 */6 * * *  steps=3
```

ノードラベルは 4 つ、**エッジ型が 1 つある**（`oil-refining` には 1 つも無い）:

```bash
jq -r '.pipelines[].steps[] | (.args.sql // .args.template // empty)' actor-manifest.jsonld \
  | grep -oE '\([a-z]+:[A-Za-z]+' | sort | uniq -c | sort -rn
jq -r '.pipelines[].steps[] | (.args.sql // .args.template // empty)' actor-manifest.jsonld \
  | grep -oE '\[[a-z]+:[A-Za-z]+' | sort | uniq -c | sort -rn
```

```
   5 (c:OilCargo
   1 (t:OilTerminal
   1 (s:Ship
   1 (c:ActorCoverageSnapshot
   2 [e:flowsTo
```

エッジ型を持つのはこの repo だけではない（**7 本中 3 本**、`flowsTo` は
`oil-midstream` と共有）:

```bash
(cd .. && for n in oil-upstream oil-midstream oil-refining oil-trading \
                   oil-shipping oil-distribution oil-coverage; do
  printf "%-17s " $n
  e=$(jq -r '.pipelines[].steps[] | (.args.sql // .args.template // empty)' $n/actor-manifest.jsonld \
      | grep -oE '\[[a-z]+:[A-Za-z]+' | sed 's/.*://' | sort -u | tr '\n' ',')
  echo "${e:-（無し）}"
done)
```

```
oil-upstream      feeds,
oil-midstream     constrainedBy,flowsTo,
oil-refining      （無し）
oil-trading       （無し）
oil-shipping      flowsTo,
oil-distribution  （無し）
oil-coverage      （無し）
```

### 書く者を数える — そして正規表現は両方向に間違える

**素朴な部分一致とラベル形の両方を出して、差を見ること。**

```bash
(cd ..
for LBL in OilCargo Ship OilTerminal; do
  echo "=== $LBL ==="
  for KIND in graph.write graph.query; do
    printf -- "--- %s ---\n" "$KIND"
    for f in */actor-manifest.jsonld; do
      n="${f%/actor-manifest.jsonld}"
      c=$(jq -r ".pipelines[]?.steps[]? | select(.fn==\"$KIND\") | (.args.sql // .args.template // empty)" "$f" 2>/dev/null \
          | grep -oE "\([a-z]+:$LBL([^A-Za-z]|\$)" | wc -l | tr -d ' ')
      [ "$c" != "0" ] && printf "  %-20s %s\n" "$n" "$c"
    done
  done
done)
```

```
=== OilCargo ===
--- graph.write ---
--- graph.query ---
  oil-shipping         5
=== Ship ===
--- graph.write ---
--- graph.query ---
  oil-shipping         1
  vessel-actor         10
=== OilTerminal ===
--- graph.write ---
--- graph.query ---
  oil-midstream        4
  oil-shipping         1
```

**3 ラベルとも writer はフリート 39 本のどこにも居ない。** `flowsTo` も同じ
（`oil-shipping` が 2 回、`oil-midstream` が 1 回読むだけで writes=0）。

**末尾を固定しない正規表現は偽陽性を出す。** `\([a-z]+:Ship` だけで数えると
`vin-actor` が「`Ship` を書いている」ように見えるが、当たっているのは別のラベルである:

```bash
(cd .. && jq -r '.pipelines[]?.steps[]? | select(.fn=="graph.write")
  | select((.args|tostring)|test("\\([a-z]+:Ship")) | .args.template' \
  vin-actor/actor-manifest.jsonld | grep -oE 'MERGE \([a-z]+:[A-Za-z]+' | sort -u)
```

```
MERGE (sc:ShipmentCohort
```

**ラベル形は逆に偽陰性を出す。** `oil-coverage` はラベル名を**文字列として**
数えており（`UNWIND [… 'OilCargo' …] AS lbl`）、上の正規表現には映らない:

```bash
(cd .. && jq -r '.pipelines[]?.steps[]? | select((.args|tostring)|test("OilCargo"))
  | .id + " :: " + ((.args.sql // .args.template)|.[0:110])' oil-coverage/actor-manifest.jsonld)
```

```
backboneCounts :: UNWIND ['OilCompany','OilField','OilPipeline','OilTerminal','Refinery','OilCargo','CrudeGrade','ProductGrade',
backbone :: UNWIND ['OilCompany','OilField','OilPipeline','OilTerminal','Refinery','OilCargo','CrudeGrade','ProductGrade',
```

---

## 手順 5 — 名乗っている risk 機能が Cypher に 1 行も無い

manifest は "Dark Fleet Risk" / "AIS anomaly and sanctions-sensitive fleet monitoring" /
"STS transfer tracking" / Hormuz・Suez・Malacca を名乗る。**どれも Cypher には無い。**

```bash
for term in AIS sanction dark Hormuz Malacca stsTransfer; do
  printf "%-14s cypher=%s  manifest全体=%s\n" "$term" \
    "$(jq -r '.pipelines[].steps[] | (.args.sql // .args.template // empty)' actor-manifest.jsonld | grep -c "$term")" \
    "$(grep -c "$term" actor-manifest.jsonld)"
done
```

```
AIS            cypher=0  manifest全体=1
sanction       cypher=0  manifest全体=4
dark           cypher=0  manifest全体=4
Hormuz         cypher=0  manifest全体=2
Malacca        cypher=0  manifest全体=2
stsTransfer    cypher=0  manifest全体=1
```

どこに居るのかを見る（**すべて散文か分類ラベル**で、query ではない）:

```bash
jq -r 'paths(scalars) as $p | [($p|join(".")), (getpath($p)|tostring)] | @tsv' \
  actor-manifest.jsonld | grep -iE 'AIS anomaly|sanction|dark|hormuz|malacca' | cut -c1-120
```

```
actors.2.description	Hormuz, Suez, Bab el-Mandeb, Malacca and Bosporus exposure.
actors.3.description	AIS anomaly and sanctions-sensitive fleet monitoring.
actors.3.displayName	Dark Fleet Risk
actors.3.path	risk:dark-fleet
convoSystemPrompt	You are the Oil Shipping agent. Track crude and product tanker flows, correlate cargoes with vessels a
description	Crude and product tanker routing, STS transfer tracking, chokepoint exposure, cargo load/discharge intellige
governance.complianceFrameworks.3	OFAC Sanctions
pipelines.0.steps.3.args.message	Oil shipping report.\n\nCargo stats:\n$cargoStats.rows\n\nRoute stats:\n$routeStats.row
profile.capabilities.4	dark-fleet-screening
profile.description	Tracks crude tanker, product tanker, LNG and LPG maritime flows. Links cargo, vessel, terminal, port
```

risk に触る query は 1 本だけで、**AIS でも制裁リストでもなく、あらかじめ計算済みの
プロパティを読むだけ**である:

```bash
jq -r '.pipelines[].steps[] | select(.id=="riskFlags") | .args.sql' actor-manifest.jsonld
jq -r '.pipelines[] | select((.trigger.nsid // "") | test("listChokepoints"))
       | .steps[0].args.sql' actor-manifest.jsonld
```

```
MATCH (s:Ship) WHERE s.riskLevel IN ['high', 'critical'] RETURN s.imoNumber AS imo, s.riskLevel AS risk, s.flagState AS flag LIMIT 20
MATCH (t:OilTerminal) WHERE t.terminal_type = 'chokepoint' RETURN t.vertex_id, t.repo, t.collection, t.status LIMIT 20
```

`riskLevel` を付ける者も、`terminal_type = 'chokepoint'` を立てる者も、フリートに
居ない（手順 4）。**海峡の名前は 1 つもデータになっていない。**

---

## 手順 6 — gate を実際に走らせる（この repo で唯一動くもの）

`/tmp/probe-shipping.cljs` を作る:

```clojure
(ns probe-shipping (:require [oil_shipping.murakumo :as m]))
(def all-gates (set m/common-gates))
(defn summarise [label atts]
  (let [p (m/cell-plan :getcargo {:attestations atts :request-id "probe-1"
                                  :computed-at "2026-08-09T00:00:00Z"})]
    (println (str label "  status=" (:status p) "  effects=" (count (:effects p))
                  "  missing=" (count (:missing-gates p))))))
(println "cells:" (count m/cell-specs) " gates:" (count m/common-gates))
(summarise "none      " #{})
(summarise "6-of-7    " (disj all-gates :kotoba-only-substrate-baseline))
(summarise "all 7     " all-gates)
(println "gate collection:" (m/collection "cargo"))
(println "actor-did:" m/actor-did)
(let [plans (m/all-cell-plans {:attestations #{}})]
  (println "blocked:" (count (filter #(= :blocked (:status %)) (vals plans))) "/" (count plans)
           " total effects:" (reduce + 0 (map #(count (:effects %)) (vals plans)))))
(let [plans (m/all-cell-plans {:attestations all-gates})]
  (println "all-7 ready:" (count (filter #(= :ready (:status %)) (vals plans))) "/" (count plans)
           " total effects:" (reduce + 0 (map #(count (:effects %)) (vals plans)))))
```

```bash
kbb --backend sci --classpath "src:/tmp" /tmp/probe-shipping.cljs
```

```
cells: 17  gates: 7
none        status=:blocked  effects=0  missing=7
6-of-7      status=:blocked  effects=0  missing=1
all 7       status=:ready  effects=1  missing=0
gate collection: com.etzhayyim.oil-shipping.cargo
actor-did: did:web:oil-shipping.etzhayyim.com
blocked: 17 / 17  total effects: 0
all-7 ready: 17 / 17  total effects: 17
```

**確かめるべきは「6 つ揃えても通らない」ことである** —— 部分的な attestation で
effect が 1 つでも出るなら、それは deny-by-default ではない。

`gate collection:` の綴りと manifest 側の綴りを見比べる。**交わりが無い**
（手順 2 の 6 件はすべて `com.etzhayyim.apps.*`）。17 cell 全部を出すと、XRPC の
メソッド名も他 actor の collection も自分の名前空間に付け替わっているのが見える:

```bash
cat > /tmp/probe-cols-shipping.cljs <<'EOF'
(ns probe-cols-shipping (:require [oil_shipping.murakumo :as m]))
(doseq [[k v] (sort-by key m/cell-specs)]
  (println (str (name k) "\t" (first (:collections v)))))
EOF
kbb --backend sci --classpath "src:/tmp" /tmp/probe-cols-shipping.cljs
```

```
cargo	com.etzhayyim.oil-shipping.cargo
domain-knowledge	com.etzhayyim.oil-shipping.domain-knowledge
getcargo	com.etzhayyim.oil-shipping.getcargo
getrouteexposure	com.etzhayyim.oil-shipping.getrouteexposure
health	com.etzhayyim.oil-shipping.health
koji	com.etzhayyim.oil-shipping.koji
kyumei	com.etzhayyim.oil-shipping.kyumei
listcargoes	com.etzhayyim.oil-shipping.listcargoes
listchokepoints	com.etzhayyim.oil-shipping.listchokepoints
portcall	com.etzhayyim.oil-shipping.portcall
portcallevent	com.etzhayyim.oil-shipping.portcallevent
route	com.etzhayyim.oil-shipping.route
shinka	com.etzhayyim.oil-shipping.shinka
shinkaevolution	com.etzhayyim.oil-shipping.shinkaevolution
shinkaknowledge	com.etzhayyim.oil-shipping.shinkaknowledge
ststransfer	com.etzhayyim.oil-shipping.ststransfer
voyage	com.etzhayyim.oil-shipping.voyage
```

cell が 17 ある理由も manifest から導ける:

```bash
jq -r '((.pipelines|map(select(.trigger.type=="xrpc"))|length) + (.requiredCollections|length)
        + (.requiredLoops|length) + (.triggers.subscribeRepos.collections|length))' actor-manifest.jsonld
```

```
17
```

**`stsTransfer` に cell が在ることに注意。** gate は「STS transfer を扱う予定の
場所」を持っているが、手順 5 のとおり STS に触る query は 1 本も無い。

---

## 手順 7 — 2 つの DID のうち、どちらが解決するか確かめる

この repo は自分を 2 通りに名乗る:

```bash
jq -r '.["@id"]' actor-manifest.jsonld       # manifest と gate が名乗る方
jq -r '.id' .well-known/did.json             # DID document が名乗る方
```

```
did:web:oil-shipping.etzhayyim.com
did:web:etzhayyim.com:actor:oil-shipping
```

```bash
dig +short oil-shipping.etzhayyim.com A          # ← 何も返らない
curl -s -o /dev/null -w '%{http_code}\n' https://oil-shipping.etzhayyim.com/.well-known/did.json
curl -s -o /dev/null -w '%{http_code}\n' https://etzhayyim.com/actor/oil-shipping/did.json
```

```
000
200
```

`000` は「HTTP status が無い」＝ 接続の前段で失敗した、という意味である。
**manifest と gate が名乗る方の DID が、この解決しない側。**

repo の did.json が live の写しでないことも見ておく（**5 か所ずれ、`diff` は exit 1**）:

```bash
curl -s https://etzhayyim.com/actor/oil-shipping/did.json > /tmp/live-did-shipping.json
diff <(jq -S . .well-known/did.json) <(jq -S . /tmp/live-did-shipping.json)
```

`ed25519-2020` vs `jws-2020` / `alsoKnownAs` 4 件 vs 空 / `_meta` と
`verificationMethod` の有無 / PDS の宛先 / 2 つ目の service が `AozoraAppView` か
`AtprotoXrpc`(libp2p) か。live 側の `_meta` はこう名乗る:

```bash
jq -rc '._meta' /tmp/live-did-shipping.json
```

```json
{"adr":["2605212030","2605241800","2606013800","2606014500"],"source":"kotoba","kind":"tier-b","status":"r0","glyph":"Oil Shipping & Tanker Intelligence","wasmCid":null,"execModel":"service","primaryLexicon":"com.etzhayyim.oil-shipping","note":"verificationMethod empty — on-chain ERC725 mirror pending; did:web trust root = TLS (no server-minted key, ADR-2605231525)"}
```

**`primaryLexicon` は gate 側の綴り**（`com.etzhayyim.oil-shipping`）であって
manifest の `apps.` 語彙ではない。宛先の生死:

```bash
for u in https://pds.etzhayyim.com/xrpc/_health https://pds.aozora.app/xrpc/_health; do
  printf "%-42s -> " $u; curl -s -o /dev/null -w '%{http_code}\n' --max-time 12 $u
done
```

```
https://pds.etzhayyim.com/xrpc/_health     -> 530
https://pds.aozora.app/xrpc/_health        -> 200
```

repo の did.json が指す方（`pds.etzhayyim.com`）が落ちていて、live が指す方
（`pds.aozora.app`）が生きている。

---

## 手順 8 — live の PDS に record があるか（そして、なぜこれが弱い証拠か）

```bash
DID="did:web:etzhayyim.com:actor:oil-shipping"
curl -s --max-time 15 "https://pds.aozora.app/xrpc/com.atproto.repo.describeRepo?repo=$DID"
```

```json
{"did":"did:web:etzhayyim.com:actor:oil-shipping","handle":"handle.invalid","collections":[],"handleIsCorrect":false}
```

**この 200 を「登録されている」と読んではいけない。** 存在しない DID でも同じ形が返る:

```bash
curl -s --max-time 15 "https://pds.aozora.app/xrpc/com.atproto.repo.describeRepo?repo=did:web:etzhayyim.com:actor:not-a-real-actor-xyz"
```

```json
{"did":"did:web:etzhayyim.com:actor:not-a-real-actor-xyz","handle":"handle.invalid","collections":[],"handleIsCorrect":false}
```

`listRecords` も同じで、manifest 語彙・gate 語彙のどちらでも `{"records":[]}`:

```bash
for c in com.etzhayyim.apps.oilShipping.cargo com.etzhayyim.oil-shipping.cargo; do
  printf "%-42s " $c
  curl -s --max-time 12 "https://pds.aozora.app/xrpc/com.atproto.repo.listRecords?repo=$DID&collection=$c"
  echo
done
```

```
com.etzhayyim.apps.oilShipping.cargo       {"records":[]}
com.etzhayyim.oil-shipping.cargo           {"records":[]}
```

言えるのは **「record は 1 件も観測できない」**までで、「登録済みだが空」と
「未登録」はこの endpoint では区別できない。

---

## 手順 9 — `.ts` テストが走らないことを確かめる

```bash
ls package.json node_modules
grep -c 'it(' actor-manifest.test.ts
```

```
ls: node_modules: No such file or directory
ls: package.json: No such file or directory
11
```

`actor-manifest.test.ts` は vitest を import しているが、その vitest がここには
無い。**この 11 の `it(` は一度も実行されていない。**

---

## ここから先に進みたい場合

`oil-refining` と違い、**移り先は無い**（手順 1）。この descriptor を動かすなら、
少なくとも次の 5 つが repo の外に要る:

1. `OilCargo` / `Ship` / `OilTerminal` と `flowsTo` を**書く**者
   （フリート 39 本のどこにも居ない、手順 4）。**4 本の兄弟がこの repo の cargo を
   待っているので、ここは 1 本ぶんの欠落ではなく hub の欠落である**（手順 3）
2. cron を撃つ scheduler と XRPC を受ける server（`runtime: k8s-langserver`）
3. 7 つの attestation を発行する主体（無ければ gate は永久に `:blocked`、手順 6）
4. `com.etzhayyim.apps.oilShipping.*` と `com.etzhayyim.oil-shipping.*` の
   どちらを lexicon の正とするかの決定（live の DID document は後者を支持、手順 7）
5. **risk のデータモデルそのもの** —— AIS 航跡、制裁リスト、STS transfer、
   海峡の地理。`riskLevel` と `terminal_type='chokepoint'` は「誰かが既に判定を
   終えている」ことを前提にした読み出しであって、判定そのものではない（手順 5）

いずれもこの repo は持っていないし、持っていると主張してもいない。
