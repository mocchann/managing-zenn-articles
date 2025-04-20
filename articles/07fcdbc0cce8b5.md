---
title: "【MySQL】分離レベルと現象についてまとめる"
emoji: "🐬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["DB", "MySQL", "トランザクション", "分離レベル"]
published: false
---

## はじめに

とある日の業務中のこと、毎時1回動作するバッチにより生成されるテーブルのレコード件数が、**正しいときもあれば2倍になるときもある**という現象に出くわしました

調査すると、以下のようにテーブルAから取得したデータを元に、新しくテーブルBのレコードを生成するという仕組みになっていることが判明

* テーブルA
    * バッチAにより、毎時00分にレコード生成
* テーブルB
    * バッチBにより、毎時00分にテーブルAのレコードを元にテーブルBのレコードを生成

上記を踏まえて原因と想定されるのはおそらく**ダーティリード**だろうという結論になり、バッチの実行タイミングをズラすことで事象自体は解消

しかし、MySQLの分離レベルについての理解がふわっとしているためこれを機に記事にまとめておこうと思います

## 分離レベルとは

ANSI(米国国家規格協会)と呼ばれる標準化組織が定義している

MySQLの場合、分離レベルと対応する現象は以下

| 分離レベル | ダーティリード | ファジーリード | ファントムリード |
| ---- | ---- | ---- | ---- |
| リードアンコミッテッド | ◯ | ◯ | ◯ |
| リードコミッテッド | × | ◯ | ◯ |
| リピータブルリード | × | × | ◯ |
| シリアライザブル | × | × | × |

## 分離レベルごとに起こりうる現象

検証用テーブルも事前に作っておきます
※MySQL8.x

```
create table test(id int not null primary key, col varchar(20));
insert into test (1, 'first');
select * from test
```

![alt text](</images/07fcdbc0cce8b5/select_test.png>)

### ダーティリード(Dirty Read)

あるトランザクションがコミットされる前に、別のトランザクションからデータを読み取れてしまう現象のこと

まずはダーティリード用のトランザクションを準備する

```
prompt DirtyRead>

// ダーティリードを起こすために、MySQLの分離レベルをリードアンコミッテッドに設定する(MySQL8.xのdefaultはリピータブルリードのため)
DirtyRead> set transaction isolation level read uncommitted;

DirtyRead> start transaction;
```

別のシェルを立ち上げて以下を実行

```
prompt Transaction>

Transaction> start transaction;

Transaction> update test set col='second' where id = 1;
```

再度、DirtyReadのトランザクションから以下を実行

```
DirtyRead> select * from test;
```

![alt text](</images/07fcdbc0cce8b5/dirty_lead.png>)

Transactionでupdateした内容をまだコミットしていないのにも関わらず、DirtyReadのトランザクションからデータを読み取れてしまいました

このように**あるトランザクションで行ったコミット前の変更を、別のトランザクションで読み取れてしまう現象をダーティリード**と呼びます

次の検証に移るのでTransaction側で一旦コミットしておきます

```
Transaction> commit;
```

### ファジーリード(Fuzzy Read)

あるトランザクションが1回目に読み取ったデータを再度読み取ったときに、2回目の結果が初回読み取りとして扱われてしまう現象

ファジーリード用のトランザクションを準備する

```
prompt FuzzyRead>

// ファジーリードを起こすため、分離レベルをリードコミッテッドに設定する
FuzzyRead> set transaction isolation level read committed;

FuzzyRead> start transaction;

// 1回目の読み取りを実行
FuzzyRead> select * from test;
```

![alt text](</images/07fcdbc0cce8b5/select_test_fuzzy_read.png>)

この時点ではレコードが1件だけ読み取られています

次に新しくシェルを追加して、別のトランザクションの中でレコードを挿入します

```
prompt Transaction>

Transaction> start transaction;

Transaction> insert into test values(2, 'fuzzy');

Transaction> commit;
```

そして再度、先ほどと同様のファジーリードのトランザクションでデータを読み取ります

```
FuzzyRead> select * from test;
```

![alt text](</images/07fcdbc0cce8b5/select_test_fuzzy_read2.png>)

同じトランザクションにも関わらず、1回目と2回目の読み取りで結果が異なっています

このように**あるトランザクションが1回目に読み込んだデータを再度読み込んだときに、結果が変わってしまう現象をファジーリード**と呼びます

### ファントムリード(Phantom Read)

あるトランザクションを複数回に渡って読み取ったときに、最初にSELECTしたレコードが3行だったのに対して2回目にSELECTしたレコードが4行だったりと、データが読み取れたり読み取れなかったりする現象

## おわりに
