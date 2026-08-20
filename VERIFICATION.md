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

# リセット: auto_dismissed と手動 dismissed を全部 open に戻す（次のサイクルの前に実行）
for s in auto_dismissed dismissed; do
  gh api "repos/$R/dependabot/alerts?per_page=100&state=$s" -q '.[].number' \
    | while read -r n; do gh api -X PATCH "repos/$R/dependabot/alerts/$n" -f state=open --silent; done
done

# リセットできたか確認（open: 41件 になれば OK）
gh api "repos/$R/dependabot/alerts?per_page=100" \
  | jq -r 'map(.state) | group_by(.) | map("\(.[0]): \(length)件") | .[]'
```

state を手で変えたアラートはルールの再適用対象から外れる可能性があります（それ自体がサイクル17の検証対象です）。もしそうだった場合、リセットを挟んだ後のサイクルは結果が信用できなくなるので、サイクル17を先に済ませて挙動を確定させてから残りに進むのが安全です。

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

## 検証サイクル（状態遷移）

ここからはルールの条件ではなく、アラートの state とルールの相互作用を見ます。「ルールがいつ評価されるのか」を切り分けるのが目的です。

時間がない場合はサイクル16〜18を優先してください。本番で詰まっている経路そのものです。

### 16. 手動 dismiss 済みのアラートにルールを作る

先に対象を手動 dismiss してからルールを作ります。

```bash
# pkg-a の #1 を手動 dismiss（本番と同じ理由・コメント）
gh api -X PATCH "repos/$R/dependabot/alerts/1" \
  -f state=dismissed -f dismissed_reason=not_used \
  -f dismissed_comment="検証用の手動 dismiss"

# 確認
gh api "repos/$R/dependabot/alerts/1" \
  -q '"\(.state)\treason=\(.dismissed_reason // "-")\tauto_dismissed_at=\(.auto_dismissed_at // "-")"'
```

この状態で `manifest:docs/legacy-app/app/webroot/js/pkg-a/package-lock.json` のルールを作成します。

期待: pkg-a の残り6件は `auto_dismissed` になる。#1 がどうなるかが焦点で、`dismissed` のまま据え置きか、`auto_dismissed` に上書きされるかを見ます。

結果:

### 17. 手動 dismiss → ルール作成 → 手動 reopen（本番 #448 と同じ経路）

サイクル16のルールを残したまま、#1 を reopen します。

```bash
gh api -X PATCH "repos/$R/dependabot/alerts/1" -f state=open
```

期待: ルールの条件には合致しているので、再評価されれば `auto_dismissed` に戻るはず。

観測は直後 / 10分後 / 1時間後 / 翌日。本番では12時間放置しても `open` のままでした。ここで同じ結果が出れば「手動で state を変えたアラートは、ルールの再適用対象にならない」と確定できます。

結果:

### 18. ルールの Disable → Enable で再評価されるか

サイクル17で `open` のまま変化しなかった場合に実施します。ルールを一度 Disabled にして、また Enabled に戻します。

期待: ルールの状態変更が再評価のトリガーになるなら `auto_dismissed` に変わる。本番では変化しませんでした。

あわせて、ルールをいったん削除して同じ内容で作り直した場合も試します。新規作成なら遡及適用（サイクル1で確認する挙動）が走るはずなので、これで戻れば「再評価はルール新規作成時のみ」と言えます。

結果:

### 19. ルール削除時に auto_dismissed はどうなるか

サイクル1のルールを削除します。

期待: `auto_dismissed` のまま残るか、`open` に戻るか。

これは検証の進め方にも影響します。削除で open に戻るなら、サイクル間のリセット作業が不要になります。

結果:

### 20. ルール有効中に新規発生したアラートは即 auto_dismissed になるか

`packages/app` を対象にしたルール（`manifest:packages/app/package-lock.json`）を作成し、有効なまま新しい脆弱依存を追加して push します。

```bash
mkdir -p packages/late-arrival
cat > packages/late-arrival/package.json <<'EOF'
{
  "name": "late-arrival",
  "version": "1.0.0",
  "private": true,
  "description": "ルール有効中に発生したアラートの扱いを見るためのダミー",
  "dependencies": {
    "ini": "1.3.5"
  }
}
EOF
(cd packages/late-arrival && npm install ini@1.3.5 --package-lock-only --no-audit --no-fund)
git add packages/late-arrival && git commit -m "test: ルール有効中の新規アラート検証用フィクスチャを追加" && git push
```

ルールの条件は `packages/app` なので、この新規アラートは対象外です。そのうえで、対象に含めたルール（`manifest:packages/late-arrival/package-lock.json`）を先に作っておいてから push する、という順序で試します。

期待: アラート発生と同時に `auto_dismissed` になる。なれば「ルールはアラート発生時に評価される」ことが確定し、サイクル17の結果とあわせて評価タイミングが特定できます。

結果:

### 状態遷移まとめ

| 起点の state | 操作 | 結果 |
| --- | --- | --- |
| `open` | ルール作成 | |
| 手動 `dismissed` | ルール作成 | |
| 手動 `dismissed` → ルール作成 | 手動 reopen | |
| `auto_dismissed` | 手動 reopen | |
| `auto_dismissed` | ルール Disable → Enable | |
| `auto_dismissed` | ルール削除 | |
| （新規発生） | ルール有効中に push | |

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
| 16 | 手動 dismiss 済みにルール作成 | 据え置きか上書きか | |
| 17 | 手動 dismiss → ルール → reopen | 再 dismiss されるか | |
| 18 | ルール Disable → Enable で再評価 | 再 dismiss されるか | |
| 19 | ルール削除時の auto_dismissed | 据え置きか open か | |
| 20 | ルール有効中の新規アラート | 即 auto_dismissed | |
