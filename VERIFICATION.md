# 検証手順と結果

GitHub の Dependabot auto-triage rules について、意図的に脆弱な依存を詰め込んだサンドボックスで挙動を実測した記録です。2026-08-24 から 08-26 にかけて実施しました。

カスタムルールを操作する API は提供されていないため、ルールの作成・削除は UI 操作（Settings → Advanced Security → Dependabot rules）で行い、結果の観測は REST API で行っています。

## 結論サマリ

### 条件の書き方

| 記法 | 結果 |
| --- | --- |
| `package:uuid package:qs`（同じキーの繰り返し） | OR |
| `package:qs scope:development`（違うキー） | AND |
| `package:uuid,qs`（カンマ区切り） | 保存できるが該当0件 |
| `severity:critical,high`（カンマ区切り） | バリデーションエラー |
| `manifest:docs/legacy-app/**`（ワイルドカード） | バリデーションエラー |

メタデータの種類ごとに AND、同じ種類の中では OR。入力欄の `matching all included metadata` の all は、値ではなく種類の単位で読みます。

カンマ区切りは弾かれ方が一貫していません。選択肢が決まっているキー（`severity` など）はエラーになりますが、`manifest` や `package` はエラーなく保存できて1件も当たりません。

### 指定できるキー

11個。実測で動作を確認したのは `malware` を除く10個です。

| キー | 記述例 | ドキュメントでの表記 |
| --- | --- | --- |
| `severity:` | `severity:critical` | severity（値は `moderate`。`medium` は不可） |
| `package:` | `package:node-forge` | package |
| `ecosystem:` | `ecosystem:pip` | ecosystem |
| `scope:` | `scope:development` | scope |
| `manifest:` | `manifest:packages/app/package-lock.json` | manifest（フルパス完全一致のみ） |
| `cwe:` | `cwe:1321` | 記載なし |
| `cve_id:` | `cve_id:CVE-2021-44906` | `CVE-ID` |
| `ghsa_id:` | `ghsa_id:GHSA-p8p7-x288-28g6` | `GHSA-ID` |
| `epss:` | `epss:>0.1` | `epss_percentage` |
| `malware:` | 未検証 | 記載なし |
| `classification:` | `classification:vulnerability` | 記載なし |

ドキュメントが挙げている Patch availability に対応するキーはありません。アクション側の `Until patch is available` として表現されています。alerts API で使える `relationship:` も指定できません。

### ルールが評価されるタイミング

3つだけです。定期的な再スキャンはありません。

1. ルールを新規作成したとき（dismiss は既存アラートに遡って効く。反映は数秒）
2. ルールを削除したとき（そのルールが閉じたアラートが `open` に戻る）
3. 新しいアラートが発生したとき（`Open a pull request` はここでのみ発火）

### アラートの state との関係

| 経緯 | 再びルールの対象になるか |
| --- | --- |
| 手動 dismiss → 手動 reopen | ならない（恒久的に対象外） |
| ルールで dismiss → ルール削除で復帰 | なる |

手動で state を触ったアラートは、以降どのルールを作っても対象外になります。ルールの動作確認を手動 reopen で行うことはできません。

### アクション

| アクション | 挙動 |
| --- | --- |
| `Dismiss alerts` → `Indefinitely` | 修正版の有無に関係なく閉じる |
| `Dismiss alerts` → `Until patch is available` | 修正版が存在しないアラートのみ対象 |
| `Open a pull request` | 新規アラートにのみ発火。既存アラートには効かない。PR を作るだけでアラートは `open` のまま |

dismiss ルールは PR ルールより先に評価されます。同じアラートに両方当たると PR は作られません。

### 確認方法

ルール一覧の表示は、スペース区切りで入力してもカンマ区切りに揃えられます。表示を見てもルールが機能するかは判別できません。編集画面を開けば入力どおりの文字列が見えます。

確実なのは実際に閉じた件数を数えることです。UI では `resolution:auto-dismissed` で絞り込めます（この構文はドキュメントに記載がなく、`is:auto-dismissed` では引っかかりません）。

以下は個別の検証記録です。

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

## Target alerts に指定できるキー（UI サジェストの実測）

New rule の Target alerts 欄をクリックすると候補が出ます。実際に指定できるキーは以下の11個。ドキュメントの記述とかなり食い違います。

| UI のキー | 値の形式 | ドキュメント側の表記 |
| --- | --- | --- |
| `severity:` | critical, high, moderate, low | `severity` |
| `package:` | package-name | `package` |
| `ecosystem:` | ecosystem-name | `ecosystem` |
| `scope:` | runtime, development | `scope`（development のみ記載） |
| `manifest:` | manifest-name | `manifest` |
| `cwe:` | cwe-number | 記載なし |
| `cve_id:` | cve-id | `CVE-ID` |
| `ghsa_id:` | ghsa-id | `GHSA-ID` |
| `epss:` | `>n` `<n` `>=n` `<=n`（0.0〜1.0） | `epss_percentage` |
| `malware:` | package, version | 記載なし |
| `classification:` | malware, vulnerability | 記載なし |

食い違いの要点。

- `CVE-ID` → 実際は `cve_id`
- `GHSA-ID` → 実際は `ghsa_id`
- `epss_percentage` → 実際は `epss`。範囲指定（`0.0..0.01`）は候補の説明に無く、比較演算子のみ
- `cwe` は alert filters のリファレンスに存在しないが、ルールでは使える
- `malware` と `classification` はどちらのドキュメントにも無い

### Patch availability に対応するキーは存在しない

auto-triage rules のドキュメントは「指定できるメタデータ」に Patch availability を挙げていますが、候補一覧に該当キーがありません。`has:` もありません。

これはアクション側の `Until patch is available` として表現されているためと考えられます。ドキュメントの一覧が条件とアクションを混ぜて記載しているせいで、フィルタとして書けるように読めてしまいます。

### severity の値は moderate（API は medium）

候補は `critical` `high` `moderate` `low` の4つ。`medium` は出てきません。

実際に `severity:medium` を入力して保存を試したところ、`severity has an invalid value: medium` でバリデーションエラーになりました（2026-08-24 確認）。

一方 REST API のクエリパラメータは `severity=medium` を受け付けて17件返します。API とルール UI で値の名前が違うため、API での下見をそのままルールに貼ると通りません。

### relationship は指定できない

alerts API では `relationship=direct` が機能しますが（direct 36 / transitive 5）、ルールの候補一覧にはありません。直接依存・推移依存での絞り込みはルールでは不可。

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

# UI で auto-dismissed だけを絞り込む検索構文（公式ドキュメントに記載なし。実測で確認）
#   resolution:auto-dismissed
#   ※ is:auto-dismissed は効かない

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

結果: カンマ区切りはルールでは使えない。2026-08-24 実施。

単一値の `manifest:` は正常に動作します（サイクル16で確認済み。6件が即 `auto_dismissed`、誤爆なし）。しかしカンマで複数値を並べると機能しません。

フィールドの型によってエラーの出方が変わります。ここが最大の落とし穴です。

| 入力 | 結果 |
| --- | --- |
| `severity:critical,high` | 保存できない。`severity has an invalid value: critical,high` |
| `manifest:docs/legacy-app/**` | 保存できない。`manifest has an invalid value: docs/legacy-app/**` |
| `package:uuid,qs` | エラーなく保存でき `Matches package:uuid,qs` と表示。ただし該当0件（21分経過しても変化なし） |
| `manifest:<pkg-a のパス>,<pkg-b のパス>` | エラーなく保存でき一覧にも表示。ただし該当0件（9分経過しても変化なし。対象9件はすべて `open` のまま） |

`manifest:` はワイルドカードを弾くので値の検証自体はしています。ところがカンマ区切りは検証をすり抜け、そのうえ何にも当たりません。もっとも危険な組み合わせです。

選択肢が決まっている `severity` はカンマをエラーとして弾きます。その場で気づけます。

一方 `package` と `manifest` はカンマを含む文字列をそのまま1つの値として受け取り、該当なしになります。ルール一覧にもそれらしく表示されるため、設定できたように見えて何も起きません。

### 同一キーをスペースで並べると OR

カンマは使えませんが、同じキーを繰り返してスペースで区切ると OR になります。

```
package:uuid package:qs
```

これで #30（qs）と #31（uuid）の2件が同時に `auto_dismissed` になりました（02:59:34Z、両件同一）。

`manifest:` でも同様です（2026-08-26 実測）。

```
manifest:docs/legacy-app/app/webroot/js/pkg-a/package-lock.json manifest:docs/legacy-app/app/webroot/js/pkg-b/package-lock.json
```

pkg-a の6件（#2〜#7）と pkg-b の2件（#8, #9）が 01:25:16〜17Z に `auto_dismissed`。手動操作済みの #1 だけ `open` のまま残りました。同じ2パスをカンマでつないだ `Manifest comma test` が2日間0件だったのと対照的です。

### ルール一覧の表示では機能するか判別できない

重要な落とし穴です。スペース区切りで入力したルールも、一覧ではカンマ区切りで表示されます。

| ルール名 | 入力した記法 | 一覧の表示 | 実際の対象 |
| --- | --- | --- | --- |
| `Manifest comma test` | カンマ区切り | `manifest:パスA,パスB` | 0件 |
| `Test manifest repeat` | スペース区切り | `manifest:パスA,パスB` | 8件 |

画面上まったく同じ文字列に見えるのに、一方は機能して一方は機能しません。

ただし編集画面（Edit rule）を開くと、入力どおりの文字列が保持されています。`manifest:` が1つならカンマ区切り、2つ並んでいればキーの繰り返しです。既存ルールの記法を確認したいときは一覧ではなく編集画面を見ること。

それでも確実なのは、実際に閉じた件数を数えることです。

スペース区切りの意味はキーが同じかどうかで変わります。

| 書き方 | 意味 |
| --- | --- |
| `package:uuid package:qs` | OR |
| `package:qs scope:development` | AND |
| `package:uuid,qs` | 使えない（0件、またはエラー） |

「メタデータの種類ごとに AND、同じ種類の中では OR」という動作です。入力欄の説明文 `Rules will be applied for alerts matching all included metadata.` は、種類単位で読む必要があります。

対処: 複数値を指定したい場合はカンマではなくキーの繰り返しで書くこと。ルールを分ける必要はありません。

なお GitHub の発表ブログに掲載されているルール作成画面のスクリーンショットには `severity:low,medium scope:development manifest:otter/package-lock.json` という作例が写っています。この記法は現在の UI ではバリデーションエラーになります。

### アクションの排他

`Dismiss alerts` → `Indefinitely` を選択すると `Open a pull request to resolve alerts` がグレーアウトして選択できなくなります。ドキュメントの記述どおりの挙動です。

### 2. 手動 reopen 後の再クローズ（検証 C）

サイクル1のルールを残したまま、`auto_dismissed` になった1件を reopen します。

```bash
gh api -X PATCH "repos/$R/dependabot/alerts/<番号>" -f state=open
```

期待: 再び `auto_dismissed` に戻る。戻らなければ「手動で state を変えたアラートはルールの再適用対象外」と確定。

観測は時間を空けて複数回（直後 / 10分後 / 翌日）。

結果: サイクル17で確定。手動で reopen したアラートは、どのルールを新規作成しても再適用されない。この方法での動作確認は成立しない。

### 3. manifest ワイルドカード

```
manifest:docs/legacy-app/**
```

期待: 保存時に `manifest has an invalid value` でエラー。`*` 単体、前方一致（`manifest:docs/legacy-app`）も同様に試す。

結果: UI で確認。`manifest has an invalid value: docs/legacy-app/**` のバリデーションエラーになり保存できない。

### 4. severity 複数値

```
severity:critical,high
```

期待: 21件（critical 6 + high 15）。manifest をまたいで効くこと、カンマ区切りが OR であることをここでも確認。

結果: `severity has an invalid value: critical,high` で保存不可。複数値はキーの繰り返し（`severity:critical severity:high`）で書く。

### 5. scope

```
scope:development
```

期待: 2件（`packages/dev-tools` の qs のみ）。

結果: 単独では未実施。ただしサイクル8の `package:qs scope:development` で development の2件だけに効くことは確認済み。

### 6. ecosystem

```
ecosystem:pip
```

期待: 9件（`services/api/requirements.txt` のみ）。

結果: 期待どおり9件。`services/api/requirements.txt` の pip 9件が即 `auto_dismissed`。npm 32件は無傷。パッケージ名の大文字小文字（PyYAML/pyyaml）もまとめて対象になった。

### 7. package

```
package:qs
```

期待: 3件（`packages/dev-tools` の2件 + `packages/unpatched` の推移依存1件）。manifest と scope をまたぐこと。

あわせて大文字小文字の扱いも確認します。pip 側は同じライブラリが advisory によって `PyYAML` と `pyyaml`、`Jinja2` と `jinja2` に分かれているため、`package:pyyaml` が両方に当たるかを見れば判定できます。

結果: `package:node-forge` で15件が即 `auto_dismissed`。単一パッケージ指定は成立。pip 側の大文字小文字の切り分けは未実施。

### 8. 異なるキーの組み合わせ（AND か）

```
package:qs scope:development
```

期待: 2件。サイクル7の3件から推移依存の1件が落ちれば、異なるキーはスペース区切りで AND。

結果: 期待どおり2件。`package:qs` の3件のうち development の2件だけが閉じ、runtime の1件（推移依存）は open のまま。異なるキーは AND で確定。

### 9. patch availability

```
has:patch
```

期待: 40件。修正版が存在しない #27（`request` / GHSA-p8p7-x288-28g6）だけが `open` で残る。

結果: このキーは存在しない。UI の候補一覧に `has:` に相当するキーが無く、Patch availability はアクション側の `Until patch is available` として表現される。条件としては書けない。

### 10. CVE ID

```
CVE-ID:CVE-2019-10744
```

期待: 1件（#1 lodash critical）。

結果: 成立。2026-08-26 実施。

`cve_id:CVE-2021-44906` で #9（minimist、critical）の1件だけが `auto_dismissed`（01:32:28Z）。同じ manifest の #8（CVE-2020-7598、medium）は `open` のまま残り、CVE 単位でのピンポイント指定が機能することを確認。

キー名はドキュメントの `CVE-ID` ではなく `cve_id`。

### 11. GHSA ID

```
GHSA-ID:GHSA-p8p7-x288-28g6
```

期待: 1件（#27 request）。

結果: `ghsa_id:GHSA-p8p7-x288-28g6` で #27 の1件だけが即 `auto_dismissed`。キー名はドキュメントの `GHSA-ID` ではなく `ghsa_id`。

### 12. CWE

```
cwe:CWE-1321
```

期待: 10件。ただし CWE は auto-triage rules のドキュメントには「指定できるメタデータ」として挙がっている一方、alert filters のリファレンスには載っていません。キー名が `cwe:` で合っているかを含めて確認します。通らなければ UI のサジェストで正しいキー名を確認してください。

結果: `cwe:1321` で成立。値は `CWE-1321` ではなく数字のみ。条件に合致する open アラートが複数マニフェストにまたがって閉じた（手動操作済みの #1 だけ対象外）。

### 13. EPSS Score

```
epss_percentage:>0.01
```

期待: 18件。`>=`、範囲指定（`0.0..0.01`）が使えるかもあわせて確認。

結果: 成立。2026-08-26 実施。

`epss:>0.1` で #2（lodash、epss=0.21333）の1件だけが `auto_dismissed`（01:31:34Z）。すぐ下の #4（epss=0.07336）は `open` のまま残り、しきい値が正しく機能することを確認。

キー名はドキュメントの `epss_percentage` ではなく `epss`。候補の説明は比較演算子（`>n` `<n` `>=n` `<=n`）のみで、範囲指定（`0.0..0.01`）の記載は無い。alerts API 側は `epss_percentage` かつ範囲指定も受け付けるため、ここでも API とルールで書き方が違う。

### 14. アクション: Until a patch is available

```
manifest:docs/legacy-app/app/webroot/js/pkg-a/package-lock.json
```

Rules: `Dismiss alerts` → `Until a patch is available`

結果: 修正版が存在しないアラートだけが対象。2026-08-24 実施。

| # | パッケージ | 修正版 | 結果 |
| --- | --- | --- | --- |
| #27 | request | なし | `auto_dismissed` 02:43:58Z（ルール作成の数秒後） |
| #29 | form-data | 2.5.4 あり | `open` のまま（7分経過） |
| #32 | form-data | 2.5.6 あり | `open` のまま（7分経過） |

`Until patch is available` は「修正版がまだ出ていないアラートを、修正版が出るまで伏せておく」という意味です。既に修正版があるアラートは最初から対象外になります。

`Indefinitely` との使い分け。

- `Indefinitely`: 修正版の有無に関係なく閉じる。使っていないディレクトリのアラートを黙らせる用途
- `Until patch is available`: 今すぐ直せないものだけ伏せる。修正版が出たら再び表に出てくる

### ルール削除で戻ったアラートは再適用の対象になる

#27 は「ルールで dismiss → ルール削除で open に復帰 → 別ルールで再び dismiss」という経路をたどりました。

サイクル17で確定した「手動で state を変えたアラートは対象外」とは対照的です。同じ `open` でも、そこに至った経緯で扱いが変わります。

| 経緯 | 再適用 |
| --- | --- |
| 手動 dismiss → 手動 reopen（#1） | されない |
| ルールで dismiss → ルール削除で復帰（#27） | される |

参考: 元の手順で確認しようとしていたコマンド。

```bash
gh api "repos/$R/dependabot/alerts?per_page=100&state=auto_dismissed" \
  | jq -r '.[] | "#\(.number)\t\(.dismissed_reason // "-")\t\(.dismissed_comment // "-")"'
```

結果: dismissed_reason / dismissed_comment の内訳確認は未実施。アクションの意味自体は上記のとおり確定。

### 15. アクション: Open a pull request

```
manifest:docs/legacy-app/app/webroot/js/pkg-b/package-lock.json
```

Rules: `Open a pull request`

結果: PR ルールは新規アラートの発生時にだけ発火する。既存アラートには効かない。2026-08-24 確定。

決定的だったのは次の対照実験です。

1. 既存アラート15件（node-forge、全て open・修正版あり・dismiss ルールなし・security updates 無効）を対象に `package:node-forge` の PR ルールを作成 → 5時間以上 PR もブランチも作られず
2. `package:ini` の PR ルールを先に作成してから、`ini@1.3.5` を含む新規 manifest を push → push 09:30:36Z、アラート #42 発生 09:30:40Z、PR #2 作成 09:31:45Z

同じアクションが、既存アラートには5時間無反応で、新規アラートには65秒で反応しました。dismiss ルールが作成時に既存アラートへ遡及するのとは対照的に、`Open a pull request` はアラート発生イベントにのみ紐づいています。

ドキュメントの「rules apply to both future and current alerts」は dismiss には当てはまりますが、PR アクションは future のみです。

なお PR が作られたあともアラート #42 は `open` のままです（PR ルールは dismiss しない）。

`package:node-forge` を対象に `Open a pull request to resolve alerts` のみを選んだルールを作成し、30分観測しましたが PR も `dependabot/` ブランチも作られませんでした。

条件は揃えてあります。

- 対象の15件はすべて `open`（dismiss ルールは削除済み）
- node-forge は `packages/app` の直接依存で、修正版 1.4.0 が存在する
- 同じパッケージで過去に Dependabot が PR を作った実績がある
- Dependabot security updates は Disabled（ルール側の注記「This will only target repositories without security updates enabled」と整合）

既存アラートの個別ページには `Review security update` ボタンが出ており更新内容は計算済みでした。つまり既存アラート側は「更新できない」のではなく「トリガーが無い」状態です。手動でボタンを押せば PR は作れます。

### 途中で判明したこと

最初は `package:uuid` を対象にしていましたが、これは検証対象の選定ミスでした。uuid は `packages/unpatched` の推移依存で、親の `request@2.88.2` に修正版が存在しません。現在 3.4.0 に対し修正版は 11.1.1 で、親を更新しない限り上げられないため、そもそも PR を作れる経路がありませんでした。

また、dismiss ルールと PR ルールが同じアラートに当たると dismiss が優先されます。ドキュメントにも `Dismissal rules always act before rules which trigger Dependabot pull requests.` と明記されています。PR が作られないときは、まず他の dismiss ルールが先に閉じていないかを疑うべきです。

### 既存ルールは open に戻ったアラートを拾わない

PR ルールを有効にしたまま、対象15件を（dismiss ルールの削除によって）`open` に戻し、15分観測しましたが PR は作られませんでした。その後ルールを削除して同じ内容で作り直すと、評価は走ります（作成時点で対象が評価される）。

ルールが評価されるのはルールの新規作成時と削除時であって、アラート側の状態が変わったときに既存ルールが再スキャンされるわけではない、という理解と整合します。

## アラートのタイムラインにルール適用が記録される

個別のアラートページを開くと、ルールによる状態変更がタイムラインに残っています。

```
dependabot dismissed this due to an alert rule
  Repository rule created and Dismiss node-forge was applied

dependabot reopened this
  Repository rule deleted: Dismiss node-forge
```

`Repository rule created and ... was applied` という文言から、適用のトリガーがルールの作成であることが読み取れます。削除時も `Repository rule deleted` として reopen が記録されます。

API の観測結果と一致しており、どのルールがどのアラートを閉じたのかを後から追う手段としても使えます。

### 21. classification

```
classification:vulnerability manifest:docs/legacy-app/app/webroot/js/pkg-a/package-lock.json
```

結果: 成立。2026-08-26 実施。

pkg-a の open 6件のうち #3〜#7 の5件が `auto_dismissed`（01:33:07Z、全件同一）。`classification:vulnerability` 単体では全アラートに当たる広い条件だが、`manifest:` との AND で pkg-a に限定され、対象外の5マニフェストは1件も動かなかった。

このキーは alert filters のリファレンスにも auto-triage rules のドキュメントにも記載がなく、UI の候補一覧でのみ確認できる。値は `malware` と `vulnerability` の2つ。通常の脆弱性アラートは `vulnerability` に該当する。

あわせてサイクル17の再確認になった。手動操作済みの #1 は条件（vulnerability かつ pkg-a）に完全に合致するにもかかわらず、5件が閉じた同じ瞬間に飛ばされている。

## 検証サイクル（状態遷移）

ここからはルールの条件ではなく、アラートの state とルールの相互作用を見ます。「ルールがいつ評価されるのか」を切り分けるのが目的です。

時間がない場合はサイクル16〜18を優先してください。実運用で最初に踏みやすい経路です。

### 16. 手動 dismiss 済みのアラートにルールを作る

先に対象を手動 dismiss してからルールを作ります。

```bash
# pkg-a の #1 を手動 dismiss
gh api -X PATCH "repos/$R/dependabot/alerts/1" \
  -f state=dismissed -f dismissed_reason=not_used \
  -f dismissed_comment="検証用の手動 dismiss"

# 確認
gh api "repos/$R/dependabot/alerts/1" \
  -q '"\(.state)\treason=\(.dismissed_reason // "-")\tauto_dismissed_at=\(.auto_dismissed_at // "-")"'
```

この状態で `manifest:docs/legacy-app/app/webroot/js/pkg-a/package-lock.json` のルールを作成します。

期待: pkg-a の残り6件は `auto_dismissed` になる。#1 がどうなるかが焦点で、`dismissed` のまま据え置きか、`auto_dismissed` に上書きされるかを見ます。

結果: 期待どおり。2026-08-24 実施。

- #2〜#7 の6件が `auto_dismissed`。`auto_dismissed_at` は全件 `2026-08-24T02:00:48Z` で同一
- ルール作成から数秒で反映された。遡及適用は即時
- 手動 dismiss した #1 は `dismissed` / `reason=not_used` のまま据え置き。ルールは上書きしない
- 対象外の5マニフェスト34件はすべて `open` のまま。誤爆なし

つまり既存 open アラートへの遡及適用は成立（検証 A）、対象外への誤爆もなし（検証 D）。手動 dismiss はルールより優先される。

なお pkg-a の7件はすべて修正版が存在します。それでも6件が dismiss されたので、アクションは `Indefinitely` として動作しています。

### 17. 手動 dismiss → ルール作成 → 手動 reopen

サイクル16のルールを残したまま、#1 を reopen します。

```bash
gh api -X PATCH "repos/$R/dependabot/alerts/1" -f state=open
```

期待: ルールの条件には合致しているので、再評価されれば `auto_dismissed` に戻るはず。

結果: 再適用されない。2026-08-24 実施。

02:08:08Z に #1 を reopen。以降25分観測しても `open` のまま変化なし。

決定的だったのは `cwe:1321` のルールを新規作成したときの挙動です。#1 は CWE-1321 を持つため条件に合致しますが、対象になりませんでした。

| # | ルール作成前の state | CWE-1321 | 結果 |
| --- | --- | --- | --- |
| #8 (pkg-b) | open | あり | `auto_dismissed` 02:33:05Z |
| #28 (unpatched) | open | あり | `auto_dismissed` 02:33:05Z |
| #1 (pkg-a) | open（手動 reopen 済み） | あり | `open` のまま |

同じルール・同じ瞬間の判定で、条件に合致する他の open アラートは閉じられ、#1 だけが飛ばされました。#1 と他の違いは「手動で state を変更した履歴があるかどうか」だけです。

結論: 一度でも手動で dismiss / reopen したアラートは、以降どのルールを新規作成しても auto-triage の対象外になります。ルールの不具合ではなく、手動操作を尊重する仕様と考えられます。

運用上の含意が2つあります。

1. ルールの動作確認を「手動 reopen」で行うことはできません。必ず自然発生したアラートか、手を触れていないアラートで確認する必要があります。

2. 過去に手動 dismiss したアラートは、あとからルールを作っても `auto_dismissed` には変わりません。手動 dismiss 済みのものはそのまま放置してよく、ルールは新規アラートに対して効きます。

### 18. ルールの Disable → Enable で再評価されるか

サイクル17で `open` のまま変化しなかった場合に実施します。ルールを一度 Disabled にして、また Enabled に戻します。

期待: ルールの状態変更が再評価のトリガーになるなら `auto_dismissed` に変わる。

あわせて、ルールをいったん削除して同じ内容で作り直した場合も試します。新規作成なら遡及適用（サイクル1で確認する挙動）が走るはずなので、これで戻れば「再評価はルール新規作成時のみ」と言えます。

結果: 未実施。サイクル17が「新規作成でも再適用されない」で確定したため、Disable→Enable で覆る見込みは薄いと判断。

### 19. ルール削除時に auto_dismissed はどうなるか

サイクル1のルールを削除します。

期待: `auto_dismissed` のまま残るか、`open` に戻るか。

結果: `open` に戻る。2026-08-24 実施。

`ghsa_id:GHSA-p8p7-x288-28g6` のルール（#27 を 02:17:37Z に `auto_dismissed` にしていた）を削除したところ、#27 は `open` に戻り `auto_dismissed_at` も `null` にクリアされました。

重要なのは、この復元が「システムによる状態変更」である点です。サイクル17で確定した「手動で state を変えたアラートは再適用対象外」の制約には引っかかりません。実際、削除後の #27 は後続のルールで再び dismiss できました。

運用上の含意。

- ルールを消せば、そのルールが閉じたアラートは open に戻る。適用範囲を間違えても取り返しがつく
- 検証時のリセットは、手動 PATCH ではなくルール削除で行うほうが安全。手動 PATCH はアラートを永久に対象外にしてしまう

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

結果: PR アクション版で実施。`package:ini` の PR ルールを先に作成し、`ini@1.3.5` の新規 manifest を push したところ、アラート発生（09:30:40Z）の65秒後に PR #2 が作成された。ルール有効中の新規アラートには即座に効く。dismiss アクション版は未実施だが、同じ発生時評価が働くと推定できる。

### 状態遷移まとめ

| 起点の state | 操作 | 結果 |
| --- | --- | --- |
| `open` | ルール作成 | 即 `auto_dismissed`（数秒） |
| 手動 `dismissed` | ルール作成 | 据え置き。上書きされない |
| 手動 `dismissed` → ルール作成 | 手動 reopen | `open` のまま。再適用されない |
| `auto_dismissed` | 手動 reopen | 未実施（手動操作した時点で以降は対象外になるため、17と同じ結果になると推定） |
| `auto_dismissed` | ルール Disable → Enable | 未実施 |
| `auto_dismissed` | ルール削除 | `open` に戻る。削除で戻ったアラートは別ルールで再適用可能 |
| （新規発生） | ルール有効中に push | PR ルールは発生65秒後に発火（dismiss ルール版は未実施） |

## まとめ

| # | 条件 | 期待 | 結果 |
| --- | --- | --- | --- |
| 1 | manifest 複数値 | 9件 | カンマ不可。キーの繰り返しで OR |
| 2 | reopen 後の再適用 | 再 dismiss | 再適用されない（17で確定）|
| 3 | manifest ワイルドカード | エラー | エラーで保存不可 |
| 4 | severity 複数値 | 21件 | バリデーションエラーで保存不可 |
| 5 | scope | 2件 | 単独未実施（8の AND で機能は確認）|
| 6 | ecosystem | 9件 | 9件で成立 |
| 7 | package | 3件 | node-forge 15件で成立 |
| 8 | 異なるキーの AND | 2件 | 2件で成立 |
| 9 | has:patch | 40件 | キー自体が存在しない |
| 10 | CVE ID | 1件 | 1件で成立（キー名は cve_id）|
| 11 | GHSA ID | 1件 | 1件で成立（ghsa_id）|
| 12 | CWE | 10件 | cwe:1321 で成立。数字のみ指定 |
| 13 | EPSS Score | 18件 | `epss:>0.1` で1件。成立（キー名は epss）|
| 14 | Until a patch is available | 7件 | 修正版なしのみ対象（確定）|
| 15 | Open a pull request | PR 作成 | 新規アラートのみ発火（65秒で PR）。既存には効かない |
| 16 | 手動 dismiss 済みにルール作成 | 据え置きか上書きか | 据え置き。残り6件は即 auto_dismissed |
| 17 | 手動 dismiss → ルール → reopen | 再 dismiss されるか | 再適用されない（確定）|
| 18 | ルール Disable → Enable で再評価 | 再 dismiss されるか | 未実施 |
| 19 | ルール削除時の auto_dismissed | 据え置きか open か | open に戻る（確定）|
| 20 | ルール有効中の新規アラート | 即 auto_dismissed | PR アクション版で実施。発生65秒後に PR 作成。dismiss 版は未実施 |
| 21 | classification | 5件 | 5件で成立。ドキュメント未記載のキー |
