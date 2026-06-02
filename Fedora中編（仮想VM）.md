ここまでやったこと
・Fedoraセットアップ
・IOMMUとvfio-pci割当

# Fedora側の設定
windows.isoとvirtio.isoやらをインストールする。
https://pve.proxmox.com/wiki/Windows_VirtIO_Drivers#virtio-win_Releases
virtio.isoはここからインストールしました。
WinFSPもファイル共有する場合はインストール。
https://github.com/winfsp/winfsp


必要ツールのインストール
```
sudo dnf install -y virt-manager libvirt qemu-kvm
```
virt-manager：仮想マシン管理
edk2-ovmf：UEFI環境エミュレート

#### ユーザー権限追加
libvirtを操作する権限がありませんとかいわれるので権限追加します。
あとlibvirtユニットを有効化します。
```
sudo usermod -aG libvirt $(whoami)
sudo usermod -aG kvm $(whoami)

sudo systemctl enable --now libvirtd
```
そしたらログアウトしてログインします
usermodというのは、ユーザーの所属グループを変更するコマンドらしいです。
libvirtとkvmを操作できるグループに加わるということ。
ユーザーはもともとグループというものに所属しているらしく、グループでアクセス権限を制御しているらしいです。
なお、すでにあるグループを上書きするとOSから追放されるので、-aGで、セカンダリグループを追加するよう。
$(whoami)は勝手にユーザー名に置き換えてくれます。タイポ防げるし便利。
グループとは：
https://www.miraiserver.ne.jp/column/about_linux-user-group/#i-5
usermodの使い方：
https://atmarkit.itmedia.co.jp/ait/articles/1612/14/news022.html
# virt-manager
パーティション構成
OS用
VM用：XFS
データ共有用：btrfs(空いてた領域使ってつくりました)

まず、VM用と、データ共有用のパーティションをホスト側にマウントします。
パーティションのUUID（ユニークな識別番号）を調べます。
参考：
https://wiki.archlinux.jp/index.php/%E6%B0%B8%E7%B6%9A%E7%9A%84%E3%81%AA%E3%83%96%E3%83%AD%E3%83%83%E3%82%AF%E3%83%87%E3%83%90%E3%82%A4%E3%82%B9%E3%81%AE%E5%91%BD%E5%90%8D
```
lsblk -f
```

/mnt直下に
/mnt/winshare
/mnt/vm
というディレクトリを作成します（ここの構成と名前はお好きな感じでいい感じに〜）

したらば、これらをホスト側に今後自動マウントするために設定を書き換えます。
```
sudo vim /etc/fstab
```
すると、vimエディターが起動します（nanoとかemacsとか好きなエディター使ってください）。
fstabとはなにかについては、Arch wiki参照
（ざっくりいうと、パーティションをどうマウントするかの設定ファイル）
https://wiki.archlinux.jp/index.php/Fstab

```
UUID=調べたID /mnt/winshare btrfs defaults,compress=zstd:3 0 0
UUID=調べたID /mnt/vm xfs defaults 0 0 
```
を追記（btrfs圧縮方式の設定はお好み）

したらば、
```
sudo systemctl daemon-reload
sudo mount -a
```
ここでエラーが出る場合は、systemctl忘れてて古い設定残ってるとか、書き方間違ってるとかです。あとサブボリューム作ってないのにサブボリュームオプションつけてたとか（＜ー私(´；ω；｀)）
sysytemctlのやつは、設定ファイル書き換えをOSに認知させるやつです。
なお、すでに手動でマウントをしている場合は、再起動時のエラー防止のため、
```
sudo mount -o remount /mnt/vm
```
とかで、再マウントさせるほうがいいかもしれない

続いて、/mnt/vmと、/mnt/winshareのアクセス権限を確認・変更します。
```
ls -la /mnt
```
これを打つと、
```
drwxr-xr-x 1 root 数字 日付 vm
drwxr-xr-x 1 root 数字 日付 winshare
```
などと出ます。
所有者がrootになっている場合は、root権限でしかアクセスできないので、所有者をユーザーに変更します。
```
sudo chown -R $(whoami):$(whoami) /mnt/vm
sudo chown -R $(whoami):$(whoami) /mnt/winshare
```
/mntではなく、/mnt以下の個別ディレクトリの所有者を変更しています。最小権限の法則ってやつかこれが（多分）。

アクセス権限：d（ディレクトリ） rwx(所有者)r-x(グループ)r-x(他人ユーザー)
	r：read、w:write、x：execute（cdでアクセス可能）
	※ファイルとディレクトリの場合で意味が異なります。所有者：root
という意味になっています。
アクセス権限は所有者はフルアクセス、ほかはrとxがあればOK.
（ただし、共有PCの場合は、ほかユーザからのアクセスを防ぎたい場合や、アクセス権限を変更したい場合などは、chmodコマンドで権限変更.）
なお、xについては、Archwikiによると、親ディレクトリから唯一継承してるパーミッションとみなすことができるらしく、親ディレクトリの一つでもxが付与されていないと、アクセスできないため、注意。
参考：
https://wiki.archlinux.jp/index.php/%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%81%AE%E3%83%91%E3%83%BC%E3%83%9F%E3%83%83%E3%82%B7%E3%83%A7%E3%83%B3%E3%81%A8%E5%B1%9E%E6%80%A7

つづいて、QEMUがユーザー権限で動けるようにします。
そうしないと、QEMUがローカルドライブにアクセスしようとしたらパーミッションで弾かれます。
```
sudo vim /etc/libvirt/qemu.conf
```
すると、qemuの設定ファイルが開きます。
user="qemu"というところを検索するか、がんばって探してください。
したらば、user=とgroup=のとこの#を外して、
```
user="username"
group="username"
```
に置き換えてください。usernameは自分のユーザー名です。
ユーザー名は
```
echo $(whoami)
```
で確認できます。
そうしたら設定反映のため、
```
sudo systemctl restart libvirtd
```
してください。

##### windows.imgの作成
https://wiki.archlinux.jp/index.php/QEMU
の3.1にあるように、QEMUの実行にはハードディスクイメージなるものが必要です。
イメージとして、rawと、qcow2という２つの形式があります。
rawはI/Oがネイティブに近い、qcow2はスナップショットが使える＋容量少なく済む
という特徴がありますが、今回はrawでやっていきます。
[Attention]
wikiでも警告されていますが、ハードディスクイメージを保存するパーティションのフォーマット形式を確認してください。
パーティションがBtrfsの場合は、ハードディスクイメージを保存するディレクトリに限り、copy-on-writeを無効にしてください。
qcow2がそもそもCoWであるためです(.imgの場合もCoWはOFF推奨)。
私の場合、/mnt/vmにマウントしているパーティションはXFSにしています。
	Btrfsは、copy-on-writeをデフォルトですべてのファイルに適応しています。
	copy-on-writeというのは、書き込みにおいて、その場のデータを上書きせず、新しい場所に書き込まれ、書き込みが終了したらメタデータのポインタを新しい場所に移す方式です。
	通常のファイル書き込みについては、データの破損リスクが抑えられるのでいいのですが、仮想マシンの.imgは頻繁に更新されるので、細かいデータをいちいち残しつ書き込みしていては外部フラグメンテーションによる断片化が大量に発生し、SSD寿命、空き領域探索によるI/Oパフォーマンスに影響する可能性があるため、vmディレクトリに限り無効にするといい？かも？
	（おまけ）
	似たようなシステムとして、ジャーナリングシステムをそういえば講義できいた事あったなぁと思いましたが、そちらでは、ジャーナル領域というところに書き込んだあとに、保存領域に転送するという仕組みなので、CoWとはまた違う方式です。
	（ジャーナル方式には、ordered, writeback, journalという３方式があり、ordered, writebackはメタデータのみ、journalはメタデータとデータ本体両方をジャーナル領域に書き込んだあとに保存領域に転送する方式）
	参考：
	https://wiki.archlinux.jp/index.php/Btrfs
	https://e-words.jp/w/QCOW2.html
	https://e-words.jp/w/%E3%82%B8%E3%83%A3%E3%83%BC%E3%83%8A%E3%83%AA%E3%83%B3%E3%82%B0%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%82%B7%E3%82%B9%E3%83%86%E3%83%A0.html

前置きが長くなりましたが、.imgを作成します。
サイズはパーティションのサイズに合わせて適宜調整してください。
```
cd /mnt/vm
qemu-img create -f raw image_file 170G
```
# virt-manager
あらかじめ入れておいたvirt-managerを起動してください。
そうしたら、新規仮想マシンの作成（テレビマーク）ー＞ローカルのisoー＞windowsのisoを選択。
メモリ割り当てとかCPUスレッド数割当とか選択
カスタムストレージの選択からさっき作った.imgを選択
インストール前に設定をカスタマイズするにチェックを入れ完了

#### Windowsの出番はまだ先
いろんなハードウェアとXMLが出る。
仮想VMとホストOSとの映像・音声・データ・キーボード・マウス制御権伝送やらなんやらを設定する必要がある。
なお、ゲストOS側で設定しない設定も存在するがとりあえずホスト側でできることをやっておく。
・映像（ただし、まだGPUパススルー自体は行いません。とりあえずwindowsをインストールします。）
	Looking Glassという仕組みを使って、GPUパススルーしてても、ホスト側でウィンドウとして表示できるようにする。
	ホスト側で、looking-glassをビルドする必要があります。
	ビルドに必要なツールをまずはインストールします（公式サイトで挙げられてるツールはDebian向けの名前なので、dnfの名前とは異なっている...）。
	ちなみに一応パッケージ一覧だしてますが、cmakeにパッケージ足りねぇと怒られたら都度追加してくださいませ....
	参考（Debian以外向けのパッケージを親切なことに書いてくださっています・・・ありがたや・・・）
	https://looking-glass.io/wiki/Installation_on_other_distributions
```
sudo dnf install \
  cmake gcc gcc-c++ make pkgconf-pkg-config \
  mesa-libEGL-devel mesa-libGL-devel mesa-libGLES-devel \
  fontconfig-devel gmp-devel spice-protocol nettle-devel \
  SDL2-devel SDL2_ttf-devel \
  libX11-devel libXext-devel libXfixes-devel libXi-devel \
  libXinerama-devel libXrandr-devel libXrender-devel \
  libXcursor-devel libXpresent-devel libxkbcommon-x11-devel \
  wayland-devel wayland-protocols-devel \
  libglvnd-devel openssl-devel binutils-devel \
  libXScrnSaver-devel libdecor-devel \
  pipewire-devel pulseaudio-libs-devel \
  libsamplerate-devel
```
	つづいて、どこかしらのディレクトリにLooking glass用のディレクトリを作りませう。
	https://looking-glass.io/docs/B6/build/
	のダウンロードページから、ソース類をダウンロードしてきて、展開します。
	そしたらサイトを参考にビルド
```
mkdir client/build
cd client/build
cmake ../
make
```
	これで、バイナリデータがビルドされるはずです。
	ついでにキーボード・マウス・クリップボード関連の設定もします。
	<devices>直下にinput type="tablet"を削除＋以下を追加してください。
```
<input type='mouse' bus='virtio'/>
<input type='keyboard' bus='virtio'/>
```
	また、<channel>直下を以下の様に書き換えるとクリップボードに対応する
```
<channel type="spicevmc">
  <target type="virtio" name="com.redhat.spice.0"/>
  <address type="virtio-serial" controller="0" bus="0" port="1"/>
</channel>
```
・音声
	ゲスト側の音声をホストのpipewireに直接流す。
	XMLを編集する（概要からXML全文が見える）。
	以下のような構造とする。
	sound modelっていうのは、ゲスト側にみせる仮想デバイス（ich9がデフォルトのはず）
	で、audio backendっていうのは、ホスト側のバックエンド。
	なので、audio backendにpipewireを指定し、かつ、ゲスト側のsoundをaudioに紐づけます。
	devicesを探し、その直下にaudioを追加。
	sound modelはもともとあるはずなので、audio idを追加（紐づけ）
```
中略
<devices>
	<audio id="1" type="pipewire" runtimeDir='/run/user/1000'>
	</audio>
中略
	<sound model="ich9">
		<audio id='1'/>
	</sound>
中略
</devices>
```
注意：sound model =のとこの最後にデフォルトで/がついてると思われるが、</sound>と重複するので外すこと、コンパイルエラーがでますぜ。
参考
https://libvirt.org/formatdomain.html#sound-devices
sound deviceセクションと、audio backend, その中のpipewireセクションを参照
・データ
	virtioを用いたI/Oの準仮想化を行うため、ゲスト側にvirtioをいれる必要がある。
	まず、.imgのディスクについては、ディスクバスをSCSIにする。
	なので、ハードウェアの追加ー＞ストレージから参照でvirtioのiso選択、CD-ROMにする
	あと、/mnt/winshareをマウントしたくば、メモリーから共有メモリを有効化したうえで、ハードウェアを追加ー＞ファイルシステムー＞マウントしたいディレクトリを指定、virtio-fsを選択(ターゲット名はお好きな名前)

# Virt-ioとは？
Virt-ioとは、I/O処理を準仮想化して、ネイティブに近い性能を出すための仕組み・・・だそうですが、そもそも準仮想化ってなんだとなったので、そもそもそこから調べました。
## 完全仮想化と準仮想化
準仮想化があるんなら完全仮想化というのもまた存在します。
わたしもあまり深く理解できてる気がしないのでもう少し理解を深めたいところですが、、、一応概要を、、、、
完全仮想化というのは、ハードウェアをゲストOS向けにエミュレートすることで、ゲストOS側に手を加えることなく仮想化することを言うそうです。
ゲストOSは自分が仮想化されていることを認識していません。
一方、準仮想化とは、ゲストOSに若干の改造(今回でいえばvirt-ioドライバのゲストOSへのインストール)
を加えることによって、ゲストOSはハイパーバイザで動作していることを認識し、かつゲストからハイパーバイザへの遷移に行える方式だそうです。
ゲストOSが物理デバイスを使用できるため、I/Oの効率が完全仮想化に比べ、よくなります(完全仮想化の場合、物理デバイスがまるであるかのようにゲストOSを騙さないといけないため、エミュレートするオーバーヘッドが発生するため）。
https://docs.redhat.com/ja/documentation/red_hat_enterprise_linux/6/html/virtualization_getting_started_guide/para-virtdevices
ちなみに今回の仮想化では、CPU, dGPUを完全仮想化、ストレージI/O、ネットワークなどを準仮想化しています。
あれさっき完全仮想化のほうがはやいっていったのになぜCPUとdGPUは完全仮想化なのかと私も思いましたが、CPUは完全仮想化を高速化する技術に特化しているから、dGPUは丸ごと貸し出せるデバイスがあるからそもそもエミュレートする規模が小さいというのが理由になります。
（SSDなどのI/Oデバイスも、丸ごとゲストOSに渡す専用ということであれば完全仮想化という手法もありますが、、、今回は1枚しかもっていないので、パススルーは使用できないため、準仮想化を使用しているという感じです。)
((((((ちなみにこんな理由づけしてますが実際は後から色々調べているので、後付けの理由だったり、、、、、))))))
CPUの完全仮想化は(Intel:VT-x,AMD:SVM), dGPUは(Intel:VT-d,AMD:AMD-VI)というCPUのハードウェア支援機能によって行われています。

仮想化支援機能についてちょっと気になったので調べてみることに
### CPU仮想化支援機能とは：リングプロテクション（書き込み途中、追記修正予定)
例えば入出力をユーザがしたいと思ったら、ユーザプロセスはOSにスーパーバイザコールというものを行い、OS側に処理を委託して、ユーザプロセスは結果をOSが受け取る形になります。
入出力など、OSでしか行えないCPUに対する命令群を特権命令といいます。
これは正確に言うと、OSでしかできないというより、特権命令を実行できるかいなかは、リングプロテクションというCPUアーキテクチャに搭載された権限構造によって管理されています。
リング0~3まであり、リング0でOS、リング3でユーザプログラムが動くという仕組みになっています。

参考
https://www.ibm.com/jp-ja/think/topics/hypervisors
https://developer.ibm.com/articles/l-virtio/
https://gihyo.jp/list/group/%E3%82%A8%E3%83%B3%E3%82%B8%E3%83%8B%E3%82%A2%E3%81%AA%E3%82%89%E7%9F%A5%E3%81%A3%E3%81%A6%E3%81%8A%E3%81%8D%E3%81%9F%E3%81%84%E4%BB%AE%E6%83%B3%E3%83%9E%E3%82%B7%E3%83%B3%E3%81%AE%E3%81%97%E3%81%8F%E3%81%BF#rt:/dev/serial/01/vm_work/0005

https://atmarkit.itmedia.co.jp/fsys/kaisetsu/085intelvt/intelvt.html
https://syuu1228.github.io/howto_implement_hypervisor/part11.html
https://docs.redhat.com/ja/documentation/red_hat_enterprise_linux/6/html/virtualization_getting_started_guide/para-virtdevices