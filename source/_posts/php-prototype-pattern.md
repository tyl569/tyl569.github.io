---
title: PHP设计模式-原型模式
tags:
  - PHP原型模式
  - PHP设计模式
id: '228'
categories:
  - - PHP设计模式
comments: false
date: 2018-02-21 20:28:08
---

#### 原型模式

原型设计模式是通过使用一种克隆技术复制实例化的对象。新对象是通过复制原型而进行创建的。原型设计模式的目的是通过克隆以减少实例化对象的开销。

![](/uploads/2018/02/%E5%8E%9F%E5%9E%8B%E7%B1%BB%E5%9B%BE.png)

<!--more-->

#### 何时使用原型模式

原型模式要求你创建某个原型对象的多个实例，这个时候就可以使用原型模式。比如，在关于进化的研究，通常使用果蝇作为研究对象。果蝇很快繁殖，基本上一个小时就能进行产卵，和其他的生物相比，找到和记录变异的几率更大。如果换做大象的话（比如大象的孕育期长达22个月），那么整个研究过程将会消耗很长的时间。因此，只需要完成两个实例化（一个雄性和一个雌性），然后就可以跟进克隆多个变异，而不需要由具体类另外创建实例。

#### 克隆函数

使用原型模式，首先就要了解如何使用PHP的\_\_clone()。

```php
abstract class CloneMe {
    public $name;
    abstract function __clone();
}

class Person extends CloneMe {

    public function __construct() {
        $this->name = "Original";
        echo 'hello';
    }

    public function display() {
        echo "\n$this->name\n";
    }

    public function __clone() {}

}
$worker = new Person();
$worker->display();

$slacker = clone $worker;
$slacker->display();
```

定义了一个抽象类`CloneMe`，然后定义一个`Person`类进行实现。定义一个变量`$worker` 进行实例化，然后使用关键字`clone`进行对象的克隆。

输出结果: hello Original

Original

不过需要注意的是，\_\_clone()不能直接调用，会出现报错`Fatal error: Cannot call __clone() method on objects - use 'clone $obj' instead`

#### 克隆不会启动构造函数

上面的输出结果可能已经见到了，clone是不会启动构造函数的。其实这个也是比较理解的。举个例子，现在有一个人A，那么这个人肯定会有手脚，那么`手脚`就算是默认构造函数进行的初始化，如果要根据A克隆一个人B，那么B不用再造`手脚`，而是克隆之后，就自带了A的`手脚`，当然，你也可以把B的`手脚`砍掉(比如$slacker->name="Tyler Teng")。

```php
abstract class CloneMe {
    public $name;
    abstract function __clone();
}

class Person extends CloneMe {

    public function __construct() {
        $this->name = "Original";
        echo 'hello';
    }

    public function display() {
        echo "\n$this->name\n";
    }

    public function __clone() {
    }

}
$worker = new Person();
$worker->display();

$slacker = clone $worker;
$slacker->name = "Tyler Teng";
$slacker->display();
```

输出结果 hello Original

Tyler Teng

#### 研究果蝇

##### 抽象类和具体实现

简单定义三个属性：眼睛的颜色，翅膀震动次数，眼睛个数

```php
abstract class IPrototype {
    public $eyeColor;
    public $wingBeat;
    public $uniyEyes;
    abstract function __clone();
}
```

除了一些基本的属性，还需要区别雄性和雌性

```php
include_once 'IPrototype.php';
class FemaleProto extends IProtoType {

    const gender = "FEMALE";
    public $fecundity;

    public function __construct() {
        $this->eyeColor = "red";
        $this->wingBeat = "220";
        $this->unitEyes = "760";
    }

    public function __clone() {}

}
```

```php
include_once "IPrototype.php";
class MaleProto extends IPrototype {

    const gender = "MALE";

    public $mated;

    public function __construct() {
        $this->eyeColor = "red";
        $this->wingBeat = "220";
        $this->unitEyes = "760";
    }

    public function __clone() {}

}
```

##### 客户端

我们定义一个`Client`类，首先从具体类中实例化$fly1和$fly2，$c1Fly、$c2Fly和$updateCloneFly则分别是这两个类实例的克隆

```php
function __autoload($class_name) {
    include_once realpath(__DIR__) . '/' .  $class_name . '.php';
}
class Client {
    private $fly1;
    private $fly2;

    private $c1Fly;
    private $c2Fly;
    private $upDatadCloneFly;

    public function __construct() {
        $this->fly1 = new MaleProto();
        $this->fly2 = new FemaleProto();

        $this->c1Fly = clone $this->fly1;
        $this->c2Fly = clone $this->fly2;

        $this->upDatadCloneFly = clone $this->fly2;
        $this->c1Fly->mated = "true";
        $this->c2Fly->fecundity = "186";
        $this->upDatadCloneFly->eyeColor = "purple";
        $this->upDatadCloneFly->wingBeat = "220";
        $this->upDatadCloneFly->unitEyes = "750";
        $this->upDatadCloneFly->fecundity = "92";

        $this->showFly($this->c1Fly);
        $this->showFly($this->c2Fly);
        $this->showFly($this->upDatadCloneFly);
    }

    public function showFly(IProtoType $fly) {
        echo "Eye color : " . $fly->eyeColor . "\n";
        echo "Wing Beats/second : " . $fly->wingBeat . "\n";
        echo "Eye units : " . $fly->unitEyes . "\n";
        $genderNow = $fly::gender;
        echo "Gender : " . $genderNow . "\n";
        if ($genderNow == "FEMALE") {
            echo "Numbers of eges : " . $fly->fecundity .  "\n";
        } else {
            echo "Mate : " . $fly->mated . "\n";
        }
    }
}

$woker = new Client();
```

输出 Eye color : red Wing Beats/second : 220 Eye units : 760 Gender : MALE Mate : true Eye color : red Wing Beats/second : 220 Eye units : 760 Gender : FEMALE Numbers of eges : 186 Eye color : purple Wing Beats/second : 220 Eye units : 750 Gender : FEMALE Numbers of eges : 92

#### 总结

作为被克隆的类，默认构造函数不应该做太多的初始化，否则结果往往不灵活，而且是过度的耦合设计。构造函数不应完成具体的工作，一种做法是忽略模式类中的构造函数，除非你有充分的理由包含这些构造函数；另外一种做法是，允许在需要的时间调用，让客户端负责实例化和克隆的有关事务。

#### 参考文献

*   Learning PHP设计模式

#### 附件

*   [PHP原型模式demo](/uploads/2018/02/test2.zip)