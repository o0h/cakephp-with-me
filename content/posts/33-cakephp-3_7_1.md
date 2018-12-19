---
title: "CakePHP 3.7.1がリリースされたので更新内容を確認"
date: 2018-12-19T23:25:17+09:00
tags: [ver-up]
categories: ["CakePHP3"]
---
※1人AdventのDay-19です

[1人advent\(CakePHP中心、PHP開発よもやま\) Advent Calendar 2018 \- Adventar](https://adventar.org/calendars/3627)

---
## 概要
CakePHP3.7.1がリリースされたので、その内容をチェックしてみます。  
今回は3.7.x系の初アップデートということもあり、コミュニティにおいて発見されたものを中心としたbugfixが多く見られました。

## イントロ
CakePHP3.7が、12月9日にリリースされました。  
その際には、以下の記事で言及しています。  
[CakePHP 3\.7の個人的な見どころ](https://cake.nichiyoubi.land/posts/28-cakephp-3_7/)

それから9日後の12月18日に、3.7.1がリリースされています。  

* bakery: [CakePHP 3\.7\.1 Released — Bakery](https://bakery.cakephp.org/2018/12/17/cakephp_371_released.html)
* GitHub Release: [Release CakePHP 3\.7\.1 released · cakephp/cakephp](https://github.com/cakephp/cakephp/releases/tag/3.7.1)
* GitHub Issues: [3\.7\.1 Milestone](https://github.com/cakephp/cakephp/milestone/185?closed=1)

この記事では、今回のアップデートで反映された修正をみてみたいと思います。


## Bugfixes
このアップデートでは、新機能の実装は行われずにBugfixesのみとなっています。

### cellがtemplateを発見できなかった場合に出力されるエラーメッセージを適切にセットできるように
```
Fixed incorrect error messages when cells cannot find their template.
```

[3\.7 cell \- exception by saeideng · Pull Request \#12783 · cakephp/cakephp](https://github.com/cakephp/cakephp/pull/12783)

### 「Cookieがセットされていない」を検査するアサーションの不具合を修正
```
Fixed a regression in assertCookieNotSet().
```
[Fixed regression in assertCookieNotSet by jeremyharris · Pull Request \#12794 · cakephp/cakephp](https://github.com/cakephp/cakephp/pull/12794)

### Emailクラスでの@deprecated漏れに対応
```
Added missing @deprecated annotations on methods on Email.
```
[Add deprecated doc strings\. · cakephp/cakephp@3db50b4](https://github.com/cakephp/cakephp/commit/3db50b4044dc4fdbad6d54cfbeb437f772ffbb3a)

### PHPDoc中の肩宣言において、`array`だった部分を`type[]` に変更
```
Improve typehints on array properties.
```
[Fix array doc blocks to be more precise where applicable\. by dereuromark · Pull Request \#12815 · cakephp/cakephp](https://github.com/cakephp/cakephp/pull/12815/files)

### 数値系のTypeへの代入時に、カンマやスペースを許容
```
Loosened type checking in integer and decimal type classes. Both these types now allow whitespace, and commas to accept more number formats.
```
[allow incorrect decimal / float formatting by segy · Pull Request \#12819 · cakephp/cakephp](https://github.com/cakephp/cakephp/pull/12819)

これは、3.7.0において「不正なフォーマットの数値文字列が来たときに例外を吐くようにする」の関連ですね。  
Issueがそこそこ盛り上がっていました。

### IntegrationTestTraitのsetUp() / tearDown()メソッドをリネームし、アノテーションを利用したフックを行うように
```
IntegrationTestTrait now uses annotations for its setup/teardown logic. This removes the need for awkward method aliasing when using the trait.
```
[Use annotitations for integration trait setup/teardown by markstory · Pull Request \#12822 · cakephp/cakephp](https://github.com/cakephp/cakephp/pull/12822)

個人的に、3.7にアップデートしたときに少し困って && 煩雑に感じていた点。  
Trait側に`setUp()` メソッドが入ることで、Controller <- `Parent::setUp()` を直接的に呼び出しにくくなっていました。  

### Consoleクラスの利用時に引数の指定状態によってはwarningが出る問題の修正
```
ConsoleConsole\Arguments::getArgument() no longer raises a notice error on missing arguments.
```
[Fix notice error on missing argument by markstory · Pull Request \#12825 · cakephp/cakephp](https://github.com/cakephp/cakephp/pull/12825/files)

### ConsoleクラスのcreateFile()実行時に、暗黙的にディレクトリを作成するように
```
Console\ConsoleIo::createFile() will now recursively create directories if necessary. This improves compatibility with Shell::createFile().
```
[createFile\(\) should create directories\. by markstory · Pull Request \#12826 · cakephp/cakephp](https://github.com/cakephp/cakephp/pull/12826/files)

## 感想
今回は、軽微なバグ修正(特に、ロジックのミスではないErrorの修正)が中心のアップデートでした。  
3.4->.5や 3.5->.6 に比べると、(その位置づけ的に)差分の小さかったアップグレードとはいえ、やはり桁が1つ上がっているので細かいところでエラーや不具合が出ているなぁという感想です。  
機能的な改善を伴うコミットhはみられないものの、もし3.7.0を利用していたらサクッと上げてしまうのが良いように思います。