# 検証手順と結果

カスタムルールを操作する API は提供されていないため、ルールの作成・削除は UI 操作のみです。1本作成して結果を観測し、削除してから次に進みます。

Settings → Advanced Security → Dependabot rules

## ベースライン

2026-08-20 時点、全41件が `open`。

| manifest | ecosystem | scope | 件数 | パッケージ |
| --- | --- | --- | --- | --- |
| `docs/legacy-app/app/webroot/js/pkg-a/package-lock.json` | npm | runtime | 7 | lodash |
| `docs/legacy-app/app/webroot/js/pkg-b/package-lock.json` | npm | runtime | 2 | minimist |
| `packages/app/package-lock.json` | npm | runtime | 15 | node-forge |
| `packages/dev-tools/package-lock.json` | npm | development | 2 | qs |
| `packages/unpatched/package-lock.json` | npm | runtime | 6 | request ほか |
| `services/api/requirements.txt` | pip | runtime | 9 | PyYAML / Jinja2 |

軸ごとの内訳。

- severity: critical 6 / high 15 / medium 17 / low 3
- scope: runtime 39 / development 2
- ecosystem: npm 32 / pip 9
- relationship: direct 36 / transitive 5
- 修正版あり 40 / 修正版なし 1（#27 `request` GHSA-p8p7-x288-28g6）

## 先行検証: alert filters の構文（REST API）

auto-triage rules の Target alerts と、Dependabot alerts の絞り込みは同じ修飾子名を共有しています。ルール本体は API が無く手で作るしかありませんが、絞り込みのほうは `GET /repos/{owner}/{repo}/dependabot/alerts` のクエリパラメータで機械的に総当たりできます。

以下は全41件（`state=open`）に対する実測値です。

注意: これは alerts API の挙動であって、auto-triage rules の Target alerts 欄の挙動そのものではありません。同じ構文族ではあるものの、ルール欄の判定は UI 側で別途確認が必要です。

### 同一キーのカンマ区切りは OR

| クエリ | 件数 | 内訳 |
| --- | --- | --- |
| `manifest:pkg-a` | 7 | |
| `manifest:pkg-b` | 2 | |
| `manifest:pkg-a,pkg-b` | 9 | 7 + 2 |
| `severity:critical` | 6 | |
| `severity:high` | 15 | |
| `severity:critical,high` | 21 | 6 + 15 |
| `ecosystem:npm` | 32 | |
| `ecosystem:pip` | 9 | |
| `ecosystem:npm,pip` | 41 | 32 + 9 |
| `package:lodash,minimist` | 9 | 7 + 2 |
| `scope:development,runtime` | 41 | 2 + 39 |

すべて単一値の合計と一致するため、カンマ区切りは OR。

### 異なるキーをスペースで並べると AND

| クエリ | 件数 |
| --- | --- |
| `package:qs` | 3 |
| `package:qs scope:development` | 2 |
| `package:qs scope:runtime` | 1 |
| `manifest:pkg-a severity:critical` | 1 |
| `manifest:pkg-a ecosystem:pip` | 0 |
| `ecosystem:pip scope:development` | 0 |

`package:qs` の3件が scope で 2 + 1 に分かれ、矛盾する組み合わせは0件。異なるキーは AND。

### manifest はフルパス完全一致のみ

| クエリ | 件数 |
| --- | --- |
| `manifest:docs/legacy-app/**` | 0 |
| `manifest:docs/legacy-app/*` | 0 |
| `manifest:docs/legacy-app`（前方一致） | 0 |
| `manifest:no/such/path.json` | 0 |

ワイルドカードも前方一致も効きません。しかもエラーにはならず、HTTP 200 で0件が返ります。

### patch availability / EPSS / relationship

| クエリ | 件数 |
| --- | --- |
| `has:patch` | 40 |
| `epss_percentage:>0.01` | 18 |
| `epss_percentage:>=0.001` | 38 |
| `epss_percentage:0.0..0.01` | 20 |
| `relationship:direct` | 36 |
| `relationship:transitive` | 5 |

`has:patch` は修正版のない #27 を正しく除外。EPSS は比較演算子と `..` の範囲指定が両方使えます。`relationship` は auto-triage rules のドキュメントには指定可能なメタデータとして挙がっていませんが、alerts API では機能します。

### 不正な値の扱いが一貫していない

ここが一番の落とし穴です。すべて HTTP 200 が返り、バリデーションエラーにはなりません。

| クエリ | 件数 | 挙動 |
| --- | --- | --- |
| `severity:bogus` | 41 | 条件が無視され全件ヒット |
| `ecosystem:bogus` | 41 | 条件が無視され全件ヒット |
| `has:bogus` | 41 | 条件が無視され全件ヒット |
| `scope:bogus` | 0 | 0件 |
| `relationship:bogus` | 0 | 0件 |
| `epss_percentage:abc` | 0 | 0件 |

`severity` / `ecosystem` / `has` は値を打ち間違えると、絞り込みが消えて全件が対象になります。dismiss ルールでこれをやると意図の正反対（全アラートが対象）になるため、ルールを作った直後に必ず対象件数を確認する必要があります。

## 準備

GitHub presets を2本とも Disabled にします。有効なままだと、プリセット由来の `auto_dismissed` とカスタムルールの効果が区別できません。

- Dismiss low-impact alerts for development-scoped dependencies
- Dismiss package malware alerts

## 毎回使うコマンド

```bash
R=yuta-ishii-cm/dependabot-auto-triage-rules-test

# 観測: manifest ごとの state 内訳
gh api "repos/$R/dependabot/alerts?per_page=100" \
  | jq -r 'group_by(.dependency.manifest_path)
    | map({path: .[0].dependency.manifest_path,
           states: (map(.state) | group_by(.) | map("\(.[0]):\(length)") | join(" "))})
    | .[] | "\(.path)\t\(.states)"'

# 観測: auto_dismissed になった件数と中身
gh api "repos/$R/dependabot/alerts?per_page=100&state=auto_dismissed" \
  | jq -r 'length as $n | "auto_dismissed: \($n)件", (.[] | "  #\(.number)\t\(.dependency.manifest_path)\t\(.dependency.package.name)\t\(.security_advisory.severity)")'

# リセット: auto_dismissed を全部 open に戻す（次のサイクルの前に実行）
gh api "repos/$R/dependabot/alerts?per_page=100&state=auto_dismissed" -q '.[].number' \
  | while read -r n; do gh api -X PATCH "repos/$R/dependabot/alerts/$n" -f state=open --silent; done
```

## 検証サイクル

各サイクルは「ルール作成 → 観測 → 結果記入 → ルール削除 → リセット」の順です。アクションは特記がなければ `Dismiss alerts` → `Indefinitely`。

期待件数はベースライン（全41件 open）に対する値です。

### 1. manifest 複数値（検証 A / B / D）

```
manifest:docs/legacy-app/app/webroot/js/pkg-a/package-lock.json,docs/legacy-app/app/webroot/js/pkg-b/package-lock.json
```

期待: 9件（pkg-a 7 + pkg-b 2）が `auto_dismissed`、残り32件は `open` のまま。

- 9件なら A（遡及適用）と B（カンマ区切りが OR）が両方成立
- 7件または2件なら B が不成立（AND または先頭のみ採用）
- 0件なら A が不成立
- 対象外の manifest が減っていれば D が不成立

結果:

### 2. 手動 reopen 後の再クローズ（検証 C）

サイクル1のルールを残したまま、`auto_dismissed` になった1件を reopen します。

```bash
gh api -X PATCH "repos/$R/dependabot/alerts/<番号>" -f state=open
```

期待: 再び `auto_dismissed` に戻る。戻らなければ「手動で state を変えたアラートはルールの再適用対象外」と確定。

観測は時間を空けて複数回（直後 / 10分後 / 翌日）。

結果:

### 3. manifest ワイルドカード

```
manifest:docs/legacy-app/**
```

期待: 保存時に `manifest has an invalid value` でエラー。`*` 単体、前方一致（`manifest:docs/legacy-app`）も同様に試す。

結果:

### 4. severity 複数値

```
severity:critical,high
```

期待: 21件（critical 6 + high 15）。manifest をまたいで効くこと、カンマ区切りが OR であることをここでも確認。

結果:

### 5. scope

```
scope:development
```

期待: 2件（`packages/dev-tools` の qs のみ）。

結果:

### 6. ecosystem

```
ecosystem:pip
```

期待: 9件（`services/api/requirements.txt` のみ）。

結果:

### 7. package

```
package:qs
```

期待: 3件（`packages/dev-tools` の2件 + `packages/unpatched` の推移依存1件）。manifest と scope をまたぐこと。

あわせて大文字小文字の扱いも確認します。pip 側は同じライブラリが advisory によって `PyYAML` と `pyyaml`、`Jinja2` と `jinja2` に分かれているため、`package:pyyaml` が両方に当たるかを見れば判定できます。

結果:

### 8. 異なるキーの組み合わせ（AND か）

```
package:qs scope:development
```

期待: 2件。サイクル7の3件から推移依存の1件が落ちれば、異なるキーはスペース区切りで AND。

結果:

### 9. patch availability

```
has:patch
```

期待: 40件。修正版が存在しない #27（`request` / GHSA-p8p7-x288-28g6）だけが `open` で残る。

結果:

### 10. CVE ID

```
CVE-ID:CVE-2019-10744
```

期待: 1件（#1 lodash critical）。

結果:

### 11. GHSA ID

```
GHSA-ID:GHSA-p8p7-x288-28g6
```

期待: 1件（#27 request）。

結果:

### 12. CWE

```
cwe:CWE-1321
```

期待: 10件。ただし CWE は auto-triage rules のドキュメントには「指定できるメタデータ」として挙がっている一方、alert filters のリファレンスには載っていません。キー名が `cwe:` で合っているかを含めて確認します。通らなければ UI のサジェストで正しいキー名を確認してください。

結果:

### 13. EPSS Score

```
epss_percentage:>0.01
```

期待: 18件。`>=`、範囲指定（`0.0..0.01`）が使えるかもあわせて確認。

結果:

### 14. アクション: Until a patch is available

```
manifest:docs/legacy-app/app/webroot/js/pkg-a/package-lock.json
```

Rules: `Dismiss alerts` → `Until a patch is available`

期待: 7件が `auto_dismissed`。`Indefinitely` との差は dismissed_reason や再オープン条件に出るはずなので、以下も確認します。

```bash
gh api "repos/$R/dependabot/alerts?per_page=100&state=auto_dismissed" \
  | jq -r '.[] | "#\(.number)\t\(.dismissed_reason // "-")\t\(.dismissed_comment // "-")"'
```

結果:

### 15. アクション: Open a pull request

```
manifest:docs/legacy-app/app/webroot/js/pkg-b/package-lock.json
```

Rules: `Open a pull request`

期待: minimist の更新 PR が作られる。Dependabot security updates は無効化済みなので選択できるはず。選択肢がグレーアウトしていればその旨を記録。

結果:

## まとめ

| # | 条件 | 期待 | 結果 |
| --- | --- | --- | --- |
| 1 | manifest 複数値 | 9件 | |
| 2 | reopen 後の再適用 | 再 dismiss | |
| 3 | manifest ワイルドカード | エラー | |
| 4 | severity 複数値 | 21件 | |
| 5 | scope | 2件 | |
| 6 | ecosystem | 9件 | |
| 7 | package | 3件 | |
| 8 | 異なるキーの AND | 2件 | |
| 9 | has:patch | 40件 | |
| 10 | CVE ID | 1件 | |
| 11 | GHSA ID | 1件 | |
| 12 | CWE | 10件 | |
| 13 | EPSS Score | 18件 | |
| 14 | Until a patch is available | 7件 | |
| 15 | Open a pull request | PR 作成 | |
