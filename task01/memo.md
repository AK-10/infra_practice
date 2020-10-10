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

### node v13.11.0
- [NodeSource](https://github.com/nodesource/distributions/blob/master/README.md)からインストールしてみる


# データベースサーバの構築
- pending

# MySQLを使い，リバースプロキシでリクエストをうけるrailsアプリケーションサーバの構築
- pending

