# `GNOME Shell`がメモリーリークしてるかも知れない

![](https://raw.githubusercontent.com/yKesamaru/gnome-shell/main/assets/eye-catch.png)
## はじめに
わたしのシステムでは時間が経過するごとに`gnome-shell`のメモリ使用量が増大します。
このため`Alt+F2`でポップアップするコマンドダイアログに`r`と入力することで`gnome-shell`を再起動させます。
この再起動ではセッションが維持されるため、セッションが初期化されてしまうログアウトやシステムの再起動より使い勝手が良いです。
![](https://raw.githubusercontent.com/yKesamaru/gnome-shell/main/assets/2024-05-25-20-00-36.png)

## 環境
```bash
inxi -S; gnome-shell --version
System:
  Host: **user** Kernel: 6.5.0-35-generic x86_64 bits: 64 Desktop: Unity
    Distro: Ubuntu 22.04.4 LTS (Jammy Jellyfish)
GNOME Shell 42.9
```

### 補足：他のシステムコマンド
`r`の他のシステムコマンドを載せておきます。
`GNOME Shell`の機能拡張を開発したりしているなら、`r`(OR `restart`), `lg`なんかは役立つかも知れません。
`gsettings`は、再インストール時に済ませてありますし、なにかの設定を切り替えたいのなら`dconfエディター`や`設定`から操作する方が見通しも良くていいのではないかと思います。
- lg
  - `gnome-shell`のデバッガ表示
- `restart`
  - `r`と同じ。`gnome-shell`の再起動。セッションは維持される
- `set loglevel <level>
  - `gnome-shell`のログレベルを指定
  - 例えば`debug`, `info`, `warn`, `error`など。
- gsettings set <schema> <key> <value>
  - システムコマンドと言えるか怪しい。`GSettings`を変更する。
  - 参考：[システム再インストールを楽にする手順: 専用BashScriptのすすめ](https://zenn.dev/ykesamaru/articles/1ab1297354d3c2)
- `gsettings reset <schema> <key>
  - `GSettings`をリセット
- `gsettings list-recursively`
  - `GSettings`設定をリスト表示
  - 仮想端末でやったほうが良くない？

- メモリ開放前: 617MB
![メモリ開放前](https://raw.githubusercontent.com/yKesamaru/gnome-shell/main/assets/2024-05-25-20-06-05.png)
- メモリ開放後:384MB
![メモリ開放後](https://raw.githubusercontent.com/yKesamaru/gnome-shell/main/assets/2024-05-25-20-07-13.png)

さて、簡単お気軽にメモリの開放ができるとはいえ、システムとしてはよろしくない状態です。
とはいえ、この問題にあまり時間をかけたくありません。

ここでは、`gnome-shell`に関する簡単な知識の整理と、ログの確認、今現在の潜在的な問題を洗い出すにとどめようと思います。
簡単なまとめを残しておくことで、同じ問題を持つかも知れない方への助けとなれば幸いです。

## `gnome-shell`とはなにか
参考：[GNOME Shell, next generation desktop shell](https://wiki.gnome.org/Projects/GnomeShell)
> Provides core interface functions like switching windows, launching applications or see your notifications.
> （`gnome-shell`は）ウィンドウの切り替え、アプリケーションの起動、通知の確認などのコアインターフェイス機能を提供します。
> [GNOME Shell, next generation desktop shell](https://wiki.gnome.org/Projects/GnomeShell)

![gnome-shell](https://wiki.gnome.org/Projects/GnomeShell/Technology?action=AttachFile&do=get&target=tech-components-diagram.png)
参考：[Technology](https://wiki.gnome.org/Projects/GnomeShell/Technology)
上記ページには、`gnome-shell`は「コンポジットマネージャである」と書かれています。
`gnome-shell`の下層には`OpenGL`に`Clutter`, `Mutter`があります。
- Window Manager: `Mutter`
- [シーングラフ](https://en.wikipedia.org/wiki/Scene_graph)ベースのAPIを提供する`Clutter`
  - 2D, 3Dグラフィック描画のためのライブラリ。
  - 強力なアニメーションフレームワークを提供→UIトランジッションやエフェクト効果
  - `gnome-shell`以外にもメディアプレーヤーやゲームなどでも使用される[Scene graphs in games and 3D applications](https://en.wikipedia.org/wiki/Scene_graph)
- `Shell Toolkit`: ボックス、ボタン、スクロールバーなどカスタムウィジェットを提供。`CSS`をサポートしているためテーマ設定が容易。
- `GObject Introspection`: `GObject`の`JavaScriptバインディング`を自動的に生成。

## ログの確認
```bash
# `journalctl`で`gnome-shell`が1日に吐き出したログをファイルに出力する
journalctl _COMM=gnome-shell --since "2024-05-24" --until "2024-05-25" > journalctl_gnome-shell.txt
```
### 補足
上記コマンドラインの`_COMM=`は`フィールド指定子`です。
`_COMM=gnome-shell`で`gnome-shell`によって生成されたログエントリをフィルタリングします。
#### 他の`フィールド指定子`
- `_PID`: プロセスIDでフィルタリング
- `_UID`: ユーザーIDでフィルタリング
- `_SYSTEMD_UNIT`: `systemdユニット`名でフィルタリング
- `_EXE`:実行ファイルのパスでフィルタリング
参考：
- [journalctlコマンドの使い方](https://hana-shin.hatenablog.com/entry/2022/02/28/203433#8-%E7%89%B9%E5%AE%9A%E3%83%97%E3%83%AD%E3%82%BB%E3%82%B9%E3%81%AE%E3%83%A1%E3%83%83%E3%82%BB%E3%83%BC%E3%82%B8%E3%82%92%E8%A1%A8%E7%A4%BA%E3%81%99%E3%82%8B%E6%96%B9%E6%B3%95_PID)
- [journalctl - systemdで起動したサービスのログを確認する](https://linuxcommand.net/journalctl/)

### ログ精査結果
メモリ増大に関連する可能性のあるログをピックアップします
1. リピ回数が多い、キーシンボルの上書き警告
```bash
Window manager warning: Overwriting existing binding of keysym 32 with keysym 32 (keycode b).
```
Window Manager(Mutter)関連のエラーと思われる。
`Input Remapper`でキーの入れ替えをしているので、ここは調査が必要。
2. `Gio.DBusError`
```bash
JS ERROR: Gio.DBusError: GDBus.Error:org.freedesktop.DBus.Error.Failed: error occurred in AboutToShow
```

`Gio.DBus`エラー
GNOMEの設定変更やシステム通知のエラー？
`error occurred in AboutToShow`: メニューやUIコンポーネントが表示される直前に発生することが多いらしい。
参考：[ JS ERROR: Gio.DBusError: GDBus.Error:org.freedesktop.DBus. Error.Failed: error occurred in AboutToShow #445 ](https://github.com/ubuntu/gnome-shell-extension-appindicator/issues/445)


3. `Stage views`の更新エラー
```bash
Can't update stage views actor <unnamed>[<MetaWindowGroup>:0x616babace360] is on because it needs an allocation.
```
特定のアクター（描画対象オブジェクト）が描画のためのメモリ空間の領域割り当てが行われていない？GPUの問題？
主メモリやVRAMの不足は見受けられない。
ウィンドウの最大化・最大化解除のたびにこのエラーが発生するとの報告もあり
参考：[Can't update stage views actor MetaWindowGroup is on because it needs an allocation](https://gitlab.gnome.org/GNOME/gnome-shell/-/issues/4281)

#### ログの対処
1に関しては`Input Remapper`の設定を見直してみて、それでもエラーが出るかどうか様子見する。
2, 3については深堀が必要そう。アップデートにより自然消滅する可能性もある。

## 最後に
結局、`Input Remmaper`の設定を見直すくらいしか対処のしようがないことになりました。
（それ以外については多くの時間が吸い込まれそうなので）
ただし`gnome-shell`のメモリーリークだとしたら、対策が必要なシステムはありそうだな…というのが気がかりです。（機能拡張など全て外していることが前提）

以上です。ありがとうございました。

## 参考文献
- [システム再インストールを楽にする手順: 専用BashScriptのすすめ](https://zenn.dev/ykesamaru/articles/1ab1297354d3c2)
- [GNOME Shell, next generation desktop shell](https://wiki.gnome.org/Projects/GnomeShell)
- [GNOME Shell, next generation desktop shell](https://wiki.gnome.org/Projects/GnomeShell)
- [Technology](https://wiki.gnome.org/Projects/GnomeShell/Technology)
- [シーングラフ](https://en.wikipedia.org/wiki/Scene_graph)
- [journalctlコマンドの使い方](https://hana-shin.hatenablog.com/entry/2022/02/28/203433#8-%E7%89%B9%E5%AE%9A%E3%83%97%E3%83%AD%E3%82%BB%E3%82%B9%E3%81%AE%E3%83%A1%E3%83%83%E3%82%BB%E3%83%BC%E3%82%B8%E3%82%92%E8%A1%A8%E7%A4%BA%E3%81%99%E3%82%8B%E6%96%B9%E6%B3%95_PID)
- [journalctl - systemdで起動したサービスのログを確認する](https://linuxcommand.net/journalctl/)
- [Can't update stage views actor MetaWindowGroup is on because it needs an allocation](https://gitlab.gnome.org/GNOME/gnome-shell/-/issues/4281)

