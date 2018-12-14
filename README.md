# mysql-training
MySQLのスレーブを理解する為のトレーニングツールです。

# Training
以下の実習をしていきましょう。

このリポジトリには `docker-composeで2台のMySQLを起動する` 為のdocker-composeのファイル一式だけ用意してあります。
課題が進むにつれて自分で修正していくスタイルでMySQLのレプリケーションを学びます。

## Requirement 
* Install docker / docker-compose (docker for Mac)

## Step1 Normal Replication
1. docker-composeで2台のMySQLを起動する。
1. MasterとSlaveで `binlog with GTID` を `on` にする。
1. レプリケーションを行ってみる。(mysql dump編)
  1. レプリケーションユーザーを作る。
  1. サンプルデータをMasterにいれる。
    `wget http://downloads.mysql.com/docs/world.sql.gz`
  1. MasterのデータをSlaveにいれる。
  1. レプリケーションを開始する。  

## Step1 Optional Step
1. テーブル単位でレプリケーションをIgnoreしてみる。
1. ROWデータのrsync方式でレプリケーションを行ってみる。

## Step2 MultiSource Replication
TBD


## Answer
[ANSWER.md](./ANSWER.md)
