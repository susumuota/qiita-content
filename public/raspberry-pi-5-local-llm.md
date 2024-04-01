---
title: Raspberry Pi 5にUbuntuをヘッドレスでセットアップしてローカルLLMを動かす
tags:
  - Ubuntu
  - RaspberryPi
  - LLM
  - LLaMA
  - llama.cpp
private: false
updated_at: '2024-04-01T22:16:03+09:00'
id: 437d4d5692b7bf010640
organization_url_name: null
slide: false
ignorePublish: false
---

# Raspberry Pi 5にUbuntuをヘッドレスでセットアップしてローカルLLMを動かす

## 概要

- Raspberry Pi 5 / 8Gをヘッドレスでセットアップ(つなぐのは電源ケーブルだけ)
- Ubuntu Server (Raspberry Pi OSは使わない)
- 最低限のセキュリティ設定(sshd, ファイアウォール)
- ローカルLLMのセットアップ(llama.cpp, gemma-2b-it)
- サーバ設定(自動マウント, スワップファイル, Samba, Apache)
- Python開発環境(pyenv, Poetry)

![qa by llama.cpp with gemma-2b-it](https://raw.githubusercontent.com/susumuota/zenn-content/main/images/llama-cpp-gemma-2b-it-qa.png)

## ハードウェア

- 本体: Raspberry Pi 5 / 8GB
- 電源: Anker PowerPort III Nano 20W (5V 3A対応)
- USBケーブル: Anker PowerLine III USB-C & USB-C 2.0 ケーブル
- microSDカード: SanDisk Ultra 128GB class 10
- ファン: Raspberry Pi 5用アクティブクーラー
- ケース: Raspberry Pi 5用公式ケース

> **Note**: 公式ケースの価格がそこそこ高く、かつ、アクティブクーラーがあればケース付属のファンが無駄になるので、サードパーティ製のケースが出てくるのを待ったほうが良いかもしれない。

## OSイメージを焼く

Raspberry Pi ImagerでOSイメージを焼く。

- https://www.raspberrypi.com/software/

以下設定でOSイメージを焼く。サーバとして利用するだけなので`Ubuntu Server`を選択。パスワードは平文でどこかにキャッシュされるといやなので、適当なパスワードを一時的に設定しておいて、sshでログイン後に真っ先に変更する。

- Raspberry Piデバイス
  - `Raspberry Pi 5`
- OS
  - `Ubuntu Server 23.10 (64-bit)`
- ストレージ
  - microSDカード
- ホスト名
  - `pi5`.local
- ユーザ名とパスワード
  - `ota`, `適当なパスワード` (→sshログイン後すぐ変える)
- Wi-Fi
  - `SSID`, `パスワード`, `JP`
- ロケール
  - `Asia/Tokyo`, `jp`
- SSH有効化
  - `公開鍵認証のみを許可する`
  - 公開鍵 (e.g. `~/.ssh/id_rsa.pub`) の中身をペースト
- テレメトリーを有効化
  - オフ

## sshでログイン

- microSDカードをRaspberry Pi 5に挿入
- 電源を入れる
- 少し待ってPCからsshでログイン
```bash
ssh ota@pi5.local
```
- 公開鍵認証でログイン出来るはず
- ホスト名が解決できなければIPアドレスを指定
- うまくいかなければ `-vvv` をつけて原因を調べる
- どうしてもだめならOSイメージから焼き直し
  - `~/.ssh/know_hosts` に残っている古いエントリを削除しておく


## パスワード変更

ログイン後、すぐにパスワードを変更する。

```bash
passwd
```

## システムのアップグレード

結構時間がかかるはず。

```bash
sudo apt-get update
sudo apt-get full-upgrade -y
sudo reboot
```

`full-upgrade` すると再起動以後ファンコントロール(温度によって強弱)が有効になる。しないとファンは回りっぱなし。

## sshの設定

`~/.ssh/authorized_keys` の確認。`~/.ssh/id_rsa.pub` の中身が含まれているはず。パーミッションも `600` になっているはず。

```bash
cat ~/.ssh/authorized_keys
ls -l ~/.ssh/authorized_keys
```

sshdの設定を変更。以下4つを変更する。
- rootユーザでのログインを禁止
- パスワード認証を禁止
- PAMを無効化(これはしなくてもOK)
- X11Forwardingを無効化

```bash
sudo cp -p /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sudo nano /etc/ssh/sshd_config
```

以下4行を変更。

```
PermitRootLogin no
PasswordAuthentication no
UsePAM no
X11Forwarding no
```

変更を確認。

```bash
diff -u /etc/ssh/sshd_config.bak /etc/ssh/sshd_config
sudo sshd -T | grep root
sudo sshd -T | grep pass
sudo sshd -T | grep pam
sudo sshd -T | grep x11
```

マシンを再起動。

```bash
sudo reboot
```

再度sshでログイン。

```bash
ssh ota@pi5.local
```

## ファイアウォールの設定

外部マシンからアクセスするポートだけ許可する。`22` だけは開けておかないとsshでログイン出来なくなり、OSイメージを焼くところからやり直しになるので注意。

- https://help.ubuntu.com/community/UFW

```bash
sudo ufw status verbose
sudo ufw enable
sudo ufw default DENY
sudo ufw allow 22      # ssh
# sudo ufw allow 80    # http for apache
# sudo ufw allow 8080  # http for llama.cpp
# sudo ufw allow 139   # samba
# sudo ufw allow 445   # samba
sudo ufw status verbose
sudo reboot
```

## bashの設定

```bash
cat >> ~/.bash_aliases <<EOF
alias ll='ls -l'
alias la='ls -A'
alias l='ls -CF'
alias rm='rm -i'
alias watch-temp='watch -n 1 -d -t vcgencmd measure_temp'
EOF

source ~/.bash_aliases
```

`LANG` を設定しておかないと、日本語ファイル名が文字化けすることがある。

```bash
cat >> ~/.bashrc <<EOF

export LANG=C.UTF-8
EOF
```

## tmuxの設定

```bash
cat >> ~/.tmux.conf <<EOF
set -g prefix C-j
unbind C-b
bind C-j send-prefix

set -g base-index 1
set -g status-right "%H:%M"
set -g window-status-current-style "underscore"
set -g default-terminal "screen-256color"
EOF
```

`tmux` を実行して `Ctrl-j d` でデタッチして `tmux a` でアタッチ出来ることを確認。

## htopの設定

`htop` でCPU温度を表示する。

`libsensors5` をインストール。

```bash
sudo apt-get install -y libsensors5
```

- `htop` を実行
- `F2 (Setup)` で設定画面
- `Display options` - `Also show CPU temperature (requires libsensors)` を有効
- `F10` で保存

## ローカルLLM(gemma-2b-it)のセットアップ

`gemma` 以外のローカルLLMについては別記事で書く予定。

> **Note**: `llama-cpp-python` より `llama.cpp` を素で使った方が推論時間が速いので `llama.cpp` をサーバモードで立ち上げることにする。

### llama.cppのインストール

- https://github.com/ggerganov/llama.cpp#usage

```bash
sudo apt-get update
sudo apt-get install -y aria2 build-essential cmake
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
make -j
```

### モデルのダウンロード

gemma を使う際は Terms of Use に同意する必要がある。

- https://huggingface.co/google/gemma-2b-it
- https://huggingface.co/google/gemma-7b-it

`gemma-2b-it` の6bit量子化版をダウンロード(1.9GB)。

```bash
aria2c -x 5 "https://huggingface.co/mlabonne/gemma-2b-it-GGUF/resolve/main/gemma-2b-it.Q6_K.gguf" -d "models" -o "gemma-2b-it.Q6_K.gguf"
```

7b 版を試す場合は、`gemma-7b-it` の2bit量子化版をダウンロード(3.2GB)。これより大きいモデルはメモリが足りず安定動作しなかった。

```bash
aria2c -x 5 "https://huggingface.co/mlabonne/gemma-7b-it-GGUF/resolve/main/gemma-7b-it.Q2_K.gguf" -d "models" -o "gemma-7b-it.Q2_K.gguf"
```

実行してみる。

```bash
./main -ngl 0 -t 4 -c 8192 -m "models/gemma-2b-it.Q6_K.gguf" -p "Where is the capital city of Japan?"
```

- `gemma-2b-it.Q6_K.gguf`
  - 毎秒5トークン程度で生成できる。
- `gemma-7b-it.Q2_K.gguf`
  - 毎秒2.5トークン程度で生成できる。
  - Raspberry Pi 5 / 8GBでまともに動くのは7bの2bit量子化版まで。3bit量子化版も試したが、メモリが足らず安定動作しなかった。

対話モードで実行。

```bash
./main -ngl 0 -t 4 -c 8192 -m "models/gemma-2b-it.Q6_K.gguf" -e -i --color \
  -p "<start_of_turn>user\nHello<end_of_turn>\n<start_of_turn>model\n" \
  --in-prefix "<start_of_turn>user\n" \
  --in-suffix "<end_of_turn>\n<start_of_turn>model\n"
```

LLMサーバ起動。

`8080` ポートを公開するので、先にファイアウォールの設定を変更してリブートしておく。

```bash
sudo ufw allow 8080  # http for llama.cpp
sudo ufw status verbose
sudo reboot
```

テンプレートを指定してサーバを起動。

- https://github.com/ggerganov/llama.cpp/tree/master/examples/server
- https://github.com/ggerganov/llama.cpp/wiki/Templates-supported-by-llama_chat_apply_template

```bash
./server -ngl 0 -t 4 -c 8192 -m "models/gemma-2b-it.Q6_K.gguf" --chat-template "gemma" --port 8080 --host 0.0.0.0
```

ブラウザで `http://pi5.local:8080` にアクセス。

gemmaの場合は以下のように設定。

- Prompt
  - (空文字列)
- User name
  - `user`
- Bot name
  - `model`
- Prompt template
```
{{prompt}}
{{history}}
<start_of_turn>{{char}}
```
- Chat history template
```
<start_of_turn>{{name}}
{{message}}
<end_of_turn>
```

`Say something...` のところに `Summarize this text. (要約したい文章)` を入力して `Send` ボタンを押すと `model:` 以下に返答が表示される。

![summarize text by llama.cpp with gemma-2b-it](https://raw.githubusercontent.com/susumuota/zenn-content/main/images/llama-cpp-gemma-2b-it-summarize.png)

> **Note**: 次節以降の作業はローカルLLMには関係なく、Raspberry Piをサーバとして運用する際のよくある設定のメモ。

## USBメモリ等の自動マウント

`autofs` を使って、ディレクトリにアクセスしたタイミングで自動的にマウントして、一定時間アクセスがなければ自動的にアンマウントする。常にマウントするなら `/etc/fstab` を使う。

- https://help.ubuntu.com/community/Autofs

USBメモリを挿して、`blkid` を実行してデバイスのパスを確認しておく。e.g. `/dev/sda1`

ひとまず、手動でマウント出来るかどうか確認。

```bash
sudo mount -t vfat /dev/sda1 /mnt
ls -l /mnt
sudo umount /mnt
```

手動でマウント出来たら `autofs` で自動マウント出来るようにする。

```bash
sudo apt-get install -y autofs
```

既に `/media` ディレクトリが存在するので、そこに `/dev/sda1` に挿された vfat の USB メモリを `/media/usb0` に自動マウントすることにする。どこにマウントするべきかは以下ページ参照。

- https://refspecs.linuxfoundation.org/FHS_3.0/fhs-3.0.html#mediaMountPoint

autofs の設定ファイルは `/etc/auto.master` で、 `man auto.master` でマニュアルが読める。設定方法には、`/etc/auto.master` を直接編集する方法と、`/etc/auto.master.d` ディレクトリにファイルを追加する方法がある。ここでは後者の方法で設定した。

マニュアルを読むと、`/etc/auto.master.d/*.autofs` に追加のマウントポイントのファイルを置けと書いてあるので、`media.autofs` というファイルを作成する。拡張子は `*.autofs` である必要がある。

```bash
cat << EOF | sudo tee -a /etc/auto.master.d/media.autofs
/media  /etc/auto.master.d/auto.media
EOF
```

さらに `/etc/auto.master.d/auto.media` ファイルを作成する。

パラメータの意味は、`fstype=vfat` でフォーマット指定。 `iocharset=utf8,codepage=932` は日本語ファイル名の文字化け対応。`uid=ota,gid=ota` はユーザ名を指定(指定しなくてもよいが指定がないとrootになる。`ota` の部分は環境に合わせて変える)。`dmask` と `fmask` はマウントしたディレクトリとファイルのデフォルトのパーミッション設定(ディレクトリは`755`、ファイルは`644`にする)。

```bash
cat << EOF | sudo tee -a /etc/auto.master.d/auto.media
usb0    -fstype=vfat,iocharset=utf8,codepage=932,uid=ota,gid=ota,dmask=0022,fmask=0133    :/dev/sda1
EOF
```

autofs のデーモンを再起動(あるいは `sudo reboot` でマシン再起動してもOK)。

```bash
sudo service autofs restart
```

`/media/usb0` にアクセスしようとすると自動的にマウントされるはず。

```bash
ls -l /media/usb0
```

として中身のファイルが見られればOK。日本語ファイル名が文字化けしていないこと、ファイルの所有者とパーミッションを確認。また、`df -h` でも確認。

## スワップファイルの設定

> **Note**: メモリ8GBモデルではおそらくスワップ不要

重いコンパイル等でメモリが足らなくなってフリーズしてしまうことがたまにあるので、その際はスワップを有効にする。コンパイル時だけ一時的にスワップ有効にして通常時はスワップ無効で運用する。

- https://help.ubuntu.com/community/SwapFaq
- https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-20-04-ja

`1g` が容量の指定。

```bash
cat /proc/swaps
ls -l /swapfile  # 既にあれば別名にする
sudo fallocate -l 1g /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
cat /proc/swaps
```

一時的にスワップ有効にするのであればここまででOK。2回目以降は `sudo swapon /swapfile` だけでOK。しばらく使わないなら `sudo swapoff /swapfile && sudo rm /swapfile` で削除しておく。

もし、永続的に有効にするには `/etc/fstab` に書き込んで再起動する。

```bash
sudo cp -p /etc/fstab /etc/fstab.bak
cat << EOF | sudo tee -a /etc/fstab
/swapfile swap swap defaults 0 0
EOF
```

## Emacsのインストール

結構時間がかかる。

```bash
sudo apt-get update
sudo apt-get install -y emacs
```

以下の設定を追加。

```bash
mkdir -p ~/.emacs.d
cat >> ~/.emacs.d/init.el <<EOF
(set-language-environment "Japanese")

(setq inhibit-startup-message t)
(setq initial-scratch-message "")

(menu-bar-mode -1)
(tool-bar-mode 0)
EOF
```

## Sambaのインストール

ファイル共有する場合はSambaをインストールする。

```bash
sudo apt-get install -y samba
sudo emacs /etc/samba/smb.conf
```

末尾に以下を追加。`/media/usb0` は共有するディレクトリ。`ota` はユーザ名。

```
[pi5usb0]
path = /media/usb0
read only = No
guest ok = Yes
force user = ota
```

ポートを開ける。

```bash
sudo ufw allow 139   # samba
sudo ufw allow 445   # samba
sudo ufw status verbose
sudo reboot
```

マシン再起動して、PCから `smb://pi5.local` にアクセスしてゲストで読み書きできればOK。

## Apacheのインストール

HTTPサーバを立てる場合はApacheをインストールする。

```bash
sudo apt-get install -y apache2
sudo cp -p /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/default.conf
sudo emacs /etc/apache2/sites-available/default.conf
```

以下のように変更。

```
        #DocumentRoot /var/www/html
        DocumentRoot /media/usb0

        <Directory /media/usb0/>
                Options Indexes FollowSymLinks
                AllowOverride None
                Require all granted
        </Directory>
```

変更を有効にする。

```bash
sudo a2dissite # 000-default を入力
sudo a2ensite # default を入力
sudo systemctl reload apache2
```

ポートを開ける。

```bash
sudo ufw allow 80    # http for apache
sudo ufw status verbose
sudo reboot
```

ブラウザで `http://pi5.local` にアクセスして、USBメモリの中身が表示されればOK。

## pyenvのインストール

Python開発環境が必要なら `pyenv` をインストールする。

メモリが足りないとフリーズするので、一度試してみて、もしメモリが足りてなかったらスワップを有効にしてからインストールする。

- https://github.com/pyenv/pyenv#automatic-installer

```bash
curl https://pyenv.run | bash
```

- https://github.com/pyenv/pyenv?tab=readme-ov-file#set-up-your-shell-environment-for-pyenv

```bash
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc
```

シェルを開き直す。

pyenv install でコンパイルする際に `cc` やライブラリが必要なのでインストールしておく。

- https://github.com/pyenv/pyenv/wiki#suggested-build-environment

```bash
sudo apt-get update
sudo apt-get install -y build-essential libssl-dev zlib1g-dev \
  libbz2-dev libreadline-dev libsqlite3-dev curl \
  libncursesw5-dev xz-utils tk-dev libxml2-dev libxmlsec1-dev libffi-dev liblzma-dev
```

Python をインストールする。数分かかる。

```bash
pyenv install --list
pyenv install 3.10.13  # 数分待つ
pyenv global 3.10.13
python -V
which python
which pip
pip list  # pip と setuptools のみ
```

## Poetryのインストール

Pythonのパッケージ管理にPoetryを使う場合。

- https://python-poetry.org/docs/#installing-with-the-official-installer

```bash
curl -sSL https://install.python-poetry.org | python3 -
```

```bash
echo 'export PATH="/home/ota/.local/bin:$PATH"' >> ~/.bashrc
```

```bash
poetry --version
```

## まとめ

- Raspberry Pi 5にUbuntuをヘッドレスでセットアップしてローカルLLMを動かす手順をまとめた
- Raspberry Pi 5 / 8GB では `gemma-7b-it` の2bit量子化版までは快適に動作
- `gemma-7b-it` の3bit以上の量子化版も試したが、メモリが足らず安定動作しなかった

以上
