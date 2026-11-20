# `--rotate_secs` / `--no_data_timeout` 変更箇所一覧

- ブランチ: `feature/rotate-secs`(ベース: `main` の `0c942456`)
- 行番号は、このブランチの変更後ファイルのものです。
- 対象は `accl_logger` のみです。`imu_logger` と `vibe_logger` は変更していません。

## 追加したオプション

| オプション | 既定値 | 内容 |
|---|---|---|
| `--rotate_secs N` | なし(分割しない) | 時計の N 秒境界で CSV を切り替えます。`3600` で毎正時です。`--max_rows` とは同時に指定できません。 |
| `--no_data_timeout N` | `600`(10分) | センサーデータが N 秒届かないと、エラーログを出して終了コード 1 で終了します。`0` で従来どおり無期限に待ちます。 |

## 変更箇所

### `src/esensorlib/example/accl_logger.py`

| 行 | 変更内容 |
|---|---|
| 38 | `accl_fn` を import(例外クラスの参照用) |
| 275-277 | `--max_rows` を `--rotate_secs` との排他グループに移動 |
| 284-290 | `--rotate_secs` を追加 |
| 291-298 | `--no_data_timeout` を追加 |
| 300-305 | `--rotate_secs` は 1 以上、`--no_data_timeout` は 0 以上かを検証 |
| 393 | `device_cfg` に `no_data_timeout` を追加 |
| 413, 416-417 | ファイル名の共通部分 `name_suffix` を保持(`set_writer()` が渡したリストを書き換えるためコピーする) |
| 419-420 | `read_sample` の選択と `exit_code` の初期化 |
| 424-427 | 最初のサンプルを受信してから CSV を作成するよう変更 |
| 433-435, 448, 462 | ヘッダは列名の行だけを書く(`meta=False`) |
| 438-440 | 現在の時間区分 `rotate_period` を初期化 |
| 443 | サンプルの読み出しをループ先頭に移動 |
| 449-462 | 時間区分が変わったら新しい CSV とヘッダを作成(ファイル名は境界の時刻) |
| 466-468 | `NoDataTimeoutError` を捕捉してエラーログを出力し、`exit_code = 1` を設定 |
| 469-470 | `log.write_footer()` の呼び出しを削除(フッターを書かない) |
| 471 | `sys.exit(0)` を `sys.exit(exit_code)` に変更 |

### `src/esensorlib/accl_fn.py`

| 行 | 変更内容 |
|---|---|
| 67-70 | 例外クラス `NoDataTimeoutError` を追加 |
| 1165-1167 | `_get_sample()` の docstring に例外を追記 |
| 1186-1188 | `_cfg` から `no_data_timeout` を取得し、待機開始時刻を記録 |
| 1191-1196 | データ待ちのループでタイムアウトを判定 |

`set_config()` に `no_data_timeout` を渡さない場合の動作は従来と同じです。

### `src/esensorlib/example/helper.py`

| 行 | 変更内容 |
|---|---|
| 69-72 | クラスの docstring に `meta` を追記 |
| 186 | `write_header()` に引数 `meta`(既定値 `True`)を追加 |
| 195-197 | docstring に `meta` を追記 |
| 431-434 | `meta=False` のときは列名の行だけを書く |

既定値は従来と同じ出力なので、`imu_logger` と `vibe_logger` の CSV は変わりません。

### `README.md`(ACCL Help Screen)

| 行 | 変更内容 |
|---|---|
| 647 | usage に 2 つのオプションを追記 |
| 683-685 | `--no_data_timeout` の説明を追加 |
| 711-713 | `--rotate_secs` の説明を追加 |

## 新オプションを指定しなくても変わる動作(`accl_logger` のみ)

- データが 600 秒(10 分)届かないと終了します。従来どおりにするには `--no_data_timeout 0` を指定します。トリガー間隔が 600 秒を超える外部トリガー運用でも指定が必要です。
- CSV は最初のサンプルを受信してから作成します。1PPS が途絶した状態で再起動を繰り返しても、ヘッダだけの CSV は増えません。
- CSV は列名の行(`Sample No.,Flags[dec],...`)とデータ行だけになります。先頭の `#Log esensorlib`、`#Creation Date:`、`#Scaled Data` の 3 行と、末尾のフッター(`#Log End`、`#Sample Count`、`#Output Rate`)は書きません(2026-11-20、ユーザー指定)。

## `--rotate_secs` の仕様

- 2 つ目以降のファイル名は、境界の時刻(`3600` なら `HH0000`)で始まります。1 つ目は起動時刻です。
  - 当初は切り替えた時点の `time.strftime()` を使っていました。`time.strftime()` は時刻を渡さないと C の `time()` を呼び、Linux では粗い時計を読むので、`time.time()` より数 ms 遅れます。1PPS 同期のサンプルは境界の直後に届くため、05 時のファイルが `045959` になっていました(2026-11-11、実機)。
- `Sample No.` はファイルをまたいで連番のままです。
- 列名の行は各ファイルの先頭に書かれます(`--max_rows` と同じ)。フッターは書きません。
- 境界は Unix 時刻基準です。`3600` は日本時間の毎正時に一致しますが、`86400` は日本時間の 9:00 になります。

## 確認状況

偽のセンサーと偽の時計で `accl_logger` 全体を動かすテストを行い、20 項目すべて合格しました。

- 確認した内容: 時刻境界での分割と欠損なし、途中で時刻が飛んだ場合の分割、`--max_rows` と分割なしの従来動作、データ途絶時の終了コードと CSV、Ctrl-C、不正な引数、`_get_sample()` の実時間でのタイムアウト
- 未確認: 実機のセンサーと Raspberry Pi での動作
