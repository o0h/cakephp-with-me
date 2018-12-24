---
title: "Entityの`$_accessible`について、もう1度。"
date: 2018-12-14T06:44:56+09:00
tags: [ORM-and-Database]
categories: ["CakePHP3"]
---
※1人AdventのDay-14です

[1人advent\(CakePHP中心、PHP開発よもやま\) Advent Calendar 2018 \- Adventar](https://adventar.org/calendars/3627)

## 概要
CakePHP3では、データの誤操作を防ぐためEntityの持つプロパティへの代入可否を設定する `$_accessible` 機構が備わっています。  
具体的な利用方法を確認していきたいと思います。

<!--more-->

## イントロ
フレームワークを利用したアプリケーション開発の最大のメリットの1つが「簡単・少量の記述で、パワフルなシステムを構築できる」点にあると思います。  
その一方で、「うっかりと変なことをやってしまう」リスクが高まる部分もあります。とりわけ、セキュリティやデータ操作の領域においての「うっかり」は取り返しのつかないダメージになりえます。

CakePHP3において、データの健全な扱いを支援するための機構の1つが `Entity::$_accessible`によるプロパティの保護です。  
これを設定することで、データを簡易に代入しつつ更新を許したくないプロパティについては渡された値を無視できるようになります。

## 利用例(Cookbookより)
まずは、Cookbookにある例をひきながらその基本的な利用の方法についておさらいをしましょう。  

[エンティティー \- 3\.7](https://book.cakephp.org/3.0/ja/orm/entities.html#mass-assignment)

最も基本的な設定方法は、以下のとおりです。

```php
namespace App\Model\Entity;

use Cake\ORM\Entity;

class Article extends Entity
{
    protected $_accessible = [
        'title' => true,
        'body' => true,
        '*' => false,
    ];
}
```

Enittyのメンバーとして `$_accessible` を設け、その連想配列の中に「key => 代入の可否」という形で値をもたせます。明示的に指定されていないプロパティはデフォルトではすべて `false`(代入不可)となり、「明示指定していないプロパティ(=「その他」のプロパティ)の挙動を変える」という場合には `*` を利用します。


### `$_accessible` が適用される場面
`$_accessible` による代入保護が働く場面は、主に以下の3つです。

1. `Table::newEntity()` によるEntityの作成、 `Table::patchEntity()` によるEntityの更新
2. newキーワードを利用したEntityの作成において、オプション(コンストラクタの第2引数となる連想配列)で `'guard' => true` を指定した場合
3. `Entity::set()` で、第1引数が連想配列の場合

1・2のケースでも内部的には`set()`メソッドを利用しており、結果的には3の挙動が適用されることになります。  

ここで注意すべきなのは、`set()`の呼び出し方によって代入の保護実施の有無が変化するという点です。  
概念として、 `$_accesible(false)`による保護は、**「値の一括代入に対する保護」**という点です。言い換えると、「指定した単一のプロパティへの代入」に関しては、考慮がなされません。  
具体例を示します。


```php
// table
namespace App\Model\Table;

use Cake\ORM\Table;

class PostsTable extends Table
{
}
?>
// entity
namespace App\Model\Entity;

use Cake\ORM\Entity;

class Post extends Entity
{
    protected $_accessible = [
        'title' => true,
        'is_deleted' => false,
    ];
}

/* -------- */
use App\Model\Entity\Post;
use Cake\ORM\TableRegistry;

$data = ['title' => 'The sample post.', 'is_deleted' => true];

// Entityのコンストラクタ
$post = new Post($data)
$post->toArray(); //['title' => 'The sample post.', 'is_deleted' => true]

$post = new Post($data, ['guard' => true]);
$post->toArray(); //['title' => 'The sample post.']

// Tableクラスの経由
$PostsTable = TableRegistry::getTableLocator()->get('Posts');
$post = $PostsTable->newEntity($data);
$post->toArray(); //['title' => 'The sample post.']

// Entityの更新

$post->set(['is_deleted' => true]);
$post->toArray(); //['title' => 'The sample post.']

$post2 = $PostsTable->patchEntity($post, ['is_deleted' => true]);
$post2->toArray(); //['title' => 'The sample post.']

$post->set('is_deleted', true);
$post->toArray(); //['title' => 'The sample post.', 'is_deleted' => true]

$post->is_deleted = false;
$post->toArray(); //['title' => 'The sample post.', 'is_deleted' => false]
```

### 保護の回避・変更
以下の方法で、保護済みのプロパティへの代入を許可することができます。

1. `Table::newEntity()`,  のオプションに`accessibleFields` を渡す
2. `Entity::setAccesible()` による保護設定の変更
3. 「`$_accessible` が適用される場面」で紹介した、単一プロパティへのミューテート各種
4. Entityのコンストラクタへの `guard` オプションの無効化指定、もしくは指定の省略

1の`accessibleFields`オプションを用いた方法については、生成するEntityの `$_accessible` 設定に作用するやり方です。そのため、「生成時の代入を許可する」ものではなく「代入可能な状態になって生成される」であることに注意してください。


## まとめ
私自身、`$_accessible` についての理解を長らく曖昧なままにしていた反省があったため、Cookbookの内容を具体的に掘り下げることでまとめてみました。  
あくまで(Cookbookでそう示されている通り)「一括代入の保護」であり、これはユーザーからのリクエストデータのマーシャル操作などを想定しているものであると思います。逆に言えば、単一のプロパティを指定して値の書き換えを行うなど、明らかに「アプリケーション実装者の意図」が介入する場合は、楽観的な作用をするということです。  
設計思想を汲み取ることで、やや複雑な仕様や紛らわしい操作についてのその違いがはっきりしやすういのではないでしょうか。