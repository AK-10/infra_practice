# (静的)webサーバの構築

## vagrantのたちあげ
- `Vagrantfile` を記述
- `$ vagrant up`で仮想環境が立ち上がる

## sshでのログイン
- key作成
    - `$ ssh-keygen -t rsa`
        - `.ssh` いかに作らないとダメだった(chmod 600とかしたけどダメ)

- ~/.ssh/configに以下を記述(`$ vagrant ssh-configに出てくるのをちょっと改変`)
```
Host web_a
    HostName 192.168.33.10
    User vagrant
    IdentityFile {your_secret_key_path}
    Port 22
    UserKnownHostsFile /dev/null
    StrictHostKeyChecking no
    PasswordAuthentication no
    IdentitiesOnly yes
    LogLevel FATAL
```
- `$ ssh web_a`でログイン

## Web Aへのnginxのインストール
### Ansibleの導入
- コントロールマシンに入っていれば良いらしいので(今回はmac),mac側にインストールする
    - `$ brew install ansible`

### 疎通確認
- `$ ansible 192.168.33.10 -m ping -u vagrant`
    - `-u` リモートユーザの指定, 指定しない場合コントロールマシンでログインしているユーザになる

### Ansibleを用いたnginxのインストール
- playbookを書く(task01/provisioning/setup.yml)
- hosts, ansible.cfgをいい感じに書く
- `ansible-playbook setup.yml` (/provisioning にて)


#### ハマったところ
- `become: yes`と書くべきところを`sudo: true`と書いていた. メッセージはsyntax errorだったのでかなりハマった


## nginxの設定
- html, web_a.confを用意
- /etc/nginx/nginx.confのhttpディレクティブにて
    - `include /home/vagrant/nginx/web_a.conf;`を追加
    - `include /etc/nginx/nginx/conf.d/*.conf;`を削除
- html, nginxディレクトリをcopy(ansible)
- nginxをrestart(ansible)

#### ハマったところ
- ansible copyアクションがファイルがすでにあると(内容にかかわらず)実行されない
    - forceつけるべきだった

## http://www.mynetに対するアクセスを受け入れる
- host(mac)の/etc/hostsに以下を追加
    - `192.168.33.10 www.mynet`
- nginxのserver_nameをwww.mynetに変更

- 二つ目はいらないのではと思っているが，どうなんだろう(実際やらなくても動く)
    - portを複数開くのであれば必要な気もする(80ではないもので受け入れる)
    - server_name: http://nginx.org/en/docs/http/request_processing.html
    - server_nameがrequest headerのHostと一致していない場合，defaultのサーバで処理される. defaultのサーバは最初に定義されたものor明示的にdefaultとしたもの
    - 今回だと一個しかないので選ばれた(1つしかないのに選ばれたとは)
        - http://www2.matsue-ct.ac.jp/home/kanayama/text/nginx/node39.html

# DNSサーバの構築
- ssh config
```
Host dns_a
    HostName 192.168.33.20
    User vagrant
    IdentityFile ~/.ssh/infra_practice/task01/id_rsa
    Port 22
    StrictHostKeyChecking no
    PasswordAuthentication no
    IdentitiesOnly yes
    LogLevel FATAL
```

## unboundのインストール
- provisioning/unbound.ymlを参照
- unboundについて: https://gihyo.jp/admin/serial/01/ubuntu-recipe/0386
- ゲスト(dns_a)の`/etc/unbound/unbound.conf`に`include: "/home/vagrant/unbound/unbound.conf"`を追記
    - これは消して/etc/unbound/unbound.conf.d/web_a.confにシンボリックリンクを貼るように変更

- 確認
- `@127.0.0.1`は`@localhost`でも可
    - digは`@*.*.*.*`でネームサーバを指定できる
```sh
vagrant@dns-a:~$ dig www.mynet @127.0.0.1

; <<>> DiG 9.11.3-1ubuntu1.11-Ubuntu <<>> www.mynet @127.0.0.1
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 34318
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.mynet.			IN	A

;; ANSWER SECTION:
www.mynet.		3600	IN	A	192.168.33.10

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Mon Sep 21 20:14:55 UTC 2020
;; MSG SIZE  rcvd: 54
```

- www.google.comへの問い合わせ

```sh
vagrant@dns-a:~$ dig www.google.com @localhost

; <<>> DiG 9.11.3-1ubuntu1.11-Ubuntu <<>> www.google.com @localhost
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 38596
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.google.com.			IN	A

;; ANSWER SECTION:
www.google.com.		67	IN	A	216.58.196.228

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Mon Sep 21 20:21:39 UTC 2020
;; MSG SIZE  rcvd: 59
```

- 正直わからんやった
    - これみた
        - https://katoko.hatenablog.com/entry/2018/05/12/121428c
        - https://hacknote.jp/archives/28932/

## web_aのリゾルバにdns_aを指定してweb_aからwww.mynetが解決できるようにする
- web_aにて`/etc/systemd/resolved.conf`に以下を追記
```
DNS=192.168.33.20
```

-

- 確認

```
vagrant@web-a:~$ systemd-resolve --status
Global
         DNS Servers: 192.168.33.20
          DNSSEC NTA: 10.in-addr.arpa
                      16.172.in-addr.arpa
                      168.192.in-addr.arpa
                      17.172.in-addr.arpa
                      18.172.in-addr.arpa
                      19.172.in-addr.arpa
                      20.172.in-addr.arpa
                      21.172.in-addr.arpa
                      22.172.in-addr.arpa
                      23.172.in-addr.arpa
                      24.172.in-addr.arpa
                      25.172.in-addr.arpa
                      26.172.in-addr.arpa
                      27.172.in-addr.arpa
                      28.172.in-addr.arpa
                      29.172.in-addr.arpa
                      30.172.in-addr.arpa
                      31.172.in-addr.arpa
                      corp
                      d.f.ip6.arpa
                      home
                      internal
```

```
vagrant@web-a:~$ dig www.mynet

; <<>> DiG 9.11.3-1ubuntu1.11-Ubuntu <<>> www.mynet
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51034
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;www.mynet.			IN	A

;; ANSWER SECTION:
www.mynet.		0	IN	A	127.0.0.1

;; Query time: 0 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
;; WHEN: Mon Sep 21 21:33:29 UTC 2020
;; MSG SIZE  rcvd: 54
```

```
vagrant@web-a:~$ dig www.mynet @192.168.33.20

; <<>> DiG 9.11.3-1ubuntu1.11-Ubuntu <<>> www.mynet @192.168.33.20
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 42332
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.mynet.			IN	A

;; ANSWER SECTION:
www.mynet.		3600	IN	A	192.168.33.10

;; Query time: 0 msec
;; SERVER: 192.168.33.20#53(192.168.33.20)
;; WHEN: Mon Sep 21 21:06:07 UTC 2020
;; MSG SIZE  rcvd: 54
```

- これでいいのかわからない, いったんこのまま進める

## ローカルマシンのhostsからwww.mynetを削除して，到達できないことを確認する
- するだけ

## ローカルマシンのリゾルバにdns_aを指定して，www.mynetに到達できることを確認する
- システム環境設定 > ネットワーク > wifi > 詳細... > DNSから192.168.33.20を追加する
- ブラウザでhttp://www.mynetに接続する
- できた

#### ハマったところ
- apparmorで分割したファイルをincludeできない
    - /etc/apparmor.d/local/usr.sbin.unboundに以下を追記
```
/home/vagrant/unbound/dns_a.conf r,
```
    - `sudo systemctl restart apparmor`で解消

- systemd-resolvedとportがバッティングしている
    - systemd-resolvedの設定を変更する
        - `/etc/systemd/resolved.conf` に `DNSStubListener=no` を追記
        - これでパブリックネットワークに繋がらなくなる(apt updateができない)
        - https://qiita.com/gkzz/items/5ac6648e87339e1dd883 これで解決
            - シンボリックリンク張り替えのとこだけやれば良さげ(再起動まで)

#### わからない点
- interface: 0.0.0.0 とは

# アプリケーションサーバの構築
## web aとの疎通確認
    - prev_proxy_aで`ping 192.168.33.10`を実行
    - 特に何もせず疎通している


## rev_proxy_aのリゾルバにdns_aを設定，www.mynetが解決できることを確認
- `/etc/systemd/resolved.conf`に`DNS=192.168.33.20`を追加
- `$ sudo systemctl restart systemd-resolved` を実行
- `$ curl http://www.mynet`

```
vagrant@rev-proxy-a:~$ curl http://www.mynet
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
</head>
<body>
  <h1>Hello www!</h1>
</body>
</html>vagrant@rev-proxy-a:~$
```

OK

## rev_proxyにansibleでnginxをインストール
- すでにやっているので割愛

## rev_proxy_aをリバースプロキシとして動くように設定し，http://www.mynetのリクエストを受けた際にweb_aにリクエストをフォーワードするように設定
- `nginx/rev_proxy_a.conf`を参照
- server_nameをhttp://www.mynetにする
- proxy_passでフォワード先をhttp://192.168.33.20:80にする
- 確認
    - rev_proxy_aにて`curl localhostを実行`
```
vagrant@rev-proxy-a:~$ curl localhost
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
</head>
<body>
  <h1>Hello www!</h1>
</body>
```

## unboundの設定を変更してwww.mynetのipをweb_a -> rev_proxy_aに変更する
- するだけ
- macでdnsサーバの設定を忘れていたのでちょっとハマった

確認(コントロールマシンから実行)
```
~/W/I/i/t/provisioning ❯❯❯ dig www.mynet @192.168.33.20

; <<>> DiG 9.10.6 <<>> www.mynet @192.168.33.20
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 7255
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.mynet.			IN	A

;; ANSWER SECTION:
www.mynet.		3600	IN	A	192.168.33.30

;; Query time: 0 msec
;; SERVER: 192.168.33.20#53(192.168.33.20)
;; WHEN: Fri Sep 25 21:37:42 JST 2020
;; MSG SIZE  rcvd: 54
```

## ブラウザからhttp://www.mynetにアクセスして Hello, www!が表示されることを確認
- web_a, rev_proxy_aの /var/log/nginx/access.logをみて両方にアクセスがきていたらOK

# アプリケーションサーバ(Rails)の構築
## rbenv, ruby-build, rubyのビルドの依存をインストール
### rbenv のインストール
- 手元にインストールするノリで入れるとうまく行かなかった

#### 失敗
- `.bash_profile`にpathを記述指定

```yml
    - name: 'install ruby 2.6.5'
      become: true
      shell: bash -lc "source /home/vagrant/.bash_profile"
```

これだとrbenvを呼べなかった.よくわからない.

#### 成功
- `/etc/profile.d/rbenv.sh` でPATHを追加するようにする
    - `/etc/profile.d/` 配下のファイルはbash起動時に`/etc/profile`によって実行されるため
    - `~/.bash_profile` も実行されるはずなんだけど，なぜか実行されなかった
        - 多分実行ユーザとかの関係でそもそも `~/.bash_profile` が実行されないと予想

```yml
      - name: 'send rbenv.sh to /etc/profile.d'
        become: true
        template:
            src: profile.d/rbenv.sh.j2
            dest: /etc/profile.d/rbenv.sh
            owner: vagrant
            group: vagrant
            mode: 0755

      - name: 'change group rbenv'
        become: true
        file:
            path: "{{ rbenv_root }}"
            owner: vagrant
            group: vagrant
            recurse: true
            state: directory
```

## node v13.11.0
- [NodeSource](https://github.com/nodesource/distributions/blob/master/README.md)からインストールしてみる
- どうもメジャーバージョンしか指定できなさそうなので諦める🥱

- `n` コマンドを使う
```yml
      # テキトーなnodejs, npmを入れる(8.10.0)
      - name: 'install old node'
        become: true
        apt:
            pkg:
            - nodejs
            - npm

      # コマンドのインストール(全体で使うのでglobalオプションを使う)
      - name: 'install n'
        become: true
        shell: npm install -g n

      # nでnode 13.11.0を入れる
      - name: 'install node 13.11.0'
        become: true
        shell: n install 13.11.0

      # テキトーに入れた方のnodejs, npmを削除
      - name: 'purge old node'
        apt:
            name:
            - nodejs
            - npm
            purge: true
```

## yarn 1.22.4のインストール

- ansibleのnpm pluginを使えるようにする
    - `ansible-galaxy collection install community.general` (コントロールマシンで)
    - リモートマシンのnpmのパスが/usr/local/binでないと動かない可能性がある
- npmで入れる
    ```yml
    name: 'install yarn 1.22.4'
    become: true
    community.general.npm:
        name: yarn
        global: true
        version: 1.22.4
    ```

## TODO: ansibleの警告を消す
```
[WARNING]: sftp transfer mechanism failed on [192.168.33.10]. Use ANSIBLE_DEBUG=1 to see detailed information
[WARNING]: scp transfer mechanism failed on [192.168.33.10]. Use ANSIBLE_DEBUG=1 to see detailed information
```
ずっとこんなのが出てる

## railsアプリケーションの作成
### 方針
- ansibleでbundlerをインストールする
- railsアプリケーションは手元で作ってgithubに置いておく
- gitモジュールでとってくる
- ansibleから構築
- ansibleから起動


### rails new
- dbなしにする
  - `bundle exec new . -O` をするとactive_recordを利用しなくなる
  - dbの接続設定もいらなくなる


## web_aからcurlでhttp://localhost:3000っでレスポンスを確認
- やるだけ

## unboundに`nodb-app.mynet` => web_aとなるIPアドレスを解決する設定を追加し，ブラウザから確認
- unbound/dns_a.confに `local-data: "nodb-app.mynet. IN A 192.168.33.10"` を追加
- sshでweb_aに入り, `bundle exec rails s -b 192.168.33.10` を実行
    - FQDNを設定しないとlocalhostからのアクセスしか受け付けず，ホストマシンからのリクエストを受け付けなくなる
    - また, railsのconfig/environments/development.rbとかに`config.hosts << "nodb-app.mynet"`を追加

- もしくは`bundle exec rails s -b nodb-app.mynet`でも行けるかも
    - これだとrails側の設定は特にいらなそう


## web_aのnginxにnodb-app.mynetの80番ポートへのリクエストを3000番ポートにフォワードする設定を追加し，ブラウザからhttp://nodb-app.mynetにアクセスして`Yay! You're on Rails`が表示されることを確認
- nginxに以下を追加

```web_a.conf
server {
    listen 80;
    server_name nodb-app.mynet;
    location / {
        proxy_pass http://localhost:3000;
    }
}
```

- serverディレクティブを二つに分けないと，元のwww.mynetが機能しなくなる(多分)

- 確認
    - 一つ前でrailsを`bundle exec rails s -b 192.168.33.10`にしていたが，今回は特にFQDNは指定しない
        - localhostに対してproxyをするようにしているため
        - `proxy_pass http://192.168.33.10:3000`だったらつける必要がある（多分）

# データベースサーバの構築
## ansibleを使いdb_aにmysql5.7をインストール
- aptでいれる

## rootにパスワードを設定する
- mysql_secure_installationを使う
    - いろいろいじって--skip-grant-tablesをつけていたのでうまく行かなかった

## mysqlコマンドでrootログイン, webappユーザを追加

- webappユーザの制約
    - パスワードを設定
    - 全てのデータベース, テーブルへのアクセスを許可
    - SELECT INSERT UPDATE DELETEのみ実行可能
    - localhost / 同一サブネットからのアクセスのみ許可


### ユーザー作成とパスワードの設定
```sql
CREATE USER 'webapp'@'localhost' IDENTIFIED BY '***';
```

## localhost / 同一サブネットからのアクセスのみ許可
ユーザ作成時点ではlocalhostからのアクセスしか許可しないのでwebappユーザをもう一つ追加
サブネットは`192.168.33.*`
```sql
CREATE USER 'webapp'@'192.168.33.%' IDENTIFIED BY '***';
```

#### 変更前
```mysql
mysql> select user, host from mysql.user;
+------------------+-----------+
| user             | host      |
+------------------+-----------+
| debian-sys-maint | localhost |
| mysql.session    | localhost |
| mysql.sys        | localhost |
| root             | localhost |
| webapp           | localhost |
+------------------+-----------+
```

#### 変更後
```mysql
mysql> select user, host from mysql.user;
+------------------+--------------+
| user             | host         |
+------------------+--------------+
| webapp           | 192.168.33.% |
| debian-sys-maint | localhost    |
| mysql.session    | localhost    |
| mysql.sys        | localhost    |
| root             | localhost    |
| webapp           | localhost    |
+------------------+--------------+
```

#### 全てのデータベース, テーブルへのアクセスを許可, SELECT INSERT UPDATE DELETEのみ実行可能
```mysql
mysql> GRANT SELECT, INSERT, UPDATE, DELETE ON *.* TO 'webapp'@'localhost';
Query OK, 0 rows affected (0.00 sec)

mysql> GRANT SELECT, INSERT, UPDATE, DELETE ON *.* TO 'webapp'@'192.168.33.%';
Query OK, 0 rows affected (0.00 sec)

mysql> SHOw GRANTS FOR 'webapp'@'localhost';
+---------------------------------------------------------------------+
| Grants for webapp@localhost                                         |
+---------------------------------------------------------------------+
| GRANT SELECT, INSERT, UPDATE, DELETE ON *.* TO 'webapp'@'localhost' |
+---------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> SHOw GRANTS FOR 'webapp'@'192.168.33.%';
+------------------------------------------------------------------------+
| Grants for webapp@192.168.33.%                                         |
+------------------------------------------------------------------------+
| GRANT SELECT, INSERT, UPDATE, DELETE ON *.* TO 'webapp'@'192.168.33.%' |
+------------------------------------------------------------------------+
1 row in set (0.00 sec)
```

## tcp接続を許可し，web_aからmysqlコマンドでdb_aにwebappでアクセスできることを確認
### tcp接続(リモートアクセス)を許可
`/etc/mysql/mysql.conf.d/mysqld.cnf`の`bind-address=127.0.0.1`を`0.0.0.0`に変更

mycnfをansibleで配布してもよかったかも．

### 確認(web_aに事前にmysql-clientを入れること)
```shell
vagrant@web-a:~$ mysql -u webapp -h 192.168.33.40 --protocol=TCP -p
Enter password:
ERROR 2003 (HY000): Can't connect to MySQL server on '192.168.33.40' (111)
vagrant@web-a:~$ mysql -u webapp -h 192.168.33.40  -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.31-0ubuntu0.18.04.1 (Ubuntu)

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> ^DBye
```

## 参考
- https://dev.mysql.com/doc/refman/5.7/en/connection-access.html

# MySQLを使い，リバースプロキシでリクエストをうけるrailsアプリケーションサーバの構築
## やるべきこと
- 自己証明書の発行
- web_aにアプリケーションサーバを構築(productionモード)
- assetsはnginxから配信
- リクエストはrev_proxyからのみ許可

段階を踏む
- web_aにアプリケーションサーバを構築
- web_aへのリクエスト(192.168.33.10:3001)で正しく動いていることを確認する
    - dbへの書き込み，読み込み
- proxy_aからのリクエストのみ許可する
- proxy_aへのリクエストで正しく動いていることを確認する
- web_aへの直接リクエストで弾かれることを確認する
- ssl証明書を発行する
- httpsでアクセスできることを確認する


### やりたいこと
- [client] -- req --> [dns_a] -- (resolve) --> [proxy_a(nginx)] -- (rev proxy) --> [web_a(nginx 192.168.33.10:80)] --> [web_a(unicorn, http://localhost:3000)] --> [web_a(rails process)] --> [db_a(mysql)]

となるように設定すれば良い.
いろいろみていると[web_a(nginx)] --> [web_a(unicorn)]はunixsocketでもできるっぽいが，今回はしない

### やること
- unicornの導入
- nginxの設定変更
- productionで動かすためのwebapp.serviceの作成
    - これはどうやればいいのかわからんので, systemdで起動するようにした

## unicornの設定
config/unicorn.rbを参照

## nginxの設定
nginx -> unicornでhttp header等もproxyする

web_a.conf
```nginx
server {
    listen 80;
    server_name www.mynet;

    location / {
        # root /home/vagrant/html;
        # index index.html;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_pass http://localhost:3001;
    }

    # error_page   500 502 503 504  /50x.html;
    # location = /50x.html {
    #     root   /usr/share/nginx/html;
    # }
}

server {
    listen 80;
    server_name nodb-app.mynet;
    location / {
        proxy_pass http://localhost:3001;
    }
}

```

## db作成
- `bundle exec rails db:create`で作成
- webappにCREATEがなかったのでつける,もしくは手で作る. (後で確認)
    - 多分マイグレーションとかができないのでCREATEも作ったほうが良さげ
    - もしくはマイグレーション用のユーザを作る

db_aでrootユーザでmysqlにログイン
```mysql
mysql> GRANT CREATE, SELECT, INSERT, UPDATE, DELETE ON *.* TO 'webapp'@'192.168.33.%';
mysql> GRANT CREATE, SELECT, INSERT, UPDATE, DELETE ON *.* TO 'webapp'@'localhost';
mysql> FLUSH PRIVILEGES;
```
## デプロイ
- web_aに対してansible-playbookを実行する
- 以下のエラーが発生

```infra-web-a/log/unicorn-stderr.log
E, [2020-10-31T09:56:25.413783 #14455] ERROR -- : /usr/local/rbenv/versions/2.6.5/bin/bundle:23:in `load'
E, [2020-10-31T09:56:25.413871 #14455] ERROR -- : /usr/local/rbenv/versions/2.6.5/bin/bundle:23:in `<main>'
E, [2020-10-31T09:56:27.039311 #14456] ERROR -- : app error: Missing `secret_key_base` for 'production' environment, set this string with `rails credentials:edit` (ArgumentError)
```

secret_key_baseがないので追加する.
実際にはcredentials.yml.encはあるのでmaster.keyを設置すれば良い

今回はコントロールマシンから送信するようにした

### Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock' (2))
デプロイして192.168.33.10:3001にアクセスすると
`Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock' (2))` のエラーが出た

確認できなかったが,RAILS_ENVを正しく設定できていないのでdbをlocalhostに繋ぎに行こうとしてsockを見に行った結果エラーになっている
`/etc/profile.d/env.sh` で`export RAILS_ENV=production`をしていたが，どうもうまくいっていなかったっぽい

- 対処
 - webappのunitfileにEnvironmentFileの指定を追加する
webapp.service
```
[Unit]
Description=webapp - rails application via unicorn
Before=nginx.service

[Service]
PIDFile=/home/vagrant/infra-web-a/tmp/pids/unicorn.pid
WorkingDirectory=/home/vagrant/infra-web-a
SyslogIdentifier=webapp-unicorn
EnvironmentFile=/home/vagrant/env/production.env <- 追加
ExecStart=/usr/local/rbenv/shims/bundle exec unicorn_rails -D -c config/unicorn.rb -E production
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID

[Install]
WantedBy=multi-user.target
```

~/env/production.env
```
RAILS_ENV='production'
DB_HOST='192.168.33.40'
DB_USER='webapp'
DB_PASSWORD='*******'
```

でうまく動いた(が，`bundle exec rails c` とかが動かなくなるので`RAILS_ENV=production bundle exec rails c`でしないといけなくなる. 多分．)

todo: update, deleteができない. 多分rails-ujsが読み込めていない

## nginx経由でアクセスする(rev_proxy_a 以外のアクセスを拒否する)
- web_aのwww.mynetに対してallow, denyを設定する
    - 192.168.33.30(rev_proxy_a)をallow. それ以外はdenyにする.

- www.mynet/todosで表示されて, 192.168.33.10/todosで表示されなければ(403なら)OK


## assetsファイルはrailsではなくnginxから返すようにする
js, css, png, jpg, みたいなのをnginxで返すように設定する
web_a.confに以下を追加

```
    location ~ ^/assets/ {
        root /home/vagrant/infra-web-a/public;
    }

```
