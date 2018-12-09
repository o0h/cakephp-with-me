---
title: "CakePHP 3.7の個人的な見どころ"
date: 2018-12-09T22:34:12+09:00
tags: [ver-up]
categories: ["CakePHP3"]
---
※1人AdventのDay-9です

[1人advent\(CakePHP中心、PHP開発よもやま\) Advent Calendar 2018 \- Adventar](https://adventar.org/calendars/3627)

---

晴れて、CakePHPの3.7がリリースされました。  
[CakePHP 3\.7\.0 Released — Bakery](https://bakery.cakephp.org/2018/12/08/cakephp_370_released.html)

ここしばらく、「4へのスムーズな移行をするために」を意識し続けてきたCakePHPコミュニティです。その様子は、別の場所に自分なりの観点でまとめてみています。

* [今からちょっとだけ先の未来、CakePHP4の話 〜Upcoming CakePHP Roadmap & Releases〜 \- コネヒト開発者ブログ](http://tech.connehito.com/entry/roadmap-for-cakephp-4)
* [CakePHP3\.6\.0のbeta1が出たのでおさらいしてみる \- コネヒト開発者ブログ](http://tech.connehito.com/entry/akephp-36-beta1)

そして、本来であれば「出さずに済ませたかった」とも言える3.7であり、これが3系のファイナルバージョンとなるはずです。

リリースノートと移行ガイドから、その内容を読み取ってみます。  
主観により取捨選択しているので、詳細は原文を参照してください。

* [CakePHP 3\.7\.0 Released — Bakery](https://bakery.cakephp.org/2018/12/08/cakephp_370_released.html)
* [3\.7 Migration Guide \- 3\.7](https://book.cakephp.org/3.0/en/appendices/3-7-migration-guide.html)

----

## CakePHP3.xの最終バージョン
>This release is the last planned feature release for 3.x. Going forward the core team will be focusing on supporting 3.7 and completing 4.0.0.

とされています。  
元々は3.6に全うさせる予定だった役割の一部を、3.7に託したのかなという印象があります。

## 振る舞いの変更
以降に際して注意すべきはこの項目なのかな、と思っています。

### Database/Datasource/ORM
* `IntegerType` にあたるカラムに非numericな値が来た場合に例外を投げるように
    * ref: https://github.com/cakephp/cakephp/pull/12543
* bool/int/float/decimal typeへのマーシャルの際に、非スカラ値はnullになるように
    * ref: https://github.com/cakephp/cakephp/pull/12478

### その他
* 5xx系例外で、status codeによってカスタムエラーハンドリング処理に入れなかったのを修正
    * ref: https://github.com/cakephp/cakephp/pull/12501
    * 今までは `$code < 506` に限ってカスタム処理に入っていた
* `Router::url()` で_methodが未指定ならGETを取るように
    * ref: https://github.com/cakephp/cakephp/pull/12580/
    * デフォ値が補完されるように
 
## 新機能
* `ArrayEngine` Cacheの提供
    * 「永続化したくない、メモリに乗せるだけなキャッシュ」
    * ストレージを使って副作用を残したりはしたくないけど、キャッシュ機能を利用するニーズが有る場面を想定
        * _this engine can be useful in tests or console tools_
    * ref: https://github.com/cakephp/cakephp/pull/12741
*  prefix付きのErrorControllerの利用が可能に
    * 例えば `App\Controller\Admin\ErrorController` みたいな
    * ref: https://github.com/cakephp/cakephp/pull/12396
* `ErrorHandlerMiddleware` がpreviousを扱えるように
    * ref: https://github.com/cakephp/cakephp/pull/12673
    * ref: https://github.com/cakephp/cakephp/pull/12738
* `Cake\Http\Client` が、拡張が存在していればデフォルトで `curl`拡張を利用するように
    * ref: https://github.com/cakephp/cakephp/pull/12209
* `$entity->hasErrors()` メソッドの追加
    * 思ったより色々やってる。「なるほど、確かに」と思ったPR
    * ref: https://github.com/cakephp/cakephp/pull/12327
* `IntegrationTestCase` にアサーションメソッドの追加
    * ref: https://github.com/cakephp/cakephp/pull/12464 https://github.com/cakephp/cakephp/pull/12136 https://github.com/cakephp/cakephp/pull/12064 あたり！
* `IntegrationTestCase` 周りのリデザ
    * constraintを利用するように
    * https://github.com/cakephp/cakephp/pull/12072/files
* `Validation` で空値の扱いに関するあれこれ
    * `allowEmptyString`, `allowEmptyArray`, `allowEmptyDate`, `allowEmptyTime`, `allowEmptyDateTime`, `allowEmptyFile`
    * ref: https://github.com/cakephp/cakephp/pull/12696
* `ViewBuilder` から viewVarを扱いやすく
    * 今までは `ViewBuilder::build()` の第1引数でしかアクセスできていなかった確か
        * メンバへの直アクセスがdeprecatedなため
    * ref: https://github.com/cakephp/cakephp/pull/12465/files

## Deprecations
3.6で概ね潰してきたとはいえ、まだ結構ありますね。  
やはり、機能追加よりもこちらの「廃止予定をはっきりさせる」「get/setを分離する」が、引き続き3.7の主命題に思います。