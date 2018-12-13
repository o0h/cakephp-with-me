---
title: "カスタムファインダーについておさらい"
date: 2018-12-13T19:09:01+09:00
tags: [ORM-and-Database]
categories: ["CakePHP3"]
---
※1人AdventのDay-13です

[1人advent\(CakePHP中心、PHP開発よもやま\) Advent Calendar 2018 \- Adventar](https://adventar.org/calendars/3627)

## 概要
CakePHP3で導入された「カスタムファインダー」は、Queryの組み立てを抽象化・パッケージ化する手法です。  
よく利用するconditonsの追加やfields、formatResultsなどの手順を一箇所にまとめ、更にメソッドチェーンによるQueryのビルドを可能にします。

## イントロ

CakePHP3のORM機能の1つに、Tableで提供されている「カスタムファインド」というものがあります。  
これは、より「鮮明で簡潔なコーディングをする」ことを支援する強力な機能だと思っています。

とはいっても、自分自身もCakePHP3を触り始めた当初はその存在[^1]を知らず、利用できていませんでした。今になって感じるのは、うまく付き合えばより「Cakeらしい」コードが書けそうだということです。  
この記事では、その具体的な実装や内部処理の流れについて触れていきます。

[^1]: 名前は知っていたのですが・・・使おう、というところまでは至らなかったのです。

## 簡単な利用例(Cookbookより)
まずは、Cookbookにある内容を例にとって、簡単なサンプルに触れてみたいと思います。
[データの取り出しと結果セット \- 3\.7](https://book.cakephp.org/3.0/ja/orm/retrieving-data-and-resultsets.html#custom-find-methods)

```php
use Cake\ORM\Query;
use Cake\ORM\Table;

class ArticlesTable extends Table
{

    public function findOwnedBy(Query $query, array $options)
    {
        $user = $options['user'];
        return $query->where(['author_id' => $user->id]);
    }

}

// コントローラーやテーブルのメソッド内で
$articles = TableRegistry::get('Articles');
$query = $articles->find('ownedBy', ['user' => $userEntity]);
```

よくありそうな、「記事の筆者によって絞り込む」という処理です。  
これは`WHERE author_id = :user_id`  という条件付けによって達成されます。

カスタムファインダーを使う際には、2つの手順が必要です。

1. `findXxx` という規則に従ってメソッド名をつける
2. `Table::find('Xxx')` という形で呼び出す

とてもシンプルですが、これだけでOKです。  
これは、CakePHP2の時代から踏襲されているものです。  

参考) [\[CakePHP2\] カスタムファインダーの使い方と利便性 \- Qiita](https://qiita.com/chinpei215/items/1f719b58e62a283ea27b)

ただし、以前のバージョンと大きく違うのは、**CakePHP3のfindメソッドは、(fetchした結果ではなく)Queryを返す** ということです。  
これにより、「汎用的で再利用性の高い処理」に対して「今この場でのみ必要な処理」を連鎖させることがとても容易になっています。

### 簡単な利用例②
これを、もう少し実際的に利用している例も紹介されています。

```php
// コントローラーやテーブルのメソッド内で
$articles = TableRegistry::get('Articles');
$query = $articles->find('published')->find('recent');
```

内容としては、「公開済み」で、かつ「最近公開されたもの」と説明されています。  
具体的な実装としては、以下のようなイメージでしょうか。


```php
use Cake\I18n\FrozenTime;
use Cake\ORM\Query;
use Cake\ORM\Table;

class ArticlesTable extends Table
{
    const STATUS_DRAFT = 1;
    const STATUS_IN_REVIEW = 2;
    const STATUS_PUBLISHED = 3;

    public function findPublished(Query $query, array $options)
    {
        return $query->where(['status' => self::STATUS_PUBLISHED]);
    }

    public function findRecent(Query $query, array $options)
    {
        $threshold = $options['threshold'] ?? new FrozenTime('-3 days');
        return $query->where(['published >=' $threshold]);
    }

}
```
これによって、「3日以内に公開された、閲覧可能状態にある記事」がごく簡単に取得できます。まだまだ単純な例ですが、「処理内容にラベルを与える」というのはコードの書き手に安心感を与えるられそうで嬉しいですね。

もちろん、通常のクエリと同様に任意のメソッドを重ねていくことも可能です。

```php
$query = $articles
    ->find('published')
    ->find('recent')
    ->orderAsc('created');
```

## 踏み込んだ利用例
Cookbook中に、

>フェッチ後に結果を変更する必要があるなら、 結果を Map/Reduce で変更する 機能を使って結果を変更してください。 map reduce 機能は、旧バージョンの CakePHP にあった 'afterFind' コールバックに代わるものです。

という言及があります。  

これの実装例が、 `findList()` です。listタイプのfindの使い方については、 [Cookbokの説明](https://book.cakephp.org/3.0/ja/orm/retrieving-data-and-resultsets.html#table-find-list)を御覧ください。

一部を抜粋して、 「map reduce」機能の部分を引用してみます。

```php
// \Cake\ORM\Table::findList()
return $query->formatResults(function ($results) use ($options) {
    /** @var \Cake\Collection\CollectionInterface $results */
    return $results->combine(
        $options['keyField'],
        $options['valueField'],
        $options['groupField']
    );
});
```

`formatResults()` は、queryの実行結果を呼び出し側に戻す前に「形を変形させる」という働きを持ちます。第1引数として`query->execute()`の実行結果を渡され、それはCollectionInterfaceを実装しているように指定されています。 
`findList`の例でいうと、「ResultSetをそのまま帰す」のではなく「`Collection::combine()` を適用して、プリミティブな連想配列を返す」ようになります。  
このように、「クエリ実行後の加工・変形処理を差し込むことができる」という余地があることで、カスタムファインダーはより「かゆいところに手が届く」ようになったと感じています。


## 内部実装を覗いてみる
実際には、Queryインスタンスはどのように渡されていっているのでしょうか？  
簡単にでも具体的な挙動を掴んでおくことは、「普段の開発で気軽に機能を使う」ためのハードルを下げると思っています。  
そこで、内容について少々踏み込んでみてみましょう。

例えば、次のようなクエリを実行したとします。

```php
$query = $this->Users->find();
$query->find('list');
```
通常であれば最初の `find()` はいらないのですが、「チェインしたときにどのような挙動をするのか」により興味があるため、あえて入れてみます。

```php
public function find($type = 'all', $options = [])
{
    $query = $this->query();
    $query->select();

    return $this->callFinder($type, $query, $options);
}
```
findメソッドは、このような流れになっています。  
引数を省略した「普通のfind」は、 `all` というタイプでの呼び出しを意味することになります。

その次に呼ばれる `callFind()` こそ、まさに「ファインダーメソッドが定義済みであるか」を検査する実態です。  
もし存在するようであれば、そのまま実行します。

```php
public function callFinder($type, Query $query, array $options = [])
{
    $query->applyOptions($options);
    $options = $query->getOptions();
    $finder = 'find' . $type;
    if (method_exists($this, $finder)) {
        return $this->{$finder}($query, $options);
    }
// 後略
```

参考までに、`findAll()` は以下のように定義されています

```php
public function findAll(Query $query, array $options)
{
    return $query;
}
```

さて、問題は次です。メソッドチェーンを用いて呼び出した場合、find()はどのような処理になるのでしょう？  
注意深く考えると、`Table` だけでなく `Query` にも 同名のメソッドが生えていることに気づきます。  
`\Cake\ORM\Query` 版の`find()`は、以下のようになります。

```php
public function find($finder, array $options = [])
{
    /** @var \Cake\ORM\Table $table */
    $table = $this->getRepository();

    return $table->callFinder($finder, $this, $options);
}
```

queryインスタンスの中から、その出生元であるtableインスタンスにアクセスをして、先に見た `callFinder()` の流れに合流するのでした。その中で、 `$query` として自分自身(`$this`)を渡しています。

これによって、チェイン処理の完成です！！

## まとめ
カスタムファインダーは、利用していくとコントローラーの見通しが抜群に良くなるように思います。ほとんど「手続きを記した」、自然文に近いような形で処理の手順を記述していきやすくなるからです。

もちろん、これはBehaviorやTraitに組み込むことで活用の幅が格段に増えていきます。
フレームワークのポテンシャルを活かして、楽しみながら本質的な開発に取り組めるようになることを望みます。