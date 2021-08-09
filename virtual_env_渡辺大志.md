## バージョン一覧
| name | version |
|:---:|:---:|
| **PHP** | 7.3 |
| **Nginx** | 1.21.1 |
| **MySQL** | 5.7.35 |
| **Laravel** | 6.20.1 |
| **CentOS** | 7.9.2009 |

## 環境構築の流れ
### 1. vagrant box ダウンロード
Linux CentOSバージョン7を指定し実行する。
```
vagrant box add centos/7
```
コマンド実行で下記の選択肢が表示されるため、3を選択する。
```shell
1) hyperv
2) libvirt
3) virtualbox
4) vmware_desktop

Enter your choice: 3
```
下記のように表示されれば導入完了。
```shell
Successfully added box 'centos/7' (v1902.01) for 'virtualbox'!
```
### 2. 作業ディレクトリ作成 && 移動
```
 mkdir vagrant_virtual_env && cd _$
 ```
### 3. Vagrant初期設定
```
　vagrant init centos/7
```
上記コマンドにてvagrantfileが作成されるため、そちらを編集する。

**編集箇所**
* 下記２行のコードをコメントイン(#を外す)
```
#変更①
config.vm.network "forwarded_port", guest: 80, host: 8080

#変更②
config.vm.network "private_network", ip: "192.168.33.19"
```
* 下記コードを編集
```
# 変更点③
config.vm.synced_folder "../data", "/vagrant_data"
# ↓ 以下に編集
config.vm.synced_folder "./", "/vagrant", type:"virtualbox"
```
### 4. vagrant up 準備
* ポート開放
既にポートが使用されている場合は先にポートを開放する。
今回はserver_lessonの時とポートが一緒(8080)であるため、先にポートを開放する。ターミナルを開き、
	```shell
	lsof -i -P | grep :8080
	```
	と入力する。ポート8080が使用されていなければ何も表示されないが、使用されていると、
	```
	VBoxHeadl 41296 username 18u  IPv4 **************  0t0  TCP *:8080 (LISTEN) 
	```
	上記のように表示される。プロセスIDは41296なので、更に
	```
	kill 41296
	```
	と入力すればポートを開放できる。
* Vagrantプラグインのインストール
	* vagrant-vbguest
		Boxの中にインストールされているGuest Additionsのバージョンを、VirtualBoxのバージョンに合わせて最新化するプラグイン。
		```
		vagrant plugin install vagrant-vbguest
		```
	* sahara
		ある時点までの仮想環境の状態を保存し、変更された後に保存した時点まで戻すことができるプラグイン。取り返しのつかない変更をしてしまった際に復元する際に有用。
		```
		vagrant plugin install sahara
		```
		上記プラグインインストール後、
		```
		vagrant plugin list
		```
		にてプラグイン一覧を確認できる。

### 5. Vagrant を使用しゲストOS起動
```shell
vagrant up
```
* **起動中にエラーが発生した場合**
	#### 原因
	----
	vagrant boxが`CentOS 7.8`で構成されており、最新の`CentOS7.9`とバージョンの相違がある。そのためvbguestでVirtualBox Guest Additionsの更新をする際に`karnel-evel`が取得できずにエラーを起こしている。
	#### 対応方法
	---
	ターミナルで`karnel`のアップデートを行う。
	1. `vagrant up` を失敗
	2. `vagrant ssh`
	3. `sudo yum -y update karnel`
	4. `exit`
	5. `vagrant reload --provision`
	---
上記対応後再び`vagrant up`を行うことでゲストOSが起動する。
#### 6. ゲストOSへログイン
```shell
$ vagrant ssh
```
コマンド実行後、下記のように表示されればログイン成功。
```
[vagrant@localhost ~]$
```

---

### 7. 各種パッケージインストール (ゲストOS内)
* `development tools`
	gitなどの開発に必要なパッケージを一括でインストールできる。
	```
	sudo yum -y groupinstall "development tools"
	```

* PHP
	PHPを下記コマンドでインストール。
	`php -v`でバージョン確認(インストールできているか確認)
	```
	sudo yum -y install epel-release wget
	sudo wget http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
	sudo rpm -Uvh remi-release-7.rpm
	sudo yum -y install --enablerepo=remi-php72 php php-pdo php-mysqlnd php-mbstring php-xml php-fpm php-common php-devel php-mysql unzip
	php -v
	```
* composer
	PHPのパッケージの依存関係を管理・解決するツール。
	下記コマンドでComposerインストール＆バージョン確認
	`sudo mv composer.phar /usr/local/bin/composer`でどのディレクトリでもcomposerを使えるようにする。
	```
	php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
	php composer-setup.php
	php -r "unlink('composer-setup.php');"
	sudo mv composer.phar /usr/local/bin/composer
	composer -v
	```
---

### 8. Laravelアプリケーションのコピー作成 (ホストOS)
`exit`でゲストOSからログアウトし、Laravelプロジェクトのコピーを`vagrant_virtual_env`ディレクトリ下へ作成する。
```shell
cd vagrant_virtual_env
cp -r laravel_appディレクトリまでの絶対パス ./
```
### 9. Laravelを動かせる状態にする(ゲストOS)
`vagrant ssh`でゲストOSへログインし、vagrantディレクトリへ移動。
```
cd /vagrant
```
---
#### laravel_appディレクトリが無かった場合
`ls`でもしlaravel_appが無かった場合、ホストOSとの同期ができていない可能性があるため、一旦`exit`でログアウトし、ホストOSにて、
```
vagrant reload
```
を実行し、同期させる。
再度`vagrant ssh`でゲストOS、`vagrant`ディレクトリで確認する。

---

* #### Laravelを6.0へアップデートする
	まず、laravel_app内の **composer.json** をviエディタで編集する。
	```
	vi laravel_app/vomposer.json
	```
	編集箇所は''require"の中身で、
	```
	"require": {
		"php": "^7.1.3",
		"fideloper/proxy": "^4.0",
		"laravel/framework": "^6.0", # 5.7 -> ^6.0へ変更
		"laravel/helpers": "^1.4",
		"laravel/tinker": "^1.0",
		"laravelcollective/html": "^6.0" # 5.7 ->6.0へ変更
	}
	```
	上記のように編集する。編集後にcomposer updadeを行う。
	```
	composer updade
	```
	完了直前に下記のようなエラーが出る。
	```
	Call to undefined function str_slug()
	```
	この場合、追加で
	```
	composer require laravel/helpers
	```
	と入力すればインストールができる。
	このコマンドは[アップグレードガイド laravel6.x](https://readouble.com/laravel/6.x/ja/upgrade.html#helpers)より、
	>アプリケーションへ新たに`laravel/helpers`パッケージを追加すれば、こうしたヘルパを今までどおり利用できます。
	
	ということである。
	アップデート時に、
	```
	Package manifest generated successfully.
	54 packages you are using are looking for funding.
	Use the `composer fund` command to find out more!
	```
	と表示されればアップデート完了。
* #### laravel authを実行できるようにする
	laravel_appディレクトリ内にて、
	```
	sudo yum install nodejs
	npm install
	npm install prettier
	npm install js-beautify 
	npm install bootstrap-sass
	```
	をそれぞれ実行し、npmをインストールする。インストール後、
	```
	npm run dev
	```
	にてnpm を実行する。エラーが表示されなければ完了。
	次に、下記コマンドを実行する。([Readouble 認証6.0より](https://readouble.com/laravel/6.x/ja/authentication.html))
	```
	composer require laravel/ui "^1.0" --
	dev php artisan ui vue --auth
	```
	`dev php artisan ui vue --auth`を実行すると下記のようにyes/noの選択肢が出てくるので、
	```
	The [auth/login.blade.php] view already exists. Do you want to replace it? (yes/no) [no]:
	> no
	The [auth/passwords/confirm.blade.php] view already exists. Do you want to replace it? (yes/no) [no]:
	> yes
	The [auth/passwords/email.blade.php] view already exists. Do you want to replace it? (yes/no) [no]:
	> yes
	The [auth/passwords/reset.blade.php] view already exists. Do you want to replace it? (yes/no) [no]:
	> yes
	The [auth/register.blade.php] view already exists. Do you want to replace it? (yes/no) [no]:
	> no
	The [auth/verify.blade.php] view already exists. Do you want to replace it? (yes/no) [no]:
	> yes
	The [home.blade.php] view already exists. Do you want to replace it? (yes/no) [no]:
	> yes
	The [layouts/app.blade.php] view already exists. Do you want to replace it? (yes/no) [no]:
	> yes
	```
	上記のように回答すれば、
	```
	Authentication scaffolding generated successfully.
	```
	と表示されるので、これでauthの導入は完了。
---

### 10. Nginxの導入
* #### Nginx をインストール
	一旦ワーキングディレクトリに戻る。
	```
	cd /vagrant
	```
	viエディタを使用し、以下のファイルを作成する。
	```
	sudo vi /etc/yum.repos.d/nginx.repo
	```
	以下の内容を書き込む。
	```
	[nginx]
	name=nginx repo
	baseurl=https://nginx.org/packages/mainline/centos/\$releasever/\$basearch/
	gpgcheck=0
	enabled=1
	```
	書き込みが完了したら以下のコマンドを入力し、Nginxをインストールする。
	```
	sudo yum install -y nginx
	nginx -v
	```
	バージョンが確認できればインストール完了。下記コマンドにてNginxを起動する。
	```
	sudo systemctl start nginx
	```
	[http://192.168.33.19](http://192.168.33.19/)でwelcomeが表示されれば起動完了。
* #### laravel を動かせるようにする
	設定ファイルである、`/etc/nginx/conf.d` ディレクトリ下の `default.conf` ファイルを編集する。
	```
	sudo vi /etc/nginx/conf.d/default.conf
	```
	以下の内容を編集する。
	```
	server {
		listen  80; server_name  192.168.33.19; # Vagranfileでコメントを外した箇所のipアドレスを記述。  
		# ApacheのDocumentRootにあたる
		root /vagrant/laravel_app/public; # 追記
		index index.html index.htm index.php; # 追記
		
		#charset koi8-r;
		#access_log /var/log/nginx/host.access.log main;
		
	location / {
		#root /usr/share/nginx/html; # コメントアウト 
		#index index.html index.htm; # コメントアウト
		try_files  $uri  $uri/ /index.php$is_args$args; # 追記
	}
	# 省略
	
	# 該当箇所のコメントを解除し、必要な箇所には変更を加える 
	# 下記は root を除いたlocation { } までのコメントが解除されていることを確認。
	
	location  ~ \.php$ {
		# root html; 
		fastcgi_pass  127.0.0.1:9000;
		fastcgi_index index.php;
		fastcgi_param SCRIPT_FILENAME /$document_root/$fastcgi_script_name; # $fastcgi_script_name以前を /$document_root/に変更
		include fastcgi_params;
	}
	# 省略
	```
	次に 、 ` php-fpm ` の設定ファイルを編集する。
	```
	sudo vi /etc/php-fpm.d/www.conf
	```
	変更箇所は、
	```plain
	;24行目近辺
	user = apache
	# ↓ 以下に編集
	user = nginx

	group = apache
	# ↓ 以下に編集
	group = nginx
	```
	で、設定ファイルの編集は以上。
	Nginxは再起動が必要なので、下記コマンドで再起動する。
	```
	sudo systemctl restart nginx
	sudo systemctl start php-fpm
	```
	[http://192.168.33.19](http://192.168.33.19/)を開くとlaravelのエラーが発生する。
	```
	The stream or file "/vagrant/laravel_app/storage/logs/laravel.log" could not be opened: failed to open stream: Permission denied
	```
	php-fpmの設定ファイルのuserとgroupのNginxが許可されていないため権限が拒否されているエラーなので、
	```shell
	cd /vagrant/laravel_app
	sudo chmod -R 777 storage
	```
	`chmod -R　７７７ storage` で、指定したディレクトリ以下のディレクトリ・ファイル全てを権限付与に一括変更できる。
	[http://192.168.33.19](http://192.168.33.19/)を開いてLravelのwelcomeが表示されれば完了。


## 環境構築の所感
### 気づいたこと、学んだこと
* Laravelのバージョンが違うだけで、プラグインやauthのコマンドなど、必要なものが全く違うことに気付いた。
* ゲストOSを立ち上げる際にポートが使用されているとポートの開放が必要であったり、ipアドレスが違うとvagrant ssh-configの情報をもとにログインする方法でpc側から中間攻撃者かを疑われるなど、ホストOSでの処理などについて新たに学んだ。

## 参考サイト
* [Qiita Vagrant + VirtualBOx で 最新のCentOS7 vbox(centos/7 2004.01)でマウントできない問題](https://qiita.com/mao172/items/f1af5bedd0e9536169ae)
* [Qiita VagrantにSaharaを導入](https://qiita.com/sudachi808/items/09cbd3dd1f5c25c23eaf)
* [Qiita【Vagrant + CentOS 7】のまっさらな状態からLaravel5.5をいれて動かすまでに詰まったことをメモ](https://qiita.com/shin1kt/items/71b42c703d8d43c5d1ec)
* [Qiita Laravelを5.8→6.0にアップデートしただけの話](https://qiita.com/June8715/items/6b3c07a022c0ef02f903)
* [Qiita Laravel npm run devでエラーが発生した話](https://qiita.com/miriwo/items/2d96062d38031e4c7944)
* [laravel 6.x アップグレードガイド](https://readouble.com/laravel/6.x/ja/upgrade.html)
* [Qiita マークダウン記法 一覧表・チートシート](https://qiita.com/kamorits/items/6f342da395ad57468ae3)
