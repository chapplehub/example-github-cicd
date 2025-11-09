# GitHub Actions の環境変数アクセス方法

## 問題の発生

```yaml
env:
  PASSWORD: ${{ secrets.PASSWORD }}
steps:
  - run: echo "${{PASSWORD}}"  # ❌ Unrecognized named-value: 'PASSWORD'
```

**エラー原因**: コンテキスト（`env`、`secrets`、`github` など）を指定していない

## 2つのアクセス方法

### 1. `${PASSWORD}` - シェル変数として

```yaml
- run: echo "${PASSWORD}"
```

- **シェルが実行時に展開**
- `env:` で定義された環境変数がシェルの環境変数として設定される
- シェルスクリプトの中で通常の環境変数としてアクセス

### 2. `${{ env.PASSWORD }}` - GitHub Actions コンテキストとして

```yaml
- run: echo "${{ env.PASSWORD }}"
```

- **ワークフロー実行前（YAML解析時）に展開**
- GitHub Actions がワークフローファイルを処理する段階で値を埋め込む

## 重要な違い

### 展開タイミング

```yaml
env:
  PASSWORD: ${{ secrets.PASSWORD }}  # ← ここで secrets から取得
steps:
  - run: echo "${PASSWORD}"           # ← シェルが環境変数を読む（実行時）
  - run: echo "${{ env.PASSWORD }}"   # ← YAML パース時に展開済み
```

### 使える場所

| アクセス方法 | 使える場所 |
|------------|----------|
| `${PASSWORD}` | `run:` の中（シェルスクリプト内）**のみ** |
| `${{ env.PASSWORD }}` | ワークフローファイルの**どこでも**（`if:` 条件、`with:` パラメータなど） |

### シェル機能の利用

```yaml
- run: echo "${PASSWORD:0:1} ${PASSWORD#?}"  # ✅ シェルの文字列操作が使える
- run: echo "${{ env.PASSWORD:0:1 }}"        # ❌ これはできない
```

## 実用上の使い分け

```yaml
jobs:
  example:
    env:
      MY_VAR: "hello"
    steps:
      # ✅ シェルスクリプト内ならシェル変数が簡潔
      - run: echo "${MY_VAR}"

      # ✅ 条件式など、YAML 属性では ${{ }} が必須
      - name: Conditional step
        if: env.MY_VAR == 'hello'
        run: echo "matched"

      # ✅ アクションのパラメータでも ${{ }} が必須
      - uses: actions/some-action@v1
        with:
          value: ${{ env.MY_VAR }}

      # ✅ 複雑なシェルスクリプトではシェル変数が便利
      - run: |
          if [ "${MY_VAR}" = "hello" ]; then
            echo "Hello World"
          fi
```

## まとめ

- **`run:` の中**: `${PASSWORD}` の方がシンプルで推奨
- **YAML 属性** (`if:`, `with:`, `name:` など): `${{ env.PASSWORD }}` が必須
- **シェル機能を使いたい場合**: `${PASSWORD}` を使う
- **エラー防止**: `${{PASSWORD}}` のようなコンテキスト指定なしは不可

## 参考

- ワークフローファイル: `.github/workflows/secret-print.yml`
- 発生したエラー: "Unrecognized named-value: 'PASSWORD'"
