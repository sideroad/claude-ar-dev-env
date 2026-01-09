# PR作成

GitHub PRを作成します。

## 引数

$ARGUMENTS にタイトルが指定されている場合はそれを使用。

## 実行手順

1. 現在のブランチとステータスを確認：

```bash
git branch --show-current
git status --short
```

2. 未コミットの変更があれば確認：
   - コミットするか
   - 変更内容の確認

3. PR情報を確認（引数がない場合）：
   - タイトル
   - ベースブランチ（デフォルト: main または develop）
   - ドラフトかどうか

4. PR作成：

```bash
TITLE="PRタイトル"
BASE="main"  # または develop
BODY=""  # 必要に応じて

# ドラフトの場合
gh pr create --title "${TITLE}" --base ${BASE} --body "${BODY}" --draft

# 通常の場合
gh pr create --title "${TITLE}" --base ${BASE} --body "${BODY}"
```

5. 作成結果を報告：

```bash
# PR番号とURLを表示
gh pr view --web
```

## オプション

- `--draft`: ドラフトPRとして作成
- `--reviewer @username`: レビュアーを指定
- `--label bug,enhancement`: ラベルを指定
