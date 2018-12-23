---
title: "ORM/Database/Datasourceの棲み分け"
date: 2018-12-21T19:40:10+09:00
tags: [ORM-and-Database]
categories: ["CakePHP3"]
---
※1人AdventのDay-21です

[1人advent\(CakePHP中心、PHP開発よもやま\) Advent Calendar 2018 \- Adventar](https://adventar.org/calendars/3627)

## 概要
CakePHP3のデータベース周りの処理を追っていくと、「ORM」「Database」「Datasource」という、似た名前のレイヤーが存在することに気づきます。  
普段は特に意識することのないこれらの違いは、どこにあるのでしょうか。  
気になったので調べてみました。

## イントロ
CakePHP3を2.xと比較すると、最も大きな革新の1つはデータベース周りの構造の改革である、ということは間違いないと思います。   
それは、普段のアプリケーションを行っている上で直接的に目にする`Tale`/`Entity`/`Query`といったオブジェクトたちの登場から、感じることができます。そこから更に注意深くみてみると、フレームワークの内部ではさらなる複雑化が実施されていました。   
これらの構造については、普段の開発では意識することはないかも知れません。しかしながら、内部構造についてイメージを掴むことで、フレームワーク自体の設計思想やポテンシャル、"らしさ"についての理解が深まるのかとも思います。  

気になったので、自分なりに整理をしてみることにしました。

## ORM層を起点に「複雑さ」を感じてみる
まずは、最も馴染みが深いであろう `\Cake\ORM`です。  
この中には、`Table` `Entity` `Query` などのクラスがあります。  

さて、これ自体は普段から目にしている通りであって特に重要ではないのですが、問題は「実は結構いろいろと継承している」というところにあります。

例えば、`Table`クラスを見てみましょう。
![](/images/posts/2018-12-24-06-26-18.png)
何かを継承しているわけではないのですが、`\Cake\Datasource\RepositoryInterface`の実装であり、内部的には`\Cake\Database`から`TableSchema` `Type`というクラスを利用し、`\Cake\Datasource`からは`ConnectionInterface`とも関係していることが見て取れます。

同様に、`Entity``Query`についてもクラス図を置いてみます。

* Entity
    * ![](/images/posts/2018-12-24-06-24-45.png)
* Query
    * ![](/images/posts/2018-12-24-06-27-45.png)

このように、ORM/Database/Datasourceは、それぞれが密接に関わり合っていることがわかります。

## どのように違うのか？
実は、これらの3層はそれぞれがcakephpのGitHubアカウントにてスタンドアロンパッケージとして切り出され、公開されています。  
そこにあるdescriptionを見ることが、互いの位置づけを掴む上での端的な説明になりそうです。

* [cakephp/orm: \[READ\-ONLY\] A flexible, lightweight and powerful Object\-Relational Mapper for PHP, implemented using the DataMapper pattern\. This repo is a split of the main code that can be found in https://github\.com/cakephp/cakephp](https://github.com/cakephp/orm)
    * The CakePHP ORM provides a powerful and flexible way to work with relational databases. Using a datamapper pattern the ORM allows you to manipulate data as entities allowing you to create expressive domain layers in your applications.
* [cakephp/database: \[READ\-ONLY\] Flexible and powerful Database abstraction library with a familiar PDO\-like API\. This repo is a split of the main code that can be found in https://github\.com/cakephp/cakephp](https://github.com/cakephp/database)
    * Flexible and powerful Database abstraction library with a familiar PDO-like API
* [cakephp/datasource: \[READ\-ONLY\] Provides connection managing and traits for Entities and Queries that can be reused for different datastores\. This repo is a split of the main code that can be found in https://github\.com/cakephp/cakephp](https://github.com/cakephp/datasource)
    * Provides connection managing and traits for Entities and Queries that can be reused for different datastores

ORMは、その名の通りORMでデータマッパーパターンをベースにした「アプリケーションユーザーに対して、オブジェクトとしてデータを操作する」ための方法を提供します。  
Databaseは、各種RDBMSとの橋渡しです。そこでは、`PDO`などの接続に関わる具体実装の提供であったり、あるいはオブジェクト<->SQLといった「翻訳」についての具体化が実現されています。端的に言えば、`Driver`のクラスを提供しているのはこのレイヤーになります。
Datasourceは、3者の中で最も汎用化された上位のデザインであり、接続の管理(Manager)であったり、Connection/Repository(table)Entity/Query/ResultSetのインターフェイス、及びEntityやQueryのトレイトを提供しています。

なんとなく、それぞれの協力関係のようなものが見えてきたでしょうか。  
もう少し具体的に見ていきます。

## 土台の設計としてのDatasource
Datasourceは最上段の抽象層であり、基本的にはInterfaceとその実装を支援するためのTraitのコレクションであるというイメージです。  
例えば、`Connection` についてはInterfaceを持っていますが「どこに何でつなぐか」という情報は一切ここにはありません。自身の使われ方として、この段ではまだ「SQLライクのリレーショナルDBで使え」という風には定義されていないのです。(もちろん、QueryInterfaceや(Table)SchemaInterfaceの要件を満たす限りにおいて、ですが・・・)

## データの橋渡し役としてのDatabase
Datasourceに対して、Databaseはずいぶんと「具体的」で「現実的にどうすべきか」という対処方法を、知識として持ち合わせています。  
その端的な例として`Driver`を持っているのはここだし、「PDOのラッパー」といった仕事はここに集まっています。データベースとPHPの境界面を担っている、と言えると思います。  
しかしながら、(意外なことに?)ここには`Entity`系のクラスはありません。「SQLを実行する」という責務は担うし、また「`Type`によってPHP界でのバリューとデータベースにおける型の通訳を行う」のもこのレイヤーです。  
READMEには以下のように書かれています。

```
This library abstracts and provides help with most aspects of dealing with relational databases such as keeping connections to the server, building queries, preventing SQL injections, inspecting and altering schemas, and with debugging and profiling queries sent to the database.
```
このように、本来的に「接続」と「実行」に関する仕事が、守備領域であるといって良いと思います。

## アプリケーション的な話をするORM
そして、これらを利用して実際に「アプリケーション」をするのがORMという訳です。  
`Table` という表現は`Database`には現れず、このレイヤーまで降りてきて初めて出てきます。
基本的に、やりたいことは「アプリケーションお内部で扱いやすいように、取り出されたデータを翻訳して保持する」というDataMapperです。  
BehaviorやMarshal、Associationといった「よりオブジェクト指向的にデータに触れていく」という役割が、ここに集約されてきます。また、「アプリケーション側を向いている」のがORMということで、`Rule`(データの整合性に関するバリデーション)について具体的な内容が定義されているのはこのレイヤーです[^1]

[^1]: `RuleChecker`がDatasourceにも存在するのですが、こちらは(他のInterrfaceやTraitと同じく)「どのようにルールを使うか・実行するか」といったものが主眼にあるように思います。Managerクラスに近いかもしれません。


## まとめ
私がこれらの棲み分けについて興味を持ったのは、自分自身が独自に`Driver`を作ろうとしたのがきっかけでした。それまでは、一枚岩的に「ORM」がそこにあっただけであり、実際にデータのやりとりは`ORM`だけと向き合っていれば事足ります。  
それでも、何となくでも自分が普段触っている道具の「内部の流れ」が掴めるようになるというのは、興味深いものです。  
責務分担や領域の境界について、「あれ、なんでこっちに・・？」という疑問があったり「この依存関係は適切なのだろうか」と感じたりする麺もあります。それでも、コードリーディングをしていると随分と勉強になるなぁといった感想です。