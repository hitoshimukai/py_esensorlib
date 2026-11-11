# py_esensorlib(fork)

Epson のセンサー用 Python ライブラリの fork です。Raspberry Pi で M-A370AD10 を 1PPS 同期で連続記録するため、`accl_logger` に機能を追加しています。ユーザーとのやり取りは日本語です。

## リポジトリとブランチ

- `origin` はユーザーの fork(`hitoshimukai/py_esensorlib`)です。fetch は HTTPS、push は SSH(`git@github.com:hitoshimukai/py_esensorlib.git`)です。この Mac には HTTPS の認証情報がなく、`gh` も入っていません。
- `upstream` は元の配布元(`cubicleguy/py_esensorlib`)です。push 権限はありません。
- 作業ブランチは `feature/rotate-secs` で、fork に push 済みです。`main` は upstream と同じ状態です。
- commit と push は、ユーザーに頼まれたときだけ行います。
- ソースと `README.md` の改行コードは CRLF です。編集しても CRLF を保ってください。LF に変わると diff が全行になります。

## 追加した機能(`accl_logger` のみ)

変更したファイルと行の一覧は `CHANGES_rotate_secs.md` にあります。`imu_logger` と `vibe_logger` は変更していません。

- `--rotate_secs N`: 時計の N 秒境界で CSV を切り替えます。`--max_rows` とは同時に指定できません。`--max_rows` は行数で区切るので、欠損があると区切りが時刻からずれます。そのため採用しませんでした。
- `--no_data_timeout N`: データが N 秒届かないと、終了コード 1 で終了します。既定値は 600 秒(ユーザー指定)で、`0` なら無期限に待ちます。実装は `accl_fn.py` の `_get_sample()` と例外クラス `NoDataTimeoutError` です。
- CSV は最初のサンプルを受信してから作成します。1PPS が途絶したまま再起動を繰り返しても、ヘッダだけの CSV が増えないようにするためです。

## コードの注意点

- `helper.LoggerHelper.set_writer(to=list)` は、渡されたリストを書き換えます(機種名などを `insert` する)。同じリストを使い回さないでください。
- 変更前の `_get_sample()` は、データが届くまで無期限に待ちます。1PPS が途絶するとセンサーは出力を止めるので、ロガーが止まったままになっていました。`--no_data_timeout` はこの対策です。
- ロガーは、Ctrl-C(SIGINT)を受けるとセンサーをコンフィグレーションモードに戻し、CSV にフッターを書いて終了します。SIGTERM ではこの処理が走りません。systemd では `KillSignal=SIGINT`、`timeout` では `-s INT` を指定します。
- 同じシリアルポートを 2 つのプロセスが開いても、エラーにならずに奪い合いになります。pyserial の `device reports readiness to read but returned no data (... multiple access on port?)` が出たら、別のロガー(古い crontab など)が動いていないか確認してください。
- cron や systemd で動かすときは `TQDM_DISABLE=1` を付けます(tqdm 4.66 以降)。付けないと進捗バーがログを埋めます。

## テスト

リポジトリにテストはありません。実機がない環境では、次の方法で `accl_logger` 全体を確認しました。テスト用のスクリプトはリポジトリに入れていません。

- `esensorlib.sensor_device.SensorDevice` を偽物に差し替えます。偽物は、`read_sample()` を呼ぶたびに偽の時計を 10 ms 進め、指定した回数で `NoDataTimeoutError` や `KeyboardInterrupt` を発生させます。
- `time.time` と `time.strftime` を偽の時計に差し替え、`runpy.run_module("esensorlib.example.accl_logger", run_name="__main__")` を別プロセスで実行して、出力された CSV を検査します。
- 依存パッケージ(tqdm、pyserial、tabulate、loguru)はリポジトリの外にインストールし、`PYTHONPATH` に `src` を加えて実行します。

Raspberry Pi の実機では、`--rotate_secs 60 --secs 200` でファイルの切り替えを確認済みです。

## Raspberry Pi での使い方

- Pi では `/home/gsi/py_esensorlib` に `feature/rotate-secs` を clone し、`/home/gsi/venv/bin/pip install -e` で入れています。変更の反映は、push のあと Pi で `git pull` するだけです。
- センサーは `/dev/ttyUSB0`(sensor1)と `/dev/ttyUSB1`(sensor2)の 2 台で、この割り当ては固定です。
- 測定条件は `--drate 100 --model a370ad10 --ext_trigger 1pps_pos --counter --tempc --ndflags --chksm` です。

連続記録の運用(systemd ユニット、USB メモリへの転送、Pi への導入手順)は、別リポジトリ `/Users/hmukai/2026/gsi-gnss-accel-sw` で管理しています。そちらの `CLAUDE.md` と `systemd/` を参照してください。

## 未追跡のファイル(commit しない)

- `deploy/raspberry-pi/`: 運用一式の最初の版です。ユーザーの指示で commit しません。`gsi-gnss-accel-sw/systemd/` に移って更新が続いているので、こちらは古いコピーです。変更しないでください。
- `m-a370_datasheet_j_rev20251023.pdf`: M-A370 のデータシート(日本語)です。1PPS は 4.11 節、FLAG レジスタは 6.4 節、COUNT レジスタは 6.5 節にあります。
