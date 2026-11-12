---
title: GitHubにPushするだけでQiitaへ自動投稿する方法【Qiita CLI × GitHub Actions】
tags:
  - QiitaCLI
  - GitHub
  - GitHubActions
  - CI
  - Markdown
private: false
updated_at: '2026-11-12T22:05:05+09:00'
id: 5f0a935c7c2a6e856151
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

# はじめに

Qiitaで技術記事を執筆する際、記事をGitHubで管理し、ローカル環境から自動投稿できる仕組みを構築しました。

今回実現したかったのは、以下の3点です。

- 記事をMarkdown形式で管理し、GitHubで変更履歴を残す
- VS Codeなどのエディタで記事を執筆する
- GitHubにPushするだけでQiitaへ自動投稿・更新する

これらは、Qiita公式の**Qiita CLI**と**GitHub Actions**を組み合わせることで実現できます。

本記事では、実際に構築した手順に加えて、途中で遭遇したエラーや、既存記事をGitHubで管理する際の注意点についても解説します。

# 1. 全体構成

今回構築する仕組みは以下のとおりです。

```text
【ローカルPC】
  VS CodeでMarkdown記事を作成・編集
           │
           │ git push
           ▼
【GitHub】
  記事のソースコードを管理
           │
           │ mainブランチへのPush
           ▼
【GitHub Actions】
  Qiita CLIを自動実行
           │
           │ Qiita API
           ▼
【Qiita】
  記事を新規投稿・更新
           │
           │ 記事IDや更新日時を反映
           ▼
【GitHub】
  Actionsが変更を自動コミット
           │
           │ git pull
           ▼
【ローカルPC】
  記事IDなどを同期
```

使用する技術は以下のとおりです。

| 技術 | 用途 |
|---|---|
| Git | ローカルでの記事のバージョン管理 |
| GitHub | 記事の保管・変更履歴管理 |
| Qiita CLI | 記事の作成・プレビュー・投稿 |
| GitHub Actions | 投稿処理の自動化 |
| Qiita API | Qiitaへの記事の投稿・更新 |

GitHub Actionsを使用するため、投稿用にサーバーを用意する必要はありません。

# 2. Qiita CLIのインストール

## 2.1 Node.jsの確認

Qiita CLIを利用するには、Node.jsが必要です。

Qiita CLIの公式ドキュメントでは、Node.js 22.22.1以上が要求されています。

まずはインストール状況を確認します。

```bash
node -v
npm -v
```

Node.jsがインストールされていない場合は、[Node.js公式サイト](https://nodejs.org/)などから導入してください。

## 2.2 作業ディレクトリの作成

記事を管理するディレクトリを作成します。

```bash
mkdir qiita-articles
cd qiita-articles
```

Node.jsのプロジェクトを初期化します。

```bash
npm init -y
```

続いて、Qiita CLIをインストールします。

```bash
npm install @qiita/qiita-cli --save-dev
```

インストール後、バージョンを確認します。

```bash
npx qiita version
```

## 2.3 Qiita CLIの初期化

以下のコマンドを実行します。

```bash
npx qiita init
```

初期化すると、主に次のファイルが生成されます。

```text
qiita-articles/
├── .github/
│   └── workflows/
│       └── publish.yml
├── node_modules/
├── .gitignore
├── package.json
├── package-lock.json
└── qiita.config.json
```

それぞれの役割は以下のとおりです。

| ファイル | 役割 |
|---|---|
| `publish.yml` | GitHub Actionsのワークフロー |
| `.gitignore` | Gitの管理対象外ファイルの指定 |
| `package.json` | Node.jsプロジェクトの設定 |
| `package-lock.json` | 依存パッケージのバージョン情報 |
| `qiita.config.json` | Qiita CLIの設定 |

特に重要なのが、`.github/workflows/publish.yml`です。

このファイルによって、GitHubへのPushをきっかけにQiitaへの自動投稿が実行されます。

# 3. Qiitaのアクセストークンを取得

GitHub ActionsからQiitaへ記事を投稿するには、Qiita APIの認証が必要です。

まず、以下のページでアクセストークンを発行します。

[Qiita アクセストークン発行ページ](https://qiita.com/settings/tokens/new)

権限には以下の2つを指定します。

- `read_qiita`：記事などの読み取り
- `write_qiita`：記事の投稿・更新

発行したトークンは安全な場所に保管してください。

**アクセストークンをGitHubのリポジトリに直接記載してはいけません。**

## 3.1 ローカル環境でログイン

Qiita CLIにログインします。

```bash
npx qiita login
```

トークンの入力を求められるので、発行したアクセストークンを入力します。

正常に認証されれば、ローカル環境からQiita CLIを使用できるようになります。

なお、このログイン操作はローカルPC用の認証です。

GitHub Actions側には、別途GitHub Secretsを登録する必要があります。

# 4. GitHubリポジトリの作成

GitHubで記事管理用のリポジトリを作成します。

今回は以下の名前とします。

```text
qiita-articles
```

[GitHubの新規リポジトリ作成画面](https://github.com/new)から作成します。

リポジトリ作成後、ローカルで以下を実行します。

```bash
git init
git branch -M main
git add .
git commit -m "Initial Qiita setup"
```

続いて、GitHubのリモートリポジトリを登録します。

```bash
git remote add origin https://github.com/USERNAME/qiita-articles.git
```

`USERNAME`は自分のGitHubユーザー名に置き換えてください。

最後にPushします。

```bash
git push -u origin main
```

これでGitHubへのアップロードが完了します。

## 補足：origin already existsが発生した場合

すでに`origin`が登録されている場合、次のエラーが発生します。

```text
error: remote origin already exists.
```

その場合は、`git remote add`ではなく、以下のコマンドで既存のURLを変更します。

```bash
git remote set-url origin https://github.com/USERNAME/qiita-articles.git
```

登録されたURLは以下で確認できます。

```bash
git remote -v
```

# 5. GitHub Secretsにトークンを登録

ここが認証設定で最も重要な部分です。

GitHubリポジトリの以下の画面にアクセスします。

`Settings` → `Secrets and variables` → `Actions`

`New repository secret`を選択し、以下の情報を登録します。

| 項目 | 設定内容 |
|---|---|
| Name | `QIITA_TOKEN` |
| Secret | Qiitaで発行したアクセストークン |

`QIITA_TOKEN`という名前は正確に指定してください。

GitHub Actionsは、以下の記述でSecretを参照します。

```yaml
qiita-token: ${{ secrets.QIITA_TOKEN }}
```

Secretに登録することで、アクセストークンをリポジトリ内のコードに直接書かずに認証できます。

# 6. GitHub Actionsの設定

`npx qiita init`を実行すると、投稿用のワークフローが自動生成されます。

ファイルの配置場所は以下のとおりです。

```text
.github/workflows/publish.yml
```

今回使用する設定例です。

```yaml
name: Publish articles

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: write

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false

jobs:
  publish_articles:
    runs-on: ubuntu-latest
    timeout-minutes: 5

    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0

      - uses: increments/qiita-cli/actions/publish@v1
        with:
          qiita-token: ${{ secrets.QIITA_TOKEN }}
          root: "."
```

重要な設定を順番に説明します。

## 6.1 自動実行のトリガー

```yaml
on:
  push:
    branches:
      - main
```

`main`ブランチにPushされたタイミングでワークフローが起動します。

つまり、ローカルから次のコマンドを実行すると、自動投稿処理が開始されます。

```bash
git push origin main
```

## 6.2 GitHubリポジトリの取得

```yaml
- uses: actions/checkout@v7
  with:
    fetch-depth: 0
```

GitHub Actionsの実行環境にリポジトリをチェックアウトします。

`fetch-depth: 0`を指定することで、Gitの履歴をすべて取得します。

## 6.3 Qiitaへの自動投稿

```yaml
- uses: increments/qiita-cli/actions/publish@v1
  with:
    qiita-token: ${{ secrets.QIITA_TOKEN }}
    root: "."
```

Qiita公式のGitHub Actionを利用しています。

このActionの内部では、Qiita CLIを使用して以下のコマンドが実行されます。

```bash
qiita publish --all --root .
```

これにより、`public/`ディレクトリ内の投稿対象記事がQiitaに反映されます。

## 6.4 contents: writeの役割

```yaml
permissions:
  contents: write
```

Qiita CLIは投稿後に、記事のIDや更新日時をMarkdownファイルに反映します。

公式Actionは、その変更をGitHubへ自動コミットします。

この処理に必要なのが`contents: write`権限です。

# 7. 記事を作成して自動投稿する

ここまでの設定が完了したら、実際に記事を作成します。

## 7.1 新規記事の作成

ローカルで以下を実行します。

```bash
npx qiita new sample-article
```

次のファイルが生成されます。

```text
public/sample-article.md
```

ファイルを開き、記事のタイトルや本文を編集します。

```markdown
---
title: "Qiita CLIの動作確認"
tags:
  - QiitaCLI
  - GitHubActions
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに

GitHub ActionsからQiitaへの
自動投稿を確認するための記事です。
```

実際には、CLIが生成したFront Matterを保持したまま編集します。

特に重要なのが以下の設定です。

```yaml
id: null
ignorePublish: false
```

`id: null`は、まだQiitaの記事IDが設定されていない状態です。

`ignorePublish: false`は、Qiita CLIの投稿処理の対象にする設定です。

## 7.2 ローカルでプレビュー

以下のコマンドを実行します。

```bash
npx qiita preview
```

ブラウザから次のURLにアクセスします。

```text
http://localhost:8888
```

Qiitaに近い表示で記事を確認できます。

なお、`qiita preview`はQiita上に存在する記事をローカルへ取得する動作も行うため、実行後に`public/`内のファイルが増えることがあります。

## 7.3 GitHubへPush

記事が完成したらコミットします。

```bash
git add public/sample-article.md
git commit -m "Add sample article"
git push origin main
```

これだけでGitHub Actionsが起動します。

GitHubのリポジトリから`Actions`タブを開くと、ワークフローの実行状況を確認できます。

正常終了すると、Qiitaへの投稿処理が完了します。

# 8. 投稿後になぜgit pullが必要なのか？

今回の仕組みで理解しておきたいのが、投稿後のGitHubとローカルの状態の違いです。

新規記事の作成時、Markdownには以下のように記載されています。

```yaml
id: null
updated_at: ""
```

GitHub ActionsがQiitaへの投稿を実行すると、Qiita側で記事IDが発行されます。

その後、GitHub ActionsがMarkdownを更新します。

```yaml
id: "発行された記事ID"
updated_at: "更新日時"
```

これらの変更はGitHub上でコミットされますが、ローカルPCにはまだ反映されていません。

そこで、次のコマンドを実行します。

```bash
git pull origin main
```

これにより、GitHub Actionsが更新した記事IDや更新日時をローカルにも反映できます。

記事IDは既存記事を識別するために重要です。

新規投稿後に`id: null`のまま編集を続けると、重複投稿などの原因になり得ます。

そのため、**新しい記事を初めて投稿した後は、次の編集を始める前にPullしておくことが重要です。**

# 9. 既存記事をGitHubで管理する場合の注意点

Qiita CLIでは、すでにQiitaへ投稿している記事も取得できます。

```bash
npx qiita pull
```

このコマンドを実行すると、既存記事がMarkdownファイルとして`public/`配下に保存されます。

例えば、以下のような構成になります。

```text
public/
├── 00be4ec4f66953f88e75.md
├── 03b184ce60d60223c11f.md
├── sample-article.md
└── ...
```

既存記事には、Qiitaで発行済みの記事IDが設定されています。

```yaml
id: 00be4ec4f66953f88e75
ignorePublish: false
```

ただし、`ignorePublish: false`のままだと、`qiita publish --all`の投稿対象になります。

Qiita CLIには、ローカルの内容がQiita上の記事より古い可能性がある場合に投稿を停止する検証機能があります。

それでも、既存記事を意図せず自動更新することを避けたい場合は、投稿対象から明示的に除外すると安全です。

```yaml
ignorePublish: true
```

この設定にすると、GitHubでファイルを管理しながら、Qiita CLIによる投稿処理から除外できます。

また、GitHubリポジトリをPublicにする場合、限定共有記事や公開したくない情報を含めないよう注意が必要です。

# 10. 実際に遭遇したエラーと対処方法

## 10.1 credentials.jsonが存在しない

GitHub Actionsの実行時、以下のエラーが発生しました。

```text
Error: ENOENT: no such file or directory,
open '/home/runner/.config/qiita-cli/credentials.json'
```

これは、Qiita CLIが認証情報を取得できなかった際に発生することがあるエラーです。

今回の構成では、GitHub Actionsは`QIITA_TOKEN`環境変数を使用するため、通常は実行環境に`credentials.json`を用意する必要はありません。

まず確認するべきなのは、GitHub Secretsです。

```text
Repository
  └── Settings
       └── Secrets and variables
            └── Actions
                 └── QIITA_TOKEN
```

Secretの名前や設定値に誤りがないか確認します。

修正後、GitHub Actionsの画面から失敗したジョブを再実行できます。

## 10.2 Repository not found

GitHubへPushする際、次のエラーが発生することがあります。

```text
remote: Repository not found.
fatal: repository not found
```

リモートURLが正しいか確認します。

```bash
git remote -v
```

URLが間違っている場合は以下で修正します。

```bash
git remote set-url origin https://github.com/USERNAME/qiita-articles.git
```

リポジトリが作成済みであることや、アクセス権限も確認してください。

## 10.3 GitHub Actions実行後にローカルと差分が発生する

GitHub Actionsが記事IDや更新日時をコミットするため、ローカルとGitHubで差分が発生することがあります。

ローカルに未コミットの変更がない状態で、以下を実行して同期します。

```bash
git pull origin main
```

なお、PullはQiitaへ投稿するための操作ではありません。

あくまでGitHub上の更新内容をローカルへ反映するための操作です。

# 11. 今後の運用方法

最終的に、記事の執筆から投稿までの基本的な流れは以下になります。

## 作業開始時

```bash
git pull origin main
```

## 新規記事の作成

```bash
npx qiita new article-name
```

## 記事の編集

```bash
code public/article-name.md
```

## プレビュー

```bash
npx qiita preview
```

## 自動投稿

```bash
git add public/article-name.md
git commit -m "Add new article"
git push origin main
```

GitHub Actionsが正常終了した後、必要に応じて次のコマンドで同期します。

```bash
git pull origin main
```

記事を更新する場合も、基本的には同様です。

また、GitHubのブランチ運用と組み合わせれば、記事をPull Requestでレビューしてから公開することも可能です。

# まとめ

Qiita CLIとGitHub Actionsを組み合わせることで、記事の作成・管理・投稿をGitHub中心のワークフローに統一できました。

今回の構築で重要だった点は以下のとおりです。

1. Qiita CLIを使用すると、記事をMarkdownファイルとして管理できる
2. GitHub Actionsによって、Pushをきっかけに自動投稿できる
3. GitHub Secretsを使用してアクセストークンを安全に管理する
4. `ignorePublish`によって記事ごとの投稿対象を制御できる
5. 投稿後に更新された記事IDをGitHubからPullすることで、ローカルとの整合性を維持できる

個人的には、単に投稿作業を自動化できることだけでなく、**記事をGitでバージョン管理し、変更履歴を追跡できること**が大きなメリットだと感じました。

今後は、GitHub Actionsを拡張してMarkdownの構文チェックやPull Request時の自動レビューなども組み込んでみたいと考えています。

# 参考資料

- [Qiita CLI公式GitHub](https://github.com/increments/qiita-cli)
- [Qiita CLI Publish Action](https://github.com/increments/qiita-cli/tree/main/actions/publish)
- [GitHub Actions公式ドキュメント](https://docs.github.com/ja/actions)
- [Qiitaアクセストークン発行ページ](https://qiita.com/settings/tokens/new)
