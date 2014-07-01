Oracle Linux 6.5のVagrant/Docker Imageを作る
=================================================

## 概要

必要最低限のOracle Linux 6.5の各種イメージを作成する手順。MacOSX上で実施しているが、恐らくWindowsでも動くはず。

* Vagrant Box: Virtualbox向けのpackage.box というファイルをローカルPC上に作成する。
* Docker Image: イメージを作成して Docker Hub にアップロードする。

以下の日本語翻訳版。
* https://github.com/yasushiyy/vagrant-docker-oracle12c-rac/blob/master/README_images.md

## 誰得？

`(∩ﾟдﾟ)ｱｰｱｰきこえなーい`

でもDocker Hubに上げると定期的にダウンロードされてるので、どこかに需要はあるのだろう。

## ダウンロード

Oracle Linux Release 6 Update 5 for x86_64 (V41362-01)のISOイメージをダウンロード。
* https://edelivery.oracle.com/linux

## インストール

VirtualboxとVagrantを先にインストールしておく。

Virtualboxで、以下の仮想マシンを作成。
* Name: oraclelinux65
* OS: Linux / Oracle (64bit)
* Mem: 512MB
* Disk: root.vdi (50GB), Dynamically Allocated

以下をSettingsで変更する。
* Network -> Advanced -> Adapter -> PCnet-FAST III
* Network -> Advanced -> Promiscuous Mode -> Allow All

起動する。Optical Diskを聞かれたら、ダウンロードした.isoを指定する。
* Install or Upgrade
* Skip media scan
* Language: English
* Keyboard: ja106
* Disk: Re-initialize all
* Time: Uncheck UTC, Timezone: Asia/Tokyo
* Password: vagrant -> Use Anyway
* Disk: Use entire drive -> Write changes to disk
* Reboot

rootでログインし、ネットワーク設定を変更。IPアドレスを確認する。

```
# vi /etc/sysconfig/network-scripts/ifcfg-eth0
ONBOOT=yes
NM_CONTROLLED=no

# service network restart

# ip addr show dev eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:ae:b9:14 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global eth0
    inet6 fe80::a00:27ff:feae:b914/64 scope link
       valid_lft forever preferred_lft forever
```

Virtualbox上で、ポートフォーワード設定を入れる。
* Settings -> Network -> Port Forwarding
* Rule 1 / TCP / 127.0.0.1 / 2222 / 10.0.2.15 / 22

この後のステップはssh経由でやればコピペができる。rootユーザでログインする。

```
$ ssh root@localhost -p 2222
vagrant
```

ログイン時の `-bash: warning: setlocale: LC_CTYPE: cannot change locale (UTF-8): No such file or directory` エラー対処を入れる。

```
[root@localhost ~]# echo LANG=en_US.utf-8 >> /etc/environment
[root@localhost ~]# echo LC_ALL=en_US.utf-8 >> /etc/environment
```

yumで最新化し、KernelをUEKR3に入れ替える。/boot/grub/grub.confにあるものを指定する。

```
[root@localhost ~]# yum update -y
[root@localhost ~]# more /boot/grub/grub.conf
[root@localhost ~]# grubby --set-default=/boot/vmlinuz3.8xxxx (上記に記載があるもの)
[root@localhost ~]# reboot
```

## Vagrant向け設定

以下を参考に。
* https://docs.vagrantup.com/v2/boxes/base.html
* https://docs.vagrantup.com/v2/virtualbox/boxes.html

Virtualbox Pluginのインストール。（そのときの最新版にすること）

```
[root@localhost ~]# yum install -y kernel-uek-devel kernel-uek-headers gcc perl openssh-clients
[root@localhost ~]# curl -O --location http://download.virtualbox.org/virtualbox/4.3.12/VBoxGuestAdditions_4.3.12.iso
[root@localhost ~]# mount -o loop,ro VBoxGuestAdditions*.iso /media
[root@localhost ~]# sh /media/VBoxLinuxAdditions.run
[root@localhost ~]# umount /media
[root@localhost ~]# rm VBoxGuestAdditions*.iso
```

Vagrantユーザを作成。

```
[root@localhost ~]# useradd vagrant
[root@localhost ~]# passwd vagrant
vagrant
```

sshd設定。 `useDNS no` にしないとSSH接続が遅い

```
[root@localhost ~]# vi /etc/ssh/sshd_config
useDNS no
```

ssh設定。Vagrantのお約束。

```
[root@localhost ~]# su - vagrant
[vagrant@localhost ~]$ mkdir .ssh
[vagrant@localhost ~]$ chmod 700 .ssh
[vagrant@localhost ~]$ cat > .ssh/authorized_keys <<EOF
ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEA6NF8iallvQVp22WDkTkyrtvp9eWW6A8YVr+kz4TjGYe7gHzIw+niNltGEFHzD8+v1I2YJ6oXevct1YeS0o9HZyN1Q9qgCgzUFtdOKLv6IedplqoPkcmF0aYet2PkEDo3MlTBckFXPITAMzF8dJSIFo9D8HfdOV0IAdx4O7PtixWKn5y2hMNG0zQPyUecp4pzC6kivAIhyfHilFR61RGL+GPXQ2MWZWFYbAGjyiYJnAmCP3NOTd0jMZEnDkbUvxhMmBYSdETk1rRgm+R4LOzFUGaHqHDLKLX+FIPKcF96hrucXzcWyLbIbEgE98OHlnVYCzRdK8jlqm8tehUc9c9WhQ== vagrant insecure public key
EOF
[vagrant@localhost ~]$ chmod 600 .ssh/authorized_keys
[vagrant@localhost ~]$ exit
```

sudo設定。requirettyをコメントアウトしないと、vagrant up(provision)時に `sudo: sorry, you must have a tty to run sudo` が出る


```
[root@localhost ~]# visudo
# Defaults requiretty
vagrant ALL=(ALL) NOPASSWD: ALL
```

shutdownする。

```
[root@localhost ~]# shutdown -h now
```

パッケージ作成。

```
$ vagrant package --base oraclelinux65
```

カレントディレクトリにpackage.boxファイルが作成される。

## Docker向け設定

以下を参考に。
* https://docs.docker.com/articles/baseimages/

あらかじめ Docker Hub 上でレポジトリを作成しておくこと。ここでは yasushiyy/oraclelinux65 という名前を利用する。
* https://hub.docker.com/

Virtualbox上で仮想マシンを起動する。ポートフォーワード設定が消えているので、再設定してからsshでログインする。

Dockerをインストールする。念のため、バイナリだけ最新版に置き換える。

```
[root@localhost ~]# rpm -iUvh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
[root@localhost ~]# yum install -y docker-io
[root@localhost ~]# curl -O --location https://get.docker.io/builds/Linux/x86_64/docker-latest
[root@localhost ~]# chmod a+x docker-latest
[root@localhost ~]# mv -f docker-latest /usr/bin/docker

[root@localhost ~]# sed -i.bak "s/--selinux-enabled//" /etc/sysconfig/docker
[root@localhost ~]# service docker start
[root@localhost ~]# rm -f /etc/yum.repos.d/epel*.repo
```

スクリプトをダウンロードして実行する。
* yum.repos.d内のファイルに変数があるとエラーになるので、定数に置き換える。
* $versionはlatestにしたいので、無理矢理置き換える。

```
[root@localhost ~]# sed -i.bak -e "s/\$uekr3/1/" -e "s/\$uek/0/" /etc/yum.repos.d/public-yum-ol6.repo
[root@localhost ~]# curl -O --location https://raw.githubusercontent.com/dotcloud/docker/master/contrib/mkimage-yum.sh
[root@localhost ~]# sed -i.bak "s/\$version/latest/" mkimage-yum.sh
[root@localhost ~]# bash mkimage-yum.sh yasushiyy/oraclelinux65
```

作成したイメージをUploadする。

```
[root@localhost ~]# docker login
Username: yasushiyy
Password:
Email: x@y.z
Login Succeeded

[root@localhost ~]# docker push yasushiyy/oraclelinux65
The push refers to a repository [yasushiyy/oraclelinux65] (len: 1)
Sending image list
Pushing repository yasushiyy/oraclelinux65 (1 tags)
a771c205a44f: Image successfully pushed
Pushing tag for rev [a771c205a44f] on {https://registry-1.docker.io/v1/repositories/yasushiyy/oraclelinux65/tags/latest}
```

Web上で確認する。
* https://registry.hub.docker.com/u/yasushiyy/oraclelinux65/

この時点で仮想マシンは消してもかまわない。

## テスト

Vagrant Boxをテスト。

```
$ vagrant box add --name oraclelinux65 package.box

$ mkdir test
$ cd test

$ cat > Vagrantfile << EOF
# -*- mode: ruby -*-
# vi: set ft=ruby :
VAGRANTFILE_API_VERSION = "2"
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "oraclelinux65"
end
EOF

$ vagrant up
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Importing base box 'oraclelinux65'...
==> default: Matching MAC address for NAT networking...
==> default: Setting the name of the VM: test_default_1403887057705_48235
==> default: Fixed port collision for 22 => 2222. Now on port 2200.
==> default: Clearing any previously set network interfaces...
==> default: Preparing network interfaces based on configuration...
    default: Adapter 1: nat
==> default: Forwarding ports...
    default: 22 => 2200 (adapter 1)
==> default: Booting VM...
==> default: Waiting for machine to boot. This may take a few minutes...
    default: SSH address: 127.0.0.1:2200
    default: SSH username: vagrant
    default: SSH auth method: private key
    default: Warning: Connection timeout. Retrying...
==> default: Machine booted and ready!
GuestAdditions 4.3.12 running --- OK.
==> default: Checking for guest additions in VM...
==> default: Mounting shared folders...
    default: /vagrant => /Users/yasushiyy/VM/test

$ vagrant ssh
[vagrant@localhost ~]$ cat /etc/oracle-release
Oracle Linux Server release 6.5
```

Docker側イメージをテスト。

```
[vagrant@localhost ~]$ sudo su -
[root@localhost ~]# rpm -iUvh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
[root@localhost ~]# yum install -y docker-io
[root@localhost ~]# service docker start

[root@localhost ~]# docker run -t -i yasushiyy/oraclelinux65 /bin/bash
Unable to find image 'yasushiyy/oraclelinux65' locally
Pulling repository yasushiyy/oraclelinux65
a771c205a44f: Download complete

bash-4.1# cat /etc/oracle-release
Oracle Linux Server release 6.5
``` 
