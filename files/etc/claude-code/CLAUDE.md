# このシステムについて

あなたが今動いているのは `grd-b580-debug` という **専用調査用 genpack artifact** です。汎用の Linux 環境ではないので、最初にこの文書で世界観を掴んでから動いてください。

## 1. このシステムの目的

Intel Arc B580 (Battlemage / Xe2) + Mesa anv (`libvulkan_intel.so`) を載せた機体で、`libva-intel-media-driver` (iHD) が同居しているときだけ `gnome-remote-desktop-daemon` が RDP セッション初期化中に **SIGSEGV する**現象の root cause を特定すること。これだけのために組まれたイメージです。

回避策 (`LIBVA_DRIVER_NAME=none` で iHD を probe させない) は既に判明しており、それを**提案するのは仕事ではありません**。本イメージは敢えて iHD を残してクラッシュさせ、anv が NULL deref する瞬間の状態をデバッガで観察するためのものです。

## 2. クラッシュの既知の特徴

```
gnome-remote-de[...]: segfault at 48 ip <X> sp <Y>
  error 4 in libvulkan_intel.so[<offset>,<base>+<size>]
```

- fault address `0x48`: 典型的に `NULL + 構造体オフセット 0x48` の deref
- ip は `libvulkan_intel.so` (anv = Mesa Intel Vulkan)
- error 4: ユーザモードでの read access page fault

スタックは PipeWire (`pw_impl_port_use_buffers`) → GRD の screencast コールバック → `vkCreateImage` 系で DMA-BUF を import するパス → anv 内部で NULL deref、の順。 **VAAPI エンコーダには到達していない**点が重要。iHD が VAAPI として probe 可能なだけで Vulkan 側 (anv) のコードパスが切り替わり、その先で死んでいる、というのが現時点の見立て。

GRD は VA probe 成功時に `hwaccel_vaapi != NULL` となり、Vulkan view creator (VkImage 作成パラメータ、DMA-BUF modifier 選択、メモリタイプ等) を VA 互換側に切り替えていると推測されている。 anv はそのパラメータの組み合わせを想定していないらしい。

## 3. このシステム上にあるもの

### デバッグ情報付きでビルドされたパッケージ

以下は `FEATURES="splitdebug installsources"` + `-ggdb3 -fno-omit-frame-pointer` でビルド済みです。

- `media-libs/mesa` (anv 本体)
- `net-misc/gnome-remote-desktop`
- `media-video/pipewire`

シンボルは `/usr/lib/debug/` に分離配置されており、gdb が `.gnu_debuglink` 経由で自動ロードします。ソースは `/usr/src/debug/<category>/<package>-<version>/` にあります。例:

```
/usr/lib/debug/usr/lib64/libvulkan_intel.so.debug
/usr/src/debug/media-libs/mesa-26.x.x/work/mesa-26.x.x/src/intel/vulkan/
```

### 観測・解析ツール

| 用途 | コマンド |
|---|---|
| クラッシュ一覧 | `coredumpctl list` |
| 個別 coredump 情報 | `coredumpctl info <PID>` |
| gdb で開く | `coredumpctl gdb <PID>` |
| GRD ログ | `journalctl -b -u gnome-remote-desktop` (system モードの場合) または `journalctl -b --user -u gnome-remote-desktop.service` |
| カーネル側 fault | `journalctl -b -k \| grep -i segfault` |

### 再現に使う仕掛け

- `gnome-remote-desktop` パッケージは **stock** (本 artifact では local patch を当てていない)
- `libva-intel-media-driver` は明示的にインストール済み → これが iHD を probe 可能にしてクラッシュの引き金になる

## 4. genpack artifact である、ということ

このシステムはルート FS が **squashfs (read-only) + tmpfs overlay** で動いています。実行中は `/` に書き込めるように見えますが、それは tmpfs です。

| 振る舞い | 結果 |
|---|---|
| 再起動 | tmpfs の変更は全消去、squashfs の素の状態に戻る |
| `emerge` で何か入れる | ビルド時間を浪費するだけで再起動で消える |
| `/etc/`, `/home/user/` に書き込んだ設定 | 永続しない |
| coredump の保存先 | `/var/lib/systemd/coredump/` (tmpfs 上)。**再起動で消えるので解析は今のセッションで完結させる** |

調査の結果見つかった事実は、ローカルファイルに書き貯めるのではなく、ユーザに対するチャット応答として伝えてください。永続化が本当に必要なものは、ユーザにファイルの転送方法 (USB マウント / scp / `cat` で標準出力に出す等) を相談すること。

`sudo` は wheel グループに対して有効です (ユーザは `user`、パスワード空)。kernel パラメータ確認や `dmesg` 読みなどには使ってかまいません。

## 5. このセッションでやるべきこと

ユーザに何か頼まれたら、それを優先してください。文脈が掴めない場合の **デフォルトの調査フロー** は以下のとおりです:

1. `coredumpctl list` で既にクラッシュがあるか確認
2. なければユーザに RDP 接続 (mstsc) で再現してもらう
3. 新しい coredump が現れたら `coredumpctl gdb <PID>` で開く
4. `bt full` でフレームをたどり、`libvulkan_intel.so` のフレームに `frame <N>` で移動
5. その関数のソースが `/usr/src/debug/media-libs/mesa-*/work/.../src/intel/vulkan/` 等にあるので参照
6. **fault address `0x48` がどの構造体のどのフィールドか**を特定する。これが今回の目的
7. NULL だったポインタが「なぜ NULL なのか」を呼び出し元 (GRD 側フレーム) に遡って追う

`/usr/src/debug/net-misc/gnome-remote-desktop-*/` の中で `hwaccel_vaapi` を参照しているコードと、Vulkan view creator が VkImage を作る箇所も併せて読むと、GRD 側で挙動が分岐している場所が見えるはず。

## 6. やってはいけないこと

- **`LIBVA_DRIVER_NAME=none` を「解決策」として提案して終わる**。既知の workaround です。root cause を掘ること。
- **`libva-intel-media-driver` をアンインストールする / 削除する**。クラッシュの引き金そのものなので、これを消したらこのイメージの存在意義がなくなる。
- **`emerge` で何かを入れて再現を試みる**。Portage は使えますがビルドしても再起動で消えます。必要なものは原則同梱済み。何か足りなければユーザに相談 (genpack.json5 への追加を本人が判断する)。
- **GRD のソースを書き換える試行錯誤**。read-only squashfs なので無理だし、修正は upstream に提案する方が筋。
- **長文の調査ノートをローカルに `.md` で書き貯める**。永続しません。発見は逐次ユーザに報告。

## 7. 不明点があったとき

ユーザに `tachikoma フォルダの Intel-Arc-B580-GRD-anv-crash-with-iHD.md` を参照しているか、あるいは新しい情報があるか聞いてください。この文書はその要約です。フル版はホスト側 (`~/Desktop/tachikoma/` 配下) にあります。
