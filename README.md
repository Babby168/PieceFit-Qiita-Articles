# PieceFit Qiita Articles

PieceFit の実装を振り返る Qiita 記事を、ローカルで書いて GitHub 経由で投稿するためのリポジトリです。

- GitHub: https://github.com/Babby168/PieceFit-Qiita-Articles
- アプリ本体: https://github.com/Babby168/PieceFit（このディレクトリは親リポジトリでは gitignore しています）

---

## 構成

```
qiita-articles/
├── .github/workflows/publish.yml  # main への push で Qiita へ公開
├── public/                        # 記事 Markdown を置く場所
├── qiita.config.json              # プレビューの host / port
├── package.json                   # @qiita/qiita-cli
└── .qiita-config/                 # ローカル認証の退避先（gitignore）
```

親の PieceFit 側で必要なもの:

| 場所 | 役割 |
| --- | --- |
| `compose.yml` の `QIITA_TOKEN` | コンテナ内の CLI にトークンを渡す |
| `compose.yml` の `8888:8888` | ホストのブラウザからプレビューする |
| `compose.yml` の `.qiita-config` マウント | `login` した場合の認証ファイルを永続化 |
| ルート `.env` の `QIITA_TOKEN` | Compose が `${QIITA_TOKEN}` を展開する元。コミットしない |
| `.gitignore` の `/qiita-articles/` | 記事は別リポジトリとして管理する |

---

## 認証の仕組み（ここが一番大事）

Qiita CLI には **2 種類の認証経路** がある。混ぜると混乱する。

```
QIITA_TOKEN がプロセスにある
        │
        ├─ ある → preview / publish / pull はこれを使う（login 不要）
        │
        └─ ない → ~/.config/qiita-cli/credentials.json を読む
                   （npx qiita login が書き込むファイル）
```

### `npx qiita login` は毎回トークンを聞く

これは仕様。`login` コマンドは入力したトークンを `credentials.json` に保存するだけで、環境変数があってもプロンプトは消えない。

「毎回入力したくない」の正解は、**毎回 login しないこと**。`QIITA_TOKEN` をコンテナに渡せば `preview` と `publish` だけで足りる。

### 環境変数を置いたのに効かなかった理由

1. CLI は `QIITA_TOKEN` だけ見る。Rails 用の別の変数名では無効。
2. CLI 内蔵の `dotenv` は **カレントディレクトリの `.env`** しか読まない。`qiita-articles/` で実行すると、PieceFit ルートの `.env` は読まれない。
3. Docker Compose は `.env` の値を **自動ではコンテナに入れない**。`environment` に `QIITA_TOKEN: ${QIITA_TOKEN}` と書いたものだけが入る。

今回の経路:

```
PieceFit/.env の QIITA_TOKEN
    → compose.yml が web コンテナへ注入
    → コンテナ内で npx qiita preview
    → プロセスの環境変数をそのまま使う（cwd の .env に依存しない）
```

`.env` を変えたあとはコンテナの作り直しが必要。

```bash
docker compose up -d --force-recreate web
```

GitHub Actions では、リポジトリ Secret の `QIITA_TOKEN` を `increments/qiita-cli/actions/publish` が同じ環境変数名で渡す。

---

## プレビューが Connection Failed だった理由

`qiita.config.json` のデフォルトは `"host": "localhost"`。

コンテナ内で localhost に bind すると、ホストから `8888:8888` でポートを開けていても届かない。コンテナのループバックはホストのループバックとは別物だから。

対処:

```json
{
  "includePrivate": false,
  "host": "0.0.0.0",
  "port": 8888
}
```

`0.0.0.0` は「コンテナの全インターフェースで待つ」という意味。ブラウザはホスト側で `http://localhost:8888` を開く。

---

## 日常の使い方

コンテナ内で実行する。ホストの Node ではなく、PieceFit の `web` を使う。

```bash
docker compose exec web bash
cd qiita-articles
npx qiita preview          # http://localhost:8888
npx qiita new 記事のファイル名
```

記事ファイルは `public/*.md`。Front Matter の `ignorePublish: true` にすると、`publish` 対象外になる。

投稿の流れ:

1. プレビューで確認する
2. このリポジトリの `main` に push する
3. GitHub Actions が `qiita publish --all` を実行する

ローカルから直接投稿する場合:

```bash
npx qiita publish 記事のファイル名
# または
npx qiita publish --all
```

---

## GitHub Actions の初回失敗について

`public/` が空の状態で first commit したため、Publish articles は失敗している。

これは Secret や workflow の接続失敗ではない。トークンがリポジトリに渡り、ジョブが走った証拠。記事を置いて `main` へ push したときに成功すればよい。空のまま workflow を成功させる修正はしていない。

---

## 入れ子リポジトリにした理由

Qiita CLI 公式は「記事ディレクトリをリポジトリのルートにする」前提。`.github/workflows/publish.yml` は **そのリポジトリのルート** にないと動かない。

PieceFit 本体の中に置きつつ別管理にするため:

- このディレクトリ自身が git リポジトリ
- 親の PieceFit は `/qiita-articles/` を gitignore
- 認証ファイル（`.qiita-config/`, `credentials.json`）も gitignore

親リポジトリで `git status` しても、ここの変更は出ない。記事の commit / push はこのディレクトリで行う。

```bash
cd qiita-articles
git status
git add public/*.md
git commit -m "Add article"
git push origin main
```

---

## 振り返り用チェックリスト

後から「なぜそうなったか」を確認するときの観点:

- [ ] `login` と `QIITA_TOKEN` は別物だと説明できるか
- [ ] Compose の `${QIITA_TOKEN}` とコンテナの環境変数の関係を説明できるか
- [ ] なぜ `host: 0.0.0.0` でないとホストのブラウザから見えないか
- [ ] なぜ PieceFit 本体に workflow を置いても動かないか
- [ ] 空の `public/` で Actions が赤くなる理由を説明できるか
