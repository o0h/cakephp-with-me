---
title: "`Query` を愛用する。(もしくは `->all()` を避ける)"
date: 2018-03-27T13:52:39+09:00
tags: ["ORM-and-Database"]
categories: ["CakePHP3"]
---
Cake2では、Modelからの返却値がすべて配列でした。  
そのため、私はCake3を利用し始めた当初、「データベースからの取得した内容」は「値の集合」に変換して処理するぞ！みたいな意識が働きがちでした。  
例えばこんな様子です
```php
$books = $this->Books->find()->all();
$this->set(compact('books'));
```
もっと酷い時は、[ResultSet](https://api.cakephp.org/3.0/class-Cake.ORM.ResultSet.html)の使い方に難儀してhydrationをいじって、そのまま配列にして処理を・・・という書き方をしたりもしました。

ref: [Class Cake\\ORM\\Query \| CakePHP 3\.4](https://api.cakephp.org/3.4/class-Cake.ORM.Query.html#_enableHydration)

しかし、`Query`はIteratorAggregateを備えているので、「それをそのままループさせる」ことでレコードを処理することが可能です

```php
// in controller
$books = $this->Books->find();
$this->set(compact('books'));

// in template
<?php foreach ($books as $book): ?>
  <div><h1><?= h($book->title) ?></h1></div>
<?php endforeach; ?>}
```

個人的には「単純にレコードを取り出したいだけ」の時は(`all()`などを使って)ResultSet化しないでQueryオブジェクトのままやり取りすることが多いです。  

1. Queryのままにしておく = 結果を確定させないままの状態を維持しておくことで利用できる機能があるため
    * `order`や`where`の追加、subqueryの利用など
2. 後にロジックの改修があったときに変更をしやすい
   * 1と同じ理由です
3. 明示的に`all()`を使っている箇所について、必ず「ResultSetが欲しい」「DBにクエリを実行させたい」と言ったような意図をコード上で表現できるようになる
4. 単純にコードが減る

1,2は昨日の話です。3,4は表現力の話です。  
これらに鑑みて、「デメリットはあまり無さそうでちょっとメリットがあるな」というのが、現時点での私の所見です。
