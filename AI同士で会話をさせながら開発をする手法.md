タミさんのポートフォリオ用に「タスク管理ツール」を題材にして、Agent Teamsで実装する具体的な流れを説明します。

## 前提: 要件を先に固める

Agent Teamsは並列作業が得意な反面、要件が曖昧だとチームメイト間で認識がズレます。着手前に簡単な仕様メモを作っておきます。

```markdown
# spec.md

## 機能要件
- タスクのCRUD(作成・一覧・更新・削除)
- ステータス管理(未着手/進行中/完了)
- 認証(Cognito想定)

## 技術スタック
- フロント: React + TypeScript
- API: API Gateway + Lambda
- DB: DynamoDB
- IaC: Terraform

## 非機能要件
- README にアーキテクチャ図と設計意図を必ず記載
- デプロイ済みURLを残す(ポートフォリオ用の必須要件)
```

## ステップ1: プロジェクトの下準備

```bash
mkdir task-manager-portfolio && cd task-manager-portfolio
git init
```

`CLAUDE.md` に、チーム全体で守るべきルールを書いておきます(前回話した安全ガードレールと同じ考え方です)。

```markdown
# CLAUDE.md

## チーム運用ルール
- 各タスク完了時にgit commit(1タスク=1コミット)
- 本番デプロイ・破壊的な変更(terraform destroy等)は行わない
- 設計判断で迷った場合は推測で進めず、shared task listに疑問点を残す
- reviewerはファイル書き込み禁止(指摘のみ、修正はdevが行う)
```

## ステップ2: Agent Teamsを有効化

```json
// ~/.claude/settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

`tmux`があると各エージェントが別ペインに表示されて見やすくなります(必須ではありません)。

```bash
brew install tmux   # 未インストールの場合
tmux new -s agent-team
```

## ステップ3: 権限設定を先に済ませておく

Agent Teamsで一番つまずきやすいのが権限プロンプトです。チームメイトが承認待ちで止まると、リードが気づかず放置されることがあります。事前にallowlistを設定しておきます。

```json
// .claude/settings.json (プロジェクト単位)
{
  "permissions": {
    "allow": [
      "Read", "Write", "Edit", "Glob", "Grep",
      "Bash(npm *)", "Bash(git *)", "Bash(terraform plan)",
      "Bash(terraform validate)"
    ],
    "deny": [
      "Bash(terraform destroy)",
      "Bash(terraform apply)"
    ]
  }
}
```

`terraform apply`は明示的に人間の確認を挟みたい操作なので、あえてdenyに入れて「チームが自動で本番反映しない」ようにしておくと安心です。

## ステップ4: Claude Codeを起動し、チーム編成を指示する

```bash
cd task-manager-portfolio
claude
```

プロンプト例:

```
spec.mdを読んで、タスク管理ツールを実装するチームを立てて。

・design: spec.mdをもとにAPI設計(エンドポイント、DynamoDBのテーブル設計、
  IAMロール設計)を行い、design.mdに書き出す。実装はしない。
・dev: designが書いたdesign.mdに基づいて実装する(フロント・API・Terraform)。
  design.mdにない判断が必要な場合は、designに直接メッセージで確認すること。
・reviewer: devが実装したコードをセキュリティ(最小権限のIAM、
  入力バリデーション)とコスト(DynamoDBのキャパシティモード等)の
  観点でレビューする。読み取り専用ツールのみ使用し、指摘はdevに
  直接メッセージで送ること。

進め方:
1. designがdesign.mdを完成させ、共有タスクリストに実装タスクを積む
2. devがタスクを1つずつ取り、実装→コミット→タスク完了
3. reviewerは各タスク完了のたびにレビューし、指摘があればdevに送る
4. 全タスク完了後、READMEにアーキテクチャ図と設計意図を書く担当を決めて仕上げる
```

## ステップ5: 進行を観察する

- `tmux`使用時は`Shift+Up/Down`でチームメイトのペインを切り替えて確認できます
- リードのターミナルにエージェントパネルが表示され、各チームメイトの状況がわかります
- 個別のエージェントに直接話しかけることも可能です(例: 「devに直接: DynamoDBのパーティションキー設計、user_idでいい?」)

## ステップ6: 詰まった時の対処

経験的によくある詰まりポイントです。

| 症状 | 対処 |
|---|---|
| タスクが完了扱いにならず後続がブロックされる | 実際に作業が終わっているか確認し、手動でタスクステータスを更新するかリードに催促 |
| チームメイトがメッセージに応答しない | リードに「該当のチームメイトを再生成して」と依頼 |
| reviewerが実装まで始めてしまう | CLAUDE.mdのルールを再度明示し、権限設定でWrite系ツールを外す |

## ステップ7: 完了後の確認

全タスク完了後、以下を必ず人間側でチェックします。

- `git log` でコミット単位が意図通り分割されているか
- `terraform plan` の内容(applyは自分で実行)
- READMEにアーキテクチャ図・設計意図・デプロイURLが入っているか(タミさんの過去のポートフォリオレビューで指摘された点です)

## 補足: 最初は小さく試す

いきなりタスク管理ツール全体を投げるより、まずは「API 1エンドポイント分の設計→実装→レビュー」くらいの小さいスコープで一度通して、チーム運用の癖を掴んでから範囲を広げるのがおすすめです。Agent Teamsはトークン消費が単一セッションより多くなるので、コスト感覚を掴む意味でも小さく始めるのが安全です。
