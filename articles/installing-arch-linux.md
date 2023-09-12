---
title: "Arch Linux のインストール"
emoji: "🅰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["archlinux"]
published: true
---

## はじめに

- システムに変更を加える前に、必要なデータをバックアップしてください。
- 記事中に登場する `[文字列]` という記述は、その部分を読者が適切な値に変更する必要があることを示します。
  - 例: `ping` コマンドを使用して、特定のホストに ICMP Echo Request を送信できます。

    ```shell
    ping [ホスト]
    # 実行例: ping 1.1.1.1
    ```
- この記事では、デスクトップ環境の導入を行いません。
    - すべての手順を終えた後で、デスクトップ環境を追加できます。

## 手順

1. イメージを USB フラッシュメモリに書き込み、Live ブートする

    - [Arch Linux JP Project - ダウンロード](https://www.archlinux.jp/download/)

1. Live ブートを設定する
    1. ネットワークに接続する
        - イーサネット: LAN ケーブルを接続すると、自動的にインターネットに接続されるでしょう。
        - Wi-Fi: `iwctl` を使用して、インターネットに接続します。
            1. `iwctl` を実行します。
            2. Wi-Fi アダプターの一覧を表示します。

                ```shell
                device list
                ```

            3. アクセスポイントを検出します。

                ```shell
                station [2 で確認したアダプターの名前] scan
                ```

            4. 3 で検出したアクセスポイントを表示します。

                ```shell
                station [2 で確認したアダプターの名前] get-networks
                ```

            5. アクセスポイントに接続します。パスワードを求められることがあります。

                ```shell
                station [2 で確認したアダプターの名前] connect [SSID]
                ```

            6. `exit` で `iwctl` を終了します。

        - 以下のコマンドで疎通を確認できるでしょう。

            ```shell
            ping 1.1.1.1
            ```

    2. セットアップを準備する
        - NTP を有効にします。

            ```shell
            timedatectl set-ntp true
            ```

        - UEFI ブートしていることを確認します。

            ```shell
            ls /sys/firmware/efi/efivars | head -n 5
            ```

        - 認識されているパーティションを確認します。

            ```shell
            lsblk
            ```

    3. パーティションを設定する

        1. `cgdisk` を実行します。

            ```shell
            cgdisk [ディスクのデバイス ファイル]
            # 例: cgdisk /dev/nvme0n1
            #     cgdisk /dev/sda
            #                           etc.
            ```

        2. 既存のパーティションをすべて削除します。
            - 表示されているすべてのパーティションに対して、`Del` を実行します。

        3. 必要なパーティションを作成します。
            `New` を実行することで、パーティションを作成できます。
            | 順番 | 開始セクタ    | 終了セクタ    | コード  | 名前        |
            |----|----------|----------|------|-----------|
            | 1  | そのままリターン | `+512M` (= ディスクの始まりから 512M まで)    | `ef00` | `efi`       |
            | 2  | そのままリターン | `-3G` (= ディスクの終わりから 3G 手前まで)      | `8300` | `Arch Linux root` |
            | 3  | そのままリターン | そのままリターン | `8200` | `swap`      |

        4. 変更を保存します。
            `Write` を実行します。

    4. パーティションをフォーマットする
        それぞれのパーティションに対して、適切なコマンドを実行します。
        - EFI パーティション

            ```shell
            mkfs.fat -F32 [EFI パーティションのデバイス ファイル]
            ```

        - root パーティション

            ```shell
            mkfs.ext4 [root パーティションのデバイス ファイル]
            ```

        - swap パーティション

            ```shell
            mkswap [swap パーティションのデバイス ファイル]
            ```

        - `fdisk -l` で、設定したパーティションを確認できます。

    5. パーティションをマウントして、swap の使用を開始する
        - EFI パーティションを `/mnt/boot` にマウントします。

            ```shell
            mount -m [EFI パーティションのデバイス ファイル] /mnt/boot
            ```
  
        - root パーティションを `/mnt` にマウントします。

            ```shell
            mount [root パーティションのデバイス ファイル] /mnt
            ```

        - swap の使用を開始します。

            ```shell
            swapon [swap パーティションのデバイス ファイル]
            ```

    7. Arch Linux の基本的なシステムをインストールする
        以下のコマンドを実行します。

        ```shell
        pacstrap /mnt base base-devel linux linux-firmware bash-completions nano sudo refind networkmanager
        ```

    8. インストールしたシステムの基本的な設定をする

        1. `/etc/fstab` の生成
            以下のコマンドを実行します。

            ```shell
            genfstab -U /mnt >> /mnt/etc/fstab
             ```

        2. chroot 環境に入る
            chroot 環境では、インストール予定のシステムと同じ `PATH` を使用できます。\
            以下のコマンドを実行します。

            ```shell
            arch-chroot /mnt
            ```

        3. タイムゾーンを設定する
            この記事の読者の多くは日本に住んでいることが予想されますから、ここでは `Asia/Tokyo` を設定します。\
            以下のコマンドを実行します。

            ```shell
            ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
            ```

            また、以下のコマンドで Arch Linux システムから UEFI に時刻をコピーします。

            ```shell
            hwclock --systohc
            ```

        4. 国際化をする
            1. `/etc/locale.gen` をテキスト エディタで開き、以下の行をコメントアウトします。

                ```plain
                en_US.UTF-8 UTF-8
                ja_JP.UTF-8 UTF-8
                ```

            2. ロケール ファイルを生成します。

                ```shell
                locale-gen
                ```

            3. システムで使用する言語を変更します。

                ```shell
                echo "LANG=en_US.UTF-8" > /etc/locale.conf
                ```

        5. その他の設定をする
            1. キーボード配列を JIS に変更します。 (任意)
                英語配列キーボードを使用している方はスキップします。

                1. `/etc/vconsole.conf` をテキスト エディタで開きます。
                2. `KEYMAP=us` の行を `KEYMAP=jp106` に変更します。

            2. コンソールのフォントを大きくします。 (任意) \

                1. フォントをインストールします。以下のコマンドを実行します。

                    ```shell
                    pacman -S terminus-font
                    ```

                2. `/etc/vconsole.conf` を開き、フォントを指定します。\
                    以下を追記します。

                    ```plain
                    FONT=ter-128n
                    ```

                - 変更は再起動後に反映されます。

            3. ホスト名を設定します。

                1. ホスト名を変更します。
                    以下のコマンドを実行します。\

                    ```shell
                    echo "[ホスト名]" > /etc/hostname
                    # 例: echo "arch" > /etc/hostname
                    ```

                2. 名前解決の設定をします。
                    `/etc/hosts` の内容を、以下のように変更します。
                    `ホスト名` には、ステップ 1 で決定したホスト名を入力します。

                    ```plain
                    127.0.0.1 localhost
                    ::1 localhost
                    127.0.1.1 ホスト名.localdomain [ホスト名]
                    ```

            4. `root` アカウントのパスワードを指定する
                以下のコマンドを実行すると、新しいパスワードを入力するよう求められます。

                ```shell
                passwd
                ```

        6. ユーザーを作成し、設定する
            普段 Arch Linux を使用する際のユーザーを作成します。

            1. ユーザーを作成します。
                - `wheel` グループに参加させます。\
                    これは、このアカウントがシステムの管理者であることを示します。

                以下のコマンドを実行します。

                ```shell
                useradd -m -G wheel [ユーザー名]
                ```

            2. パスワードを変更します。
                以下のコマンドを実行すると、新しいパスワードを入力するよう求められます。

                ```shell
                passwd [ユーザー名]
                ```

            3. `sudo` を実行できるようにする
                `sudo` は、一時的に権限昇格を行うコマンドです。

                1. `visudo` を実行します。
                    - `nano` を使用する場合、`EDITOR=nano visudo` とします。

                2. 以下の行をコメントアウトします。

                    ```plain
                    %wheel  ALL=(ALL)       ALL
                    ```

        7. ブートローダーを導入する

            1. rEFInd のファイルを EFI パーティションにコピーします。
                以下のコマンドを実行します。

                ```shell
                refind-install
                ```

            2. 既定の設定ファイルをバックアップします。
                以下のコマンドを実行します。

                ```plain
                cp /boot/refind_linux.conf /boot/refind_linux.conf.bak
                ```

            3. rEFInd の設定を書き込みます。
                以下のコマンドを実行します。

                ```plain
                echo "\"Boot with standard options\" \"root=UUID=$(blkid [Linux をインストールしたパーティションのデバイス ファイルの場所] -s UUID -o value) rw splash\"" > /boot/refind_linux.conf
                ```

        8. システムを起動する
            `reboot` で再起動します。\
            Arch Linux が起動できないときは、以下を御覧ください。

            - Arch Linux のセットアップは、再度 Live ブートすることで全体的に、あるいは部分的に修正できます。
            - 以下の点が、多くの場合において致命的にブートを失敗させる要因となります。
                - rEFInd の設定の間違い
                - `/etc/fstab` の記述の間違い

    9. ブート後の基本的な設定を行う
        - 以下の操作は、ブート後ログインして行います。

        1. インターネットに接続する
            1. NetworkManager を起動する
                以下のコマンドを実行します。

                ```shell
                sudo systemctl enable --now NetworkManager
                ```

                多くの場合、自動的にインターネットに接続されるはずです。\
                Wi-Fi 環境である場合や、なにか問題が発生した場合は、以下のコマンドを実行して NetworkManager を設定します。

                ```shell
                sudo nmtui
                ```
