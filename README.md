# jetson-aion-ansible-unification

## 概要
jetson-aion-ansible-unificationは、OSがインストールされたJetson/Ubuntuへの、aion-core、kubernetes、および関連リソースの環境構築を、自動で行うツールです。   
この環境構築には、必要なパッケージのインストールや、docker/kubernetesのセットアップ、その他リポジトリのクローンやdocker imageのpull等が含まれています。

## セットアップ手順
I. local host(mac)にansibleをインストールする   
II. remote host(Jetson/Ubuntu)でssh接続のための設定をする   
III. local host(mac)でansibleの設定をする

### I.local hostにansibleをインストールする   
1. local host(mac)にansibleをインストールする
```
# install ansible
brew install ansible

# install sshpass
brew tap esolitos/ipa
brew install sshpass
```
2. local host(mac)に、jetson-aion-ansible-unification内で使用するモジュールをインストールする
```
# docker_imageモジュールを使うためにインストール
ansible-galaxy collection install community.docker

# set_timezoneモジュールを使うためにインストール   
ansible-galaxy collection install community.general   
```
3. ansible coreを2.11.1にする   
  （[kubernetes : Add Google Cloud Offical key]でエラーが出るため）
```
pip3 install ansible-core
```

### II.remote hostでssh接続のための設定をする   
1. remote host(Jetson/Ubuntu等の対象端末)上でbitbucket SSH鍵を作成する
```
cd ~
mkdir .ssh
cd .ssh
vim config
⇒以下の内容でファイルを作成する
==========
Host bitbucket.org
    User git
    Port 22
    HostName bitbucket.org
    identityFile ~/.ssh/id_rsa
    TCPKeepAlive yes
    IdentitiesOnly yes
==========
ssh-keygen -t rsa
  ⇒質問されるので、最初だけ/home/ユーザ名/.ssh/id_rsaと入力。それ以外はすべてエンターで抜ける
cat id_rsa.pub
  ⇒公開鍵が表示されるので、選択してコピーする
```
2. bitbucketに公開鍵を登録する
  Bitbucketを開き、左下ユーザアイコン>Personal settings>SSH鍵>鍵を追加を選択   
    Label: 「案件*-端末名（例：aisin-cat01）」
    Key: 「コピーした公開鍵」
  を入力し、鍵を追加する   
  ※ 開発ならdevelop

### III.local hostでansibleの設定をする   
1. このリポジトリをlocal host(mac)上にクローンし、該当のブランチに切り替える
```
git clone git@bitbucket.org:latonaio/jetson-aion-ansible-unification.git
git checkout {ブランチ名}
```
2. クローンしたリポジトリ内の `inventory` ファイルに端末情報を記載する   
  remote host(対象端末)のIPアドレスは、remote hostのterminalで `ifconfig` をすると検索できる
```
[edge]
192.168.xxx.xxx             # remote host(対象端末)のIPアドレス

[edge:vars]
ansible_port=22
ansible_user=XXXXXX         # remote host(対象端末)のユーザー名
ansible_ssh_pass=XXXXXXX    # remote host(対象端末)のログインパスワード
ansible_ssh_private_key_file=~/.ssh/id_rsa
```
3. クローンしたリポジトリ内の `group_vars／all.yml` に変数を設定する
```
user_name: "XXXXXX"             # remote host(対象端末)のユーザー名
host_name: "XXXXX"              # remote host(対象端末)のデバイス名
ip_address: 192.168.XXX.XXX     # remote host(対象端末)のIPアドレス
azure_iot_connection_string: HostName=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
                                # Azure IoTの接続情報
dbmysql_user: XXXXXXX           # Mysql DBユーザー名
dbmysql_password: XXXXXXXX      # Mysql DBユーザーログインパスワード
dbmysql_storage_size: 1Gi       # Mysql PersistentVolume サイズ
smb_ip_address: 192.168.XXX.XX  # SC連携 WindowsのIPアドレス
smb_workgroup: WORKGROUP        # SC連携 Windowsのワークグループ名
smb_user: test                  # SC連携 Windowsのユーザー名
smb_password: XXXXXXXX          # SC連携 Windowsのパスワード
```
4. microSDやSSDから起動させる場合、`run.yaml`の`- sdboot`や`- ssdboot`行のコメントアウトを外す

## 使用方法
jetson-aion-ansible-unificationディレクトリ配下で以下のコマンドを実行する
```
ansible-playbook run.yaml  -i inventory --ask-become-pass --ask-vault-pass
BECOME password: [remote host(対象端末)のpasswordを入力]
Vault password: [api_access_keyの暗号復元のためのpasswordを入力]
```

### エラー対処法
1. 以下のような、python3を使ってね系のwarning
```
[DEPRECATION WARNING]: Distribution Ubuntu 18.04 on host 192.168.XXX.XXX should use /usr/bin/python3, but is using /usr/bin/python for backward
compatibility with prior Ansible releases.
```
`whereis python3`の実行結果を、`ansible.cfg`内の interpreter_python= 以降に追加   
例）
```
interpreter_python=/usr/bin/python3
```


## ファイル構成
### 実行するrole,taskの一覧
roleの一覧は `run.yaml` に記載、   
各role内で行うtaskの一覧は、`roles/{各role名のフォルダ}/tasks/main.yaml` に記載、   
実行するtaskの詳細はそれぞれのtaskフォルダ内の各yamlファイルに記載されている。 
  
**sdboot**:   
  - `common-setting.yaml` ：スクリーンロック無効化などの設定   
  - `format.yaml`         ：microSDのフォーマット   
  - `install.yaml`        ：microSDに全データのコピー・microSDから起動するための設定等   

**ssdboot**   
  - `common-setting.yaml` ：スクリーンロック無効化などの設定   
  - `format.yaml`         ：SSDのフォーマット   
  - `install.yaml`        ：SSDに全データのコピー・SSDから起動するための設定等   

**common**    
  - `common-packages.yaml`：curlやpython3等必要なパッケージのインストール   
  - `common-setting.yaml` ：タイムゾーンやキーボードの設定   
  - `go-install.yaml`     ：goenvを使用してgolangのインストール      

**cuda**    
  - `install.yaml`        ：cudaのインストール      

**docker**   
  - `install.yaml`        ：docker.ioのインストール      
  - `setting.yaml `       ：ユーザーのdockerグループへの追加、dockerの起動等      

**kubernetes** 
  - `install.yaml`        ：kubelet,kubeadm,kubectlのインストール   
  - `setup.yaml`          ：Kubeadmでのセットアップ等   

**mysql** 
  - `clone-repositories.yaml`：必要なリポジトリのクローン(MySQL内のリポジトリ)   
  - `setup.yaml`            ：mysqlの起動等   

**aion**    
  - `created-directories.yaml`：ディレクトリの作成(AionCore,Runtime等)   
  - `clone-repositories.yaml` ：必要なリポジトリのクローン(AionCore,DataSweeper内のリポジトリ。クローンするリポジトリ一覧については以下)   
  - `pull-images.yaml`        ：必要なdocker imageをpull   
  - `setup-data-sweeper.yaml`: 以下のファイルの書き換え   
    - data-sweeper-batch-kube/mysql-config.json   
    - data-sweeper-batch-kube/data-sweeper-batch-kube.yaml   
    - data-sweeper-batch-kube/sql/DataSweeper_sweep_directories.sqlの読み込み      
  - `docker-build.yaml`      ：AionCore, DataSweeperのimageをbuild   
   
### ansible-playbookの使い方
一部のtaskのみを実行したい場合、実行しない部分をコメントアウトすれば良い。   
例えばdockerタスクのsetting.yamlの内容のみ実行したい場合、以下の通りコメントアウトする。
```
[run.yaml]
---
- hosts: edge
  gather_facts: False
  roles: # 実行するコマンド群の定義
    # - ex
    # - common
    # - cuda
    - docker
    # - kubernetes
    # - aion
    # - mysql
```
```
[roles/docker/tasks/main.yaml]
---
# - include: install.yaml
- include: setting.yaml
```

### クローンするリポジトリやpullするdocker imageの設定
クローンするリポジトリ名とブランチの一覧、pullするdocker imageの一覧は、`roles/{各role名のフォルダ}/vars/clone-repositories.yaml` または `roles/{各role名のフォルダ}/vars/pull-images.yaml` 内に記載されている。   
クローンやpullの必要がないものはコメントアウトしたり、リポジトリごとにクローンしてくるブランチを設定したりして使用できる。

