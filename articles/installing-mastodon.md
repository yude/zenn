---
title: "Mastodon インスタンスをセットアップする"
emoji: "💬"
type: "tech"
topics: ["Mastodon", "Docker", "cloudflared"]
published: true
---

## 事前準備

この記事に基づいて Mastodon インスタンスを設定するには、以下の 3 点が必要です。

- [Cloudflare](https://cloudflare.com)
  - 1. [Cloudflare アカウント](https://dash.cloudflare.com/sign-up?lang=en-US)
  - 2. Cloudflare で管理されている独自ドメイン
- Linux サーバ
    以下をインストールしてください。
  - 3. [Docker](https://docs.docker.com/engine/install/)
    - コマンド `docker compose` を実行できるようにしてください。

## 設定

1. 作業ディレクトリの作成

    作成したディレクトリに、Mastodon インスタンスに関連するすべてのファイルが保存されます。

    ```shell
    mkdir /path/to/mastodon
    cd /path/to/mastodon
    ```

1. Mastodon 用 `docker-compose.yaml` ファイルの取得
    Docker Compose は、`docker-compose.yaml` という名前の YAML ファイルに基づいて、自動的にアプリケーションを構成します。

    ```shell
    wget https://raw.githubusercontent.com/mastodon/mastodon/main/docker-compose.yml
    ```

1. nginx の設定
    nginx は、Mastodon の前段に置かれるリバースプロキシとして使われます。\
    以下のコマンドを実行して、設定ファイルを格納するディレクトリを作成し、また Mastodon 用の設定ファイルをダウンロードします。

    ```shell
    mkdir nginx
    wget https://gist.githubusercontent.com/yude/7d0de7f85cc75a6a8e17ff59a610a6d4/raw/7fec9cacfb060d824840aa1d029f9848481cf5c7/gistfile1.txt nginx/mastodon.conf
    ```

    また、`docker-compose.yaml` に以下を追記します。\
    追記の際は、インデントに注意してください。

    ```yaml
    nginx:
        image: nginx:latest
        volumes:
        - ./nginx/mastodon.conf:/etc/nginx/conf.d/default.conf
        restart: always
        depends_on:
        - web
    ```

1. 認証情報の設定
    1. `.env.production` ファイルの雛形を用意します。
        これは、Mastodon の認証情報や、インスタンスの基本的な設定が記述されるファイルです。
        ```shell
        wget https://raw.githubusercontent.com/mastodon/mastodon/main/.env.production.sample .env.production
        ```
    2. 後の設定で必要となる文字列を生成します。
        これらの文字列は、ステップ 3.3 で使用しますから、安全な場所に保管してください。
        1. 文字列 A を生成します。
            2 箇所で使用します。別々の文字列でなければなりませんから、**2 回実行してそれぞれを保管**してください。
            ```shell
            docker compose run --rm web bundle exec rake secret
            ```
        2. 文字列 B を生成します。
            ```shell
            docker compose run --rm web bundle exec rake mastodon:webpush:generate_vapid_key
            ```
    3. `.env.production` を編集します。
        `vi` や `nano` 等のエディタを使用して、このファイルを開いてください。

        **必須の設定項目**
        - Federation
            - `LOCAL_DOMAIN`: Mastodon インスタンスを公開するドメインを指定します。
        - Redis
            - `REDIS_HOST`: `redis` に変更します。
        - PostgreSQL
            - `DB_HOST`: `db` に変更します。
        - Elasticsearch
            - `ES_ENABLED`: `false` に変更します。
        - Secrets
            - `SECRET_KEY_BASE`: 文字列 A のうち、1 つ目の文字列を指定します。
            - `OTP_SECRET`: 文字列 A のうち、2 つ目の文字列を指定します。
        - Web Push
            - `VAPID_PRIVATE_KEY`, `VAPID_PUBLIC_KEY`:
                文字列 B を指定します。\
                この文字列は、そのまま貼り付けられる形式で出力されているはずですから、そのままペーストします。
        - File storage
            - `S3_ENABLED`: `false` に変更します。
        
        **任意の設定項目**
        以下の項目を設定せずとも、Mastodon インスタンスは実行できます。しかし、アカウントの認証等で混乱が生じる可能性がありますから、設定することを強く推奨します。
        - Sending mail
            - `SMTP_SERVER`: SMTP サーバーのアドレスを指定します。
            - `SMTP_PORT`: SMTP サーバーのポート番号を指定します。
            - `SMTP_LOGIN`: SMTP サーバーへのログインに使用するアカウント名を指定します。
            - `SMTP_PASSWORD`: SMTP サーバーへのログインに使用するパスワードを指定します。
            - `SMTP_FROM_ADDRESS`: Mastodon インスタンスから送信されるメールの、送信元アドレスを指定します。
        - その他
            - 自分しか使用しない Mastodon インスタンスである場合は、以下を追記して新規登録を無効化できます。
              ```
              SINGLE_USER_MODE=true
              ```

1. データベースの初期化、アセットの生成
    以下のコマンドを実行します。
    ```shell
    docker compose run --rm web rails db:migrate
    docker compose run --rm web rails assets:precompile
    ```

1. Cloudflare の設定
    `cloudflared` (Cloudflare Tunnel) を用いて、Mastodon インスタンスを公開します。
    1. Tunnels ページを開く
        以下の順でページを遷移します。
        1. Cloudflare にログイン
        2. Cloudflare アカウント ホーム (`https://dash.cloudflare.com/[アカウント ID]`)
        3. Zero Trust
        4. Access
        5. Tunnels
    2. トンネルの作成
        `Create a tunnel` をクリックします。\
        必要な項目を入力して、`Save tunnel` をクリックします。
        - Tunnel name: トンネルの名前。任意の文字列を指定できます (例: `mastodon`)
    3. トークンの保管
        `If you already have cloudflared installed on your machine:` の下に表示されているコマンドを、一旦コピーします。\
        **まだ Cloudflare のページを開いているブラウザは閉じないでください。**
    4. トークン文字列の編集
        コピーした文字列は `cloudflared` コマンドの書式になっています。\
        今回使用するのは **トークンの部分のみ** ですので、文字列から以下の部分を除去します。
        ```
        sudo cloudflared service install 
        ```
        `install` の後の半角スペースも除去します。
    5. トークンの保管
        `docker-compose.yaml` が保管されているディレクトリに `.env` というファイルを作成し、以下の内容で保存します。
        ```
        CLOUDFLARED_TOKEN=ステップ 4 で編集済みの文字列
        ```
    6. トンネルの作成作業を完了する
        Cloudflare のページに戻り、`Next` をクリックします。\
        いくつかの入力欄が現れますから、必要な項目を入力します。
        - `Subdomain`: `mstdn.example.com` のように、サブドメイン上に Mastodon インスタンスを設置する場合は指定します。
        - `Domain (required)`: Mastodon インスタンスを設定するドメインの第 2 レベル部 (`mstdn.example.com` であれば `example.com` の部分) を指定します。
        - `Path`: 何も入力しません。
        - `Type`: `HTTP` を選択します。
        - `URL`: `nginx` を入力します。

        作業が終了したら、`Save tunnel` をクリックします。


    7. `cloudflared` サービスの追加
        `docker-compose.yaml` に以下を追記し、保存します。\
        追記の際は、インデントに注意してください。
        ```yaml
        cloudflared:
            image: cloudflare/cloudflared:latest
            user: root
            restart: always
            command: tunnel --no-autoupdate run --token $CLOUDFLARED_TOKEN
        ```

1. Mastodon インスタンスの起動
    `docker compose up -d` で起動します。\
    正しく起動していることが確認できなければ、`docker compose logs` でログを参照できます。

1. 自分のアカウントの作成
    インスタンスのセットアップが完了した後は、できる限り早く自分のアカウントを作成します。\
    以下のコマンドで指定したアカウントを `Owner` に昇格できます。

    ```shell
    docker exec -it $(docker compose ps -q web) bin/tootctl accounts modify アカウント名 --role Owner
    ```

セットアップは以上です。お疲れ様でした。
