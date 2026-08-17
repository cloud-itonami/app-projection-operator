# app-projection-operator

**この repo は記録（record）の保管庫であって、実行される projection operator ではない。**
`CLAUDE.md` が記述する MCP actor 群 —— `projection-manager-mcp`（34 tool）・
`projection-operator-mcp`・`po-ui` —— の**コードは 1 行もここに無い。**
ここに在るのは、`etzhayyim/root` から切り出された 6 個の記述ファイルと、
その切り出しを記録した 2 個の生成レコードだけである。

`CLAUDE.md` を読んで実装を探しに来た読み手が最初に必要とするのはこの事実なので、
名乗りの直後に置く。

| | |
|---|---|
| 保管している記録 | 8 ファイル / 17,185 バイト（**うち実行コード 0**） |
| そのほか | `README.md`（この文書）と `docs/operator-quickstart.md` |
| 種別 | `README.edn` の宣言は `:kind :standalone-app-artifact` |
| 出所 | `etzhayyim/root` の `60-apps/etzhayyim-project-projection-operator` |
| 出所 revision | `691c245da48f3acb11dd757218f189ff2482b1c8` |
| 出所 tree | `f93de9901bcea2a30fa1488725ed07ac6fb1ef65`（**照合済み** —— §2） |
| 宣言 URL | `https://po.etzhayyim.com`（**名前解決しない** —— §3） |
| nanoid | `po1x9k2m`（operator） / `pm7k3x9n`（manager） |

手順は `docs/operator-quickstart.md`。以下は 2026-08-17 (UTC) に**実際に測って**分かった
現在地であり、推測は含まない。測り方は各項に書いてある。

---

## 1. 読み手が最初に踏む地雷 —— `CLAUDE.md` は、ここに無いものを記述している

`CLAUDE.md` は次のディレクトリ構造と build 手順を書いている:

```
wasm/
├── projection-manager-mcp-component/
│   ├── src/app.ts                    # ~1400 lines, 34 MCP tools
│   └── wit/world.wit
├── projection-operator-mcp-component/
└── po-ui-po1x9k2m/
```

```bash
cd wasm/projection-manager-mcp-component
etzhayyim build
go vet ./...
```

**この `wasm/` は存在しない。** そして重要なのは、*切り出しのときに落ちた*のではなく、
**出所の revision にも最初から無かった**ということである。出所 revision の全ツリーを
検索して確認した:

```
691c245d の全パスのうち "projection-manager|projection-operator" に一致  → 6 件（下記の記録ファイルのみ）
691c245d の全パスのうち nanoid "pm7k3x9n|po1x9k2m" に一致                → 0 件
691c245d の wasm/ 配下で "projection" に一致                              → 0 件
（対照）691c245d に存在する wasm/ パスの総数                              → 91 件
```

最後の 1 行が effort floor である —— 検索は 91 件の `wasm/` パスを見つけられている。
つまり 0 件は「検索が壊れていた」のではなく**本当に無い**。

したがって `CLAUDE.md` の build 手順・34 tool 一覧・`go vet ./...` は、この repo に対しては
**踏めない**。読み物としての設計記録であって、操作手順ではない。

## 2. ここに在るもの —— そして、それは出所と正確に一致する

```
CLAUDE.md                              6,709   設計記述（§1 のとおり実装は伴わない）
PROJECT.jsonld                         8,062   JSON-LD の project 記録（DoDAF DM2 対応付き）
appview/README.md                        741   統合構成の説明
appview/migration-plan-manager.yaml      493   spinkube 移行計画（§4 参照）
NOTICE                                   513   Apache-2.0 + etzhayyim Charter Rider v3.1
OWNERS                                    37   @etzhayyim/platform
                                     ───────
                                      16,555   ← 出所からそのまま来た 6 ファイル
README.edn                               209   ┐ 切り出し時に生成された記録
migration.edn                            421   ┘（出所には無い）
                                     ───────
                                      17,185   = 記録 8 ファイルの総バイト
```

（この `README.md` と `docs/operator-quickstart.md` はこの 8 個には含まれない。
保管対象ではなく、保管対象について書いた文書である。）

`migration.edn` が主張する出所（`:tracked-files 6` / `:bytes 16555` / `:git-tree f93de990…`）は、
**3 つとも実際の出所ツリーと一致する**。照合は `docs/operator-quickstart.md` §5 で、
1 コマンドで再現できる。上の足し算（16,555 + 630 = 17,185）も §6 で検査できる。

つまりこの repo は**内容は薄いが、来歴は完全に閉じている**。薄さと不確かさは別のことである。

## 3. 宣言されている URL は、どれも名前解決しない

`PROJECT.jsonld` の `url`、`CLAUDE.md` の endpoint 表、`appview/README.md` の
「MCP エンドポイントは `po.etzhayyim.com` を正とします」は、すべて生きていない。
2026-08-17 (UTC) の実測:

| URL | 結果 |
|---|---|
| `https://po.etzhayyim.com` | **NXDOMAIN**（`Could not resolve host`） |
| `https://pm7k3x9n.etzhayyim.com/api/mcp` | **NXDOMAIN** |
| `https://po1x9k2m.etzhayyim.com/api/mcp` | **NXDOMAIN** |
| `https://etzhayyim.com` | `200`（対照 —— 親ドメインと DNS 経路は生きている） |

最後の行が対照である。`etzhayyim.com` が 200 を返す以上、上の 3 件は
「ネットワークが無い」のではなく**そのホストが存在しない**。

## 4. 内部の食い違い —— `status: implemented`

`appview/migration-plan-manager.yaml` は次を宣言している:

```yaml
services:
  - name: projection-manager-pm7k3x9n
    status: implemented
```

§1 と §3 のとおり、実装コードはこの repo にも出所 revision にも無く、endpoint も
解決しない。この `implemented` は**この repo の中で裏付けられない**。
値を書き換えていないのは、これが出所からそのまま来た記録であり（§2）、
書き換えると `migration.edn` の byte 照合が壊れるからである —— 記録の保管庫としては、
食い違いを黙って直すより、**食い違いとして可視にしておく方が正しい。**

## 5. いまはこの repo が唯一の保管者である

出所側では、この 6 ファイルは既に**消えている**。`etzhayyim/root` の現 `origin/main`
（`b9f6afbe`、全 27,183 ファイル、full 履歴）を測った結果:

```
origin/main で "projection-manager|projection-operator" に一致  → 0 件
origin/main の 60-apps/ 直下のエントリ                          → 1 件（etzhayyim-project-organism のみ）
691c245d は origin/main の祖先か                                → YES
```

`691c245d` が祖先であることを確かめてあるので、この 0 件は「別系統を見ていた」結果ではない。
出所から取り除かれたのであって、**この repo が失うと記録は他のどこにも残らない。**

## 6. この repo に対して意味のある作業

- **記録の完全性を検査する** —— `docs/operator-quickstart.md`。パース検査・来歴照合・
  バイト恒等式で、8 ファイルが改竄されていないことを確認できる。
- **`CLAUDE.md` の記述を実装する** —— そのときは §1 を書き換えること。
- **記録として畳む** —— 実装しないと決めるなら、`status: implemented`（§4）と
  死んだ URL（§3）を訂正した上で、そう宣言する。

**やってはいけないのは、`CLAUDE.md` を実装済みの説明として引用することである。**
34 tool の表は仕様であって、在庫ではない。
