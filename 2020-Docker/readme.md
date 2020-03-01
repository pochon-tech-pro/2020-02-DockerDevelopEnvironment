## 2020-Docker

### 概要

- Docker関連の知見

### Volumesの問題

<details>
<summary>docker-volumes-owner-linux</summary>

## dockerでvolumeをマウントするときの問題点
- `docker run`するときに-v オプションをつけることによってホストのディレクトリをコンテナ内にマウントすることができる
- ホスト側のファイルをコンテナ内で使いたい場合や、逆にコンテナで作ったファイルにホストからアクセスしたい場合に有用
- しかし、ファイルのアクセス権限についてちゃんと考えておかないと問題が起きることがある

**例: ホスト内でのユーザーのuidが500の場合**
```sh:
$ id
uid=500(ec2-user) gid=500(ec2-user) groups=500(ec2-user),10(wheel),497(docker)
```
- volumeをマウントしたコンテナを作ってみると
```sh:
$ mkdir -p temp && touch temp/foo # 実験用に適当なディレクトリを作ってみる
$ docker run -it -v $(pwd)/temp:/temp busybox # 先ほど作ったディレクトリを /temp にマウントする
（コンテナ内にて）
# ls -la
total 48
drwxr-xr-x    1 root     root          4096 Mar  6 23:58 .
drwxr-xr-x    1 root     root          4096 Mar  6 23:58 ..
-rwxr-xr-x    1 root     root             0 Mar  6 23:58 .dockerenv
drwxr-xr-x    2 root     root         12288 Feb 20 17:57 bin
（省略）
drwxrwxr-x    2 500      500           4096 Mar  6 23:56 temp
（省略）
```
- コンテナ内からはマウントしたファイルのownerが`uid=500,gid=500`になっている
- ここでファイルを作成してみる
```sh:
# touch temp/bar
# ls -l temp
total 0
-rw-r--r--    1 root     root             0 Mar  7 00:02 bar
-rw-rw-r--    1 500      500              0 Mar  6 23:56 foo
```
- rootがownerのファイルとして作成される
- ホストOSから同じファイルを見て見ると
```sh:
$ ls -l temp
total 0
-rw-r--r-- 1 root     root     0 Mar  7 00:02 bar
-rw-rw-r-- 1 ec2-user ec2-user 0 Mar  6 23:56 foo
```
- となり、rootの持ち物として作成される
- 当然このファイルは一般ユーザーからは編集不可
- **コンテナとホストで相互にファイルのやりとりをしたいときにこの挙動は困ることが多い**
- **ちなみにdocker for macで試したところ、上記の問題は起きない**
- コンテナ内からはownerがrootとして表示されるが、mac上からは自ユーザーがownerとして表示されている
- docker for macの中でうまく解決してくれている

## うまくいかない方法1 : Dockerfile内でuseraddする

```Dockerfile:
RUN useradd --shell /bin/bash -u 1024 -o -c "" -m myuser
RUN mkdir -p /shared/tmp && chown myuser. /shared/ -R
USER myuser
CMD /usr/local/bin/myprocess
```
- この方法はイメージをビルドしたマシンと実行するマシンが同じならば問題がない
- しかし、**イメージをビルドする段階でuidを決定しなければならない**
- 別のマシンでビルドしたイメージは使えず実用的ではない

## うまくいかない方法2 : docker runに-uオプションをつける

- docker runコマンドには実行するユーザーを指定する-uオプションがある
- この-uでホストOS上のUIDを指定すればよいような気がする
```sh:
$ docker run -it -u `id -u $USER` debian:jessie /bin/bash
I have no name!@dcb415bad433:/$ id
uid=500 gid=0(root) groups=0(root)
```
- 上記のやり方で、コンテナ内のUIDがホストOSと同じUIDになるが下記の問題がある
  - **gid (group id)が変わっていない**
  - **/etc/passwd の情報とuidが一貫していない**
- ２つ目の問題はファイルを使っているだけなら問題ないが、いくつかのアプリで/etc/passwdを参照することがあり問題が起きることもある
- つまり、**-uのようなオプションでuidを指定したいが、実際にuseraddを使ってユーザーを作ることが必要となる**ので実用的ではない

## うまくいく方法1 : ENTRYPOINTでuseraddでユーザーを作る

- 基本的な方針: 
  - ホストOSでのユーザーのUIDを環境変数で渡す（必要であればGIDも）
  - コンテナ内で`useradd`で`uidを`指定して一般ユーザーを作る
  - その一般ユーザーでコマンドを実行する
- Dockerfile
```Dockerfile
FROM ubuntu:latest

RUN apt-get update && apt-get -y install gosu
COPY entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
```
- entrypoint.sh
```sh:
#!/bin/bash

USER_ID=${LOCAL_UID:-9001}
GROUP_ID=${LOCAL_GID:-9001}

echo "Starting with UID : $USER_ID, GID: $GROUP_ID"
useradd -u $USER_ID -o -m user
groupmod -g $GROUP_ID user
export HOME=/home/user

exec /usr/sbin/gosu user "$@"
```
- これらのファイルを準備して`docker build . -t mybase`コマンドでmybaseイメージを作る
- entrypoint.sh:
  - 環境変数`LOCAL_UID`で指定されたuidでユーザーを作っている
  - [gosu](https://github.com/tianon/gosu)というツールはgoで書かれたsuコマンドのようなもの
  - **普通のsuコマンドはTTYの問題とか色々と奇妙なことがおきるらしい...**
  - gosuはやっていることはsuと同じだと思えばよい
- 環境変数を何も設定しないで実行してみると...
```sh:
$ docker run -it mybase bash
Starting with UID : 9001
user@d45b2d3198c9:/$ id
uid=9001(user) gid=9001(user) groups=9001(user)
```
- デフォルトの9001番でユーザーが作られている
- 環境変数を設定して、ボリュームをマウントして見ると...
```sh:
$ docker run -it -v $(pwd)/temp:/temp -e LOCAL_UID=$(id -u $USER) -e LOCAL_GID=$(id -g $USER) mybase bash
# コンテナにて
$ id
uid=500(user) gid=500(user) groups=500(user)
$ ls -l /temp
total 0
-rw-rw-r-- 1 user user 0 Mar  7 01:49 foo
```
- ホストOS上でのUID,GIDが適切に設定されるようになったことがわかる
- コンテナ内でマウントされたディレクトリにファイルを作成しても、ホストOSからは自ユーザーがownerのファイルのように見える
- **コンテナ内とホストOSではユーザー名は異なるが、ファイルシステムはUIDを使ってファイルを管理しているのでこれでうまくいく**

## うまくいく方法2 : /etc/passwdと/etc/groupをコンテナにマウントする
- `docker run`に-uをつけてもうまくいかなかったのは、`/etc/passwd`との不整合が起きるためであった
- うまくいくようにするには、ホストOSの/etc/passwdをマウントするという方法がある
```sh:
docker run -it -v /etc/group:/etc/group:ro -v /etc/passwd:/etc/passwd:ro -u $(id -u $USER):$(id -g $USER) ubuntu bash
```
- コンテナが勝手に/etc/passwdや/etc/groupを書き換えないように`read only`でマウントしている
- これでほとんどの場合うまくいく
- ただし、上記のコマンドは`Docker for mac`では、**/etcがマウントできないため失敗する**
- しかし、コンテナのイメージを作成する際にすでに一般ユーザーを作って作業を行っている場合、その一般ユーザーで作ったファイルにアクセスできなくなるため問題が起きる (次で解説)

## うまくいく方法3: Dockerfile内ですでに一般ユーザーが作られている場合
- イメージを作る段階ですでに一般ユーザーを作って、そのホーム以下でいろいろと設定しているケースを考える
- その場合、**すでにコンテナ内に存在する一般ユーザーのUIDをusermodコマンドで変更し、ホストOSのUIDと合わせるとよい**
- 方法1とほぼ同様
- スクリプトを以下のように準備し、usermodコマンドで既存ユーザーのUIDを変更する
- このとき、**usermodコマンドはHOME以下のファイルのownerも自動的に切り替えてくれるのでファイルのオーナーを書き換えたりする必要はない**
- entrypoint.sh
```sh:
usermod -u $USER_ID -o -m user
groupmod -g $GROUP_ID user
```
</details>


