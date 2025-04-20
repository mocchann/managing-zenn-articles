---
title: "【MySQL】分離レベルと現象についてまとめる"
emoji: "🐬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["DB", "MySQL", "トランザクション", "分離レベル"]
published: false
---

## はじめに

とある日の業務中のこと、毎時1回動作するバッチにより生成されるテーブルのレコード件数が、**正しいときもあれば2倍になるときもある**という現象に出くわしました
(業務内容なので詳細は省きかなり抽象的に書きます)

調査すると、以下のようにテーブルAから取得したデータを元に、新しくテーブルBのレコードを生成するという仕組みになっていることが判明

* テーブルA
    * バッチAにより、毎時00分にレコード生成
* テーブルB
    * バッチBにより、毎時00分にテーブルAのレコードを元にテーブルBのレコードを生成

上記を踏まえて原因と想定されるのはおそらく**ファントムリード**だろうという結論になり、バッチの実行タイミングをズラすことで事象自体は解消しました

そこで個人的にMySQLの分離レベルについての理解がふわっとしているためこれを機に記事にまとめておこうと思います

## 分離レベルとは

ANSI(米国国家規格協会)と呼ばれる標準化組織が定義している

ANSIが定義する分離レベルと対応する現象は以下

| 分離レベル | ダーティリード | ファジーリード | ファントムリード |
| ---- | ---- | ---- | ---- |
| リードアンコミッテッド | ◯ | ◯ | ◯ |
| リードコミッテッド | × | ◯ | ◯ |
| リピータブルリード | × | × | ◯ |
| シリアライザブル | × | × | × |

MySQLのdocsでは以下のように記載されています

> 分離レベルは、複数のトランザクションで変更が行われ、クエリーが同時に実行される場合に、パフォーマンスと信頼性、一貫性および結果の再現性のバランスを微調整する設定です。

https://dev.mysql.com/doc/refman/8.0/ja/innodb-transaction-isolation-levels.html

## 分離レベルごとに起こりうる現象

検証用テーブルも事前に作っておきます
※MySQL8.xを使用します

```
mysql> create table test(id int not null primary key, col varchar(20));
mysql> insert into test (1, 'first');
mysql> select * from test
```

![alt text](</images/07fcdbc0cce8b5/select_test.png>)

### ダーティリード(Dirty Read)

:::message alert
**あるトランザクションがコミットされる前に、別のトランザクションからデータを読み取れてしまう現象のこと**
:::

まずはダーティリード用のトランザクションを準備する

```
mysql> prompt DirtyRead>

// ダーティリードを起こすために、MySQLの分離レベルをリードアンコミッテッドに設定する(MySQL8.xのdefaultはリピータブルリードのため)
DirtyRead> set transaction isolation level read uncommitted;

DirtyRead> start transaction;
```

別のシェルを立ち上げて以下を実行

```
mysql> prompt Transaction>

Transaction> start transaction;

Transaction> update test set col='second' where id = 1;
```

再度、DirtyReadのトランザクションから以下を実行

```
DirtyRead> select * from test;
```

![alt text](</images/07fcdbc0cce8b5/dirty_lead.png>)

Transactionでupdateした内容をまだコミットしていないにも関わらず、DirtyReadのトランザクションからデータを読み取れてしまいました

:::message
このように**あるトランザクションで行ったコミット前の変更を、別のトランザクションで読み取れてしまう現象をダーティリード**と呼びます
:::

次の検証に移るのでTransaction側で一旦コミットしておきます

```
Transaction> commit;
```

### ファジーリード(Fuzzy Read)

:::message alert
**あるトランザクションが1回目に読み取ったデータを再度読み取ったときに、2回目の結果が初回読み取りとして扱われてしまう現象**
:::

ファジーリード用のトランザクションを準備する

```
mysql> prompt FuzzyRead>

// ファジーリードを起こすため、分離レベルをリードコミッテッドに設定する
FuzzyRead> set transaction isolation level read committed;

FuzzyRead> start transaction;

// 1回目の読み取りを実行
FuzzyRead> select * from test;
```

![alt text](</images/07fcdbc0cce8b5/select_test_fuzzy_read.png>)

次に新しくシェルを追加して、別のトランザクションの中でレコードを更新します

```
mysql> prompt Transaction>

Transaction> start transaction;

Transaction> update test set col='updated_value' where id = 1;

Transaction> commit;
```

そして再度、先ほどと同様のファジーリードのトランザクションでデータを読み取ります

```
FuzzyRead> select * from test;
```

![alt text](</images/07fcdbc0cce8b5/select_test_fuzzy_read2.png>)

同じトランザクションにも関わらず、1回目と2回目の読み取りで結果が異なっています

:::message
このように**あるトランザクションが1回目に読み込んだデータを再度読み込んだときに、2回目以降の結果が1回目となってしまう現象をファジーリード**と呼びます
:::

### ファントムリード(Phantom Read)

:::message alert
**あるトランザクションを複数回に渡って読み取ったときに、最初にSELECTしたレコードが3行だったのに対して2回目にSELECTしたレコードが4行だったりと、データが読み取れたり読み取れなかったりする現象**
:::

ファントムリード用のトランザクションを準備する

```
mysql> prompt PhantomRead>

// ファントムーリードを起こすため、分離レベルをリピータブルリードに設定する
PhantomRead> set transaction isolation level repeatable read;

PhantomRead> start transaction;

// 1回目の読み取りを実行
PhantomRead> select * from test;
```

![alt text](</images/07fcdbc0cce8b5/select_test_phantom_read.png>)

この時点ではレコードが1件だけ読み取られています

次に新しくシェルを追加して、別のトランザクションの中でレコードを挿入します

```
mysql> prompt Transaction>

Transaction> start transaction;

Transaction> insert into test values(2, 'phantom');

Transaction> commit;
```

そして再度、先ほどと同様のファントムーリードのトランザクションでデータを読み取ります

```
// 2回目の読み取りを実行
PhantomRead> select * from test;
```

![alt text](</images/07fcdbc0cce8b5/select_test_phantom_read2.png>)

ここでファントムリードが起こり、先ほどinsertしたレコードが見える想定でしたが**select結果は変わりませんでした**

実はこれ、MySQLのinnoDB型のテーブルが**MVCC**という仕組みで動作しているため、ファントムリードが起きないようになっているからです

```
// MySQLでファントムリードを再現するためリードコミッテッドに設定する
PhantomRead> set transaction isolation level read committed;

PhantomRead> start transaction;

// 1回目の読み取りを実行
PhantomRead> select * from test;
```

![alt text](</images/07fcdbc0cce8b5/select_test_phantom_read3.png>)

```
Transaction> start transaction;

Transaction> insert into test values(3, 'phantom lead');

Transaction> commit;
```

```
2回目の読み取りを実行
PhantomRead> select * from test;
```

![alt text](</images/07fcdbc0cce8b5/select_test_phantom_read4.png>)

MySQLの分離レベルをリードコミッテッドに設定するとファントムリードを再現しました

同じトランザクションにも関わらず、1回目と2回目の読み取りで結果が異なり新しいレコードが表示されています

つまり、MySQLにおける分離レベルと現象の対応表は以下が正しいということになります

| 分離レベル | ダーティリード | ファジーリード | ファントムリード |
| ---- | ---- | ---- | ---- |
| リードアンコミッテッド | ◯ | ◯ | ◯ |
| リードコミッテッド | × | ◯ | ◯ |
| リピータブルリード | × | × | × | // リピータブルリードでもファントムリードは起きない
| シリアライザブル | × | × | × |


:::message
このように**あるトランザクションで複数回読み取りを行ったときに、データが現れたり(INSERT)消えたり(DELETE)してしまう現象をファントムリード**と呼びます
:::

## おわりに

分離レベルと現象について記事にまとめてきましたが、ここでひとつ新たな疑問が浮かびました

冒頭で記載した「とある日の業務中のこと、毎時1回動作するバッチにより生成されるテーブルのレコード件数が、**正しいときもあれば2倍になるときもある**という現象に出くわしました」について、おそらくファントムリードだろうとの結論になっていました

そこで改めてMySQLの分離レベルを確認してみたのですが、**REPEATABLE READ**でした

この記事での検証でもうお分かりかと思いますが**MySQLの分離レベルがREPEATABLE READ = ファントムリードは発生しない**ということになります

なので今回の現象はDBの分離レベルの挙動そのものというよりは、**並行して動作するバッチのロジック自体に起因する問題**である可能性が高そうです(データ作成前のDELETE処理が何らかの原因で失敗していた等)

同じテーブルを参照するバッチ処理の実行タイミングを同時刻に設定することは、データ競合などを引き起こす可能性があるので、**バッチ処理の実行タイミングは適切に分散させましょう**ということで締めの言葉としたいと思います
