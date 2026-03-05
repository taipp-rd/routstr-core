# サブドメインで Routstr を公開する手順（bulbulpp.com の場合）

LN Address で使っている **bulbulpp.com** のサブドメイン（例: **routstr.bulbulpp.com**）で、Routstr を 80/443 だけ開放して公開する手順です。  
**8000 番ポートは外に開けず、nginx が中継するので安全です。**

---

## 全体のイメージ

```
インターネット
    │
    │  https://routstr.bulbulpp.com  (443番だけ開放)
    ▼
┌─────────────────┐
│  nginx           │  ← ここで SSL 終端＆中継
│  (80 / 443)      │
└────────┬────────┘
         │  http://localhost:8000  (外から見えない)
         ▼
┌─────────────────┐
│  Routstr (Docker)│  ポート 8000 は「このサーバー内」だけ
└─────────────────┘
```

- **外から見えるのは 80 と 443 だけ** → ファイアウォールで 8000 は開けない。
- **Routstr は localhost:8000** → nginx が「中継役」になって、外からの 443 を 8000 に渡す。

---

## 前提

- **Routstr を動かすサーバー**が 1 台ある（VPS や自宅サーバーなど）。
- そのサーバーに **SSH でログイン**できる。
- **bulbulpp.com の DNS を変更できる**（お名前.com や Cloudflare などで管理している想定）。

---

## このドキュメントで想定している VPS（LNVPS #1035）

本手順では **LNVPS #1035**（`vm-1035.lnvps.cloud`）を例にしています。同じ VM で bulbulpp.com / lnbits を運用している想定です。

| 項目 | 値 |
|------|-----|
| **ホスト名** | vm-1035.lnvps.cloud |
| **IPv4** | 185.18.221.177 |
| **SSH ユーザー** | ubuntu |
| **SSH ログイン** | `ssh ubuntu@vm-1035.lnvps.cloud` |
| **リージョン** | Dublin (IE) |
| **スペック** | 4 vCPU, 8GB RAM, 160GB SSD |

DNS の A レコードで「routstr.bulbulpp.com → このサーバー」とするときの **値（ポイント先）** は **185.18.221.177** です。

---

## 既存の VM（LNVPS など）で別プロジェクトと同居させる場合

**同じ VM に Routstr を追加して問題ありません。** よくあるパターンです。

- **ポートの取り合いにならない**: Routstr は **8000 番**をサーバー内だけで使います。既存の LN ノード（9730 など）や Web（80/443）とは別です。nginx が 80/443 を受け、**サブドメイン（routstr.bulbulpp.com）ごとに** どのサービスに渡すかを振り分けます。
- **nginx が既にある場合**: すでに nginx と certbot で bulbulpp.com やサブドメインを運用していれば、**新しい server ブロックを 1 つ追加**するだけで、routstr.bulbulpp.com を Routstr(8000) に割り当てられます。ステップ 2〜3 の「nginx インストール」「certbot 初回」は飛ばして、ステップ 4 の設定追加だけ行えばよいです。
- **リソース**: 同じ VM で複数サービスを動かすので、**メモリ・CPU に余裕があるか**だけ確認してください。Routstr（Docker + FastAPI）は軽めですが、既存の LN ノードなどと合算して、スワップが出ない程度が目安です。
- **セキュリティ**: 1 台の VM で複数サービスを動かす場合、どれか 1 つが侵入されると他にも影響し得ます。OS とソフトの更新、ファイアウォール（80/443 のみ開放）、強いパスワード・鍵運用はこれまで通り行ってください。

---

## ノート PC から VPS へ移して「年中稼働」にする

**ノート PC の Docker で動かしていると、PC の電源を落としたらノードも止まります。** 第三者に使ってもらうには、**LNVPS 上で Routstr を動かす**と年中稼働になります。

### やることの流れ（nginx は既に bulbulpp.com / lnbits で運用中の場合）

1. **LNVPS で Routstr を動かす**（下記「VPS で Routstr を起動する」）
2. **DNS**: routstr.bulbulpp.com を LNVPS の IP に向ける（ステップ 1）
3. **nginx**: routstr.bulbulpp.com 用の server ブロックを 1 つ追加（ステップ 4）、certbot で証明書取得（ステップ 3）
4. **ダッシュボード**: ブラウザで **https://routstr.bulbulpp.com/admin** を開く
5. **第三者に使ってもらう**: ダッシュボードで HTTP URL を設定し、相手に「接続先 URL」と「API キーの取り方」を伝える（下記「第三者に使ってもらう設定」）

### ダッシュボードはどこで見るか

Routstr を **https://routstr.bulbulpp.com** で公開した場合：

| 用途 | URL |
|------|-----|
| **管理ダッシュボード** | **https://routstr.bulbulpp.com/admin** |
| API 情報（動作確認） | https://routstr.bulbulpp.com/v1/info |
| ログイン画面 | https://routstr.bulbulpp.com/login |

`.env` の `ADMIN_PASSWORD` でログインします。

### 第三者に使ってもらう設定

1. **ダッシュボード**（https://routstr.bulbulpp.com/admin）で **Settings** → **HTTP URL** に `https://routstr.bulbulpp.com` を設定する。
2. **相手に伝えること**:
   - **接続先（Base URL）**: `https://routstr.bulbulpp.com/v1`
   - **API キーの取り方**: ノードの Lightning インボイスで支払ってセッションキー（`sk-...`）を取得するか、Cashu トークン（`cashuA...`）をそのノードで使う。  
   （Routstr のクライアント向けドキュメントや、ノードのトップページ・インボイス発行ページの URL を共有するとよいです。）

---

## VPS で Routstr を起動する（LNVPS で年中稼働）

LNVPS に SSH ログインして、次の順で行います。

```bash
# ログイン（LNVPS #1035 の場合）
ssh ubuntu@vm-1035.lnvps.cloud
```

### 1. Docker が入っているか確認

```bash
docker --version
```

無ければインストール（Ubuntu/Debian の例）:

```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker
# 自分のユーザーで docker を叩く場合
sudo usermod -aG docker $USER
# いったんログアウトして再ログインするか、以下は sudo docker で実行
```

### 2. プロジェクトと .env を用意する

- **オプション A（新規）**: VPS に `routstr-core` を clone し、`.env.example` をコピーして `.env` を編集（ADMIN_PASSWORD、UPSTREAM_API_KEY、RECEIVE_LN_ADDRESS など）。
- **オプション B（PC から移行）**: ノート PC の `routstr-core` にある **`.env`** を VPS の同じ場所にコピーする。**データ（ウォレット等）** も引き継ぐ場合は、PC の `routstr-data` ボリュームや `./data` の中身を VPS にコピーする。

例（PC から .env をコピーする場合）:

**※ このコマンドは「.env があるノート PC（Windows）」のターミナル（PowerShell や WSL）で実行します。** VPS に SSH 接続した先では実行しません。PC から VPS へファイルを送るイメージです。

```bash
# ノート PC の routstr-core フォルダで実行（LNVPS #1035 の場合）
scp .env ubuntu@vm-1035.lnvps.cloud:/home/ubuntu/routstr-core/.env
```

### 3. イメージをビルドしてコンテナを起動（VPS 上で）

```bash
cd /home/ubuntu/routstr-core   # LNVPS #1035 の場合（ubuntu ユーザー）
docker build -f Dockerfile.full -t routstr-local .
docker run -d -p 8000:8000 --dns 8.8.8.8 --dns 8.8.4.4 --env-file .env -v routstr-data:/app/data --name routstr --restart unless-stopped routstr-local
```

- `--restart unless-stopped` で、VPS 再起動後もコンテナが自動で立ち上がります（年中稼働のため）。
- **`--dns 8.8.8.8 --dns 8.8.4.4`**: コンテナ内で Cashu の mint（例: mint.minibits.cash）の名前解決を行うため。省略すると「Name does not resolve」「Mint unreachable」などのエラーになることがあります。既存の公式ノードでもコンテナに DNS を渡して mint に接続しています。

### 4. 動作確認

**VPS に SSH 接続したまま**、同じターミナルで実行します（Routstr が動いているサーバー内から、localhost:8000 に届くか確認するため）。

```bash
curl http://127.0.0.1:8000/v1/info
```

JSON が返れば OK です。このあと nginx で routstr.bulbulpp.com を 8000 番に振り向ければ、外から https://routstr.bulbulpp.com でアクセスできます。

---

## ステップ 1: サブドメインを DNS で用意する

**やること**: 「routstr.bulbulpp.com にアクセスしたら、Routstr が動いているサーバーに届く」ようにする。

1. **ドメイン管理画面**を開く。  
   **name.com** の場合は [name.com](https://www.name.com) にログイン → **My Domains** → **bulbulpp.com** をクリック → **Manage DNS**（または **DNS Records**）を開く。
2. **bulbulpp.com** の **DNS 設定**（A レコードや CNAME）を開く。
3. **新規で 1 件追加**（Add Record など）する：
   - **名前（ホスト）**: `routstr`  
     → これで **routstr.bulbulpp.com** になる。
   - **タイプ**: **A**
   - **値（ポイント先）**: **Routstr を動かしているサーバーの IP アドレス**  
     （LNVPS #1035 の場合は **185.18.221.177**）
   - **TTL**: 300 や 3600 など（そのままでよい）

保存して、**数分〜最大 24 時間**待つと、routstr.bulbulpp.com がそのサーバーを向くようになります。

> **用語**: 「A レコード」＝「この名前（api）はこの IP のサーバーだよ」と教える設定です。

---

## ステップ 2: サーバーに nginx を入れる

**やること**: インターネットから来た 443 番の通信を、サーバー内の Routstr(8000) に渡す「入口」を用意する。

Routstr を動かしている **同じサーバー**で、次のどちらかを行う。

### Ubuntu / Debian の場合

```bash
sudo apt update
sudo apt install nginx -y
```

### CentOS / RHEL の場合

```bash
sudo dnf install nginx -y
# または
sudo yum install nginx -y
```

インストール後、nginx を有効化して起動：

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

---

## ステップ 3: SSL 証明書を取る（HTTPS にする）

**やること**: 「https://routstr.bulbulpp.com」で暗号化して繋がるようにする。無料の Let's Encrypt を使う。

```bash
sudo apt install certbot python3-certbot-nginx -y   # Ubuntu/Debian
# または
sudo dnf install certbot python3-certbot-nginx -y  # CentOS/RHEL
```

証明書を取得（**routstr.bulbulpp.com** 用）：

```bash
sudo certbot --nginx -d routstr.bulbulpp.com
```

画面の指示に従う（メールアドレス入力、規約同意など）。  
終わると、証明書のパスが表示されます（後で nginx の設定で使います）。  
多くの場合、certbot が **自動で nginx の設定を書き換え**してくれるので、そのまま使えます。

> **用語**: 「SSL 証明書」＝「このサーバーは本当に routstr.bulbulpp.com です」と証明するもの。HTTPS で必須。

---

## ステップ 4: nginx で「routstr.bulbulpp.com だけ Routstr に渡す」設定をする

**やること**: 「routstr.bulbulpp.com へのアクセス」だけを localhost:8000（Routstr）に転送する。

設定ファイルを編集：

```bash
sudo nano /etc/nginx/sites-available/routstr
```

（`sites-available` が無い場合は `sudo nano /etc/nginx/conf.d/routstr.conf` など、既存の nginx 設定の置き場に 1 ファイル作る。）

**中身**（certbot が既に 443 用の server を用意している場合は、その中に `location /` を追加する形でもよい）：

```nginx
server {
    listen 80;
    server_name routstr.bulbulpp.com;
    # 80 は HTTPS にリダイレクト（certbot が書いてくれる場合もある）
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name routstr.bulbulpp.com;

    # certbot が入れたパス（環境によって違うので、certbot の表示を確認）
    ssl_certificate     /etc/letsencrypt/live/routstr.bulbulpp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/routstr.bulbulpp.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

保存したら、有効化して nginx を再読み込み：

```bash
# sites-available を使っている場合
sudo ln -s /etc/nginx/sites-available/routstr /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

> **用語**: `proxy_pass http://127.0.0.1:8000` ＝「このサーバー内の 8000 番（Routstr）に転送する」という意味。

---

## ステップ 5: Routstr が動いているか確認する

- **Routstr は同じサーバーで** `docker run -d -p 8000:8000 ...` などで **localhost:8000** で動いている想定です。
- 動いていなければ、先に Routstr のコンテナを起動する。

ブラウザで次の URL を開く：

- **https://routstr.bulbulpp.com/v1/info**

Routstr の情報が JSON で表示されれば成功です。  
ダッシュボードは **https://routstr.bulbulpp.com/admin** で開けます。

---

## ステップ 6: ダッシュボードで HTTP URL を設定する

1. **https://routstr.bulbulpp.com/admin** にログインする。
2. **Settings** の **Node / Nostr** あたりで **HTTP URL** を次のように設定する：
   - `https://routstr.bulbulpp.com`
   - （末尾に `/v1` を付けるかは、Routstr の画面の説明に合わせる）

これで、Nostr などでノードを告知するときに、この URL が使われます。

---

## 安全の確認（ファイアウォール）

**8000 番を外に開けていないか**確認する。

- **ufw を使っている場合**:
  ```bash
  sudo ufw allow 80
  sudo ufw allow 443
  sudo ufw enable
  sudo ufw status
  ```
  → 80 と 443 だけ許可し、**8000 は一覧に無い**ことを確認。

- **クラウドの「セキュリティグループ」**で 8000 を開放していないかも確認する。  
  **外に開放するのは 80 と 443 だけ**にすると安全です。

---

## まとめチェックリスト

| 項目 | 内容 |
|------|------|
| DNS | routstr.bulbulpp.com → Routstr サーバーの IP（A レコード） |
| ポート | 外に開くのは **80 と 443 だけ**。8000 は開けない。 |
| nginx | 443 で受け、`proxy_pass http://127.0.0.1:8000` で Routstr に転送。 |
| SSL | certbot で routstr.bulbulpp.com の証明書を取得。 |
| Routstr | 同じサーバーで localhost:8000 で起動。 |
| HTTP URL | ダッシュボードで `https://routstr.bulbulpp.com` を設定。 |

この運用なら、**サブドメインを用意して 80/443 だけ開ける**形になっており、安全です。
