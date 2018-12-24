---
title: "ObjectRegistryについて"
date: 2018-12-24T07:35:23+09:00
tags: [Core]
categories: ["CakePHP3"]
---
※1人AdventのDay-23です

[1人advent\(CakePHP中心、PHP開発よもやま\) Advent Calendar 2018 \- Adventar](https://adventar.org/calendars/3627)

## 概要
ObjectRegistryというものがあります。  
CakePHPで内部的にかなり頻繁に利用されているクラスであり、インスタンス管理の根幹を担っていると言っても過言ではありません。  
どんな使われ方をされていて、どんな処理をしているのかを見てみました。

## イントロ
CakePHPを用いたアプリケーションを作成していくに当たり、自身で直接触れることはあまりなさそうなクラスです。
class docを見ると、以下のように書かれています。

```
Acts as a registry/factory for objects.

Provides registry & factory functionality for object types. Used
as a super class for various composition based re-use features in CakePHP.

Each subclass needs to implement the various abstract methods to complete
the template method load().
 ```

 1つとして「普段は抽象的に操作をしている部分を少し具体的に知ることができたら、より納得感をもって開発ができるだろう」という視点、2つ目に「自身が実装や設計を考えるにあたって学びがありそうだ」という視点で内部構造を見てみたいと思いました。

## ObjectRegistryはどこで利用されているか
ObjectRegistryは抽象クラスであり、Countable/IteratorAggregateといったインターフェイスを実装します。ここから分かる通り、「コレクション」としての性格を備えたものです。  
この利用先は、どのようなクラスがあるでしょうか。

一覧を列挙してみます。


* \Cake\Cache\CacheRegistry
* \Cake\Console\HelperRegistry
* \Cake\Console\TaskRegistry
* \Cake\Controller\ComponentRegistry
* \Cake\Datasource\ConnectionRegistry
* \Cake\Log\LogEngineRegistry
* \Cake\Mailer\TransportRegistry
* \Cake\ORM\BehaviorRegistry
* \Cake\View\HelperRegistry

このように、「〜Registry」というsuffixをつける、という規則が見いだせます。  
そしてその内容は、 処理を委譲するために、何かのロジックを持ったオブジェクトを管理してアクセス方法を提供するものだというのが名前から汲み取れると思います。


## ObjectRegistryの内容
「どんな仕事をしているのか」をみるには、そのクラスが持っているabstractメソッドやpublicメソッドを見るのが良いでしょう。    

### 抽象メソッド
まず、abstractメソッドは次の3つを持っています。

#### _resolveClassName
```php
/**
 * Should resolve the classname for a given object type.
 *
 * @param string $class The class to resolve.
 * @return string|bool The resolved name or false for failure.
 */
abstract protected function _resolveClassName($class);
```

このメソッドは、「与えられたオブジェクト名(クラス名)に対して、どのように探索を行うか」という具体的な手段を実装することを求めています。  
FQCN等により直接的なクラスの読み込みを行うと矛盾を招き入れる可能性があるためです。例えば、「ヘルパーのレジストリにビヘイビアを指定された」という場合は、登録が成功すべきではありません。  
実際には、クラス名の解決より後の読み込みフェーズにおいて型のチェックを行っている例の方が多く見られますが、いずれにせよこのメソッドは「ある程度の暗黙的な動きをカプセル化することによって、利用者に対して間違いの少ないように利便性を与える」という役割を担っています。

利用例をみていきましょう。   
ConnectionRegistryはデータベースへの接続を管理するクラスです。その「クラス名の解決」方法は以下のようになっています。

```php
/**
 * Resolve a datasource classname.
 *
 * Part of the template method for Cake\Core\ObjectRegistry::load()
 * @param string $class Partial classname to resolve.
 * @return string|false Either the correct classname or false.
 */
protected function _resolveClassName($class)
{
    if (is_object($class)) {
        return $class;
    }

    return App::className($class, 'Datasource');
}
```
特徴としては、何らかのインスタンスを直接引き渡すことができる点でしょうか。  
`App::className()`は、よく利用されるメソッドです。これは、プラグインの指定(ドット区切りの指定)を解決したり、第2引数にパッケージ名を渡すことで読み無スペースを限定するといった機能を持ちます。  
ただし、第1引数に`\\`を含む場合はそれをそのまま利用するので、FQCNを渡した場合は直接的なクラス指定も可能になっています。

他のももう1つ見てみましょう。

```php
protected function _resolveClassName($class)
{
    if (is_object($class)) {
        return $class;
    }

    return App::className($class, 'Cache/Engine', 'Engine');
}
```
これは、CacheRegistryの例です。  
App::className()に第3引数を渡しています。これはクラス名のsuffixであり、例えば`$class = 'Redis'` とした場合にはsuffixの付与により`RedisEngine`を探索されるという格好です。

#### _throwMissingClassError
```php
/**
 * Throw an exception when the requested object name is missing.
 *
 * @param string $class The class that is missing.
 * @param string $plugin The plugin $class is missing from.
 * @return void
 * @throws \Exception
 */
abstract protected function _throwMissingClassError($class, $plugin);
```
このメソッドは、 読み込みもしくは読み込み解除時において該当クラス/オブジェクトが見つからなかった処理を定義します。名称の通り、例外を吐くように求められます。

具体例を見ていきます。

```php
// ComponentRegistry
protected function _throwMissingClassError($class, $plugin)
{
    throw new MissingComponentException([
        'class' => $class . 'Component',
        'plugin' => $plugin
    ]);
}

// LogEngineRegistry
protected function _throwMissingClassError($class, $plugin)
{
    throw new RuntimeException(sprintf('Could not load class %s', $class));
}

// CacheRegistry
protected function _throwMissingClassError($class, $plugin)
{
    throw new BadMethodCallException(sprintf('Cache engine %s is not available.', $class));
}
```
最も一般的な例は、ComponentRegistryのように自身の属する空間に応じた独自の例外を利用することだと思います。  
他方で、クラスの役割によって、ランタイムエラーやロジックエクセプションを利用しているという格好です。

#### _create(
```php
/**
 * Create an instance of a given classname.
 *
 * This method should construct and do any other initialization logic
 * required.
 *
 * @param string $class The class to build.
 * @param string $alias The alias of the object.
 * @param array $config The Configuration settings for construction
 * @return mixed
 */
abstract protected function _create($class, $alias, $config);
```
実際にインスタンスを作成する方法について実装したメソッドです。  
最も単純化した例ではクラスのインスタンス化を行います。実際にはconfigの注入なども合わせて行う事が多いようです。

CacheRegistryとBehaviorRegistryの例を見てみます。

```php
protected function _create($class, $alias, $config)
{
    if (is_object($class)) {
        $instance = $class;
    }

    unset($config['className']);
    if (!isset($instance)) {
        $instance = new $class($config);
    }

    if (!($instance instanceof CacheEngine)) {
        throw new RuntimeException(
            'Cache engines must use Cake\Cache\CacheEngine as a base class.'
        );
    }

    if (!$instance->init($config)) {
        throw new RuntimeException(
            sprintf('Cache engine %s is not properly configured.', get_class($instance))
        );
    }

    $config = $instance->getConfig();
    if ($config['probability'] && time() % $config['probability'] === 0) {
        $instance->gc();
    }

    return $instance;
}
```
インスタンスの生成と、そのクラスのチェックまではどのRegistryにも共通して見られる例です。  
そこから`init()`メソッドを呼び、コンフィグの内容によって`gc()`うぃ行うといった処理が独自に加えられています。

```php
protected function _create($class, $alias, $config)
{
    $instance = new $class($this->_table, $config);
    $enable = isset($config['enabled']) ? $config['enabled'] : true;
    if ($enable) {
        $this->getEventManager()->on($instance);
    }
    $methods = $this->_getMethods($instance, $class, $alias);
    $this->_methodMap += $methods['methods'];
    $this->_finderMap += $methods['finders'];

    return $instance;
}
```
こちらはBehaviorRegistryです。イベントのフックフックもサポートします。  
特筆すべきはcustom finderやmethodの登録を行っていることで、これはBehaviorならではの要件になります。Behaviorの場合は、Tableオブジェクトに対するマジックメソッドにて、提供されているメソッドを把握できる必要がります。そのため、提供可能なAPIとして、Registry自体にメソッドリストの登録を行っているわけです。

抽象メソッドは以上の3つになります。

### 公開メソッド
abstractメソッドに加えて、公開のAPIも実装されています。  
その中にはCountableやIteratorAggregateインターフェスからの妖精で実装されているものや開発/デバッグ用のものもいくつか含まれます。

この記事では「ObjectRegistryとはなにか」を描き出すのが目的のため、本質的な機能を掴むためのメソッドをいくつかピックアップして言及します。

#### load()
与えられたクラス名と設定を使って、実際にオブジェクトをRegistryに登録・管理するメソッドです。

1. クラス名を解決し
2. すでに同名のオブジェクトが読み込まれているが、設定が異なる場合は違反を報告し
3. 同名・同設定の場合は既存のオブジェクトを返却し
4. 新規のオブジェクトとなる場合は生成・登録を行う

という流れです。  
もちろん、この中で先述の`_resolveClassName()`や`_create()`が利用されています。

例えば、ControllerにおけるComponentの登録は以下のように記述されています。

```php
// \Cake\Controller\Controller
public function loadComponent($name, array $config = [])
{
    list(, $prop) = pluginSplit($name);

    return $this->{$prop} = $this->components()->load($name, $config);
}
```

#### set() / __set()
マジックメソッドが設定されているのは呼び出しを簡易にするためで、実際にはバイパスをしているだけに見えます。

`set()`については、単純に「渡されたオブジェクトを食わせる」だけではなく、その内部で`unload()`の実行とEventのディスパッチを行っています。これによって、ObjectRegistryが自身の状態の管理を堅牢にしている様子です。

```php
public function set($objectName, $object)
{
    list(, $name) = pluginSplit($objectName);

    // Just call unload if the object was loaded before
    if (array_key_exists($objectName, $this->_loaded)) {
        $this->unload($objectName);
    }
    if ($this instanceof EventDispatcherInterface && $object instanceof EventListenerInterface) {
        $this->getEventManager()->on($object);
    }
    $this->_loaded[$name] = $object;

    return $this;
}
```

#### get() / __get()
とてもシンプルに、内部のコレクションへのアクセスにバイパスを行っています。

```php
public function get($name)
{
    if (isset($this->_loaded[$name])) {
        return $this->_loaded[$name];
    }

    return null;
}
```

他にも `loaded()`や`reset()``has()`といったメソッドが備わっていますが、凡そ名前のとおりです。

## 関連: TableRegistry
テーブルオブジェクトの管理を行うクラスTableRegistryが存在しますが、こちらはObjectRegistryの具象クラスにはなっていません。  
内部的にはTableLocatorを利用しており、これはLocatorInterfaceの実装となります。

## まとめ
ObjectRegistryは、CakePHP「らしく」オブジェクトの管理やコレクションを実現するのに役立つ仕組みだと思います。  
サードパーティのDIを利用するという方法も存在しますが、そこまで高度なものが必要でない場合や学習コストをフレームワーク本体のそれに寄せたい場合などに、目的や状況によってはとても簡単に手を出せるかもなと感じました。