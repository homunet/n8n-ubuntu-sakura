# VPSを使用したn8nセットアップ

こういうドキュメントはqiitaとかに掲載されることが多いと思いますが、私としてはシェアしてなんぼ、シェアされてなんぼだと思っています。明示的にMITライセンスを設定しましたので、必要に応じてより良い内容にアップードして使ってください。

## 注意事項
サーバー管理を全くやったことがない人がこの文書だけ読んで構築しても、サーバーのセキュリティを確保するのは難しいと思います。セルフホストならn8nの利用料を節約できますが、情報漏洩によって発生するコストの方が普通は高いです。くれぐれもお気を付けください。

## 参考にしたウェブサイト

いくつかのウェブサイトでインストール方法が公開されているのですが、SSLのウェブソケットが使用できないことが原因でインストール後にn8nのアクセスエラーが発生しました。この記事の存在価値はほとんどその回避策です。

[n8nのインストール方法：自動化を自分でホスティングするには](https://www.hostinger.com/jp/tutorials/how-to-install-n8n/)

* 概ねこのページを参考にしましたが、マウントポイントのアドレスが異なります。

[Using n8n with NGINX Reverse Proxy](https://pliszko.com/blog/post/2024-03-14-using-n8n-with-nginx-reverse-proxy)

* SSLをNGINXで使用する際にリバースプロキシでwebsoketを通す必要がある
    * 筆者はここでハマって時間を溶かしました
    * ただしproxy_set_header Forwarded         $proxy_add_forwarded;の行は不要

## 構築の前提環境・構築する環境
* ローカルはMac
* SSL使用
* SSLはLet's Encrypt
     * 自分のサーバーに自分で設定したDNSでアクセスするのでホストの真正性検証は割愛
* SSLを使うのでドメイン名あり
* なので使用するドメイン名のネームサーバーの設定できる必要あり
* VPSはさくらインターネットを使用
    * 筆者は最初カゴヤと契約しようとしたのだけれど、管理画面ログインができなくて諦めた）
* OSはUbuntu(24.04)
    * 特にこだわりはないけれども情報量が比較的多い印象
* 公開鍵の公開にはGithubを使用
* n8nはDockerのイメージからインストール
* https用のwebサーバーはNGINX

## 鍵ペアの生成
Macのターミナルで鍵を生成。ディレクトリはそのままのでリターン。パスフレーズは使わないのでリターン2回。
```
ssh-keygen -t ed25519 -C "your@mail.net"
```
生成結果を確認。
```
cat .ssh/id_ed25519.pub 
cat .ssh/id_ed25519
```

## Githubへの登録
公開鍵(id_ed25519.pubの方 拡張子無しの方ではない)をコピーしてGithubの登録画面で登録
* Githubホーム右上の自分のアイコン→Settings→SSH and GPG Keys→New SSH Key
* メモ用の名前を適当に付けてAuthenticationを選択してさっきの.pubのcatの出力をコピペ


## ローカルのホスト認証の情報を削除
インストールを２回以上実施している場合にはSSHの際にエラーとなるので、認証情報を削除しておく
```
ssh-keygen -R xxx.xxxx.xxx.xxx
```

## VPS契約
1Gで契約(512Mで快適かは未検証)

## VPSへのOSインストール
* 標準OS
* Ubuntu(24.04)
* 管理ユーザー名は自動的に
```
ubuntu
```
* 管理ユーザーのパスワードには安全なものを
    * 筆者のポリシーとしてはMacの自動生成記法で大文字小文字記号数字ありの20文字以上で生成させる
* スタートアップスクリプトなし(アップデートは後々コマンドで実行)
* パスワードを使用したログインを許可する→チェックしない
* コンパネでパケットフィルタ
    * SSH TCP 22 送信元IPアドレス: すべて許可する
    * カスタム TCP 5678 送信元IPアドレス: すべて許可する
    * Web TCP 80/443 送信元IPアドレス: すべて許可する

## ドメイン設定
使用するドメインのサブドメインのAレコードにVPSのアドレスを設定

## SSHログイン
```
ssh ubuntu@xxx.xxxx.xxx.xxx
```
## OSアップデート
管理ユーザーパスワードを聞かれるのでコピペで入力
```
sudo apt-get update && sudo apt-get upgrade -y
```

## OSのセキュリティアップデートを自動化
```
sudo vi /etc/apt/apt.conf.d/50unattended-upgrades
```
コメントを外してtrueに変更
```
// Automatically reboot *WITHOUT CONFIRMATION* if
//  the file /var/run/reboot-required is found after the upgrade
Unattended-Upgrade::Automatic-Reboot "true";
```
コメントを外す
```
// If automatic reboot is enabled and needed, reboot at the specific
// time instead of immediately
//  Default: "now"
Unattended-Upgrade::Automatic-Reboot-Time "02:00";
```
プロセス再起動
```
sudo systemctl restart unattended-upgrades
```

## 必要なコマンド類をインストール
```
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
```

## Dockerのインストール
Dockerの公式GPGキーを追加
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

Dockerリポジトリを追加し、Dockerをインストール
```
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
```
Dockerがインストールされているか確認
```
docker --version
```

## n8nインストール
```
sudo docker pull n8nio/n8n
```

n8nテスト実行（このアクセス方法は継続して使用しないけれども、実行してここまでの作業が正しいかを確認する）
```
sudo docker run -d --name n8n -p 5678:5678 n8nio/n8n
```
docker用のpsでプロセスを確認
```
ubuntu@os3-304-41878:~$ sudo docker ps
CONTAINER ID   IMAGE       COMMAND                  CREATED          STATUS          PORTS                                         NAMES
6aa2239321b4   n8nio/n8n   "tini -- /docker-ent…"   44 seconds ago   Up 44 seconds   0.0.0.0:5678->5678/tcp, [::]:5678->5678/tcp   n8n
```
アクセス（動き出すまで若干時間がかかる気がするのでエラーになっても焦らず待つ）
```
http://xxx.xxx.xxx:5678
```
* （画像を表示）

n8nではhttpでのアクセスが非推奨なのでエラーが表示される。ここで設定を変更して強制的にhttpでもアクセスできるように変更できるけれども、今回はちゃんとSSLで通信できるようになるまで設定する。

停止してコンテナを削除
```
sudo docker stop n8n
sudo docker rm n8n
```

## n8nをボリュームと環境変数を指定して実行
ボリュームをマウントして実行しないとデータが永続化されないので、専用のディレクトリを作成
```
mkdir -p ./n8n/.n8n
```
環境変数指定して実行（  -d --restart unless-stopped はサーバー再起動時にコンテナを自動起動させるためのもの）
```
sudo docker run -d --name n8n \
  -p 5678:5678 \
  -e N8N_BASIC_AUTH_ACTIVE=true \
  -e N8N_BASIC_AUTH_USER=xxx@xxx.xxx \
  -e N8N_BASIC_AUTH_PASSWORD=xxxxxx \
  -e N8N_HOST=xxx.xxx.xxx \
  -e N8N_PORT=5678 \
  -e WEBHOOK_URL=https://xxx.xxx.xxx/ \
  -e GENERIC_TIMEZONE=UTC \
  -v ~/n8n/.n8n:/home/node/.n8n \
  -d --restart unless-stopped \
  n8nio/n8n
```

## NGINXとCertbotのインストール
```
sudo apt update && sudo apt install nginx certbot python3-certbot-nginx -y
```
 NGINXを有効化して起動します：
```
sudo systemctl enable nginx
sudo systemctl start nginx
```
アクセスして動作を確認。xxx.xxx.xxxは使用するドメイン名。
```
http://xxx.xxx.xxx
```
（画像を表示）

## NGINXにリバースプロキシを設定
n8n公開用の設定ファイルを作成
```
sudo vi /etc/nginx/sites-available/n8n
```
新規ファイルなので以下の内容をペーストする。xxx.xxx.xxxは使用するドメイン名。
```
server {
    server_name xxx.xxx.xxx;

    location / {
        proxy_pass http://localhost:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    listen 80;
}
```
設定が読み込まれるようにする
```
sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
```
NGINXを再起動
```
sudo systemctl restart nginx
```
アクセス。xxx.xxx.xxxは使用するドメイン名。先程と同じアドレスでアクセスしているのに今度はn8nの画面が表示されていることを確認できればＯＫ。ただしこの時点では80番での通信。
```
http://xxx.xxx.xxx
```

## httpsへの対応
Let’s Encryptの無料SSL証明書の作成と設定の自動実行。xxx.xxx.xxxは使用するドメイン名。
```
sudo certbot --nginx -d xxx.xxx.xxx
```
```
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Enter email address (used for urgent renewal and security notices)
 (Enter 'c' to cancel): xxx@xxx.xxx

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.5-February-24-2025.pdf. You must
agree in order to register with the ACME server. Do you agree?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: Y

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing, once your first certificate is successfully issued, to
share your email address with the Electronic Frontier Foundation, a founding
partner of the Let's Encrypt project and the non-profit organization that
develops Certbot? We'd like to send you email about our work encrypting the web,
EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: Y
Account registered.
Requesting a certificate for xxx.xxx.xxx

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/xxx.xxx.xxx/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/xxx.xxx.xxx/privkey.pem
This certificate expires on 2025-10-03.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

Deploying certificate
Successfully deployed certificate for xxx.xxx.xxx to /etc/nginx/sites-enabled/n8n
Congratulations! You have successfully enabled HTTPS on https://xxx.xxx.xxx

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

```
これにより先程のnginxの設定ファイルに必要な設定が追記される
```
server {
    server_name xxx.xxx.xxx;

    location / {
        proxy_pass http://localhost:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/xxx.xxx.xxx/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/xxx.xxx.xxx/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}

server {
    if ($host = xxx.xxx.xxx) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    server_name xxx.xxx.xxx;

    listen 80;
    return 404; # managed by Certbot


}

```

証明書更新
```
sudo certbot renew
```

NGINXを再起動
```
sudo systemctl restart nginx
```
アクセス。httpでアクセスするとhttpsにリダイレクトされることを確認する。
```
https://xxx.xxx.xxx
```
(画像)
最初からhttpsでアクセスしても同じ結果が得られる。
```
https://xxx.xxx.xxx
```

## n8nのエラーを確認
この時点で既にn8nが正常稼働しているように見えるものの、ワークフローの作成画面に行くと、以下のエラーが表示される。外部との通信が発生するパーツが使用できない状態。調査したところ、n8nが外部との通信にwebsocketを使っていて、NGINXが通信を通していないために発生する模様（断定できる状態には至らず）。
（画像）
対策のため設定を追加する
```
sudo vi /etc/nginx/nginx.conf
```
httpのセクションのどこかに以下のmapを追記
```
http {
    
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ""      close;
    }

}
```
```
sudo vi /etc/nginx/nginx.conf
```
locationのhttp://localhost:5678が入っている方を以下の内容に書き換え。
```
    location / {
        proxy_pass http://localhost:5678;
        
        chunked_transfer_encoding off;
        proxy_http_version 1.1;
        proxy_cache_bypass $http_upgrade;
        proxy_ssl_server_name on;

        proxy_set_header Host $host;
        proxy_set_header Connection '';
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header X-Real-IP $remote_addr;

    }
```
この変更を行った時点で再起動無しで先程のエラーが消える。

