---
title: "Behaviorを使うか、Traitにするか"
date: 2018-12-16T16:04:42+09:00
tags: [ORM-and-Database]
categories: ["CakePHP3"]
---
※1人AdventのDay-16です

[1人advent\(CakePHP中心、PHP開発よもやま\) Advent Calendar 2018 \- Adventar](https://adventar.org/calendars/3627)

## 概要
CakePHP3ではBehaviorを利用することで、Tableクラスにmixinすることができます。また、PHPにはTraitの仕組みがあり、これを利用することで継承を用いずにメソッドやプロパティの再利用を実現することができます。  
現時点で考えている、個人的な「どう使い分けるか」というポイントをまとめてみます。

## イントロ
CakePHPには3.xより前のバージョンから、Modelレイヤーに「Behavior」という機構があります。複数のModel(Table)クラスで同様に利用されるような機能を、Modelを通じて透過的に利用できるようにしようというものです。これを利用することで、呼び出し側は、その機能がModelクラスに直接所属しているのかBehaviorから提供されているかを全く意識することなく利用ができます。  
例えば、 `TreeBehavior` などはCakePHPにおいて過去から継続して提供されているユニークな例の1つでしょう。 `$Model->find('children', ['for' => 100])` などとすれば、「アイテム100番に属する子アイテムを取得する」という操作が可能です。

一方で、PHP5.4からTrait機構が言語から提供されています。  
CakePHP2.xはPHP5.3.0での利用を前提としているためにTraitの利用は選択肢に含まれませんが、CakePHP3.xにおいては状況が異なります。「Traitを利用して、Tableクラスに直接メソッドを植え付ける」ことが可能になりました。  
単純なメソッド提供だけに絞れば同等の利用ができそうな両者は、どのように共存していけばよいのでしょうか？

## 個人的なシンプルな結論
個人的には、「Traitで済むものはTraitで良いのではないか」と考えるようにしています。その範疇外のものや、出来なくはないが考慮すべきことが増える場合などにはBehaviorを利用します。

## PHPにおけるTraitの位置付け
Traitが言語から提供されるようになってから、「どう使うか」というのは議論の的です。とりわけ、Abstract/Interfaceと交えて「デザインの仕方が大きく変わりそう」という関心を集めているように感じます。

必ずしもTrait(のみ)の話ではないですが、個人的な「Classの役割が狭まり、Traitの利用箇所は多岐に渡って拡大していくのではないか」という考え方は、こちらのエントリーにインスパイアされた面があります。

[PHP 7 の無名クラスから考えるクラスの在り方 \- Shin x Blog](https://blog.shin1x1.com/entry/anonymous-class-change-class-in-php7)

今回は無名クラスの話ではないので、スコープが異なるのですが、それでも「Traitでやれること─とりわけ、具体実装についてはTraitを利用していく」という意味で合致しています。  

## Traitで済むこと
「メソッドの提供」のみであれば、Traitでもできそうです。また、visible/non-visibleなクラスメンバ・インスタンスメンバを利用する場合でも、traitは十分に活用できます。

例えばカスタムファインダメソッドの提供などは、Traitでも行けそうです。  
「Queryインスタンスが渡されてきて」「呼び出し時に$optionを受け取ることができる」というインターフェイスにすれば問題なく動作します。その中に、formatResults()やwhere()/order()のデコレーションを実装することができるし、面倒くさいQueryExpressionを内部で組み立てて上げることも出来るわけです。

こう考えると、従来的なBehaviorの役割は、その多くをTraitに託すことも可能なのではないでしょうか？

## Behaviorを使う場面①: ライフサイクルへの
CakePHPにはModelの利用に関するライフサイクルが存在し、イベントベースで処理を割り込ませることが可能です。  
[コールバックのライフサイクル](https://book.cakephp.org/3.0/ja/orm/table-objects.html#table-callbacks)

これらを活用する場合、Bahaviorでの機能提供を行ったほうが無難かと考えています。  
Bahaviorでは、各イベントの発火時にTableクラス自体とBehaviorが持つそれぞれの処理が考慮されるようにデザインされています。例えば、Behaviorに `beforeFind`メソッドをもたせた場合、`Model.beforeFind` にぶら下げて実行させる事は容易です。
他方で、Traitに定義された `beforeFind()` メソッドは、それを利用するTable自体の `beforeFind()` と見分けが付きません。そのため、「TableとBehaviorの`beforeFind()`を明示的に双方とも呼び出す」ことが必要になります。これは、実装時にうっかりと忘れてしまいそうではないでしょうか・・?

そういう訳で、もし「ライフサイクルに則ってイベントを購読する」ことが重要な場合は、Behaviorを利用したほうがストレスなく開発を勧めていけるのではないかと思います。もちろん、具体的な実装をTraitに持たせつつ、Tabelの内部のメソッド(例えば`initialize()`や`beforeMarshal()`)、implemetedEventsに登録されているメソッドの内部からTraitの処理を呼び出す」といった方法は可能です。

## Behaviorを使う場面②: 設定値の管理
Behaviorの中での何らかの設定値を扱う場面もあります。その際に、 `ConfigTrait`を利用するなど、インスタンス内部に `$_config` というプロパティを作成することで管理するでしょうか。

もしTraitで同様のことを行おうとすると、利用側となるTableの持つプロパティと区別をできずに競合する可能性があります。  
これを無難に回避するために、設定や状態管理を伴うある程度複雑な機能については、Behaviorとして「別クラス」を定義してしまうほうが良いのかなと感じます。

## Behaviorを使う場面③: (静的)プロパティの利用・共有
設定値の管理の変形バージョンといえる話かもしれません。  
ある機能を専任的に扱う存在としてのBehaviorを考える場合、「他のクラスに対しても影響をもたらす値の管理」を行いたい場合があります。  

例えば「管理者モードを扱えるようにするBehavior」において「一時的に非管理者モードでのアクセスを行う」ような実装を考えていたとします[^1]。この状態切り替えを、すべて「読み込み時に完了できる」もしくは「操作実行時に、都度参照できる」ようになれば、問題なくモードの切替が達成できると思います。  
後者のアプローチを考える場合、Behaviorなら「自身の中に値を保持する」「それを都度参照するようにする」ことが可能です。各Tableからみて、「1箇所だけ見れば済む」ようになりました。他方で、Traitでは「自身の中に持つ」という状況を作り出すのが困難です。Traitの中にプロパティを生やすことはもちろん可能ですが、それは「各Tableに複製される」ようになります。すなわち、どれか1つを変更したところで、他Tableの手が届くところへの影響はあません。

このようなケースに置いては、Behaviorを扱うのが良いのかな？と思います。

[^1]: モードの状態管理をModel側で行いたい需要があるかは不明ですが・・あくまで、この場限りの説明として

## まとめ
今回の内容は、あくまで今の時点における個人的な考え方です。  
CakePHPのコアコード内で、ORM\TableにおいてはBaahaviorの投入例がないので、フレームワーク側の設計思想を読み取るにはいたりませんでした。
そうした中で、「Traitの活用方法は」という原則論に立ち返ることで、「Traitで可能な範囲の具体的な実装は、Traitにまかせてしまってよいのでは」という様に考えています。  
Behaviorの機能やその呼び出され方、協調の仕方を学ぶことで、すっきりとした実装の基準が描いていければいいなと思います。