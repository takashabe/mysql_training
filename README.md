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

ワークショップで用意するサーバと同構成のサーバをローカルに立てる事が出来ます。
`vagrant`, `virtualbox`, `ansible` を利用するのでそれらが入っていない場合は別途インストールしておいてください。

```
git clone https://github.com/mynet-inc/mysql_training
cd mysql_training/provisioning
vagrant up
cd ansible
ansible-playbook -i hosts site.yml
```
