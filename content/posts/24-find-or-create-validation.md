---
title: "findOrCreate()時にvalidationを行う"
date: 2018-12-02T18:12:32+09:00
tags: [ORM-and-Database]
categories: ["CakePHP3"]
---

※1人AdventのDay-2です

[1人advent\(CakePHP中心、PHP開発よもやま\) Advent Calendar 2018 \- Adventar](https://adventar.org/calendars/3627)

先日、Modelを書いているときにfindOrCreate()の挙動でハマった部分があったので調べてみました。
バリデーションが期待通りに動かなかったので、その対応を書いています。

## findOrCreateメソッドについて

CakePHP3のTableClassに、 `findOrCreate()` というメソッドがあります。

code: https://github.com/cakephp/cakephp/blob/3.6.13/src/ORM/Table.php#L1686

これは、「第1引数で渡されたデータを検索する。DB上に存在しなければ、新規にEntityを作成・保存し、それを返却する」というものです。知っていると、多くの場面で使いたくなります。

```php
$data = ['name' => 'new太郎'];
$entity = $this->Table->findOrCreate($data);
```

この結果として、(DB上にすでにレコードがあるか無いかにかかわらず) `{name: new太郎}` のEntityインスタンスが取得される、というわけです。なお、 すでに _persistent_ 済みであるため、`$entity->isNew()` はfalseとなります。

### 実装内容について詳しく

もう少し、内部処理について詳しく見てみましょう。  

```php
// \Cake\ORM\Table
public function findOrCreate($search, callable $callback = null, $options = [])
    {
        $options = new ArrayObject($options + [
            'atomic' => true,
            'defaults' => true,
        ]);

        $entity = $this->_executeTransaction(function () use ($search, $callback, $options) {
            return $this->_processFindOrCreate($search, $callback, $options->getArrayCopy());
        }, $options['atomic']);

        if ($entity && $this->_transactionCommitted($options['atomic'], true)) {
            $this->dispatchEvent('Model.afterSaveCommit', compact('entity', 'options'));
        }

        return $entity;
    }
```

optionsをデフォルトとマージした後に、具体的な処理は `_processFindOrCreate()` に委譲しています。

* processをラップしている `executeTransaciton()` は、「$atomicフラグに応じて、トランザクションを利用した関連データ更新込みの処理を行う」というものです。
  CakePHP2でいえば、saveにおいては `saveAssociated()` `saveAssociated()` も $atomicを受け付けていたことを想起します。
* `_transactionCommitted()` は、「トランザクションを抜けてるか」というものです。「$atomicモードであるか、もしくは$atomicモードではなくて$primaryとして呼び出されている」という条件でtrueを返します。対偶は「$atomicモードじゃない時に非$primary扱いの処理」はfalseです。

ということで、 `_processFindOrCreate()` を見れば内容がわかるという事になります。

#### _processFindOrCreate()

内部はこのようになっています。

```php
    protected function _processFindOrCreate($search, callable $callback = null, $options = [])
    {
        if (is_callable($search)) {
            $query = $this->find();
            $search($query);
        } elseif (is_array($search)) {
            $query = $this->find()->where($search);
        } elseif ($search instanceof Query) {
            $query = $search;
        } else {
            throw new InvalidArgumentException('Search criteria must be an array, callable or Query');
        }
        $row = $query->first();
        if ($row !== null) {
            return $row;
        }
        $entity = $this->newEntity();
        if ($options['defaults'] && is_array($search)) {
            $entity->set($search, ['guard' => false]);
        }
        if ($callback !== null) {
            $entity = $callback($entity) ?: $entity;
        }
        unset($options['defaults']);

        return $this->save($entity, $options) ?: $entity;
    }
```



やっていることは、大まかに3つです。

1. $searchを、DBへのSELECTクエリの実行を可能なように前処理をする
2. 1で得た$queryを用いて、レコードの検索を行う。
   1. 該当があれば、その1件目を返す
3. レコードが見つからなかった場合、Entityを作成して `Table::save()` した結果を返す

`findOrCreate()` の主たる使い方は「渡したデータ($search)で検索をし、なかったらそのデータを入力したEntityを返す」だと思っています。
しかし `_processFindOrCreate()` を見てみると、その例に従わないケースが2つあることに気付きます。

> if ($options['defaults'] && is_array($search)) 

の部分です。
すなわち、「$searchにQueryオブジェクトやcallbackを渡した場合」と「`$options['defaults']` に`false`をセットした場合」です。その場合は、Entityへのデータ入力をスキップして、「空のままのEntityをsave()する」ことになります。



### Entityへのデータセットの仕方に注意が必要

さて、問題は「Entityの作り方」です。

個人的には、コードの内容を見る前は `Table::newEntity()`を用いた操作が行われているものだと思っていました。
しかし、実際には `(new Entity())->set($data, ['guard' => false])`  となっています。
ここから感じた「思っていたのと違う」点は2つです。

#### guardが無効になる

ここでは明示的に guard:falseオプションが指定されているため、Entityクラスで規定されている保護(`Entity::$_accesible`)設定が無効化されています。すなわち、「どんなフィールドでも代入可能」ということです。

* CF: 
  * [エンティティー \- 3\.6](https://book.cakephp.org/3.0/ja/orm/entities.html#mass-assignment)
  * [一括代入に対する保護の回避](https://book.cakephp.org/3.0/ja/orm/entities.html#id9)

ただし、「その値を持っているEntityを取得する」という目的のメソッドなので、「一括代入不可能なようなフィールドを、引数として渡す」というニーズもあまりないように思いました。
「一括代入不可能にする」というのは、システム上で自動的に作成するもの(idやダイジェスト値など)などの「ユーザーの入力による変更が発生したら困るもの」が多いはずだからです。

#### Validationが実行されない

もう1つの点は、「validationが実行されない」 という点です。 
「Entityインスタンスへのメソッド実行によるミューテーションを行う」というサイクルにより、同じ「新規にEntityを作る」という行為でも、こちらは 「`newEntity()`に内包されている処理を通さない」ということになります。

内部実装としては [Marshaller::one()](https://github.com/cakephp/cakephp/blob/c15a2e9c1933fa5a874c33d2bcb6d63efacf8876/src/ORM/Marshaller.php#L168-L228)メソッドになりますが、`newEntity()` の場合はこの内部でvalidationを行うことになります。[^1]つまり、`findOrCreate()`の場合は、部分的に「不正な値の入力を通してしまう」ということになります。[^2]

[^1]: 既存のEntityインスタンスへの値を更新し、かつvalidationを実行したい場合は、`Table::patchEntity()`を利用することになります。

[^2]: applicatoin rule(domain rule)による検証は、`Table::save()`メソッド内部での作用なので、通常通りに行われます。そのため、existsInルールやisUniqueルール等についてはハンドリング可能になります。

ここが、私の「思っていたのと違う」ポイントでした。

また、関連して `beforeMarshal`も呼び出されないことになります。
「レコード作成前のデフォルト値の生成・代入を行う」といった処理をhookしている場合、「値がない」状態のままでsave()まで進んでいくのを防げません。期待通りの挙動をなさず、データベースレイヤーでの例外発生やPHPのError(undefind indexやnullへのアクセスなど)に至ることになります。

このメソッドは高機能であり、一見とても便利ではあるのですが、利用する際には入力するデータに関しては事前に検証するなど慎重に扱わないと危険かもしれません。
例えば、CGMのようなサービスでの「ハッシュタグの作成」といったシナリオは、「なかったら作って、そのIDやインスタンスを扱いたい」といったニーズが有るように思います。これに対して、文字長制限や空文字チェックなどを素通りしてしまう可能性があります。
システム内部の処理や、ユーザーの入力値を直接扱わない場面に活躍の場を留めておいたほうが無難でしょうか。

## findOrCreateでバリデーションを行う

さて、`findOrCreate()`はその第2引数に「callback」を取ることができます。
`_processFindOrCreate()`における該当箇は以下のとおりです

```php
        if ($callback !== null) {
            $entity = $callback($entity) ?: $entity;
        }
        unset($options['defaults']);

        return $this->save($entity, $options) ?: $entity;

```

「Entity作成・値の代入後〜save()の実行前に処理を挟む」ことができます。  
これを利用すれば、vlaidationも可能になります。
例えば、ざっくりとやるなら「生成されたEntityインスタンスを1度捨ててしまい、改めてTableインスタンス経由で`newEntity()`を実行する」ことでも目的が達成されます。

```php
$callback = function (Entity $entity) use ($table) {
    return $table->getValidator()->newEntity($entity->toArray());
};
$entity = $table->findOrCreate($data, $callback);
```

こうすることで、save()に渡される$entityにエラーがマークされ、INSERTING処理の実行より手前で「保存に失敗する」ようになります。
もしくは、callback内で独自に `Cake\ORM\PersistenceFailedException`などの例外をスローする方法でも、ハンドリングしやすいかもしれません。