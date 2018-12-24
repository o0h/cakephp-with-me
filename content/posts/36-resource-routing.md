---
title: "Resource Routing"
date: 2018-12-25T00:00:06+09:00
tags: [Http/Route]
categories: ["CakePHP3"]
---
※1人AdventのDay-24です

[1人advent\(CakePHP中心、PHP開発よもやま\) Advent Calendar 2018 \- Adventar](https://adventar.org/calendars/3627)

---
## 概要
CakePHPのRouterには、ある名前のリソースに対してRESTfulなアクセスを簡単に提供する機能 `resources()` があります。  
その内部実装がどのようになっているのか、処理を追ってみました。

## イントロ
CakePHPは伝統的に設定より規約を重んじるフレームワークであり、Routingにもその色が濃く出ています。  
CakePHP2の時代と違い、交換・取り外し可能にはなりましたが、fallbackを用いて「コントローラー/アクション」を自動的にマッピングし、起動する機構は健在です。  

そうして、CakePHPにおいてはRoutingとは「書かなくても動くもの」としての性質がありますが、その方向性を更に強化するのが`Resource Routing`だと感じます。  
Resource Routingは、対象の「リソース」の名前を指定することで、RESTfulなエンドポイントを仕立てる機能です。

Bookにある例を引用し、概要の紹介とします。

[ルーティング \- 3\.7](https://book.cakephp.org/3.0/ja/development/routing.html#restful)

```php
// config/routes.php 内で...

Router::scope('/', function ($routes) {
    // 3.5.0 より前は `extensions()` を使用
    $routes->setExtensions(['json']);
    $routes->resources('Recipes');
});
```

| HTTP format | URL.format | 対応するコントローラーアクション |
| ---- | ---- | ---- |
| GET | /recipes.format | RecipesController::index() |
| GET | /recipes/123.format | RecipesController::view(123) |
| POST | /recipes.format | RecipesController::add() |
| PUT | /recipes/123.format	| RecipesController::edit(123) |
| PATCH  | /recipes/123.format | RecipesController::edit(123) |
| DELETE | /recipes/123.format | RecipesController::delete(123) |

この記事では、「Resource Routingを使ったときに、内部的には、具体的に何が起きているのか」を見ていきます。

## source
まずは、単純にソースコードを貼ってみます。

https://github.com/cakephp/cakephp/blob/3.7.1/src/Routing/RouteBuilder.php#L389

```php
public function resources($name, $options = [], $callback = null)
{
    if (is_callable($options)) {
        $callback = $options;
        $options = [];
    }
    $options += [
        'connectOptions' => [],
        'inflect' => 'underscore',
        'id' => static::ID . '|' . static::UUID,
        'only' => [],
        'actions' => [],
        'map' => [],
        'prefix' => null,
        'path' => null,
    ];

    foreach ($options['map'] as $k => $mapped) {
        $options['map'][$k] += ['method' => 'GET', 'path' => $k, 'action' => ''];
    }

    $ext = null;
    if (!empty($options['_ext'])) {
        $ext = $options['_ext'];
    }

    $connectOptions = $options['connectOptions'];
    if (empty($options['path'])) {
        $method = $options['inflect'];
        $options['path'] = Inflector::$method($name);
    }
    $resourceMap = array_merge(static::$_resourceMap, $options['map']);

    $only = (array)$options['only'];
    if (empty($only)) {
        $only = array_keys($resourceMap);
    }

    $prefix = '';
    if ($options['prefix']) {
        $prefix = $options['prefix'];
    }
    if (isset($this->_params['prefix']) && $prefix) {
        $prefix = $this->_params['prefix'] . '/' . $prefix;
    }

    foreach ($resourceMap as $method => $params) {
        if (!in_array($method, $only, true)) {
            continue;
        }

        $action = $params['action'];
        if (isset($options['actions'][$method])) {
            $action = $options['actions'][$method];
        }

        $url = '/' . implode('/', array_filter([$options['path'], $params['path']]));
        $params = [
            'controller' => $name,
            'action' => $action,
            '_method' => $params['method'],
        ];
        if ($prefix) {
            $params['prefix'] = $prefix;
        }
        $routeOptions = $connectOptions + [
            'id' => $options['id'],
            'pass' => ['id'],
            '_ext' => $ext,
        ];
        $this->connect($url, $params, $routeOptions);
    }

    if (is_callable($callback)) {
        $idName = Inflector::singularize(Inflector::underscore($name)) . '_id';
        $path = '/' . $options['path'] . '/:' . $idName;
        $this->scope($path, [], $callback);
    }
}
```

ざっと眺めると、いくつかのポイントが有ることがわかります。

1. `$name`パラメータは、対応するコントローラーの指定とpathの指定に利用されていること
2. 第二引数でいくつかのオプションをとること
3. 加えて、callbackが指定できること
4. 最終的には、クラスメンバ及びoptionで渡された`$_resourceMap`に従い、`scope()`を設定していること

原理を見れば単純なもので、これによって、あの「パワフルなRESTfulルーティング」が実現されています。  
というわけで、設定の方法と処理内容について見ていきます。


## $options
デフォルトオプションは、以下の項目になっているようです。

```php
[
    'connectOptions' => [],
    'inflect' => 'underscore',
    'id' => static::ID . '|' . static::UUID,
    'only' => [],
    'actions' => [],
    'map' => [],
    'prefix' => null,
    'path' => null,
]
```

`connectOptions` は、最終的にはいくつかのデフォルト項目とマージされて`$this->connect()`メソッドの第3引数に渡されます。  
例えば `action` `_ext` `_name` `routeClass` `_middleware` といった項目が、ここから注入可能ということです。

`inflect` は、名前の通りURLの「語形変化」について設定します。・・・ここだとデフォルトが`underscore`なんですね。もしチェインケースでのURLを生成する場合は、 `dasherize` を指定することで実現されます。

`id` はクラス定数を参照していますが、以下のように定義されています。

```php
/**
 * Regular expression for auto increment IDs
 *
 * @var string
 */
const ID = '[0-9]+';

/**
 * Regular expression for UUIDs
 *
 * @var string
 */
const UUID = '[A-Fa-f0-9]{8}-[A-Fa-f0-9]{4}-[A-Fa-f0-9]{4}-[A-Fa-f0-9]{4}-[A-Fa-f0-9]{12}';
```
これは、例えば `articles/:id` (GETアクセス・view()アクション)時に「許容する」文字列のパターンをどうするかという話になります。デフォルトだと、インクリメントを想定して数字のキーかUUIDベースでのIDが想定されていることになります。  
「screen_id」のようなリソース識別子を用いたアクセスを提供する場合など、自分で注入する必要があるでしょう。

`only`は「どのアクションに対してResource Routingを提供するか」を指定するリストで、例えば「一覧と個別表示だけ許可」といった場合は`['only' => ['index', 'view']]` といった形になるのではないでしょうか。  

`actions` は、規定の「操作とactionの紐づけ」を変更するための指定です。  
例えば、元々は `['index' => ['action' => 'index']]`というように指定されています。同様に、index/create/view/update/deleteの5種類の「操作」に対して、「規定のaction」が設定されているわけです。  
これについて、「index操作はlist()メソッドで」「create操作はgenerate()メソッドで」というような、上書き設定を注入することができます。  
これ自体は「mapの再指定」ではなく、あくまで「対応の書き換え」として動作することに注意してください。すなわち、「actionsで指定のないものは提供されない」のではなくて、「元々mapで規定されているものを利用する」ことになります。

`map`は、「提供する操作の一覧・定義」をするものです。これはRouteBuilderがもつクラスメンバ`$_resourceMap`とマージされる形で利用されます。  
mapの基本的な形として、`RouterBuilder::$resourceMap` を引用します。

```php
protected static $_resourceMap = [
    'index' => ['action' => 'index', 'method' => 'GET', 'path' => ''],
    'create' => ['action' => 'add', 'method' => 'POST', 'path' => ''],
    'view' => ['action' => 'view', 'method' => 'GET', 'path' => ':id'],
    'update' => ['action' => 'edit', 'method' => ['PUT', 'PATCH'], 'path' => ':id'],
    'delete' => ['action' => 'delete', 'method' => 'DELETE', 'path' => ':id'],
];
```
これをforeachで回しながら、`connect()`に引き渡す形です。iterateされたvalueが、`connect()`の第二引数へと渡されます。ただし、すべての値がそのまま渡されるわけではないので注意が必要です。適用されるのは、`action` `path` `method`の3項目になります。また、`$params['action']`に関しては、先述の`$options['actions']`で宣言されている内容が優先されることになります。  
また、もしmap内で`path`を省略した場合は、暗黙的に操作名を`path`に割り当てるような挙動をします。

```php
foreach ($options['map'] as $k => $mapped) {
    $options['map'][$k] += ['method' => 'GET', 'path' => $k, 'action' => ''];
}
```

`prefix` `path` は、その名の通り定義するルーティングの設定内容です。最終的には、`/$prefx/$options['path]/$params['path']/current($map)['path']` という形のURLが構築されます(prefixだけ、connect()の第1引数でなく第3引数で設定されることに注意

## $callback
resources()メソッドは第3引数に`callback`を取ることができます。また、第2引数にcallableを渡し第3引数を省略した場合にも動作が可能です。

```php
if (is_callable($options)) {
   $callback = $options;
   $options = [];
}
```
これは何でしょうか？  
(通常のroutingの設定で見るような)`scope()` に直接コールバックを渡すような形で、RESTfulなルーティングを構築するようになります。  

例えば、サンプルとして以下のような例がありました。

```php
/**
  * You can create nested resources by passing a callback in:
  *
  * $routes->resources('Articles', function ($routes) {
  *   $routes->resources('Comments');
  * });
  *//
```

これは、実際にどのような routingを構築するでしょう？試してみました。

```sh
 bin/cake routes
+-----------------+------------------------------------+-----------------------------------------------------------------------------------+
| Route name      | URI template                       | Defaults                                                                          |
+-----------------+------------------------------------+-----------------------------------------------------------------------------------+
| articles:index  | /articles                          | {"_method":"GET","action":"index","controller":"Articles","plugin":null}          |
| articles:add    | /articles                          | {"_method":"POST","action":"add","controller":"Articles","plugin":null}           |
| articles:view   | /articles/:id                      | {"_method":"GET","action":"view","controller":"Articles","plugin":null}           |
| articles:edit   | /articles/:id                      | {"_method":["PUT","PATCH"],"action":"edit","controller":"Articles","plugin":null} |
| articles:delete | /articles/:id                      | {"_method":"DELETE","action":"delete","controller":"Articles","plugin":null}      |
| comments:index  | /articles/:article_id/comments     | {"_method":"GET","action":"index","controller":"Comments","plugin":null}          |
| comments:add    | /articles/:article_id/comments     | {"_method":"POST","action":"add","controller":"Comments","plugin":null}           |
| comments:view   | /articles/:article_id/comments/:id | {"_method":"GET","action":"view","controller":"Comments","plugin":null}           |
| comments:edit   | /articles/:article_id/comments/:id | {"_method":["PUT","PATCH"],"action":"edit","controller":"Comments","plugin":null} |
| comments:delete | /articles/:article_id/comments/:id | {"_method":"DELETE","action":"delete","controller":"Comments","plugin":null}      |
+-----------------+------------------------------------+-----------------------------------------------------------------------------------+
```
articlesと、その単一リソースの下に更にcommentsへのアクセスが提供されているのがわかります。  
更に、この場合に`article_id` `comment_id` のいずれにも、コントローラーのアクション内からアクセス可能になっているのが気の利いている部分です。
例えば、`GET articles/2/comments/12` にアクセスした場合に、`$this->getRequest()->params`の内容は次のようになります。

![](/images/posts/2018-12-25-04-13-37.png)

まさに、「自動的にRESTfulなroutingを提供してくれる」といった体感です。


## まとめ
Resource Routingは、機能自体は知っていて一部利用したことがあります。しかしながら、あまり使いこなせているという実感がなく、「なんとなく」で利用していました。  
そこで、今回コールドリーディングをしてみた次第です。

使いこなそうとすると、やや癖がありそうなものの、柔軟で開かれた機能だなぁという印象を受けました。  

便利なのであれば使わない手もないし、内部構造を知れたことで今後はより「無駄のない設定」をしつつ付き合っていけそうに思います。