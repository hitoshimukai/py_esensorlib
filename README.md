# py_esensorlib(fork)

Epson のセンシングデバイス用 Python ライブラリ [esensorlib](https://github.com/cubicleguy/py_esensorlib) の fork です。加速度センサー M-A370AD10 を 1PPS に同期させて長期間連続で記録するために、加速度センサー用のロガー `accl_logger` に機能を追加しています。

ライブラリ本体の使い方、対応機種、ロガーの全オプション、Windows での注意点などは、[fork 元の README](https://github.com/cubicleguy/py_esensorlib/blob/main/README.md) を参照してください。この README では、fork で追加・変更した部分だけを説明します。

## 追加・変更の概要

変更したのは `accl_logger` だけです。`imu_logger` と `vibe_logger` の動作は fork 元と同じです。

| 項目 | 内容 |
|---|---|
| `--rotate_secs N` | 時計の N 秒境界で CSV を切り替えます(新しいオプション) |
| `--no_data_timeout N` | データが N 秒届かないと、終了コード 1 で終了します(新しいオプション、既定値 600) |
| CSV の作成時期 | 最初のサンプルを受信してから CSV を作成します |
| CSV の形式 | 列名の行とデータ行だけを書きます。`#` で始まるヘッダとフッターは書きません |

ファイルと行ごとの変更内容は [CHANGES_rotate_secs.md](CHANGES_rotate_secs.md) にあります。

## インストール

PyPI の `esensorlib` は fork 元の版なので、この fork はリポジトリから入れます。

```
git clone https://github.com/hitoshimukai/py_esensorlib.git
cd py_esensorlib
python3 -m pip install -e .
```

必要な環境は fork 元と同じです(Python 3.8 以上、pyserial、tqdm、tabulate、loguru)。

## 使い方の例

M-A370AD10 を 100 sps、1PPS 同期で記録し、毎正時に CSV を切り替える例です。

```
python3 -m esensorlib.example.accl_logger -s /dev/ttyUSB0 \
    --drate 100 --model a370ad10 --ext_trigger 1pps_pos \
    --counter --tempc --ndflags --chksm \
    --secs 31536000 --rotate_secs 3600 --csv --tag sensor1
```

`--secs` は記録する総時間です。長期間の連続記録では、上の例のように大きな値(この例では 1 年)を指定します。

## 追加したオプション

### `--rotate_secs N`

時計の N 秒境界で CSV を切り替えます。`3600` なら毎正時、`600` なら 10 分ごとです。

- `--max_rows` とは同時に指定できません。`--max_rows` は行数で区切るので、データに欠損があると区切りが時刻からずれます。
- 1 つ目のファイル名は起動時刻です。2 つ目以降のファイル名は境界の時刻(`3600` なら `HH0000`)です。
- 境界は Unix 時刻が基準です。日本時間では、`3600` は毎正時に一致しますが、`86400` は毎日 9:00 になります。
- 境界の判定にはホストの時計を使います。区切りの精度は、ホストの時計の同期の精度で決まります。
- `Sample No.` はファイルが変わっても通し番号のままです。
- N には 1 以上の整数を指定します。

### `--no_data_timeout N`

センサーデータが N 秒届かないと、エラーログを出して終了コード 1 で終了します。既定値は 600 秒で、`0` を指定すると fork 元と同じく無期限に待ちます。

外部トリガーや 1PPS を使っているとき、信号が途絶えるとセンサーは出力を止めます。fork 元のロガーはこのとき止まったままになります。このオプションで終了させ、systemd などで再起動できるようにしています。

トリガーの間隔が N 秒より長い使い方をする場合は、N を大きくするか `0` を指定してください。

## 新しいオプションを指定しなくても変わる動作

`accl_logger` では、次の動作が fork 元と異なります。

- データが 600 秒届かないと終了します(`--no_data_timeout` の既定値)。
- CSV は最初のサンプルを受信してから作成します。1PPS が途絶えたまま再起動を繰り返しても、空の CSV は増えません。
- CSV は列名の行とデータ行だけになります。fork 元が書いていた次の行は書きません。
  - 先頭の `#Log esensorlib`、`#Creation Date:`、`#Scaled Data` の 3 行
  - 末尾の `#Log End`、`#Sample Count`、`#Output Rate` の 3 行

上の例の条件では、各 CSV の 1 行目は次のようになります。

```
Sample No.,Flags[dec],Ts[degC],Ax[mG],Ay[mG],Az[mG],Counter[dec],Chksm16[dec]
```

CSV から消えた機種、シリアル番号、スケール係数などの情報は、ロガーの終了時にコンソールに出力される表で確認できます。

## 常駐させるときの注意

systemd や cron でロガーを動かすときは、次の点に注意してください。

- **停止は SIGINT で行います。** ロガーは Ctrl-C(SIGINT)を受けると、センサーをコンフィグレーションモードに戻してから終了します。SIGTERM ではこの処理が行われません。systemd では `KillSignal=SIGINT`、`timeout` コマンドでは `-s INT` を指定してください。
- **`TQDM_DISABLE=1` を設定します。** 設定しないと、進捗バーがログを埋めます(tqdm 4.66 以降)。
- **1 つのシリアルポートは 1 つのプロセスだけで開きます。** 2 つのプロセスが同じポートを開いてもエラーにならず、データを奪い合います。pyserial の `device reports readiness to read but returned no data (... multiple access on port?)` が出たら、別のロガーが動いていないか確認してください。

systemd ユニットの例です(パスとポートは環境に合わせてください)。

```
[Service]
Environment=TQDM_DISABLE=1
WorkingDirectory=/path/to/output
ExecStart=/path/to/python3 -m esensorlib.example.accl_logger -s /dev/ttyUSB0 --drate 100 --model a370ad10 --ext_trigger 1pps_pos --counter --tempc --ndflags --chksm --secs 31536000 --rotate_secs 3600 --csv
KillSignal=SIGINT
TimeoutStopSec=30
Restart=always
RestartSec=10
```

## ライセンス

fork 元と同じ MIT License です。[LICENSE](LICENSE) を参照してください。
