---
title: "コレクションを任意の順番で並び替える"
date: 2018-03-25T20:35:59+09:00
tags: ["tips", "Utility", "Collection"]
categories: ["CakePHP3"]
---
# コレクションを任意の順番で並び替える

PHPを使っていると、「任意に順番を並び替えたキーに沿って、連想配列を並び替えたい」ということが意外と面倒くさかったりする。  
MySQLでいう ORDER BY FIELD(field, idx1, idx2, idx3) みたいなやつ。

Collectionクラスを用いると簡単にできそうだな〜と思ったのでメモ。

```php
$order = [2, 1, 3];
$data = collection([
    ['id' => 1, 'name' => 'John'],
    ['id' => 2, 'name' => 'Alice'],
    ['id' => 3, 'name' => 'Yui'],
]);
$orderMap = array_flip($order);
$data = $data->sortBy(
    function ($datum) use ($orderMap) {
        return $orderMap[$datum->id];
    },
    SORT_ASC
    )
    ->compile(false);
```

何をしているかというと、

1. array_flipでもとの並び順を覚えさせて
2. 対応順にarrayに位置を教えてあげる

というだけのもの。


Collectionクラスは、以前にQiitaにも書いた。  
とても便利で強力なUtilityなので、使いこなしたい。

[\[CakePHP3\]現場で使えるCollectionクラスの15選 \- Qiita](https://qiita.com/o0h/items/26f582986aec1ac2a67f)
