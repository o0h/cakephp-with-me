---
title: "CakePHP 3.7.2がリリースされたので更新内容を確認"
date: 2019-01-05T02:01:40+09:00
tags: [ver-up]
categories: ["CakePHP3"]
draft: true
---
## 概要
akePHP3.7.2がリリースされたので、その内容をチェックしてみます。  
3.7.1から2週間半振りののアップデートです.

23個のPR/issueがcloseされており、CSやTypeHint、docなどの軽微な修正に加えてbugfixが盛り込まれています。

<!--more-->

## イントロ
日付が変わりまして昨日の1月4日、CakePHPの3.7.2がリリースされました。

* [CakePHP 3\.7\.2 Released — Bakery](https://bakery.cakephp.org/2019/01/03/cakephp_372_released.html)
* [3\.7\.2 Milestone](https://github.com/cakephp/cakephp/milestone/187?closed=1)

実際に自分の関わっているアプリケーションでのupdate実施に備えて、今回の変更の内容を見ていきたいと思います。

## Bugfixes
このアップデートでは、新機能の実装は行われずにBugfixesのみとなっています。

### Consoleのサブコマンド名にスペースを利用できるように

```
Commands added in an Application or plugin console() hook method can now use spaces in the command name.
```
[Enable subcommands to use spaces\. by markstory · Pull Request \#12824 · cakephp/cakephp](https://github.com/cakephp/cakephp/pull/12824)

「これまではスペースを含む名前を使えなかったけど、使えるようにしたよ」「例えば `bin/cake bake:test`や`bin/cake bake.test`は動いたけど、`bin/cake bake test`は動かなかったでしょ」とのこと。  

・・・なのですが、これは「スペースを含むネストされたコマンド名」は、ドキュメントを見ると今までも使えているように見えるんですよね。

https://book.cakephp.org/3.0/ja/console-and-shells.html#index-0
![](/images/posts/2019-01-05-02-49-21.png)

これが紛らわしく思い実際に手元で動作確認をしてみたところ、3.7.1では上記コードは

```sh
bin/cake "user dump"
```
と、「1つのコマンド名」として認識できるようにしないとダメそうでした。  
3.7.2からは、(利用可能な区切り数は現実的な範囲で制限されているものの)ちゃんと、クオテーション等もなく動作させられそうです。

### IntegrationTestTraitでの$_FILESの扱いの改善
```
IntegrationTestTrait now forwards files defined in configRequest().
```
[Make PSR7 uploaded files accessible in integration tests by robertpustulka · Pull Request \#12833 · cakephp/cakephp](https://github.com/cakephp/cakephp/pull/12833)

### .html/.htm拡張子のURLにおけるエラーハンドリングの修正
```
Error pages for parsed extensions are once again rendering correctly.
```
[Fix handling of \`html\` and \`htm\` extensions\. by ndm2 · Pull Request \#12845 · cakephp/cakephp](https://github.com/cakephp/cakephp/pull/12845)

### FileEngineCacheの内部処理における無用なTTLチェック処理の排除
```
A redundant expiry time check in FileEngine was removed.
```
[Remove redundant check in FileEngine by markstory · Pull Request \#12839 · cakephp/cakephp](https://github.com/cakephp/cakephp/pull/12839)

元Issue -> [File cache engine: do not check duration on read · Issue \#12827 · cakephp/cakephp](https://github.com/cakephp/cakephp/issues/12827)  
これは結構、踏んでしまうと切なそうだ・・・

### RequestHandlerComponentを用いても拡張子付きリクエストがエラーレンダリング時にHTMLコンテンツを生成する問題

```
RequestHandlerComponent once again handles explicit htm and html extensions correctly.
```
[Fix \.json routes not rendering error pages properly by markstory · Pull Request \#12836 · cakephp/cakephp](https://github.com/cakephp/cakephp/pull/12836)

(これ、正に・・ちょうど今日踏んでたなぁ。。WAF側の問題だったのね。)

### `assertUrl()`メソッドで、CDN等の「ホスト違い」系のURLの検査に対応しやすく
```
UrlHelper::assetUrl() now supports a fullBase option enabling CDN assets to be linked more easily.
```
[Allow assetUrl\(\) to use CDNs easier\. by dereuromark · Pull Request \#12849 · cakephp/cakephp](https://github.com/cakephp/cakephp/pull/12849)

### Console/EmailのIntegrationTestTraitでもbefore/afterアノテーションを利用する
```
EmailTrait and ConsoleIntegrationTestTrait now uses before and after annotations to apply its setup/teardown logic.
```
[Trait annotations by jeremyharris · Pull Request \#12865 · cakephp/cakephp](https://github.com/cakephp/cakephp/pull/12865)  
cf: [Use annotitations for integration trait setup/teardown by markstory · Pull Request \#12822 · cakephp/cakephp](https://github.com/cakephp/cakephp/pull/12822)

## 感想
いつもより「バグ」っぽいバグが多いような気もしました。(FileCacheEngine, .htmlページエラーハンドリング, 非HTMLコンテンツへのExceptionRender)  
・・・元よりこんなもんですっけ。。。  
当然、期待通りに動いてもらえないのは「困る」わけで、他人が困っているように自分も・またその逆も・・というのが容易に想像できます。  
実際、「なんか変だなぁ気持ち悪いなぁ」と思いながらスルーしていた部分が今回のバージョンでパッチを当てられていたりしたので、これは裏を返せば「自分が貢献できたかもしれない」と感じる部分でもあります。  
通常業務とのバランスになっていきますが、やはり、気になった部分はコードを読んでいく姿勢が大事ですね。また、Slack#supportチャンネルなどでも非常にアクティブに対応してくれる印象があるので、「何かあったら気軽に聞いてみる(ただし誠実にやる！)」くらいの温度感を保っておくのがよいのかな〜とも思いました。