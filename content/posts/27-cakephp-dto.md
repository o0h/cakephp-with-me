---
title: "dereuromark/cakephp-dtoに触ってみる"
date: 2018-12-08T18:12:32+09:00
tags: []
categories: ["Plugins"]
---

※1人AdventのDay-8です

[1人advent\(CakePHP中心、PHP開発よもやま\) Advent Calendar 2018 \- Adventar](https://adventar.org/calendars/3627)

CakePHP開発者であるMark Sch.さんが、新しいプラグインを公開されていました。
<blockquote class="twitter-tweet" data-lang="ja"><p lang="en" dir="ltr">[New]dereuromark/cakephp-dto CakePHP DTO Plugin <a href="https://t.co/vBFe8DJUPE">https://t.co/vBFe8DJUPE</a></p>&mdash; function(){exit;} (@call_user_func) <a href="https://twitter.com/call_user_func/status/1071454532830253056?ref_src=twsrc%5Etfw">2018年12月8日</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

 名前の通り、CakePHPでDTOを扱うための実装のようです。  
 cf) [Data Transfer Object \- Wikipedia](https://ja.wikipedia.org/wiki/Data_Transfer_Object)

 おもしろそうなので、早速触ってみました。

### ざっくりいうと何？
* 決められたプロパティを持つmutable/immutableなオブジェクトを扱いやすくするためのもの
* 決められたプロパティ = 型は、設定ファイルに記述していく
* それらの設定を、実クラス生成コマンドによって作成する
* 実際のクラスを生成するからIDE上での保管や静的解析との相性が良い

### CakeDTOに触ってみる

#### setup
まずは、インストールです

```
composer require dereuromark/cakephp-dto:dev-master
```

Pluginを有効化します。[^1]

[^1]: なぜPSR-4準拠でないnamespaceで通るかというと、plugin-installerによってpath mappingがなされているからです。具体的には、vendor/composer/autoload_psr4.phpに `'CakeDto\\' => array($vendorDir . '/dereuromark/cakephp-dto/src')` が追加されています

```
bin/cake plugin load CakeDto -b
```

#### はじめてのDTO作成
ファイル初期生成用のコマンドが用意されています。
```
bin/cake dto init
```

これを実行すると、config/dto.xmlに以下のようなファイルが設置されます

```xml
<?xml version="1.0"?>
<dtos
	xmlns="cakephp-dto"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="cakephp-dto https://github.com/dereuromark/cakephp-dto">

</dtos>
```
・・・と言っておいて何ですが、個人的にYAMLでいきたいのでYAMLに書き換えます。
こちらのExampleを参考にしましょう。
[/examples/basic.dto.yml](https://github.com/dereuromark/cakephp-dto/blob/fbf2604e53ab800ed0a35ddf482b8d62548b48fa/docs/examples/basic.dto.yml)

##### YAML Engineの利用
(まず前提として、peclでyamlエクステンションをインストールしておいてください。)  

yamlのDTO定義を利用するには、configでデフォルトであるXmlEngineからYamlに切り替えるよう、指示をする必要があります。  
`config/app.php` にて、以下の3行を追記してください。
```
  'CakeDto' => [
        'engine' => CakeDto\Engine\YamlEngine::class,
    ],
```
`Configure::read('CakeDto.engine')` で読み取れる位置に設定値が入ればOKです。


##### DTOの定義
早速、DTOの定義の仕方を見ていきます。

```yaml
Car:
  fields:
    isNew: bool
    value: float
    distanceTravelled: int
    attributes: string[]
    manufactured: \Cake\I18n\FrozenDate
    owner: Owner
```
※ 一部、依存解消のためにexampleの内容を変更しています

「Car」という型が定義されました。  
プリミティブなタイプを使用できるのはもちろん、既存のクラスを利用できることと、配列を持つことができるのが見て取れます。  
これを、`config/dto.yml`に保存します。

##### クラスの生成
定義ファイルができたら、これを元にして実際のPHPクラスを生成します。

```sh
bin/cake dto generate
```
これだけです！
`src/DTO/CarDto.php` というファイル名で、以下の内容が生成されます。  
(長いのでリンク先をご確認ください)
[gist](https://gist.github.com/o0h/0cd23ead77e756ec1c1ad0caa92a8a66)

#### DTOに軽く触れてみる

「いちいち、コード生成をしなければならないのは手間ではないか」「実行時に設定ファイルを読み出して、簡単に使えるようにしてしまえばいい」という見方もあるかと思います。  

IDEから補完が効いて便利です。
![](/images/posts/2018-12-09-14-05-06.png)

コンストラクタに連想配列を渡してあげることで、値をそのままセットできます。
```
$ bin/cake console
You can exit with `CTRL-C` or `exit`

Psy Shell v0.9.9 (PHP 7.2.6 — cli) by Justin Hileman
>>> use App\Dto\CarDto;
>>> $car = new CarDto(['isNew' => false, 'value' => 2.5]);
=> App\Dto\CarDto {#2353
     +"data": [
       "isNew" => false,
       "value" => 2.5,
       "distanceTravelled" => null,
       "attributes" => null,
       "manufactured" => null,
       "owner" => null,
     ],
     +"touched": [
       "isNew",
       "value",
     ],
     +"extends": "CakeDto\Dto\AbstractDto",
     +"immutable": false,
   }
>>> $car->getValue()
=> 2.5
```

インスタンスへのミューテーション

```
>>> $car->setValue(3.2)
=> App\Dto\CarDto {#2353
     +"data": [
       "isNew" => false,
       "value" => 3.2,
       "distanceTravelled" => null,
       "attributes" => null,
       "manufactured" => null,
       "owner" => null,
     ],
     +"touched": [
       "isNew",
       "value",
     ],
     +"extends": "CakeDto\Dto\AbstractDto",
     +"immutable": false,
   }
>>> $car->getValue()
=> 3.2
```

#### もう少し触れてみる
次のオブジェクトを定義してみましょう。  
設定内で、「継承」と「イミュータブル化」が可能です。この機能を織り交ぜて、 `SmallCar` `FrozenCar` を定義してみます。

```yaml
SmallCar:
  extends: Car
  fields:
    countryCode: int

FrozenCar:
  immutable: true
  fields:
    isNew: bool
    value: float
    distanceTravelled: int
    attributes: string[]
    manufactured: \Cake\I18n\FrozenDate
    owner: Owner
```

```sh
$ bin/cake dto generate
Creating: SmallCar DTO
Creating: FrozenCar DTO
```

SmallCarの方には、Carの持つプロパティに加えて`countryCode`が定義されました。

![](/images/posts/2018-12-09-14-43-05.png)

immutableはどのような挙動になるでしょう。

こちらには、`setXxx()` メソッドが生えていないことがわかります。
![](/images/posts/2018-12-09-14-30-57.png)

そのかわりに、 `withXxx()` が提供されています。
![](/images/posts/2018-12-09-14-40-13.png)

その他にも、様々な機能が備えられています。  
詳細はレポジトリを参照してください。

* [README](https://github.com/dereuromark/cakephp-dto/tree/fbf2604e53ab800ed0a35ddf482b8d62548b48fa/docs) 
* [Example](https://github.com/dereuromark/cakephp-dto/tree/fbf2604e53ab800ed0a35ddf482b8d62548b48fa/docs)
    * 同様に、test appも参考に [test_app/src](https://github.com/dereuromark/cakephp-dto/tree/fbf2604e53ab800ed0a35ddf482b8d62548b48fa/tests/test_app/src)
* [Wiki](https://github.com/dereuromark/cakephp-dto/wiki)

### cakephp-dtoのモチベーション
「このプラグインを作ったのはなぜか？」「どうして、こういう実装になっているのか？」について、作者が明言をしています。
[Motivation and Background](https://github.com/dereuromark/cakephp-dto/blob/fbf2604e53ab800ed0a35ddf482b8d62548b48fa/docs/Motivation.md)

思想を知ることで、より豊かにソフトウェアを使えるようになると思いますので、個人的に特徴を感じた部分をかいつまんでみたいと思います。  
(翻訳をしているわけではなく、あくまで主観的な解釈を織り交ぜたまとめですのであしからず。)


#### 「連想配列で何でもやる」が辛い問題
* 複雑で、何層にもネストされた配列というのはとてもつらい
* 特にtemplateの中で「どのキーが(確実に)使えるのか？」というのは、知る術がない
* IDEがタイプヒントや補完をしてくれても・・
* 素朴な配列というのは、代入された要素が何の型か？というのを検証する術がない

この辺りに対するアプローチとして、DTO！

#### なぜコードを生成するの？
* 他のライブラリだと、設定ファイルを読み込んで実行時に生成したり決定したりしている。なぜ、このプラグインは事前定義が必要なのか？
    * IDEや静的解析で利用シーンを考えると、強力な利点がある。grepで利用箇所の検索が容易にできるし、静的解析を行う上でトリックのようなことも必要なくなる

#### Entityじゃだめなの？
* Entityというのは、名前空間が示すように強くORMと関連付けて考えられている。すなわち、永続化するというのも主要な目的の1つ。Entityにある「無用な部分」を削り、その上で「必要とされる部分」を付け足す。
    * 型制約の問題など(はDTOのほうが強力)
    * nullの使い方を明示的にしている
    * 現状のEntityに関する議論について、https://github.com/cakephp/cakephp/issues/11792 も参照

#### なぜ `Dto` サフィックスをつけているの？
* Entityとぶつかりやすくなるのを避けるため。明示的なサフィックスがあることで、コード中でなんのクラスであるかが判明しやすくなる

#### パフォーマンス問題はどうですか？
* 動作速度やメモリ消費について気になる
    * 「多層化された配列」の方が確かにずっと高パフォーマンスだと思う！
    * ただ、大抵のWebサービスのリクエスト処理に目を向けると、**開発スピード、コードの可読性というのは、些細な実行時パフォーマンスなんかよりずっと重要** である。

### 個人的な感想
もっと深く触れてみたい！という状況です。個人的には、値の管理や型定義については、強力な手法を用意できるとPHPプログラミングを格段に進化させていけると考えている立場です。  
個人的に開発を進めているライブラリも、最終的な目的が異なるものの、このプラグインのアプローチは非常に参考にしたいと感じる部分が多々ありました。そういう観点も込で、引き続き学んだりプラクティスを積み重ねていきたいプロダクトだなぁ、と思います。
