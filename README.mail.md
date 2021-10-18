# mailserver インストール

## postfix dovecot設定
ココを参照 https://monmon.jp/296/launching-the-mail-server-sasl-authentication-on-the-postfix-and-dovecot/
上記webの手順+カスタマイズした結果が↓

### postfix と dovecot のインストール
```
$ sudo apt install postfix
#  選択は
#  internet mail
#  mail.sample-domain.com FQDN ドメイン名
$ sudo install dovecot-core dovecot-imapd dovecot-pop3d
$ sudo systemctl status postfix
$ sudo systemctl status dovecot
```
#### postfixを設定 /etc/postfix/main.cf
##### メールアドレスを隠す セキュリティーのためなんだけどなんだかわからない
```
#smtpd_banner = $myhostname ESMTP $mail_name (Ubuntu)
smtpd_banner = $myhostname ESMTP
```
##### TLS parameters
```
#smtpd_tls_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
#smtpd_tls_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
smtpd_tls_cert_file = /etc/letsencrypt/live/mail.sample-domain.jp/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/mail.sample-domain.jp/privkey.pem
```
##### ホストネームなどなど
```
myhostname = mail.sample-domain.com
mydomain = sample-domain.com
myorigin = $myhostname
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
```
##### Maildir形式を使います
```
home_mailbox= Maildir/
```
##### 内部ユーザーを隠します
```
disable_vrfy_command = yes
```

##### SASL認証の設定も諸々有効にしておきます。
```
smtpd_sasl_auth_enable = yes
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
broken_sasl_auth_clients = yes
smtpd_recipient_restrictions =
    permit_mynetworks
    permit_sasl_authenticated
    reject_unauth_destination

smtpd_use_tls = yes
```
#### SMTP認証の設定　/etc/postfix/master.cf

##### プロトコルを有効化
```
smtps     inet  n       -       y       -       -       smtpd
#  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
#  -o smtpd_reject_unlisted_recipient=no
```
SMTPs 465 用設定
587 submission portも動かす場合はsubmissonを有効にする
```
submission inet n       -       y       -       -       smtpd
#  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
#  -o smtpd_tls_auth_only=yes
```
##### 終わったら再起動を忘れずに
```
$ sudo postfix check   # 設定のチェック
$ sudo systemctl restart postfix
```
#### dovecot設定 /etc/dovecot/conf.d/10-master.conf
##### SASLの設定
```
# SSLでないポートは無効
service imap-login {
  inet_listener imap {
    #port = 143
    port = 0
  }
  inet_listener imaps {
    #port = 993
    #ssl = yes
    port = 993
    ssl = yes
  }
}

service pop3-login {
  inet_listener pop3 {
    #port = 110
    port = 0
  }
  inet_listener pop3s {
    #port = 995
    #ssl = yes
    port = 995
    ssl = yes
  }
}

 # Postfix smtp-auth
  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
  }
```
##### 証明書の設定 /etc/dovecot/conf.d/10-ssl.conf
```
#ssl_cert = </etc/dovecot/private/dovecot.pem
#ssl_key = </etc/dovecot/private/dovecot.key
ssl_cert = </etc/letsencrypt/live/mail.sample-domain.com/fullchain.pem
ssl_key = </etc/letsencrypt/live/mail.sample-domain.com/privkey.pem
```
##### メールボックスをMaildirに変更 /etc/dovecot/conf.d/10-mail.conf
```
#mail_location = mbox:~/mail:INBOX=/var/mail/%u
mail_location = maildir:~/Maildir
```
##### 再起動を忘れずに
```
sudo systemctl restart dovecot
```
##### 新規ユーザ作成時にディレクトリが作られるようにします
```
sudo mkdir /etc/skel/Maildir
sudo chmod 700 /etc/skel/Maildir
```
#### メールユーザー追加
```
adduser [ユーザー名]
#パスワード以外はほぼエンター
```

# OP25 outbound25に対する対応 AWSに解除の申請が必要
いまのところgmailにたいする受信ができない
たぶんec2 アウトバウンド port25の制限のためと思われるので下記を参考に制限解除中　４８時間まち　8/31
  --> 結局２回目の申請で解除できた　多分ちゃんとメール来る？とかで確認している気がする
https://qiita.com/BlueBaybridge/items/9a97c82ff611d9f28d55
もっとちゃんとかけって怒られて解除してくんない
https://it.becrazy.jp/article/ec2-port-25-throttle
とりあえずこの通りにやってみる
解除できました　gmailだと迷惑メールになるけどとりあえずメールサーバの機能は動いた
SPF,DKIMなどの対応でgmailも迷惑メールにならずに送受信できました。

Request to remove email sending limitations
https://aws-portal.amazon.com/gp/aws/html-forms-controller/contactus/ec2-email-limit-rdns-request

## 申請した内容
### 1回目
下記を参考に記入して下さい。

Request information
・Email address
　AWSからの連絡を受信できるメールアドレスを入力してください。

・Use case description
　Please removal of Email sending limit and create reverse DNS record.

Elastic IPs information
・Elastic IP address - optional
　対象メールサーバのElasticIPを入力してください。
　例）123.124.125.126

・Reverse DNS record - optional
　対象メールサーバのDNS名を入力してください。
　例）mail.example.com

submitで送信してから、数時間～最大で48時間の営業時間待ちが必要です。
承認されるとAWSからメールが届きます。

### ２回目の use case
なんとなく２回やれば解除してくれるっぽい
メールアドレスで連絡できるのかを見ているのかもしれない
Build a mail server for office workers.

The following measures will be taken to prevent the sending of spam emails

* Build MTA and take necessary measures such as SMTP-AUTH / SPF / DKIM

* rDNS is set up

# よりセキュアに gmailで迷惑メールに振り分けられないために
## SPF
https://monmon.jp/301/introspates-spf-dkim-dmarc-to-avoid-spam-maetization/
DNSにtxtレコード追加 value "v=spf1 ip4:IPアドレス -all"

## DKIM
まずはインストール
```
$ sudo apt install opendkim opendkim-tools
```

鍵を作るためにディレクトリを用意 ドメインである必要はないぞ
```
$ sudo mkdir /etc/dkimkeys/sample-domain.com
```

selectorは指定せずにdefaultで鍵を作った　複数サーバだとselector指定しなきゃですね
```
$ sudo opendkim-genkey --directory=/etc/dkimkeys/sample-domain.com --domain=sample-domain.com --bits=2048
```
鍵を作るためのディレクトリはここでだけ必要だから適当な名前でいいっすよね

/etc/dkimkeys/sample-domain.com/ 以下に下記２つのファイルができてるはず
default.private
default.txt
このファイルはopendkimから参照されるっぽいのでopendkimのユーザー、グループに変更してアクセス可能に
```
$ sudo chown -R opendkim:opendkim /etc/dkimkeys/sample-domain.com
```

### opendkimの設定 /etc/opendkim.conf
```
Socket                  inet:8892@localhost
#Socket                 local:/run/opendkim/opendkim.sock

#Mode                    sv
Mode                    sv
```
最下行に追加
```
KeyTable    /etc/dkimkeys/KeyTable
SigningTable    refile:/etc/dkimkeys/SigningTable
ExternalIgnoreList    refile:/etc/dkimkeys/TrustedHosts
InternalHosts    refile:/etc/dkimkeys/TrustedHosts
```
#### 上記の設定ファイルの内容を作成
##### /etc/dkimkeys/KeyTable
こにはドメインキーと秘密鍵を紐付け
```
default._domainkey.sample-domain.com sample-domain.com:default:/etc/dkimkeys/sample-domain.com/default.private
```
##### /etc/dkimkeys/SigningTable
ドメインキーとメールユーザの紐付け ----
```
*@sample-domain.com default._domainkey.sample-domain.com
```
##### /etc/dkimkeys/TrustedHosts
信頼できるホスト指定 ここはローカルのみ ----
```
127.0.0.1
::1
```
#### postfixにdkimを設定する /etc/postfix/main.cf
最下行に追加
```
milter_default_action = accept
smtpd_milters = inet:127.0.0.1:8892
non_smtpd_milters = $smtpd_milters
```
### 再起動を忘れずに
```
$ sudo systemctl restart postfix
$ sudo systemctl restart opendkim
```

### DNS(route53)に公開鍵を登録
/etc/dkimkeys/sample-domain.com/default.txtの内容をDNS TXTレコードに登録
default._dmainkey.sample-domain.com txt "v=DKIM1; h=sha256; k=rsa; p=...."
#valueはダブルクォーテーションで区切る必要あり　長いため　その時に改行が含まれているとダメ！って仕様らしい
#一度エディタで編集して改行を消すこと

## ADSPレコードも追加
_adsp.domainkey.sample-domain.com txt dkim=unknown

## DMARCも追加 ---
_dmarc.sample-domain.com txt "v=DMARC1; p=none"

## 以上で一通りの設定は終了 ---
# まだスパムメール受信対策ができていないのでどーするか？？　(9/1)

# SPF,DKIMなど　設定し終わったら確認はこちら↓
https://qiita.com/toshihirock/items/ad46e32721d1d301786b


