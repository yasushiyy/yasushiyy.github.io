Vagrant + CoreOS + Dockerを利用した開発環境セットアップ
===================================================

MacOSX + Vagrant + CoreOS + Docker + Ubuntuの環境。

2014年6月11日時点での情報。

* Version: CoreOS 343.0.0
* Kernel: 3.14.5
* Docker: 1.0

## 技術要素の説明

独断と偏見での説明。

* Vagrant - 仮想マシンの作成・起動・停止などを簡単に行うためのツール
* VirtualBox - 仮想化ソフトウェア
* CoreOS - Dockerを実行するのに特化した最低限のLinuxイメージ
* Docker - コンテナ型実行環境を提供するツール

## Why Docker?

* いろいろな環境を仮想OSで準備するのはだるい
* そのためにVagrantがあるが、OSイメージが乱立するとディスクスペースを食う
* 最近Dockerが流行っている
* 環境を分けるだけならやりやすい
* Immutable Infrastructureを理解できる
  * コンテナは実行完了後に破棄される（イメージ）ので、再実行するたびに新規環境になる（イメージ）

## 構築イメージ

```
      コンテナ1 | コンテナ2 | …          ← 環境ごとにコンテナを準備
---------------------------------------
                  Docker                  ← CoreOSに最初から入っている
---------------------------------------
                  CoreOS                  ← このセットアップにVagrantを利用
---------------------------------------
                VirtualBox
---------------------------------------
                  MacOSX
```

## インストール
* Vagrant http://www.vagrantup.com/
* Virtualbox https://www.virtualbox.org/
* hub.docker.comでユーザとレポジトリを作成 https://hub.docker.com/
  * 下記では yasushiyy/test というレポジトリを利用 https://registry.hub.docker.com/u/yasushiyy/test/

### CoreOSセットアップ

注：もし古いVagrant情報を消してやり直したい場合
```
$ rm -r ~/.vagrant.d
```

CoreOS用のVagrantfileをダウンロードする。

```
$ mkdir ~/VM
$ cd ~/VM
$ git clone https://github.com/coreos/coreos-vagrant.git
Cloning into 'coreos-vagrant'...
remote: Reusing existing pack: 241, done.
remote: Counting objects: 4, done.
remote: Compressing objects: 100% (4/4), done.
remote: Total 245 (delta 0), reused 0 (delta 0)
Receiving objects: 100% (245/245), 49.73 KiB | 0 bytes/s, done.
Resolving deltas: 100% (101/101), done.
Checking connectivity... done.
```

Vagrantfileを書き換えて、NFS経由でHOST(MacOS)側のフォルダが見えるようにする。ここでは ~/Dropbox/Code というMaxOS側のフォルダを、CoreOS上の /home/core/share から見えるようにする。

```
$ cd coreos-vagrant
$ vi Vagrantfile

config.vm.network :private_network, ip: ip
# Uncomment below to enable NFS for sharing the host machine into the coreos-vagrant VM.
#config.vm.synced_folder ".", "/home/core/share", id: "core", :nfs => true, :mount_options => ['nolock,vers=3,udp’]

  ↓

config.vm.network "private_network", ip: "172.12.8.150"
config.vm.synced_folder "/Users/yasushiyy/Dropbox/Code", "/home/core/share", id: "core", :nfs => true,  :mount_options => ['nolock,vers=3,udp']
```

CoreOSを起動する。

```
$ vagrant up
Bringing machine 'core-01' up with 'virtualbox' provider...
==> core-01: Box 'coreos-alpha' could not be found. Attempting to find and install...
    core-01: Box Provider: virtualbox
    core-01: Box Version: >= 308.0.1
==> core-01: Loading metadata for box 'http://alpha.release.core-os.net/amd64-usr/current/coreos_production_vagrant.json'
    core-01: URL: http://alpha.release.core-os.net/amd64-usr/current/coreos_production_vagrant.json
==> core-01: Adding box 'coreos-alpha' (v343.0.0) for provider: virtualbox
    core-01: Downloading: http://alpha.release.core-os.net/amd64-usr/343.0.0/coreos_production_vagrant.box
    core-01: Calculating and comparing box checksum...
==> core-01: Successfully added box 'coreos-alpha' (v343.0.0) for 'virtualbox'!
==> core-01: Importing base box 'coreos-alpha'...
==> core-01: Matching MAC address for NAT networking...
==> core-01: Checking if box 'coreos-alpha' is up to date...
==> core-01: Setting the name of the VM: coreos-vagrant_core-01_1402449154428_2561
==> core-01: Fixed port collision for 22 => 2222. Now on port 2200.
==> core-01: Clearing any previously set network interfaces...
==> core-01: Preparing network interfaces based on configuration...
    core-01: Adapter 1: nat
    core-01: Adapter 2: hostonly
==> core-01: Forwarding ports...
    core-01: 22 => 2200 (adapter 1)
==> core-01: Running 'pre-boot' VM customizations...
==> core-01: Booting VM...
==> core-01: Waiting for machine to boot. This may take a few minutes...
    core-01: SSH address: 127.0.0.1:2200
    core-01: SSH username: core
    core-01: SSH auth method: private key
    core-01: Warning: Connection timeout. Retrying...
==> core-01: Machine booted and ready!
==> core-01: Setting hostname...
==> core-01: Configuring and enabling network interfaces...
==> core-01: Exporting NFS shared folders...
==> core-01: Preparing to edit /etc/exports. Administrator privileges will be required…

Password:

-> パスワード入力

==> core-01: Mounting NFS shared folders…
```

CoreOSの起動を確認する。


```
$ vagrant status
Current machine states:

core-01                   running (virtualbox)

The VM is running. To stop this VM, you can run `vagrant halt` to
shut it down forcefully, or you can run `vagrant suspend` to simply
suspend the virtual machine. In either case, to restart it again,
simply run `vagrant up`.
```

CoreOSに接続する。vagrant ssh で可能。CoreOS上で、先ほど設定したNFS共有フォルダが見えることを確認する。

```
$ vagrant ssh
CoreOS (alpha)
core@core-01 ~ $ df -h
Filesystem                                Size  Used Avail Use% Mounted on
rootfs                                     17G   16M   16G   1% /
devtmpfs                                  489M     0  489M   0% /dev
tmpfs                                     500M     0  500M   0% /dev/shm
tmpfs                                     500M  224K  499M   1% /run
tmpfs                                     500M     0  500M   0% /sys/fs/cgroup
/dev/sda9                                  17G   16M   16G   1% /
/dev/sda3                                1008M  292M  666M  31% /usr
tmpfs                                     500M     0  500M   0% /tmp
tmpfs                                     500M     0  500M   0% /media
/dev/sda6                                 108M   88K   99M   1% /usr/share/oem
172.12.8.1:/Users/yasushiyy/Dropbox/Code  233G  125G  109G  54% /home/core/share
```

### Dockerセットアップ

CoreOS上には、既に最新版のDockerがインストールされている。以下で、Dockerのバージョンを確認する。

```
core@core-01 ~ $ docker version
Client version: 1.0.0
Client API version: 1.12
Go version (client): go1.2
Git commit (client): 63fe64c
Server version: 1.0.0
Server API version: 1.12
Go version (server): go1.2
Git commit (server): 63fe64c
```

最初のコンテナに利用する、Ubuntuのイメージをダウンロードする。

```
core@core-01 ~ $ docker pull ubuntu
Unable to find image 'ubuntu' locally
Pulling repository ubuntu
145762641db9: Download complete
6b0a59aa7c48: Download complete
ad892dd21d60: Download complete
3db9c44f4520: Download complete
ae8682f4ff20: Download complete
e314931015bd: Download complete
511136ea3c5a: Download complete
6cfa4d1f33fb: Download complete
ede2acc744ed: Download complete
663644853bf8: Download complete
97d42f9f1f6a: Download complete
e465fff03bce: Download complete
1ba25bde4355: Download complete
7c4b0d0eb402: Download complete
1ad85ea956a5: Download complete
e3c9f99b6410: Download complete
962c031a986e: Download complete
6ead103d9a34: Download complete
b5b8a196824c: Download complete
431aaca25584: Download complete
2b039c014364: Download complete
23f361102fae: Download complete
9db365ecbcbb: Download complete
```

コンテナを実行してみる。-vオプションでマウント設定を書くことで、CoreOS上のフォルダを、コンテナからも参照することが可能。そこでもNFS共有フォルダが見えることを確認する。

```
core@core-01 ~ $ docker run -v /home/core/share:/opt/share -t -i ubuntu df -h
Filesystem                                Size  Used Avail Use% Mounted on
/dev/sda9                                  17G  1.3G   15G   9% /
tmpfs                                     500M     0  500M   0% /dev
shm                                        64M     0   64M   0% /dev/shm
/dev/sda9                                  17G  1.3G   15G   9% /etc/hosts
tmpfs                                     500M  272K  499M   1% /etc/resolv.conf
172.12.8.1:/Users/yasushiyy/Dropbox/Code  233G  126G  108G  54% /opt/share
tmpfs                                     500M     0  500M   0% /proc/kcore
```

このコマンド実行時にコンテナが作成されて実行されるが、コマンド完了時点でそのコンテナは破棄（イメージ）される。再実行するたびに、コンテナは新規作成される（イメージ）。

## ゲストコンテナのセットアップ

毎回新規作成されるとだるいので、特定の環境を入れたイメージを作成し、永続化する。

以下で、python実行環境用のイメージを作成。他にもやり方はある（Dockerfile）。

```
core@core-01 ~ $ docker run -v /home/core/share:/opt/share -t -i ubuntu /bin/bash

# apt-get -y install python2.7
# apt-get -y install python-setuptools
# apt-get -y install python-pip

# pip install mechanize
# pip install beautifulsoup4
# pip install web.py
# pip install pexpect

# exit
exit
```

exitした時点でコンテナは停止するが、ps -aにて停止済みコンテナのIDを確認できる。

```
core@core-01 ~ $ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                      PORTS               NAMES
4b6b95f3ceed        ubuntu:14.04        /bin/bash           7 minutes ago       Exited (0) 2 seconds ago                        high_leakey
66a133a257d3        ubuntu:14.04        df -h               13 minutes ago      Exited (0) 13 minutes ago                       desperate_brown
```

新しいイメージとしてコミットすると、永続化される。ここではイメージ名を「yasushiyy/test」というDocker Hub上のレポジトリ名と合わせることで、後でHub上にPUSHできる。

```
core@core-01 ~ $ docker commit 4b6b95f3ceed yasushiyy/test
1eda666ecece60c2343572af4bd685de951e1257bae22fdae1c863ef3a7e14eb
```

docker images コマンドにて、ローカルに保持しているイメージを参照できる。

```
core@core-01 ~ $ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
yasushiyy/test      latest              1eda666ecece        18 seconds ago      415.4 MB
ubuntu              12.10               e314931015bd        6 days ago          172.1 MB
ubuntu              quantal             e314931015bd        6 days ago          172.1 MB
ubuntu              13.10               145762641db9        6 days ago          180.2 MB

  :
```

一度永続化したイメージは、以下にて利用できる。

```
core@core-01 ~ $ docker run -v /home/core/share:/opt/share -t -i yasushiyy/test /bin/bash
```

ここでexitしていしまうと、コンテナ自体が停止してしまう。代わりにctrl-p ctrl-q とやると、停止せずに抜けることができる。抜けた後は、docker attach コンテナID で再接続できる。

## イメージのpush

永続化したイメージを、Docker Hub上で共有することができる。

Docker Hubにログインする。

```
core@core-01 ~ $ docker login
Username: yasushiyy
Password:
Email: x@y.z
Login Succeeded
```

先ほど作成したyasushi/testイメージを、レポジトリにpushする。

```
core@core-01 ~ $ docker push yasushiyy/test
The push refers to a repository [yasushiyy/test] (len: 1)
Sending image list
Pushing repository yasushiyy/test (1 tags)
Image 511136ea3c5a already pushed, skipping
Image e465fff03bce already pushed, skipping
Image 23f361102fae already pushed, skipping
Image 9db365ecbcbb already pushed, skipping
Image ad892dd21d60 already pushed, skipping
1eda666ecece: Image successfully pushed
Pushing tag for rev [1eda666ecece] on {https://registry-1.docker.io/v1/repositories/yasushiyy/test/tags/latest}
```

PUSHされたことを確認する。

```
core@core-01 ~ $ docker search yasushiyy
NAME             DESCRIPTION       STARS     OFFICIAL   AUTOMATED
yasushiyy/test   test repository   0
```

Web上でも確認可能。
https://registry.hub.docker.com/u/yasushiyy/test/

## 古いコンテナの削除

何度もコンテナの起動・停止を試していると、古いコンテナ情報が蓄積され、ディスク領域等々を圧迫する。永続化する予定のないコンテナは、以下で削除可能。

```
core@core-01 ~ $ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                      PORTS               NAMES
4b6b95f3ceed        ubuntu:14.04        /bin/bash           24 minutes ago      Exited (0) 17 minutes ago                       high_leakey
66a133a257d3        ubuntu:14.04        df -h               30 minutes ago      Exited (0) 30 minutes ago                       desperate_brown


core@core-01 ~ $ docker rm $(docker ps -a -q)
4b6b95f3ceed
66a133a257d3

core@core-01 ~ $ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

## docker関連ファイルはどこに入っているか

/var/lib/dockerに入っている。

```
core@core-01 ~ $ sudo su -
core-01 ~ # ls -l /var/lib/docker
total 12
drwx------ 1 root root   20 Jun 12 04:38 btrfs
drwx------ 1 root root  128 Jun 12 04:41 containers
drwx------ 1 root root   12 Jun 12 04:35 execdriver
drwx------ 1 root root  520 Jun 12 04:39 graph
drwx------ 1 root root   32 Jun 12 04:35 init
-rw-r--r-- 1 root root 5120 Jun 12 04:41 linkgraph.db
-rw------- 1 root root  333 Jun 12 04:39 repositories-btrfs
drwx------ 1 root root    0 Jun 12 04:35 volumes
```

docker pullしたイメージのディレクトリ構造は、/var/lib/docker/btrfs内にある。（これは1.0から？）

```
core-01 ~ # docker images --no-trunc
REPOSITORY          TAG                 IMAGE ID                                                           CREATED             VIRTUAL SIZE
fedora              20                  3f2fed40e4b0941403cd928b6b94e0fd236dfc54656c00e456747093d10157ac   7 days ago          372.7 MB
fedora              heisenbug           3f2fed40e4b0941403cd928b6b94e0fd236dfc54656c00e456747093d10157ac   7 days ago          372.7 MB
fedora              latest              3f2fed40e4b0941403cd928b6b94e0fd236dfc54656c00e456747093d10157ac   7 days ago          372.7 MB
fedora              rawhide             64fd7993bcaf5d76aa430d360e118fe54aee25d9a91c54940d25691116f6325f   7 days ago          366.8 MB

core-01 ~ # ls -l /var/lib/docker/btrfs/subvolumes/3f2fed40e4b0941403cd928b6b94e0fd236dfc54656c00e456747093d10157ac
total 16
lrwxrwxrwx 1 root root    7 Apr  9 13:11 bin -> usr/bin
drwxr-xr-x 1 root root  156 Apr  9 13:11 dev
drwxr-xr-x 1 root root 2208 Apr  9 13:11 etc
drwxr-xr-x 1 root root    0 Aug  7  2013 home
lrwxrwxrwx 1 root root    7 Apr  9 13:11 lib -> usr/lib
lrwxrwxrwx 1 root root    9 Apr  9 13:11 lib64 -> usr/lib64
drwx------ 1 root root    0 Apr  9 13:08 lost+found
drwxr-xr-x 1 root root    0 Aug  7  2013 media
drwxr-xr-x 1 root root    0 Aug  7  2013 mnt
drwxr-xr-x 1 root root    0 Aug  7  2013 opt
drwxr-xr-x 1 root root    0 Apr  9 13:08 proc
dr-xr-x--- 1 root root  120 Apr  9 13:11 root
drwxr-xr-x 1 root root   96 Apr  9 13:11 run
lrwxrwxrwx 1 root root    8 Apr  9 13:11 sbin -> usr/sbin
drwxr-xr-x 1 root root    0 Aug  7  2013 srv
drwxr-xr-x 1 root root    0 Apr  9 13:08 sys
drwxrwxrwt 1 root root    0 Apr  9 13:11 tmp
drwxr-xr-x 1 root root  100 Apr  9 13:11 usr
drwxr-xr-x 1 root root  160 Apr  9 13:11 var
```

## 参考資料

* http://coreos.com/docs/running-coreos/platforms/vagrant/
* http://coreos.com/using-coreos/docker/
* http://docs.docker.com/userguide/dockerrepos/
