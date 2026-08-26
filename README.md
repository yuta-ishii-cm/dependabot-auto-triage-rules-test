# dependabot-auto-triage-rules-test

GitHub の Dependabot auto-triage rules（自動トリアージルール）の挙動を検証するためのサンドボックスリポジトリです。

意図的に脆弱な依存を6つのディレクトリに配置して41件のアラートを発生させ、ルールの条件をどう書けるのか・いつ評価されるのかを 2026-08-24 〜 08-26 に実測しました。

検証の手順と全記録は [VERIFICATION.md](VERIFICATION.md) にあります。この README はその結論だけを抜き出したものです。

> [!WARNING]
> このリポジトリには、意図的に既知の脆弱性を含む古いバージョンの依存パッケージが含まれています。Dependabot alerts を発生させることが目的で、コードはビルドも実行もされません。実プロダクトでの利用を想定したものではありません。

カスタムルールを操作する API は提供されていないため、ルールの作成・削除は UI（Settings → Advanced Security → Dependabot rules）で行い、結果の観測は REST API で行っています。

## 条件（Target alerts）の書き方

メタデータの種類ごとに AND、同じ種類の中では OR。入力欄の `matching all included metadata` の all は、値ではなく種類の単位で読みます。

| 記法 | 例 | 結果 |
| --- | --- | --- |
| 同じキーの繰り返し | `package:uuid package:qs` | OR。2件とも対象になる |
| 違うキーをスペース区切り | `package:qs scope:development` | AND。3件から2件に絞られる |
| カンマ区切り（値が自由入力のキー） | `package:uuid,qs` | 保存できるが該当0件 |
| カンマ区切り（値が選択式のキー） | `severity:critical,high` | `severity has an invalid value` で保存不可 |
| ワイルドカード | `manifest:docs/legacy-app/**` | `manifest has an invalid value` で保存不可 |

カンマ区切りの弾かれ方は一貫していません。`severity` のように値が決まっているキーはその場でエラーになりますが、`manifest` や `package` はエラーなく保存できて1件も当たりません。ルール一覧の表示は入力した記法にかかわらずカンマ区切りに揃えられるため、一覧を眺めても機能しているかは判別できません。

## 指定できるキーと構文例

UI のサジェストに出るのは11個。うち `malware` を除く10個の動作を実測しました。

| キー | 構文例 | 実測でわかったこと |
| --- | --- | --- |
| `severity:` | `severity:critical` | 値は `critical` `high` `moderate` `low`。`medium` は保存不可（REST API 側は `medium` を受け付けるため、API での下見をそのまま貼ると通らない） |
| `package:` | `package:node-forge` | 単一パッケージ指定で15件が対象。manifest と scope をまたいで当たる |
| `ecosystem:` | `ecosystem:pip` | pip の9件のみ対象、npm 32件は無傷。`PyYAML` / `pyyaml` の表記ゆれもまとめて拾う |
| `scope:` | `scope:development` | 値は `runtime` `development`。`package:qs scope:development` の AND で development の2件だけに絞れた |
| `manifest:` | `manifest:packages/app/package-lock.json` | フルパス完全一致のみ。ワイルドカードと前方一致は不可 |
| `cwe:` | `cwe:1321` | 値は数字のみで `CWE-1321` は不可。alert filters のリファレンスには無いがルールでは使える |
| `cve_id:` | `cve_id:CVE-2021-44906` | 該当1件だけをピンポイントで閉じられる。ドキュメント表記は `CVE-ID` |
| `ghsa_id:` | `ghsa_id:GHSA-p8p7-x288-28g6` | 同上。ドキュメント表記は `GHSA-ID` |
| `epss:` | `epss:>0.1` | 比較演算子（`>` `<` `>=` `<=`）のみ。範囲指定 `0.0..0.01` は不可。ドキュメント表記は `epss_percentage` |
| `classification:` | `classification:vulnerability` | 値は `vulnerability` `malware`。単体では全アラートに当たるので `manifest:` 等との AND で使う。ドキュメント記載なし |
| `malware:` | （未検証） | サジェストの値の形式は `package, version`。ドキュメント記載なし |

書けそうで書けないものが3つあります。

- Patch availability: 対応するキーがありません。アクション側の `Until patch is available` として表現されています。ドキュメントが条件とアクションを混ぜて記載しているため、フィルタとして書けるように読めてしまいます
- `has:patch`: キー自体が存在しません（alerts API では機能します）
- `relationship:`: 直接依存・推移依存での絞り込みはできません（alerts API では機能します）

## アクション

| アクション | 挙動 |
| --- | --- |
| `Dismiss alerts` → `Indefinitely` | 修正版の有無に関係なく閉じる |
| `Dismiss alerts` → `Until patch is available` | 修正版が存在しないアラートのみ対象 |
| `Open a pull request` | 新規アラートにのみ発火。既存アラートには効かない。PR を作るだけでアラートは `open` のまま |

dismiss ルールは PR ルールより先に評価されます。同じアラートに両方当たると PR は作られません。

## ルールが評価されるタイミング

3つだけです。定期的な再スキャンはありません。

1. ルールを新規作成したとき（dismiss は既存アラートに遡って効く。反映は数秒）

2. ルールを削除したとき（そのルールが閉じたアラートが `open` に戻る）

3. 新しいアラートが発生したとき（`Open a pull request` はここでのみ発火。実測ではアラート発生の65秒後に PR が作られた）

アラートの state との関係は次のとおりです。

| 経緯 | 再びルールの対象になるか |
| --- | --- |
| 手動 dismiss → 手動 reopen | ならない（恒久的に対象外） |
| ルールで dismiss → ルール削除で復帰 | なる |

手動で state を触ったアラートは、以降どのルールを作っても対象外になります。ルールの動作確認を手動 reopen で行うことはできません。

## ルール数の上限

ドキュメントの記載どおり1リポジトリあたり10本でした。上限に達すると `New rule` ボタンが非活性になります。

`Disabled` に切り替えたルールも枠を消費するため、使わないルールを無効化して保管しておくことはできません。枠を空けるには削除するしかありません。GitHub presets の2本は別枠でカウントされません。

## サンドボックスの構成

ルールの対象パスと対象外パスを両方用意し、条件ごとに切り分けられるようディレクトリを分けています。

```
docs/legacy-app/app/webroot/js/pkg-a/   lodash@4.17.11    npm  runtime      ← manifest 検証用
docs/legacy-app/app/webroot/js/pkg-b/   minimist@0.0.8    npm  runtime      ← 同一キー OR の検証用
packages/app/                           node-forge@0.9.0  npm  runtime      ← 誤爆チェック用（ルール対象外）
packages/dev-tools/                     qs@6.5.1          npm  development  ← scope 検証用
packages/unpatched/                     request@2.88.2    npm  runtime      ← patch availability 検証用
packages/late-arrival/                  ini@1.3.5         npm  runtime      ← 新規アラートの扱いの検証用
services/api/requirements.txt           PyYAML / Jinja2   pip  runtime      ← ecosystem 検証用
```

ベースラインは全41件で、内訳は severity が critical 6 / high 15 / medium 17 / low 3、scope が runtime 39 / development 2、ecosystem が npm 32 / pip 9 です。

パッケージ選定の意図は以下のとおりです。

`packages/dev-tools` 以外の脆弱な依存は `devDependencies` ではなく `dependencies` に置いています。GitHub presets の「Dismiss low-impact alerts for development-scoped dependencies」が有効だと、開発依存の低リスクアラートがプリセット側で自動 dismiss され、カスタムルールの効果と区別できなくなるためです。検証中は presets を2本とも Disabled にしています。

`packages/unpatched` の `request@2.88.2` は、GHSA-p8p7-x288-28g6（Server-Side Request Forgery in Request）に修正版が存在しません。`Until patch is available` で対象に残る側の材料として使います。あわせて transitive な脆弱依存（`form-data` / `tough-cookie` / `uuid` / `qs`）も引き込みます。

`qs` は `packages/dev-tools`（直接依存・development）と `packages/unpatched`（推移依存・runtime）の両方に現れます。同じパッケージ名で scope と manifest だけが違うので、AND 判定の確認に使えます。

`packages/late-arrival` だけは後から追加したものです。`package:ini` の PR ルールを有効にした状態で `ini@1.3.5` を push し、ルール有効中に発生した新規アラートがどう扱われるかを見ました（現在は Dependabot の PR がマージ済みで `1.3.6`）。

## 状態の確認方法

```bash
R=yuta-ishii-cm/dependabot-auto-triage-rules-test

# manifest ごとの state 内訳
gh api "repos/$R/dependabot/alerts?per_page=100" \
  | jq -r 'group_by(.dependency.manifest_path)
    | map({path: .[0].dependency.manifest_path,
           states: (map(.state) | group_by(.) | map("\(.[0]):\(length)") | join(" "))})
    | .[] | "\(.path)\t\(.states)"'

# auto_dismissed になった件数と中身
gh api "repos/$R/dependabot/alerts?per_page=100&state=auto_dismissed" \
  | jq -r 'length as $n | "auto_dismissed: \($n)件", (.[] | "  #\(.number)\t\(.dependency.manifest_path)\t\(.dependency.package.name)\t\(.security_advisory.severity)")'
```

UI で絞り込む場合は `resolution:auto-dismissed` を使います（この構文はドキュメントに記載がなく、`is:auto-dismissed` では引っかかりません）。

アラートを open に戻すときは、REST API での手動 PATCH ではなくルールの削除で行ってください。手動で state を触ったアラートは、以降どのルールを作っても対象外になります。

## 参考

- [About Dependabot auto-triage rules](https://docs.github.com/en/code-security/dependabot/dependabot-auto-triage-rules/about-dependabot-auto-triage-rules)
- [Customizing auto-triage rules to prioritize Dependabot alerts](https://docs.github.com/en/code-security/dependabot/dependabot-auto-triage-rules/customizing-auto-triage-rules-to-prioritize-dependabot-alerts)
- [Dependabot alert filters](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-alerts-filters)
