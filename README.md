# MySQLワークショップ

## 目次

### MySQL基礎1

#### 内容

* [レプリケーション](training1_replication.md)
  * 1. レプリケーションとは
  * 2. レプリケーションの仕組み
  * 3. レプリケーションの種類
  * 4. レプリケーションの設定
  * 5. レプリケーション障害
  * 6. スレーブ昇格
  * 7. Global Transaction ID(GTID)
  * 8. Tips

### MySQL基礎2

#### 内容

* ハイパフォーマンスMySQL(仮)


### サーバセットアップ

1. `vagrant`, `virtualbox`, `ansible` が入っていない場合はインストールしておく
2. `git clone https://github.com/mynet-inc/mysql_training`
3. `cd mysql_training/provisioning`
4. `vagrant up`
5. `cd ansible`
6. `ansible-playbook -i hosts site.yml`
