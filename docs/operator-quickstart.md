# operator quickstart — app-projection-operator

**build も deploy も起動もしない。この repo にはコードが無い**（`README.md` §1）。
operator にできる作業は 1 つで、それは**保管している 8 個の記録が壊れていないことを
検査する**ことである。この文書はその手順である。

下の手順は 2026-08-17 (UTC) に、`cloud-itonami/main`（`8a139e2`）から切った
worktree で実走した結果であり、掲載している値は**そのとき実際に出た値**である。

実測環境: macOS (darwin 25.3.0) / node **v26.3.0** / nbb **v1.4.210** / python3 の PyYAML **6.0.3**。

## 0. 先に知っておくこと —— stderr に出るノイズ

このワークスペースでは `git` が次を stderr に吐くことがある:

```
error: could not read IPC response
```

これは `~/.gitconfig` の `core.fsmonitor = true`（**このマシンの設定であって、この repo の
問題ではない**）に由来する。**exit code は 0 で、標準出力は正しい**。実測で確認済み:

```bash
git ls-files >/dev/null 2>/tmp/noise.txt; echo "exit=$?"; cat /tmp/noise.txt
```
```
exit=0
error: could not read IPC response
```

以降の手順では、**このメッセージが出ても合格判定に影響しない**。判定は各手順の
標準出力と exit code で行う。

## 1. 記録がパースできるか（EDN 2 本）

```bash
kbb --backend sci -e '(require (quote ["fs" :as fs]) (quote [clojure.edn :as edn]))
(doseq [f ["README.edn" "migration.edn"]]
  (let [forms (edn/read-string (str "[" (fs/readFileSync f "utf8") "]"))]
    (when-not (= 1 (count forms))
      (throw (js/Error. (str f ": expected exactly 1 top-level form, got " (count forms)))))
    (println f "→ 1 form," (count (keys (first forms))) "keys")))'
```

実測（exit 0）:

```
README.edn → 1 form, 5 keys
migration.edn → 1 form, 7 keys
```

⚠ **ファイル全体を `[` `]` で包んでいるのには理由がある。包まないとこの検査は無意味になる。**
`edn/read-string` は**先頭の 1 フォームしか読まず、残りを黙って捨てる**ので、
`README.edn` の末尾に壊れたテキストを足しても素通りする。実際、最初にこの検査を
素朴に書いたとき、末尾破壊の変異 2 種が**どちらも合格した**。包めばリーダは
全バイトを消費させられ、フォーム数を数えれば「余計なフォームが増えた」も捕まる。

## 2. `PROJECT.jsonld` が JSON として読めるか

```bash
node -e 'const j=JSON.parse(require("fs").readFileSync("PROJECT.jsonld","utf8"));
console.log("@id        :",j["@id"]);
console.log("url        :",j.url);
console.log("components :",j.component.map(c=>c.name).join(", "))'
```

実測（exit 0）:

```
@id        : urn:etzhayyim:project:etzhayyim-project-projection-operator
url        : https://po.etzhayyim.com
components : projection-operator-mcp, projection-manager-mcp, project-mail-ingest-resend
```

⚠ ここで出る `url` は**名前解決しない**（`README.md` §3）。この手順が確かめているのは
**JSON として妥当か**だけで、宣言先が生きているかではない。

## 3. `migration-plan-manager.yaml` が YAML として読めるか

```bash
python3 -c 'import yaml
d=yaml.safe_load(open("appview/migration-plan-manager.yaml"))
print("project  :",d["project"])
print("target   :",d["strategy"]["target"])
print("services :",[(s["name"],s["status"]) for s in d["services"]])'
```

実測（exit 0）:

```
project  : etzhayyim-project-projection-manager
target   : spinkube
services : [('projection-manager-pm7k3x9n', 'implemented')]
```

⚠ `implemented` は**この repo の中で裏付けられない**（`README.md` §4）。

## 4. 在庫

```bash
git ls-files | wc -l | tr -d ' '
cat CLAUDE.md NOTICE OWNERS PROJECT.jsonld \
    appview/README.md appview/migration-plan-manager.yaml | wc -c | tr -d ' '
cat README.edn migration.edn | wc -c | tr -d ' '
```

実測: `10` / `16555` / `630`。内訳は 3 層である:

| 層 | ファイル | バイト |
|---|---|---|
| 保管対象（出所からそのまま） | 6 | **16,555** ← `migration.edn` の `:bytes` と一致 |
| 切り出し時の生成レコード（`README.edn` / `migration.edn`） | 2 | 630 |
| この文書と `README.md`（保管対象ではない） | 2 | 可変 |

⚠ **repo 全体のバイト総数はここに書かない。** この文書自身がその総数に含まれるので、
値を書き足した瞬間に値が変わる —— 自己言及で必ず陳腐化する数である。
上の 3 行のうち**意味があるのは第 1 層の 16,555 だけ**で、それは
`migration.edn` に記録された値であり、この文書を何度書き換えても動かない。

**保管対象の 6 ファイルだけは、増えても減っても変わってもいけない。**
それを検査するのが §6 である。第 3 層と違い、`README.md` やこの文書は
後から足してよい —— §6 はその区別を実際に付けられる
（§7 の表「無関係な文書を 1 個足す」の行）。

## 5. 来歴の照合（出所 repo の checkout が要る）

これが**この repo で一番強い検査**である。`migration.edn` が主張する出所ツリーの
ハッシュを、実際の `etzhayyim/root` に問い合わせて突き合わせる。

```bash
ETZ=<etzhayyim/root の checkout パス>          # 例: ~/github/com-junkawasaki/orgs/etzhayyim/root
REV=691c245da48f3acb11dd757218f189ff2482b1c8
SRC=60-apps/etzhayyim-project-projection-operator

echo "recorded : $(kbb --backend sci -e '(require (quote ["fs" :as fs]) (quote [clojure.edn :as edn])) (println (get-in (edn/read-string (fs/readFileSync "migration.edn" "utf8")) [:source :git-tree]))')"
echo "actual   : $(git -C $ETZ rev-parse $REV:$SRC)"
```

実測 —— **一致する**:

```
recorded : f93de9901bcea2a30fa1488725ed07ac6fb1ef65
actual   : f93de9901bcea2a30fa1488725ed07ac6fb1ef65
```

ファイル数とバイト数も同じく一致する:

```bash
git -C $ETZ ls-tree -r --name-only $REV:$SRC | wc -l | tr -d ' '   # → 6   （migration.edn の :tracked-files 6）
git -C $ETZ ls-tree -r $REV:$SRC | awk '{print $3}' \
  | while read o; do git -C $ETZ cat-file -s $o; done \
  | awk '{s+=$1} END {print s}'                                     # → 16555（migration.edn の :bytes 16555）
```

⚠ **出所 repo が手元に無いときは、この手順を「合格」と読んではならない。**
`git -C` は失敗して空文字列を返すので、`recorded` と素朴に比較する書き方だと
**「両方空でない」ことを確かめない限り、無検査が一致に見える。** 上のように
`actual` を印字して**目で 40 桁を見る**か、スクリプト化するなら
`[ -n "$actual" ]` を必ず先に置く。実測でこの穴を作りかけ、`ETZ` を存在しない
パスにする対照実験で潰した（出所不在のとき検査は FAIL になる、が正しい）。

## 6. 保管対象 6 ファイルの逐一照合

§5 はツリー全体のハッシュを 1 個突き合わせた。§6 は**ファイルごとに** blob OID を
突き合わせる。どのファイルが壊れたかまで分かり、`README.md` のような後から足した
文書とは区別される。

```bash
#!/bin/bash
set -u
ETZ=<etzhayyim/root の checkout パス>
REV=691c245da48f3acb11dd757218f189ff2482b1c8
SRC=60-apps/etzhayyim-project-projection-operator

n=0; bad=0
while IFS=$'\t' read -r meta path; do
  oid=$(echo "$meta" | awk '{print $3}')
  here=$(git hash-object "$path" 2>/dev/null || echo MISSING)
  n=$((n+1))
  if [ "$oid" = "$here" ]; then printf '  OK    %s\n' "$path"
  else printf '  DIFF  %s  src=%s here=%s\n' "$path" "$oid" "$here"; bad=$((bad+1)); fi
done < <(git -C "$ETZ" ls-tree -r "$REV:$SRC")

echo "checked=$n  mismatched=$bad"
[ "$n" -eq 6 ] || { echo "REFUSING: expected 6 custody files, saw $n"; exit 3; }
[ "$bad" -eq 0 ] || exit 1
```

実測（exit 0）:

```
  OK    CLAUDE.md
  OK    NOTICE
  OK    OWNERS
  OK    PROJECT.jsonld
  OK    appview/README.md
  OK    appview/migration-plan-manager.yaml
checked=6  mismatched=0
```

⚠ **`n -eq 6` の行が evidence floor である。** 出所 repo が手元に無いと `ls-tree` は
何も出さず、ループは 1 度も回らない —— `bad=0` のまま**「不一致ゼロ＝合格」に見える。**
件数を検査して `exit 3`（0 でも 1 でもない値 ＝「答えられなかった」）で終わることで、
**測れなかったことと、測って問題が無かったことを出力で区別する。**

## 7. この検査群が捕まえるもの / 捕まえないもの

上の手順が本当に判別するかを、**壊したコピーに対して実際に確かめた**。
1 変異ごとに、落ちるべき手順だけが落ちることを見ている:

| 壊し方 | 結果 |
|---|---|
| （無改変） | **どれも落ちない** |
| `README.edn` の末尾を途中で切る | §1 のみ落ちる |
| `migration.edn` の末尾にゴミ | §1 のみ落ちる |
| `README.edn` にフォームをもう 1 つ足す | §1 のみ落ちる |
| `PROJECT.jsonld` を JSON として壊す | §2 が落ちる |
| yaml を壊す | §3 が落ちる |
| `:git-tree` の 1 桁を変える | §5 のみ落ちる |
| `CLAUDE.md` に 1 バイト足す | §6 が `DIFF CLAUDE.md` / exit 1 |
| `NOTICE` から 1 バイト削る | §6 が `DIFF NOTICE` / exit 1 |
| yaml の `implemented` を `planned` に書き換える | §6 が `DIFF …yaml` / exit 1 |
| 保管対象を 1 個消す | §6 が `here=MISSING` / exit 1 |
| **無関係な文書を 1 個足す** | **どれも落ちない（正しい）** |
| 出所 repo を外す | §6 が `REFUSING` / **exit 3** |

先頭行と末尾から 2 行目が対照である。無改変で落ちないことを確かめていないなら
残りは「常に落ちる検査」と区別がつかず、文書を足して落ちるなら
この repo は文書を足せない repo になってしまう。**両方向を実際に見た。**

最終行は 3 つ目の対照で、**「落ちる」と「答えられない」を分けている**
（exit 1 と exit 3）。

**捕まえないもの:**

- **どの手順も内容の正しさを見ていない。** `CLAUDE.md` が実在しないコードを
  記述していること（`README.md` §1）は、これらの検査を全部通る。**保管されているのは
  正しさではなく、出所と同一であるという事実だけである。**
- **§5 / §6 は出所 repo の checkout が要る。** 手元に無いときは
  「合格」ではなく「未検査」であり、§6 はそれを exit 3 で申告する。
- **`README.edn` と `migration.edn` は出所に無いので §6 の対象外。**
  この 2 つを守るのは §1（構文）と §5（記録された値と実際の出所の一致）だけで、
  たとえば `:destination` を書き換えても、どの手順も落ちない。
