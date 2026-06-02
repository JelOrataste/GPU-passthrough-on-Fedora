# Virtioの有効化
仮想マシンを起動してwindowsをインストールします。
今の時点では、looking-glassではなくvirt-managerの通常の画面（QXL等)で画面表示されます。
CD-ROMの中に、virtio-win-guest-tools.exeがあるので実行して、インストールします。
そしたら再起動します。
また、ファイル共有を行う場合は、WinFSPもインストールします。
デバイスマネージャーで、virtio関係のドライバが正常に割り当てられてるか確認してください。
なお、ファイル共有を行う場合は、ターミナルで以下のコマンドを実行すると、Zドライブなどとして見れるようになります。

```
sc.exe start VirtioFsSvc
```
https://docs.redhat.com/ja/documentation/red_hat_enterprise_linux/10/html/configuring_and_managing_windows_virtual_machines/sharing-files-between-the-host-and-its-virtual-machines
こちらのドキュメントには、scのあとに.exeつけてないコマンドが紹介されているのですが、PowerShellの場合は、scのあとに.exeをつける必要があるようです。
なお、管理者権限のターミナルでないとアクセス拒否されます。

# いよいよGPUパススルー
あと、looking-glass-hostもWindows側にインストール。
そしたら一旦シャットダウン。
GPUにケーブルがモニタにつながってるか確認。
したらvirt-managerからPCIデバイスでRTX5060追加。
あとXMLのdevices直下に以下追記。
looking-glassが共有メモリを利用できるように権限設定する。
	参考：https://looking-glass.io/docs/B7/ivshmem_shm/
	以下の設定ファイルをつくりvimなどで開く。
```
sudo vim /etc/tmpfiles.d/looking-glass.conf
```
	そしたら以下を追記(0660はアクセス権限、下三桁)
```
f /dev/shm/looking-glass 0660 ユーザー名 kvm -
```
	そしたらシステム更新
```
sudo systemd-tmpfiles --create /etc/tmpfiles.d/looking-glass.conf
```
続いて各種設定をしていきます。
https://looking-glass.io/docs/B6/install/
	libvirtのXMLの<devices>直下に以下を追加（IVSHMEMという画面転送に使う共有メモリ設定）
 [Attention!!]size unitサイズは上記サイトの表を参考に変更
```
<shmem name='looking-glass'>
  <model type='ivshmem-plain'/>
  <size unit='M'>256</size>
  </shmem>
```
	<video>セクション探して以下に変更（たぶん、model typeをqxlからvgaにするだけ）
```
<video>
      <model type="vga"/>
</video>
```

# SELinuxの権限設定
> [!WARNING]
> 本項はセキュリティ・権限管理について触れていますが、筆者はセキュリティど素人であり、かつ、筆者の作業環境は個人用かつ非共有端末を想定しているため、厳密なセキュリティ管理を前提としていません。
> 厳密なセキュリティ管理が必要な場合において、SELinux設定を行う場合は、必ず公式ドキュメントや信頼性のあるソースをご確認・学習の上、自己責任で行ってください。
> 記事のはじめでも述べている通り、筆者は本記事・本項の記事内容について一切保証しません。
> 本記事を参考に実行した場合の損害について、筆者は一切の責任を負いません。

さて、このまま、仮想マシンを起動しようとすると、多分右下にSELinuxくんがめっちゃ通知だしてくれると思います。
SELinuxというのは、強制アクセス制御（MAC）を実施するものらしいです。
アクセス制御といえば、中編でもいじった、rwxやらユーザーグループでのアクセス管理（DACと呼ぶらしい）が思い浮かびます。
DACとMACの違いは権限昇格に対する耐性です。
DACの場合は権限昇格でrootまたはsudoでrootを用いた場合、システム全体に制限なくアクセスできるようになってしまいます。
一方で、MACの場合は、セキュリティポリシーというものを用いて、ファイルのパーミッションを凍結できるらしく、rootすらもアクセスを防ぐことが可能だそう。
その関係上、GPUパススルーとかやろうとすると、SELinuxがブロックしてきます。
なので、必要なプロセスについて、ラベルや、ポリシーを書き換えるなどが必要です。

SELinuxとは（使い方は都度調べるとして、ここでは概念を理解しようという試み）
https://www.redhat.com/ja/topics/linux/what-is-selinux
https://wiki.archlinux.jp/index.php/SELinux
（2005年の記事ですが）
https://atmarkit.itmedia.co.jp/fsecurity/rensai/selinux01/selinux01.html
アクセス制御全般についてはこちら
https://www.redhat.com/ja/topics/security/what-is-access-control


## 親切なSELinux通知とsemanage, semodule
SELinuxくんは親切なことにどのプロセスがどこにアクセスするのを拒否してるのかを教えてくれるので、それを頼りました。
一瞬で消える通知を頑張ってクリックしてもいいですが（<ー私）、普通にAlt+Spaceで”SELinux通知ブラウザー"というものを立ち上げることで見ることができます。
だいたい同時刻にいくつかまとまってブロック履歴があると思います。
で、トラブルシュートというボタンがあるのでそこを押すと、いくつか選択肢をだしてくれます。
touchつかう選択肢とsemanage fcontextつかう選択肢と、semoduleつかう選択肢がでると思います
（でない選択肢もある）。
touchは個別許可ではなくすべてラベルをつけ直すことになってしまうので除外。
だいたいsemanageの選択肢がでてるものは、その指示どおりに、semoduleの選択肢しかないものはsemoduleの指示どおりにやりました。

なお、semanageする際、FILE_TYPEというものを指定する必要がある箇所もあるとSELinux通知くんが教えてくれるのですが、Geminiくんと以下のドキュメントから判断するに、virt_image_tでいけました。
（ドキュメントで紹介されてるxen_image_tはKVMとは異なるハイパーバイザー用なので）
https://docs.redhat.com/ja/documentation/red_hat_enterprise_linux/5/html/virtualization/chap-virtualization-security_for_virtualization
あと、restoreconするのお忘れなく。
そしたら、 
```
ls -Z　該当パス
```
でラベルがみれるので反映されるか確認ください。
ただ、/dev/shm/looking-glassだけはvirt_image_tではうまく行きませんでした。
これは、/dev/shm/looking-glassがメモリとして機能する、tmpfsというファイルシステムだからだそうで、ラベルはsvirt_tmpfs_tにする必要があるようです。
参考
https://wiki.archlinux.jp/index.php/Tmpfs
https://www.linuxcampus.net/documentation/selinux-policy/svirt.html
#### ラベルとポリシー
詳しくは、上記の参考記事を参照してください。
とっても大雑把にいうと、SElinuxというのは、アクセス元のプロセスのラベルのtype識別子と、アクセス先のtype識別子の組み合わせをポリシーに問い合わせて、それをもとにアクセス可否を決定しているらしいです多分。
つまり、SELinuxの通知としては、既存のポリシーで対応できるものについては、ラベルの変更で対応できるため、semanageの選択肢を含め提示しており、既存のポリシーで対応できないものは、ポリシーの追加が必要なため、semoduleの選択肢しか表示していないと思われます（多分）。

