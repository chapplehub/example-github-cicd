# GitHub Actions の Secret マスク回避問題と対策

## 問題の概要

GitHub Actions は Secret をログ出力時に自動でマスクしますが、**シェルの文字列操作を使うとマスクを回避できてしまう**という既知のセキュリティ制限があります。

### マスクされるケース

```yaml
env:
  PASSWORD: ${{ secrets.PASSWORD }}
steps:
  - run: echo "${PASSWORD}"           # ✅ マスクされる (****)
  - run: echo "${{ env.PASSWORD }}"   # ✅ マスクされる (****)
```

### マスクが回避されるケース

```yaml
- run: echo "${PASSWORD:0:1} ${PASSWORD#?}"  # ❌ マスクされない！
```

**理由**: GitHub Actions は**完全一致**でマスクを行うため、文字列を分割・変形すると検出できない

```
password123     → マスクされる
p assword23     → マスクされない（空白が入っているため）
```

## なぜ問題なのか

- 悪意のあるPRや開発者がこの手法を使えば、意図的にSecretを漏洩できる
- 「ログ出力しないように」と注意喚起しても、技術的に防げない

## 対策方法

### 1. Conftest/OPA でポリシーチェック（自動検出）

このリポジトリには既に Conftest が導入されているので、それを拡張して危険なパターンを検出できます。

**現状のポリシー**: `example-github-cicd/policy/workflow.rego`
```rego
package main

deny[msg] {
    not input.permissions
    msg = "Workflow permissions are missing"
}
```

**拡張案**: Secret の危険な使用を検出
```rego
# Secretを含むechoコマンドを検出
deny[msg] {
    job := input.jobs[_]
    step := job.steps[_]
    contains(step.run, "echo")
    regex.match(`echo.*(\$\{.*PASSWORD.*\}|\$\{\{.*secret)`, step.run)
    msg = sprintf("Potentially unsafe secret output detected in step: %v", [step])
}

# シェルの文字列操作パターンを検出
deny[msg] {
    job := input.jobs[_]
    step := job.steps[_]
    # ${VAR:offset:length} や ${VAR#pattern} などのパターン
    regex.match(`\$\{[^}]*(:|#|\%|\/)`, step.run)
    contains(step.run, "PASSWORD")
    msg = "Shell parameter expansion on PASSWORD detected - potential secret leak"
}
```

**実行方法**: PR時に自動チェック（`.github/workflows/conftest.yml` で既に設定済み）
```yaml
- run: |
    docker run --rm -v "$(pwd):$(pwd)" -w "$(pwd)" \
    openpolicyagent/conftest test --policy policy/ .github/workflows/
```

### 2. GitHub Environments + Required Reviewers（承認制）

Secret を使うジョブに Environment を設定し、実行前に承認を必須にする。

```yaml
jobs:
  secret-print:
    runs-on: ubuntu-latest
    environment: production  # ← Environment を指定
    env:
      PASSWORD: ${{ secrets.PASSWORD }}
    steps:
      - run: echo "${PASSWORD}"
```

**設定方法**:
1. GitHub リポジトリ → Settings → Environments
2. `production` Environment を作成
3. "Required reviewers" を設定
4. Secret を使うワークフローは承認がないと実行できない

**メリット**:
- 実行前に人間がコードをレビュー
- 怪しいスクリプトを発見できる

**デメリット**:
- 手動承認が必要で CI/CD が遅くなる

### 3. CODEOWNERS でワークフロー変更を保護（レビュー必須化）

`.github/CODEOWNERS`:
```
# ワークフローファイルの変更はセキュリティチームの承認が必須
/.github/workflows/*.yml @security-team
/.github/workflows/*.yaml @security-team
```

**設定方法**:
1. `.github/CODEOWNERS` ファイルを作成
2. ブランチ保護ルールで "Require review from Code Owners" を有効化

**メリット**:
- マージ前に必ずレビューが入る
- 危険なコードを検出できる

### 4. Repository Rulesets（最強の保護）

GitHub の比較的新しい機能で、Admin でもバイパスできないルールを設定可能。

**設定方法**:
1. Settings → Rules → Rulesets
2. Create ruleset
3. Target: `.github/workflows/**`
4. Rules:
   - Require status checks to pass (Conftest など)
   - Require pull request before merging
   - Require approval from specific users/teams
   - Block force pushes
   - **Bypass list を空にする**（Admin でもバイパス不可）

**メリット**:
- 最も強力な保護
- 組織全体に一貫したポリシーを適用可能

### 5. ワークフロー実行ログの監視（事後検出）

GitHub API を使ってワークフロー実行ログを監視し、怪しいパターンを検出。

```yaml
# .github/workflows/audit.yml
name: Audit Workflow Logs
on:
  workflow_run:
    workflows: ["*"]
    types: [completed]
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - name: Check for suspicious patterns
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          # ログを取得
          gh run view ${{ github.event.workflow_run.id }} --log > logs.txt

          # 怪しいパターンを検出（例: 複数の文字が連続して表示されている）
          if grep -E '[a-zA-Z0-9]{8,}' logs.txt | grep -v '***'; then
            echo "Potential secret leak detected!"
            # Slack通知、Issue作成など
          fi
```

**メリット**:
- 事後的に漏洩を検出できる

**デメリット**:
- すでに漏洩した後の検出

### 6. Secret の使用を最小化（根本対策）

そもそも Secret をワークフロー内で直接使わない設計にする。

**代替案**:
- **OIDC (OpenID Connect)** を使う（パスワード不要）
- **GitHub App** を使う（短命トークン）
- **外部 Secret Manager** を使う（AWS Secrets Manager、HashiCorp Vault など）

例: AWS へのデプロイは OIDC を使う（このリポジトリの Chapter 11-12 で実践）
```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::${{ secrets.AWS_ID }}:role/${{ secrets.ROLE_NAME }}
    aws-region: ap-northeast-1
```

## 実用的な推奨対策（優先順位順）

### レベル1: すぐできる対策
1. **Conftest ポリシー拡張** - 危険パターンの自動検出
2. **CODEOWNERS 設定** - ワークフロー変更時のレビュー必須化

### レベル2: 組織レベルの対策
3. **Repository Rulesets** - 強力な保護ルール
4. **ブランチ保護** - 直接 push を禁止

### レベル3: 根本対策
5. **OIDC への移行** - パスワードレス認証
6. **Secret 使用の最小化** - 設計レベルでの改善

## まとめ

| 対策 | 効果 | 導入コスト | 推奨度 |
|-----|------|----------|--------|
| Conftest ポリシー | 自動検出 | 低 | ⭐⭐⭐ |
| CODEOWNERS | レビュー必須化 | 低 | ⭐⭐⭐ |
| Repository Rulesets | 最強の保護 | 中 | ⭐⭐⭐ |
| Environment 承認 | 実行前チェック | 中 | ⭐⭐ |
| ログ監視 | 事後検出 | 高 | ⭐ |
| OIDC 移行 | 根本解決 | 高 | ⭐⭐⭐ |

**結論**: 完全に防ぐのは技術的に不可能だが、**レビュープロセス + ポリシーチェック**の組み合わせでリスクを大幅に軽減できる。

## 参考

- ワークフローファイル: `.github/workflows/secret-print.yml`
- 既存ポリシー: `example-github-cicd/policy/workflow.rego`
- Conftest ワークフロー: `example-github-cicd/.github/workflows/conftest.yml`
- GitHub Docs: [Encrypted secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- GitHub Docs: [Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
