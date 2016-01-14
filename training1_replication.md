## レプリケーションとは

* 1つのMySQLサーバ(マスタ)のデータを、1つまたは複数のMySQLサーバ(スレーブ)に複製出来る

## レプリケーションのメリット

* パフォーマンス
  * マスタは更新専用(UPDATE, INSERT, DELETE等), スレーブは参照専用(SELECT)にすることで負荷が分散する
  * 一般的なWebアプリケーションでは読み取り負荷の方が多い傾向にあるため、レプリケーションでは参照性能向上の恩恵が大きい
* バックアップ
  * データはスレーブに複製され、スレーブはレプリケーションプロセスを一時停止できるため、スレーブで安全なコールドバックアップが取得可能
* 分析
  * マスタに負荷をかけずにスレーブの情報を分析可能

## レプリケーションの仕組み

* マスタ
  * Binlogダンプスレッド
    * バイナリログを生成
* スレーブ
  * スレーブI/Oスレッド
    * マスタに接続し、対象のバイナリログの更新を送信要求する
    * マスタのBinlogダンプスレッドから送信される更新を読み取り、リレーログとしてスレーブ側のローカルにコピーする
  * スレーブSQLスレッド: スレーブI/Oスレッドによって
    * スレーブI/Oスレッドによって書き込まれたリレーログを読み取り、SQLを実行する

## レプリケーションの種類

* バイナリログの種類
  * ステートメントベースレプリケーション(SBR)
    * デフォルト
    * SQLをバイナリログに書き込む
    * ファイルサイズは小さい
    * 非決定性関数を使った場合にスレーブで不整合が生じる可能性がある
      * UUID(), RAND()など
      * `UPDATE foo SET id = UUID()...` のようなクエリがそのまま転送されるため
  * 行ベースレプリケーション(RBR)
    * 更新結果をバイナリログに書き込む
    * ファイルサイズは大きい
  * 混合(Mixed)
    * 通常時はSBRで書き込み
    * 関数にUUID()が含まれる、FOUND_ROWS()などが含まれるときはRBRに切り替わる
* レプリケーション同期の種類
  * 非同期レプリケーション
    * デフォルト
    * マスタ側でトランザクションが終了したら非同期にスレーブに送信する
    * マスタで障害が発生した場合、障害発生直前のトランザクションがスレーブに送信されていない可能性がある
  * 準同期レプリケーション
    * マスタ側でコミット後、トランザクションはブロックされ、少なくとも1つのスレーブがリレーログを書き込んだタイミングでマスタに通知、トランザクションが完了する
    * 複数のスレーブが存在する場合、いずれか1つのスレーブから応答が返った時点で完了と見なす
      * 一部のスレーブに障害が起こっていてもレプリケーション自体への支障はない
    * スレーブからの受信通知がタイムアウトした場合は非同期レプリケーションになる
      * その後受信通知が届いた段階で準同期レプリケーションに戻る

## レプリケーションの設定

### マスタ

* my.cnfの設定

```
[mysqld]
# 各MySQLサーバを識別するためのIDの設定
# レプリケーションクラスタ内で一意である必要がある
server-id=1

# バイナリログの有効化
# ログファイル名のprefixを `mysql_bin` として設定する場合の例
# prefixを省略した場合はホスト名が使われ、仮にホスト名が変更された場合に一貫性を保つのが面倒になるため、何か設定することを推奨
log_bin=mysql_bin

# バイナリログをコミットと同時にディスクに書き込む
# マスタをクラッシュセーフにするために必要
innodb_flush_log_at_trx_commit=1
sync_binlog=1

# リレーログをテーブル(mysql.slave_relay_log_info)に書き込む
# スレーブをクラッシュセーフにするために必要
relay_log_info_repository=TABLE
relay_log_recovery=ON
```

変更後、サーバを再起動する

* レプリケーション用ユーザの作成

```
# スレーブがlocalhostにいる前提
mysql> CREATE USER 'repl'@'localhost' IDENTIFIED BY 'password';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'localhost';
```

* マスタのダンプ

```
$ mysqldump --all-databases --master-data > ~/master_dump.db
```

### スレーブ

* my.cnfの設定

```
[mysqld]
# 各MySQLサーバを識別するためのIDの設定
# レプリケーションクラスタ内で一意である必要がある
server-id=2

# 更新を禁止する
read-only
```

変更後、サーバを再起動する

* ダンプの読み込み

```
$ mysql -uroot < ~/master_dump.db
```

* レプリケーションの設定

マスタのバイナリログファイル名と開始位置を確認する

```
$ cat ~/master_dump.db | grep 'CHANGE MASTER TO'
```

レプリケーション設定

```
# 1つ前で確認したログファイル名、開始位置をそれぞれ MASTER_LOG_FILE, MASTER_LOG_POS に指定する
mysql> CHANGE MASTER TO
    ->   MASTER_HOST='localhost',
    ->   MASTER_USER='repl',
    ->   MASTER_PASSWORD='password',
    ->   MASTER_LOG_FILE='foo',
    ->   MASTER_LOG_POS=bar;
```

* レプリケーションの開始

```
mysql> START SLAVE;
```

設定変更などのためにレプリケーションを初期化したい場合は下記を実行する

```
mysql> RESET MASTER; # マスタバイナリログの受信を止める
mysql> RESET SLAVE;  # リレーログの初期化
```

一時的にレプリケーションを停止したい場合は以下を実行する

```
mysql> STOP SLAVE;
```

### 動作確認

* マスタ側で確認

```
# Commandフィールドに Binlog Dump がある
mysql> SHOW PROCESSLIST\G

# スレーブサーバの情報が出力される
mysql> SHOW SLAVE HOSTS;
```

* スレーブ側で確認

```
mysql> SHOW SLAVE STATUS\G
```

`SHOW SLAVE STATUS` で見るべき項目

| 項目 | 内容 |
|------|------|
| Slave_IO_State | スレーブの現在のステータス\n詳しくは https://dev.mysql.com/doc/refman/5.6/ja/slave-io-thread-states.html |
| Slave_IO_Running | スレーブI/Oスレッドが実行中かどうか |
| Slave_SQL_Running | スレーブSQLスレッドが実行中かどうか |
| Last_IO_Error | リレーログを処理するときにI/Oスレッドによって登録された最後のエラー |
| Last_SQL_Error | リレーログを処理するときにSQLスレッドによって登録された最後のエラー |
| Seconds_Behind_Master | スレーブSQLスレッドがマスタバイナリログの処理より何秒遅れているか |
| Read_Master_Log_Pos | スレーブ I/O スレッドがマスタバイナリログ(Master_Log_file)からどのくらい離れてイベントを読み取ったかを示す、ログ内の座標 |
| Exec_Master_Log_Pos | スレーブ SQL スレッドがマスタバイナリログ(Relay_Master_Log_File)から受け取ったイベントをどのくらい離れて実行したかを示す、ログ内の座標 |
| Relay_Log_Pos | スレーブ SQL スレッドがスレーブリレーログ(Relay_Log_File)をどのくらい離れて実行したかを示す、リレーログ内の座標 |

* リレーログテーブルの確認

```
mysql> SELECT * FROM mysql.slave_relay_log_info;
```
