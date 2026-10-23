# paperang-p1-usb

PAPERANG P1 サーマルプリンタに、**USB 経由で**テキストや画像を印刷するコマンドラインツールです。
公式アプリも Bluetooth も使わず、Mac から USB ケーブル 1 本で印刷できます。

A tiny command-line tool to print text and images on a PAPERANG P1 thermal printer over USB (macOS).

```
./paperang text "こんにちは"
./paperang image photo.jpg
```

## 特長

- **単一ファイル**：本体は `paperang` 1 つだけ。[uv](https://docs.astral.sh/uv/) のスクリプト形式（PEP 723）なので、仮想環境の作成や `pip install` は不要です
- **日本語 OK**：テキストはヒラギノ角ゴシックで描画し、幅に合わせて自動で折り返します
- **画像印刷**：用紙幅 384 ドットに合わせて縮小し、ディザリングで白黒に変換します（ロゴや QR コード向けに 2 値化だけのモードもあります）
- **紙送りを mm で指定**：印刷後にカッターで切りやすい位置まで紙を送ります
- **MCP サーバ付き**：`paperang-mcp` で Claude やローカル LLM から印刷できます（Tailscale で tailnet 内に公開する手順つき）

## 動作環境

- macOS（Apple シリコンの macOS 15 で動作確認）
- [uv](https://docs.astral.sh/uv/) と [libusb](https://libusb.info/)
- PAPERANG P1（USB ID `4348:5584`。Mac からは「MPTII Printer」として見えます）

Linux でも動く可能性はありますが、未確認です（USB デバイスへのアクセス権の設定が別途必要になるはずです）。

## インストール

```sh
brew install uv libusb
git clone https://github.com/Shinichi-Ohki/paperang-p1-usb.git
cd paperang-p1-usb
./paperang selftest   # セルフテストページが出れば OK
```

初回実行時に uv が依存ライブラリ（pyusb, pillow。MCP サーバはこれに加えて mcp）を自動でダウンロードします。2 回目以降はキャッシュを使うのでオフラインでも動きます。

どこからでも使いたい場合は、PATH の通ったディレクトリにリンクを張ってください。

```sh
ln -s "$PWD/paperang" ~/bin/paperang
```

## 使い方

```sh
./paperang text "こんにちは"           # テキスト（自動折り返し）
echo hello | ./paperang text           # 標準入力から
./paperang text --size 40 "大きい字"   # 文字サイズ（px、既定 24）
./paperang text --font /path/to/font.ttf "..."  # フォントを指定
./paperang file memo.txt               # テキストファイル（UTF-8）
./paperang image photo.jpg             # 画像（幅 384px に縮小・ディザリング）
./paperang image qr.png --no-dither    # ロゴや QR コードは 2 値化のみ
./paperang --density 80 text "濃く"    # 印字濃度 0〜100
./paperang --feed 20 text "..."        # 印刷後の紙送り（mm、既定 15）
./paperang feed                        # 紙送り 10mm（./paperang feed 25 で 25mm）
./paperang selftest                    # セルフテストページ
```

`--density` と `--feed` はサブコマンドの前に書きます。

## MCP サーバ

`paperang-mcp` は、プリンタを MCP（Model Context Protocol）のツールとして公開するサーバです。Claude や、MCP に対応したローカル LLM のクライアントから印刷できます。通信方式は Streamable HTTP です。

| ツール | 内容 |
|---|---|
| `print_text` | テキストを印刷（`text`, `size`, `density`, `feed_mm`） |
| `print_image` | 画像を印刷（`image_base64` か `image_url` のどちらか, `dither`, `density`, `feed_mm`） |
| `feed` | 紙送り（`mm`） |
| `selftest` | セルフテストページ |

### 起動

```sh
./paperang-mcp                 # http://127.0.0.1:9900/mcp で待ち受け
./paperang-mcp --port 9000     # ポートを変える
```

サーバは `127.0.0.1` でしか待ち受けないので、そのままでは同じ Mac からしか使えません。

### Tailscale で tailnet 内に公開する

[Tailscale](https://tailscale.com/) の `tailscale serve` を使うと、同じ tailnet の機器からだけ接続できるようになります（インターネットには公開されません）。

```sh
# Mac アプリ版の CLI は /Applications/Tailscale.app/Contents/MacOS/Tailscale
tailscale serve --bg --http=9900 http://127.0.0.1:9900

# tailnet 上の名前を Host ヘッダとして許可して起動する
./paperang-mcp --allow-host <machine>.<tailnet>.ts.net --allow-host <machine>
```

クライアントからは `http://<machine>.<tailnet>.ts.net:9900/mcp` に接続します。

- `--allow-host` が必要なのは、DNS リバインディング対策で、知らない Host ヘッダのリクエストを拒否しているためです
- IP アドレス（`100.x.y.z`）では接続できません。`tailscale serve` は名前でアクセスされたときだけ転送します
- 公開をやめるときは `tailscale serve --http=9900 off`

Claude Code に登録する例：

```sh
claude mcp add --transport http paperang http://<machine>.<tailnet>.ts.net:9900/mcp
```

### 注意

- プリンタは状態を返さないので、ツールが成功しても「データを送った」という意味でしかありません
- 1 回の印刷は 4000 行（約 50cm）、テキストは 5000 文字までに制限しています
- 認証はありません。tailnet 内の機器なら誰でも印刷できます

## トラブルシューティング

- **「PAPERANG P1 が見つかりません」と出る**：電源が入っているか確認してください。電源が切れていると、USB につないでも充電されるだけで Mac から見えません。充電専用の USB ケーブルでも認識されません
- **`Permission denied` になる**：コピーの途中で実行権限が外れています。`chmod +x paperang` を実行するか、`uv run paperang ...` で起動してください
- **`No backend available` になる**：libusb が見つかっていません。`brew install libusb` を確認してください
- **コマンドは成功するのに何も印刷されない**：プリンタは USB で応答を返さないため、ツール側では失敗を検知できません。用紙切れやフタの開きを確認し、`./paperang selftest` を試してください

## プロトコル

P1 は USB 接続でも Bluetooth と同じ独自パケットを使います。USB の Bulk OUT エンドポイント（`0x02`）にそのまま書き込みます。USB のデバイス情報には「ESC/POS」と書かれていますが、ESC/POS のコマンドは無視されます。

```
0x02 | cmd | index | len (u16 LE) | payload | crc32 (u32 LE) | 0x03
```

- CRC32 は zlib の `crc32(payload, 0x35769521)`（初期値を `0x35769521` にしたもの）
- `PRINT_DATA (0)`：1 行 48 バイト（384 ドット、ビット 1 が黒、MSB が左端）。1 パケット 1008 バイト（21 行）ずつ送ります
  - 印刷が追いつかないときは、プリンタが USB の書き込みを待たせて受け取るペースを調整するようです。パケット間の待ち時間が 20ms でも、ほぼ全面が黒い 800 行（約 10cm）の画像を欠けなく印刷できました（送信に約 8 秒）
- `SET_CRC_KEY (24)`：payload は「新しいキー XOR `0x35769521`」。このツールは起動時に payload 0 を送って、キーを標準値に戻しています
- `SET_HEAT_DENSITY (25)`：payload は濃度 1 バイト（0〜100）
- `FEED_LINE (26)`：payload は送り量（u16 LE）。単位は約 1/25mm（実測）
- `PRINT_TEST_PAGE (27)`：セルフテストページ
- USB の Bulk IN（`0x81`）には応答が返ってきません

## 謝辞

パケット形式とコマンド番号は [tinyprinter/python-paperang](https://github.com/tinyprinter/python-paperang)（Bluetooth 版の実装）を参考にしました。

## 免責

非公式のツールです。PAPERANG とは関係ありません。利用は自己責任でお願いします。

## ライセンス

[MIT License](LICENSE)
