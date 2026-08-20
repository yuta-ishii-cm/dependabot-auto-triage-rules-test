# dependabot-auto-triage-rules-test

GitHub の Dependabot auto-triage rules（自動トリアージルール）の挙動を検証するためのサンドボックスリポジトリです。

> [!WARNING]
> このリポジトリには、意図的に既知の脆弱性を含む古いバージョンの依存パッケージが含まれています。Dependabot alerts を発生させることが目的で、コードはビルドも実行もされません。実プロダクトでの利用を想定したものではありません。
>
> 検証が完了したら、このリポジトリは削除します。

## 検証の目的

Dependabot alerts はスキャン対象のパスを除外できません。そのため「参考用に保持しているだけで、ビルドにも実行にも使われないディレクトリ」から大量のアラートが発生します。

これを auto-triage rules の `manifest:` 条件で自動 dismiss できるか、その挙動を実際に確認します。

## 確認すること

| ID | 確認項目 |
| --- | --- |
| A | ルール作成時に、既存の open アラートが遡って `auto_dismissed` になるか |
| B | 同一キーのカンマ区切り複数値（`manifest:a,b`）が OR として機能するか |
| C | 手動 reopen したアラートがルールで再クローズされるか |
| D | ルール対象外のパスのアラートが open のまま残るか（誤爆しないこと） |

## 構成

ルールの対象パスと対象外パスを両方用意しています。

```
docs/legacy-app/app/webroot/js/pkg-a/     ← ルール対象1
docs/legacy-app/app/webroot/js/pkg-b/     ← ルール対象2（B の OR 検証用）
packages/app/                             ← ルール対象外（D の誤爆チェック用）
```

ecosystem は npm に統一しています。

脆弱な依存は `devDependencies` ではなく `dependencies` に置いています。GitHub presets の「Dismiss low-impact alerts for development-scoped dependencies」が有効だと、開発依存の低リスクアラートがプリセット側で自動 dismiss され、カスタムルールの効果と区別できなくなるためです。

## 検証で使うルール

Settings → Advanced Security → Dependabot rules から、以下の内容で1本作成します（カスタムルールを操作する API は提供されていないため、UI 操作のみ）。

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
