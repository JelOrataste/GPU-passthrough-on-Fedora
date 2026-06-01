
# FedoraとArchのデュアルブート＋GPUパススルー
Ryzen 7700内蔵AMD製GPU（以下iGPU）とNVIDIA RTX5060（以下dGPU）を同時にマザボに繋いだ状態
ー＞iGPUから映像出力＋仮想VM（Windows）はdGPUに処理させて、iGPUで映像出力したい

## UEFIいじり
・オンデバイスボード：内蔵ディスプレイ優先
・AMD CBS->NBIO Common option
	・IOMMU...enabled
	・GFX conf igpu関連
# パーティションわけ
EFI (FAT32) 
ext4 (boot)
Btrfs
	@home 
	@docker (chatter+C)(/var/lib/docker)
XFS(Windows.img用)
fedora live usbつかってパーティションわけを行う。
インストーラの右上の３点からストレージエディタ起動してパーティションを設定する。
設定終わったら、ストレージエディタ閉じて進むとストレージ構成が設定された通りになっているので、それを確認して進み、OSをインストール。
その後シャットダウン。

# いざ起動
usbひっこぬいて起動したらeキーを連打する（カーネルパラメータを変更するため）
デフォルトだとgrubが一瞬で消えてfedoraが起動するのでeを連打します（あとから待機時間変更できます）
```
中略
linux ()..... nomodeset
```
rhgb quietは削除（起動ログがみえるようになるよ！ロマンだね！）
nomodesetは、グラフィックドライバ関連の起動トラブルを回避するためのものです。
AMDのiGPUから出力しておきながらRTXをつないでいる状態なのでトラブル回避します。

書き換えたらctrl + Xおす(Emacsらしい)
# IOMMUとVfio関係
参考
https://wiki.archlinux.jp/index.php/OVMF_%E3%81%AB%E3%82%88%E3%82%8B_PCI_%E3%83%91%E3%82%B9%E3%82%B9%E3%83%AB%E3%83%BC

grubの場合

pciチェック
```
lspci -nnk | grep NVIDIA
```
これででてくるRTXとaudio deviceのシリアル番号をおぼえておく

grub(/etc/default_grub)
(vimとかお好きなエディタで編集ください、vimは別途dnf installですが)
```
GRUB_TIMEOUT=60
中略
GRUB_CMDLINE_LINUX="modprobe.blacklist=nouveau rd.driver.blacklist=nouveau amd_iommu=on iommu=pt vfio-pci.ids=XXXX:XXXX,XXXX:XXXX
 amdgpu.dcdebugmask=0x10"
```
iommu有効化と、nouveauつかわないでねってやつ(modprobeってのがモジュール読み込みに関連するので）と、vfioのやつ
maskのやつはcpuの省電力OFFなのでモニタのちらつきある場合のみ


/etc/modprobe.d/vfio.conf
```
options vfio-pci ids=XXXX:XXXX,XXXX:XXXX
softdep nouveau pre: vfio-pci
softdep snd_hda_intel pre: vfio-pci
```
ワイの環境ではgrubだけでなくこのconfかかんとvpciにあたってくれへんかった。
https://wiki.archlinux.jp/index.php/OVMF_%E3%81%AB%E3%82%88%E3%82%8B_PCI_%E3%83%91%E3%82%B9%E3%82%B9%E3%83%AB%E3%83%BC
（３．１）
softdep A pre Bとすると、B が優先になるらしい？
https://linuxjm.sourceforge.io/html/kmod/man5/modprobe.d.5.html

grub書き込み
```
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```
マザボramリセット？
```
sudo dracut -f --verbose
```
これをやらないといくらgrubにかきこんだところで前の設定のこってて全然反映してくれへんことあります
わたしはこれの存在を知らず、AI君もなかなかだしてくれずクソ時間かかりました。
rebootする（その前にしたの注意確認）

### 注意
rebootするとvfio-pciにあてたgpuをホストがつかえなくなるので注意
e連打しなくてもgrubが60秒まってくれてたら書き換え成功です

。。。がしかし、grub通過後画面真っ黒となり、左上にはアンダーバーが出ている場合、
残念ながらamdのドライバとnouveauのドライバが衝突しているなどが発生しているかも。。。。。。
（つまりnouveauを無効化しきれてない, blacklistあたりの抜け落ちがないか確認、dracutやったかも確認）
もしくは、rtxをvpciに渡した結果、マザボによっては、pciの依存関係によっては他の部分が道連れになっているのかもしれない、、、、、

ちなみにわたしはインストール作業２回してまして、１回目はblacklist追記＋sudo dracutで直った記憶、２回目はnouveauのno"u"を"n"にしてたのが原因でした泣


vfio-pciに当たってるか確認
```
lspci -nnk | grep -i -A 3 nvidia
```
kernel in useがvpciになってればOK
IOMMUが作動してるか確認
```
sudo dmesg | grep -i iommu
```
GPUのIOMMUグループが分離してるか確認
```
for d in /sys/kernel/iommu_groups/*/devices/*; do n=${d#*/iommu_groups/*}; n=${n%%/*}; printf 'IOMMU Group %s ' "$n"; lspci -nns "${d##*/}"; done
```


## そもそもなぜIOMMUが必要なのか？
ざっくり言ってしまえば、GPUがゲストOSの言うとおりに直接メモリにアクセスした結果、ホストOSのメモリ空間を破壊してしまうという悲劇を防ぐためです。
ここからは、ハイパーバイザー上で動くゲストOSのメモリアクセスと、GPUのDMAという機能の観点から見てみます。
### ハイパーバイザーのメモリアクセスの特性
まず、ハイパーバイザー関係なしに、OSがどのようにメモリ空間を利用しているのか整理してみます。
アプリケーションが動くためには、メモリ上に必要なデータを配置する必要があります。でも、物理メモリ上の資源は有限です。なので、本来メモリを利用するには、メモリのどこが空いているか、どのアドレスからどのアドレスまでデータを配置しているか、ほかのプロセスとの協調など、さまざまなことを考えなくてはなりません。しかし、アプリケーション開発において、これらの要素をいちいち意識して開発してたら大変なので、これらの仕事はOSが裏で頑張ってやってくれています。実際、WindowsやLinuxなどのOS上で動くプログラムを書く際、変数宣言するときにいちいちどこがあいてるのかというプログラムは書かかないですものね。
ではOSはどのようにメモリ割り当てを管理しているのかと言うと、一般にはページテーブルとMMUというものを使っています。
ページテーブルというのは、アプリケーション側から見える仮想アドレスと、物理メモリ上のアドレスを対応づけるテーブルです。
アプリケーション側には仮想アドレスをみせ、OSカーネル側でそれを物理アドレスに変換することで、アプリケーション側がいちいち物理アドレス上にデータがあるかないか判断して、ページフォールトを起こして、二次記憶からスワップインという処理を意識する必要なくなるわけです。全部OSが隠蔽してくれてます。
これら、仮想アドレスから物理アドレスへの変換は、CPUに搭載されたMMUというハードウェアが行っています。
では、ここでGPUのメモリアクセス方法を見てみましょう。
GPUは、DMA(Direct Memory Access)という仕組みを使って、物理メモリに直接アクセスすることができます。
つまり、CPUのMMUを介さずにアクセスすることになります。GPUがホストOSによって制御されている場合は、ホストOSが仮想アドレスと物理アドレスとの対応をちゃんと把握しているので、GPUに正しい物理アドレスを教えるはずなので、理論上は問題ないはずです（運用上は問題が起こる可能性がありますが）。
がしかし、ゲストOSとなると話は変わってきます。ゲストOSはなんと本物のページテーブルを持っていません。ゲストOSの仮想アドレス ー＞ゲストOSから見た物理アドレスー＞本物の物理アドレスという変換手順を踏んでいます。これによって、ゲストOSのメモリアクセスをハイパーバイザが制御できるようになっています。
ただし、DMAの場合は、アドレス指定が物理アドレスによって直接的になされます。すなわち、このゲストOSがdGPUの制御権を持つ状況では、ゲストOSの物理アドレスを指定することになってしまいますから、ハイパーバイザの制御外になり、ホストOSのメモリ領域にアクセスしてしまうなどの不正領域アクセスにつながります。
そこで登場するのがIOMMUです。
IOMMUとは、I/Oデバイス用のMMUのようなもので、dGPUなどのデバイスと物理メモリの間に入ることによって、DMAでdGPUが物理メモリにアクセスしようとする前に、安全なアドレスに変換してくれます。
参考
https://syuu1228.github.io/howto_implement_hypervisor/part16.html
https://syuu1228.github.io/howto_implement_hypervisor/part2.html
# dnfパッケージ取得時のサーバーregionと並行ダウンロード数変更
sudo dnf upgradeとかするときに、デフォルトのままだと、めちゃ遅いミラーサーバにつながったり、並行ダウンロード数が少なくて、速度が遅いことがあるので、サーバーを日本サーバに限定し、並行ダウンロード数の上限数を増やします。

fastmirror(一番はやいサーバーからインストールするオプションの有効化)
```
sudo sed -i 's/fastestmirror=False/fastestmirror=True/g' /etc/dnf/dnf.conf
```
sed -i 's/old/new/g' ファイルパス：ファイルパスのファイルの"old"を"new"に変更
-i：ファイルの書き換え（-i.bakとすると変更前の保存も可能）

ミラーサーバを日本限定にする
```
sudo sed -i 's/metalink=\(.*\)/metalink=\1\&country=jp/g' /etc/yum.repos.d/*.repo
```

並行ダウンロード上限数をふやす
```
if grep -q "max_parallel_downloads" /etc/dnf/dnf.conf; 
	then sudo sed -i 's/max_parallel_downloads=.*/max_parallel_downloads=10/' /etc/d  
nf/dnf.conf; 
	else echo "max_parallel_downloads=10" | sudo tee -a /etc/dnf/dnf.conf; fi
```
if grep -q then 命令; fi：grepが文字列を発見した場合、thenを実行(grep -qは標準出力をシェルに書き出さないオプション)

echo | tee -a　パス：echoの内容を|でteeにわたし、パスファイルに追記（-aオプション）
# Mozcの設定
日本語入力がデフォルトだとできないのでできるようにします。
```
sudo dnf install fcitx5 fcitx5-mozc fcitx5-autostart fcitx5-configtool fcitx5-gtk fcitx5-qt
```
インストール後再起動したほうがいい
ー＞設定ー＞fcitxにする

# Bluetoothデバイスのスリープ無効化
PCをスリープから復帰させた際、Bluetoothが無効化されていて、有効化しようとボタンを押しても反応しませんでした。
どうやら、一部のBluetoothコントローラにはバグがあるらしく（bluetoothctl: No default controller available）、Arch wikiの通り設定して、bluetoothのオートサスペンドを無効化すると治りました。
```
echo "options btusb enable_autosuspend=n" | sudo tee /etc/modprobe.d/btusb_no_as.conf
```
Realtek製bluetoothドライバを搭載している場合、PCをスリープさせると、不具合が発生するので、このドライバだけスリープしないように設定
（Linuxカーネルに起こりやすい問題らしい）
なお、不具合によりbluetooth接続できなくなった場合は、PCシャットダウンのみならず、背面の電源スイッチをOFFにして、コンセントを抜いて、電源ボタン長押しして放電させる必要がある。
参考
https://wiki.archlinux.jp/index.php/Bluetooth#%E3%83%88%E3%83%A9%E3%83%96%E3%83%AB%E3%82%B7%E3%83%A5%E3%83%BC%E3%83%86%E3%82%A3%E3%83%B3%E3%82%B0
(6.3.2 bluetoothctl: No default controller available)