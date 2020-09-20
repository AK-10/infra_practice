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

# アプリケーションサーバの構築

# データベースサーバの構築

# MySQLを使い，リバースプロキシでリクエストをうけるrailsアプリケーションサーバの構築

