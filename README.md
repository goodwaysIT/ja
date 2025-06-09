# Goodways ITテクニカルハブ

## 概要

Goodways ITテクニカルハブへようこそ！当サイトは、Goodways ITチームの公式技術ナレッジベースおよびブログプラットフォームです。私たちは、Oracle Databaseお・Oracle GoldenGateソリューションに特化した技術コンサルティング会社です。私たちの使命は、Oracle技術を活用して、組織のデータインフラストラクチャの最適化、高可用性の確保、データ統合プロセスの効率化を支援することです。

このプラットフォームでは、私たちの専門知識、洞察、オープンソースへの貢献、およびプロフェッショナルサービスに関する詳細を共有しています。

## 主な機能

*   **詳細な技術記事**: Oracle Database管理、Oracle GoldenGate、高可用性、パフォーマンスチューニング、データ移行などを網羅したブログ記事をご覧ください。
*   **オープンソースツール**: Oracle環境向けに設計された、以下を含む私たちのオープンソースツール群について学べます：
    *   `inspect4oracle`: 軽量Oracleデータベースヘルスチェックツール。
    *   `adg-dashboard`: プロフェッショナルOracle Active Data Guard監視ダッシュボード。
    *   `oracle-adgmgr`: Oracle Active Data Guardスイッチオーバー管理プラットフォーム。
    *   `ggutil`: GoldenGate Classic Editionマルチインスタンス管理ツール。
*   **プロフェッショナルサービス**: 以下を含む私たちの幅広いサービスをご覧ください：
    *   Oracle Database管理（セットアップ、最適化、HA、監視）。
    *   Oracle GoldenGateソリューション（アーキテクチャ、実装、チューニング、サポート）。
    *   私たちのオープンソースツールのカスタマイズ。
*   **検索機能**: ニーズに合った記事や情報を簡単に見つけることができます。

## ローカル開発とはじめに

このJekyllサイトをローカルで実行するには、RubyとBundlerがインストールされている必要があります。このサイトはGitHub Pagesと互換性があるように構築されています。

### 前提条件

*   **Ruby**: バージョン `3.3.4` ([GitHub Pagesの依存関係バージョン](https://pages.github.com/versions/) に準拠)
    *   Rubyのバージョン管理には、`rbenv` や `rvm` のようなRubyバージョンマネージャーの使用をお勧めします。
*   **Bundler**: プロジェクトの依存関係を管理するRuby gemです。`gem install bundler` でインストールしてください。

### インストールとセットアップ

1.  **リポジトリをクローンする:**
    ```bash
    git clone https://github.com/goodwaysit/en.git
    cd en
    ```

2.  **依存関係をインストールする:**
    `en` ディレクトリ（またはJekyllプロジェクトのルートディレクトリが異なる構造の場合はそちら）に移動し、以下を実行します：
    ```bash
    bundle install
    ```
    このコマンドは、`Gemfile` と `Gemfile.lock` で指定されているJekyll（バージョン `3.10.0`）およびその他の必要なgemをインストールします。

### サイトをローカルで実行する

依存関係がインストールされたら、サイトをローカルでサーブできます：

```bash
bundle exec jekyll serve
```

デフォルトでは、サイトは `http://localhost:4000` で利用可能になります。

## コンテンツ投稿ガイドライン

私たちは知識の共有と貢献を歓迎します。新しいブログ記事を追加したい場合は、以下のガイドラインに従ってください：

1.  **ディレクトリ**: 新しいMarkdownファイルを `_posts` ディレクトリに作成して下さい。
2.  **ファイル名形式**: ファイル名には `YYYY-MM-DD-your-post-title.md` 形式を使用ししてください。
    *   例: `2025-06-06-optimizing-oracle-performance.md`
3.  **Front Matter**: 各投稿には適切なYAML Front Matterを含めるようにしてください。基本的なテンプレートは以下の通りです：

    ```yaml
    ---
    layout: post
    title: "魅力的な投稿タイトル"
    excerpt: "一覧ページに表示される投稿の簡単な要約またはティーザー。"
    date: YYYY-MM-DD HH:MM:SS +ZZZZ # 例: 2025-06-06 10:00:00 +0800
    categories: [関連, カテゴリー] # 例: [Oracle, GoldenGate]
    tags: [関連, タグ, 検索用] # 例: [oracle, performance, tuning]
    # image: /assets/images/posts/your-image-name.jpg # オプション: 関連画像へのパス
    ---

    Markdown形式の投稿内容はここから始めます...
    ```

## お問い合わせ

Goodways ITチームは、OracleデータベースおよびGoldenGateに関する最適なソリューションをご提案いたします。気になることやお困りのことがあれば、お気軽に[ご連絡ください](https://it.goodways.co.jp/contact/)
