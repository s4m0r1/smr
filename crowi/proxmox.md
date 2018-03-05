# さもり的Proxmox使えるメモ


###環境
Thinkpad t470s
CPU:i5-7500u
RAM:20GB
にて動作させた。
メモリ20Gbなんて基本ないと思うんでどうにか調整してみてください

##最終的には
複数のProxMoxを接続しBackUp用などそれぞれ分けていきます。


##仮想環境について
ここでかなり引っかかりました。仕様はよく調べる用にしよう（笑）
Oracle社のVirtualBoxではProxmoxは正常に動作しません。
理由は、VT-x/AMD-Vが有効化できないためです

ProxMox以外でもESXiなどの環境を立ててみたいときはVMWareを使用しましょう。

##ネットワーク環境について
図２こんな形で行っています
node1: xx.xx.xx.1
node2: xx.xx.xx.2


##ダウンロード
ProxMoxのダウンロードはここ

##インストール
その他の環境を立てるときと同じような形で大丈夫です。
メモリは少し多めにしておくとよいです。
またVT-x/AMD-Vを有効化しておきます
図１
インストールはここなどを参考に。

##インストールが終わったら
そのままログインし

apt update
apt upgrade

を実行

後Exitにてログアウト

##ブラウザからの操作
＊Node1での操作です
https://XX.XX.XX.1:8006 へアクセス
注意:httpsでのみ接続できます
エラーでますが各自例外などに追加しましょう

##ブラウザからnoVNCによるクラスター作成からノード追加まで
図
から
図
へ

noVNCが立ち上がります
図

user:root
password:インストール時の設定

##クラスターの作成
pvecm create "exaple"

##初回の状態を確認
pvecm nodes

##ノードを追加
＊この操作はNode2で
pvecm add xx.xx.xx.1

このあとパスワードが求められるのでNode1のパスワードを入力

##クラスターの確認
＊この操作はNode１で
pvecm status
と入力

Menbership infomation
"---------------------
Nodeid       Votes  name
0x00000001       1  xx.xx.xx.1(local)
0x00000002       2  xx.xx.xx.2

となり追加されていることが確認できます

##実際にWeb上から確認
図
やった～～～

##コマンドまとめ

pvecm status
>>クラスターの確認

pvecm nodes
>>ノードの状態確認

pvecm create XX
>>クラスターの作成

pvecm add xx.xx.xx.xx
>>操作中のノードをxx.xx.xx.xxクラスターに参加

##有効なサブスクリプションがどうたらこうたら
/usr/share/pve-manager/ext4/pvemanagerlib.js
519行あたり

if (data.status !== 'Active') {
を
if (false) {
へ

##おすすめするインストールの仕方
１つのHDDを選択するとすべてパーティションが割られてしまい
データーストア用やその他にわけられないので
HDDをまずインストール時に設定できるので小さい10GBなどにパーティションを割ってインスコ
残ったパーティションについては以下の方法で割当を行う。

（あくまでさもりのおすすめです）

fdisk -lコマンドにてそれぞれのパーティションを確認
（今回は/dev/sdbという空のHDDで作成

fdisk /dev/sdb
にてfdiskコマンドをスタートさせる

からの場合KVMに変換できないのでまず
Command (m for help): n

物理か論理と聞かれるので今回は物理パーティションにします

Partiton type
  p  primary (0 primary, 0 extended, 4 free)
  e  extended (container for logical partitions)
Select (default p):p

パーティションの番号を聞かれます(1-4)

Partiton number (1-4, default 1): 1

Point
物理パーティションは構造上4つまでしか分けることができません
４つ以上分ける際は一旦物理で割ってからさらに論理で分割するようにしましょう。



パーティション開始セクターを指定
First sector (2048-XXXX, default 2048):
＊ここではEnterでdefault(2048)が指定されます

パーティション終了セクターを指定
Last sector, +sectors or +size{K,M,G,T,P} (2048-XXX, default XXX):
＊ここではEnterでdefault(XXX)が指定されます

Point
ここは終了セクターの他にもデーターサイズで指定することも可能です


Created a new partition 1 of type 'Linux' and of size 100 GiB.

と成功のメッセージが出ればパーティション作成できてます。

Point
確認したい場合はcommand: の状態にて
i コマンドを使ってみましょう
以下のように確認することができます
図

次にパーティションをLVM形式に変換します。
Command (m for help):t
を実行すると

Selected partition 1
ここでは先程フォーマットしたパーティションを

Partition type (type L to list all types): 8e
ここではLinux LVMを表す 8eを

Point
１６進数２桁によって表しているパーティションタイプですがここで"L"と打ち込むと変更できる一覧を確認することができます。
ぜひ確認してみましょう

成功すれば
Changed type of partition 'Linux' to 'Linux LVM'となります。

確認する場合
Command (m for help): p
にて確認

ここまでで以下のようになっています
図

これにてFdiskによるパーティション操作は終了なので
Command (help for m): w
にて終了します。
図


LVMを設定していきます
論理ボリュームグループを作成

今回はBackupという名前で作成しています
vgcreate backup /dev/sdb1

作成されているか確認
vgdisplay -v backup
図

LVMを作成
今回読み間違えミスしてしまいました
lvdatabaseも任意の名前にできます
Backupフォルダの中にlvdatabaseスペースを作るよ！って意味になりますね。
（ちゃんと読もう・・・）
lvcreate -L 99.996G -n lvdatabase backup
図

Point
100GBすべては選択できません4m取られているので今回4mを引いた99.996Gで行ってます

フォーマットします
mkfs.ext3 /dev/backup/lvdatabase

lvdisplay
図

最後にマウントします
mkdir -p /mnt/backup
マウント用のフォルダを作成します

mount /dev/backup/lvdatabase

df -k
図


最後に自動マウントできるようにします
/etc/fstab
に
/etc/backup/lvdatabase /mnt/backup ext3 defaults 0 0
と追記

rebootしてマウントできてればOKです