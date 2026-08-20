# dependabot-auto-triage-rules-test

GitHub の Dependabot auto-triage rules（自動トリアージルール）の挙動を検証するためのサンドボックスリポジトリです。

> [!WARNING]
> このリポジトリには、意図的に既知の脆弱性を含む古いバージョンの依存パッケージが含まれています。Dependabot alerts を発生させることが目的で、コードはビルドも実行もされません。実プロダクトでの利用を想定したものではありません。
>
> 検証が完了したら、このリポジトリは削除します。

## 検証の目的

Dependabot alerts はスキャン対象のパスを除外できません。そのため「参考用に保持しているだけで、ビルドにも実行にも使われないディレクトリ」から大量のアラートが発生します。

これを auto-triage rules で自動 dismiss できるか、そして Target alerts にどんな条件をどう書けるのかを実際に確認します。

## 確認すること

### ルールの適用タイミング

| ID | 確認項目 |
| --- | --- |
| A | ルール作成時に、既存の open アラートが遡って `auto_dismissed` になるか |
| B | 同一キーのカンマ区切り複数値（`manifest:a,b`）が OR として機能するか |
| C | 手動 reopen したアラートがルールで再クローズされるか |
| D | ルール対象外のパスのアラートが open のまま残るか（誤爆しないこと） |

### Target alerts に指定できる条件

ドキュメント上、カスタムルールの Target alerts には以下のメタデータを指定できます。それぞれが実際にどう書けて、どう効くのかを1本ずつ検証します。

| # | 条件 | 検証用フィクスチャ |
| --- | --- | --- |
| 1 | Manifest path（リポジトリレベルのルール限定） | 6つの manifest |
| 2 | Severity | critical / high / medium / low が揃う |
| 3 | Package name | パッケージごとにディレクトリを分離 |
| 4 | Ecosystem | npm と pip |
| 5 | Dependency scope | `packages/dev-tools` のみ development、他は runtime |
| 6 | Patch availability | `packages/unpatched` に修正版が存在しない advisory |
| 7 | CVE ID | 各アラートに付与済み |
| 8 | GHSA ID | 各アラートに付与済み |
| 9 | CWE | 15種類以上 |
| 10 | EPSS Score | 各アラートに付与済み |

アクション側も2種類あります。

- `Dismiss alerts` → Indefinitely / Until a patch is available
- `Open a pull request`（Dismiss indefinitely を選んだ場合、または Dependabot security updates が有効な場合は選べない）

## 構成

ルールの対象パスと対象外パスを両方用意し、条件ごとに切り分けられるようディレクトリを分けています。

```
docs/legacy-app/app/webroot/js/pkg-a/   lodash@4.17.11    npm  runtime      ← ルール対象1
docs/legacy-app/app/webroot/js/pkg-b/   minimist@0.0.8    npm  runtime      ← ルール対象2（B の OR 検証用）
packages/app/                           node-forge@0.9.0  npm  runtime      ← ルール対象外（D の誤爆チェック用）
packages/dev-tools/                     qs@6.5.1          npm  development  ← scope 検証用
packages/unpatched/                     request@2.88.2    npm  runtime      ← patch availability 検証用
services/api/requirements.txt           PyYAML / Jinja2   pip  runtime      ← ecosystem 検証用
```

パッケージ選定の意図は以下のとおりです。

`packages/dev-tools` 以外の脆弱な依存は `devDependencies` ではなく `dependencies` に置いています。GitHub presets の「Dismiss low-impact alerts for development-scoped dependencies」が有効だと、開発依存の低リスクアラートがプリセット側で自動 dismiss され、カスタムルールの効果と区別できなくなるためです。

`packages/unpatched` の `request@2.88.2` は、GHSA-p8p7-x288-28g6（Server-Side Request Forgery in Request）に修正版が存在しません。`has:patch` で絞り込んだときに除外される側の材料として使います。あわせて transitive な脆弱依存（`form-data` / `tough-cookie` / `uuid` / `qs`）も引き込むため、直接依存と推移依存の区別にも使えます。

`qs` は `packages/dev-tools`（直接依存・development）と `packages/unpatched`（推移依存・runtime）の両方に現れます。同じパッケージ名で scope と manifest だけが違うので、条件を組み合わせたときの AND 判定の確認に使えます。

ecosystem は npm を中心に、pip を1つだけ混ぜています。

## 検証で使うルール

Settings → Advanced Security → Dependabot rules から作成します。カスタムルールを操作する API は提供されていないため、UI 操作のみです。public preview 中はリポジトリあたり10本までという制限があるので、1本ずつ作成して結果を観測し、削除してから次に進みます。

A〜D の検証で使う基本のルールは以下のとおりです。

```
Rule name:     Dismiss alerts in docs/legacy-app
Target alerts: manifest:docs/legacy-app/app/webroot/js/pkg-a/package-lock.json,docs/legacy-app/app/webroot/js/pkg-b/package-lock.json
Rules:         Dismiss alerts → Indefinitely
```

## 状態の確認方法

```bash
# manifest ごとの state 内訳
gh api "repos/{owner}/{repo}/dependabot/alerts?per_page=100" --paginate --slurp \
  | jq -r '[.[][]] | group_by(.dependency.manifest_path)
    | map({path: .[0].dependency.manifest_path,
           states: (map(.state) | group_by(.) | map({(.[0]): length}) | add)})
    | .[] | "\(.path)\t\(.states)"'

# 個別アラートの状態
gh api repos/{owner}/{repo}/dependabot/alerts/<番号> \
  -q '"\(.state)\tauto_dismissed_at=\(.auto_dismissed_at // "-")"'
```

## 参考

- [About Dependabot auto-triage rules](https://docs.github.com/en/code-security/dependabot/dependabot-auto-triage-rules/about-dependabot-auto-triage-rules)
- [Customizing auto-triage rules to prioritize Dependabot alerts](https://docs.github.com/en/code-security/dependabot/dependabot-auto-triage-rules/customizing-auto-triage-rules-to-prioritize-dependabot-alerts)
- [Dependabot alert filters](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-alerts-filters)
