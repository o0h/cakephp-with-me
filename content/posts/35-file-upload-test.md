---
title: "Fileのアップロードに関する結合テストをどうするか？"
date: 2019-05-18T19:11:11+09:00
tags: [Tests]
categories: ["CakePHP3"]
---

## 概要

Controller のテストにおいて、IntegrationTestTrait(IntegrationTestCase)の `get` / `posts` といったメソッドを利用する機会は多いと思います。  
この「POST や PUT のリクエスト」において、ファイルアップロード処理についてはどのように扱うべきでしょうか？  
`post()`に渡すデータ = リクエストボディとなるデータとは別に、`configRequest()`によるリクエストコンテキストへの `$_FILES` へのアップロードファイルの注入が必要です。

<!--more-->

## イントロ

CakePHP のコントローラーのテストは、IntegrationTestTrait(以前は IntegrationTestCase)を利用することで、「任意の HTTP メソッドで」「特定の URL に、query や payload 込みでリクエストした場合」をエミュレーションし、かつ「Response Header」「ResponseBody(描画される内容や view 変数にセットされた値)」を検査するテストが実装できます。  
その内容は、CakeBook の[コントローラーの統合テスト](https://book.cakephp.org/3.0/ja/development/testing.html#integration-testing)に詳しいです。

これは内部的には ServerRequest クラスとうまく協調することで実現されていますので、「ファイルのアップロード」に関しても再現することが可能です。  
これは Book 上では詳細な言及がなかったので、調べてみました。

## IntegrationTestTrait での「ファイルのアップロード」

### IntegrationTestTrait を利用した基礎的なリクエストの例

IntegrationTestTrait での「リクエストのテスト」のためのメソッドには、以下のようなものがあります。

- get()
- head()
- options()
- patch()
- post()
- put()

[Trait Cake\\TestSuite\\IntegrationTestTrait \| CakePHP 3\.7](https://api.cakephp.org/3.7/class-Cake.TestSuite.IntegrationTestTrait.html#_patch)
このうち `patch` `post` `put` は第 2 引数をとり、リクエストコンテンツ(BODY)を渡すことが可能です。  
ざっくりといって、 PHP でいえば `$_POST` に入ってくる内容をセットできるのだとイメージしてください。

```php
$this->post('/api/post/add.json', $data);
```

### ServerRequest と \$\_FILE

ここで、少しだけ詳しく「`post()` の中で何が行われているのか」を見てみましょう。  
簡単にまとめると

1. リクエスト内容を組み立てて
2. それを `ServerRequest` に変換し
3. `ServerRequest` を利用して、実際のリクエスト処理の実行(`Server` = Middleware のスタックを順に処理する)
4. 実行結果をセットする

という内容が取り扱われています。

今回の関心領域で言えば、「`ServerRequest` はどのように組み立てられるのか？」という部分になります。  
それをサマリーとして表したのが、以下のシーケンス図です。

![](/images/posts/2019-06-07-13-50-16.png)

ここから、 `ServerRequest` インスタンスは`ServerRequestFactory::fromGlobals()` というメソッドを介して取得されていることがわかります。

`ServerRequestFactory::fromGlobals()` は、「 PHP のスーパーグローバル変数(`$_FILES`, `$_COOKIE`, `$_GET`, `$_POST`)と明示的に渡された各引数を合成して、 `ServerRequest`(PSR-7)のインスタンスを組み立てる、というものです。

```php
public static function fromGlobals(
    array $server = null,
    array $query = null,
    array $body = null,
    array $cookies = null,
    array $files = null
)
```

API: [ServerRequestFactory::fromGlobals\(\)](https://api.cakephp.org/3.7/source-class-Cake.Http.ServerRequestFactory.html#30-60)

さて、「 `IntegrationTestTrait`の各種リクエストメソッドは、 HTTP メソッド名・リクエスト先 URL 情報、リクエストボディをとる」ということを想起してください。  
すなわち `$server` `$cookies` `$files` は、別の手段を用いて渡して上げる必要がありそうです。

### IntegrationTestTrait::configRequest()

ここで登場するのが、 `IntegrationTestTrait::configRequest()`というメソッドです。  
これは、「次に実行されるリクエストのエミュレーションの内容を設定する」というものです。

```php
    /**
     * Configures the data for the *next* request.
     *
     * This data is cleared in the tearDown() method.
     *
     * You can call this method multiple times to append into
     * the current state.
     *
     * @param array $data The request data to use.
     * @return void
     */
     public function configRequest(array $data)
```

例えば、CakeBook を参照すると「リクエストヘッダーをセットする」といった用途で利用する例を紹介しています。

```php
// ヘッダーの設定
$this->configRequest([
    'headers' => ['Accept' => 'application/json']
]);
```

Book: [リクエストの設定](https://book.cakephp.org/3.0/ja/development/testing.html#id22)

このメソッドに渡された `$data` は、一時的に TestCase クラス(IntegrationTestTrait)のインスタンスメンバとして保持されることになり、最終的に dispatcher に渡される「リクエスト情報」として URL やリクエストボディとマージされることになります。

いいかえると、`configRequest()` に `files` というキーで渡された値が、 `ServerRequestFactory::fromGlobals()` に `$files`(第 5 引数)として渡されていくことになります。

これで準備が整ったので、実際にファイルアップロードを試してみましょう。

### 結合テストでファイルアップロードを試す

やることは 2 つです。

1. 実際に `$_FILES` に入ってくるような形式の連想配列を用意する
2. その内容を、 `configRequest()` と `post()` に同一のキーで渡す

#### データ形式

利用する値は、 `error` `tmp_name` `size` というキーを持つ連想配列である必要があります。  
これは、`IntegrationTestTrait`がリクエストデータを組み立てるときに簡易なバリデーションを行っていて、その条件を満たさなければ「配列でなくてスカラデータとして扱えるように変換する」という機構になっているためです。  
非常にシンプルな実装なので、実コードを示します。

```php
if (is_array($value)) {
    $looksLikeFile = isset($value['error'], $value['tmp_name'], $value['size']);
    if ($looksLikeFile) {
        continue;
    }
    $data[$key] = $this->_castToString($value);
}
```

Source: [IntegrationTestTrait](https://github.com/cakephp/cakephp/blob/cedcb681b3e383dfa272956b002a0448aa4443c6/src/TestSuite/IntegrationTestTrait.php#L701-L720)

#### コードサンプル

```php
//$srcはCake\Filesystem\Fileのインスタンスなどを想定しています
$file = [
    'error' => 0,
    'name' => $src->name,
    'size' => $src->size(),
    'tmp_name' => $src->path,
    'type' => $src->mime(),
];
$this->configRequest([
    'Content-Type' => 'multipart/form-data',
    'files' => [
        'field-name' => $file,
    ]
]);

$this->post(
    '/api/add',
    [
        'field-name' => $file,
    ]
);
```

このコードがあれば、`ServerRequest::getUploadedFile('field-name')` (コントローラー内部からであれば `$this->getRequest()->getUploadedFile('field-name')`)によるファイルへのアクセスを伴うコードもテストが可能になります。

API: [getUploadedFile\(\)](https://api.cakephp.org/3.7/source-class-Cake.Http.ServerRequest.html#2215-2229)

## まとめ

ファイルアップロードのテストはやりづらさを感じる場面が多いものと思いますが、CakePHP ではこうすることでグローバル変数を汚染することなく検査ができることがわかりました。  
少々の冗長さも感じるので、もしファイル関連の操作を頻繁に行うアプリケーションであるなら、TestSuite やそれに類するレイヤーに「ファイルのアップロードのテストを行う」ためのヘルパーを用意しても、便利かもしれません。
