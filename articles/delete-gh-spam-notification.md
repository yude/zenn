---
title: "GitHub 上のスパムにより発生した通知を既読にする"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["github"]
published: true
---

# 消せない通知

![GitHub 上 Notifications ページの状態](https://storage.googleapis.com/zenn-user-upload/725da2ca4dc3-20250924.png)

- 先日 GitHub の Web サイトにおいて、未読の通知があるように表示されるが、その内容が表示されず、したがって既読にできない状態になっていることに気づきました。  
- これは GitHub のモデレーションチームがスパムに対して処置を行った結果、通知のステートに不具合が生じ発生したものと推察できます。 (左下のリポジトリ名に基づいて調査しました。)  
- 少なくとも私が探索した限りでは Web の機能を用いて通知欄を修正することはできなかったため、GitHub の REST API 経由で修正を試みることとしました。

## REST API を操作するための token を入手する

* [Personal Access Tokens (classic)](https://github.com/settings/tokens) から以下の capability を付与した token を生成します
    * `notifications`


## 特定の通知のみ既読にする

* 不可能のようです。
* 上記スクリーンショット左下に表示されているリポジトリに対して操作を実行しましたが、404 ステータス コードが返却されました

    ```
    $ curl -X PUT -H "Authorization: token ghp_<redacted>" \
        -H "Accept: application/vnd.github+json" \
        https://api.github.com/repos/notifier-mail/gitcoin.com/notifications
    {
    "message": "Not Found",
    "documentation_url": "https://docs.github.com/rest",
    "status": "404"
    }
    ```

## すべての通知を既読にする

* 正常なリポジトリに関する通知についても未読になるため、このことに対して留意してください。

    ```
    $ curl -X PUT -H "Authorization: token <token, starts with ghp_>" \
     -H "Accept: application/vnd.github+json" \
     https://api.github.com/notifications
    ```
* この操作を実行後、未読ランプが消えたことを確認しました (2025/9/24 22:40 頃)
* 左下に表示されている Repositories は依然として表示されています
