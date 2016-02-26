## 1. パフォーマンス観点

単純にパフォーマンスと言った場合の捉え方は人様々である。
例えば以下のような項目が考えられる。

* 1秒あたりのクエリ数
* CPU使用率
* スケーラビリティ

前提としてここではパフォーマンスは `1クエリあたりの応答時間` を指標とする。
データベースサーバとして最も興味があるであろう、クエリを解釈してデータを返すというタスクのパフォーマンスを見るのに適当だからである。

また特に断りのない限り、本内容はInnoDBを想定したものとする。

## 2. プロファイリング

何をチューニングすべきか、チューニングした結果どう変わったか、その指標とするためにプロファイリングを用いる。

### 2.1. スロークエリログ

クエリにどのくらい時間がかかったかなどの情報を出力するためのログ。後述するプロファイリングツールはこのログを解析して結果を出しているものが多い。

#### 主要な設定項目

* `slow_query_log`
  * スロークエリログを出力するかどうか
* `long_query_time`
  * スロークエリログに出力するための閾値とする実行時間
  * 0 を設定することで全クエリが出力される
  * 0.5秒以上かかったクエリを出力したいのであれば `0.5` と指定する
* `log_queries_not_using_indexes`
  * インデックスが使われていないクエリを出力するかどうか
  * 有効化すると `long_query_time` に該当しないクエリでも出力されるようになる
* `slow_query_log_file`
  * スロークエリログの出力先
  * デフォルトでは `datadir/<hostname>-slow.log` に出力される
* `log_slow_admin_statements`
  * 管理ステートメントのログ出力を行うかどうか
  * 対象のステートメント
    * ALTER TABLE
    * ANALYZE TABLE
    * CHECK TABLE
    * CREATE INDEX
    * DROP INDEX
    * OPTIMIZE TABLE
    * REPAIR TABLE

### 2.2. スロークエリログの解析

スロークエリログ単体ではログの量が膨大になるため、通常ログと合わせて解析ツールが用いられる。

#### mysqldumpslow

* 特徴
  * 公式の解析ツール
  * mysqlに付属している
  * 以下いずれかの項目でソートした結果を出力出来る
    * 総ロックタイム
    * 総行数
    * 総実行時間
    * 総クエリ数
    * 平均ロックタイム
    * 平均行数
    * 平均実行時間(デフォルト)

実行例

```
# 総実行時間が長い順でソート
$ mysqldumpslow -s t slow_query.log
```

#### pt-query-digest

* 特徴
  * Percona Serverを作っているPercona社が作っている
  * スロークエリログ以外にも `PROCESSLIST` や `tcpdump` のデータを元に解析が可能
  * 総実行時間、総クエリ数などいくつかの軸でクエリをランキングしてくれる
  * 解析と同時に各クエリのサマリの `explain` 結果が出力可能

* インストール

RHEL系

```
yum install perl-Time-HiRes perl-IO-Socket-SSL
yum localinstall http://www.percona.com/redir/downloads/percona-toolkit/LATEST/RPM/percona-toolkit-2.2.7-1.noarch.rpm
```

実行例

```
# スロークエリログから読む
$ pt-query-digest --type slowlog slow.log

# processlistから読む
$ pt-query-digest --processlist h=localhost,u=root,p=password --run-time 60

# tcpdumpから読む
$ tcpdump -s 65535 -x -nn -q -tttt -i any -c 1000 port 3306 mysql.tcpdump
$ pt-query-digest --type tcpdump mysql.tcpdump
```

## 3. インデックス

* 前提はInnoDBとする
  * インデックスはB+Tree構造を使っているものとする
* どういう場合に有効か
  * 多くの場合で有効。とりあえずやっていない場合はやって良い
  * インデックスを貼っても検索行数が大きすぎる場合は効果が薄い
    * パーティション、シャーディングでテーブルを分割させて母数のデータ数を減らす
    * 巨大なユーザテーブルを分解して、特定のグループだけが集まるようなユーザテーブルを抽出するなど
* インデックスの構造
  * インデックスの持ち方
  * クラスタインデックス
  * セカンダリインデックス
* カーディナリティ
* 複合インデックスの順番
* 実際にインデックスを使うときのコーナーケース
  * プレフィックスインデックス
    * BLOB型やTEXT型などの非常に大きいデータを持つカラムにインデックスを貼りたい場合
    * InnoDBはインデックスをキーとするので余計にディスクを食う。そもそも付けれるインデックス名に制限長がある

## 4. パーティション

* 母数を減らすために有効
* 設定方法

## 5. シャーディング

* 母数を減らすために有効
* Updateの負荷が分散出来る
* 設定方法
  * キーの振り分け方式など
    * ユーザID % 100 が計算しやすくて運用にも便利だよという結論

## 6. サーバ設定

* メモリに載せたほうが良い

## 7. Tips

* 何か

## 参考メモ

http://naokirin.hatenablog.com/entry/2015/02/07/193609
http://www.mysql.gr.jp/frame/modules/bwiki/index.php?plugin=attach&refer=matsunobu&openfile=Session-Index.pdf
