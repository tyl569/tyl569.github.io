---
title: PHP设计模式-适配器模式
tags:
  - PHP
  - PHP设计模式
id: '235'
categories:
  - - PHP
  - - PHP设计模式
comments: false
date: 2018-03-09 00:12:25
---

#### 适配器模式

##### 组合优于继承

学习设计模式，最经常听到的一句话就是：`组合优于继承`，因为使用组合，可以使参与者之间的绑定更宽松，在重用、结构和修改等方面会有很多的有点。这个和继承不同，继承类或者所继承的类中包含已经实现的方法，这其实也是一种绑定，使用组合，就没有这种紧密绑定的缺点。

<!--more-->

![使用继承的适配器类图](/uploads/2018/02/%E4%BD%BF%E7%94%A8%E7%BB%A7%E6%89%BF%E7%9A%84%E9%80%82%E9%85%8D%E5%99%A8%E7%B1%BB%E5%9B%BE.png)

使用继承的适配器类图

![使用组合的适配器类图](/uploads/2018/02/%E4%BD%BF%E7%94%A8%E7%BB%A7%E6%89%BF%E7%9A%84%E9%80%82%E9%85%8D%E5%99%A8%E7%B1%BB%E5%9B%BE-2.png)

使用组合的适配器类图

#### 组合适配器的例子

有一家温湿度传感器公司，这个传感器可以测试空气的温度和湿度。分别调用getTemperature()方法和geThumidity()

```php
class Sensor {

    public function getThumidity() {
        echo "the thumidity is 1234... \n";
    }

    public function getTemperature() {
        echo "the temperature is 45678... \n";
    }
}
```

这个时候呢，由于公司扩大业务，有一家智能家居的公司前来谈合作，想要使用遥控，来读取温度和湿度，分别调用方法 getThumidityByRemote() 和getTemperatureByRemote() ,没办法，只能扩充Sensor的类了。

```php
class Sensor {

    public function getThumidity() {
        echo "the thumidity is 1234... \n";
    }

    public function getTemperature() {
        echo "the temperature is 45678... \n";
    }

    public function getThumidityByRemote() {
        echo "the thumidity is 1234... \n";
    }

    public function getTemperatureByRemote() {
        echo "the temperature is 45678... \n";
    }
}
```

紧接着，又来了一家智能音箱的公司，也要接入温湿度传感器，但是这家公司是根据用户的话的内容来区分是否查询温湿度getByAnswer($code), $code = 0的时候，查询温度，当$code = 1的时候，查询湿度。

```php
class Sensor {

    public function getThumidity() {
        echo "the thumidity is 1234... \n";
    }

    public function getTemperature() {
        echo "the temperature is 45678... \n";
    }

    public function getThumidityByRemote() {
        echo "the thumidity is 1234... \n";
    }

    public function getTemperatureByRemote() {
        echo "the temperature is 45678... \n";
    }

    public function getByAsnwer($code) {
        if ($code == 0) {
            echo "the temperature is 45678... \n";
        } else if ($code == 1) {
            echo "the thumidity is 1234... \n";
        }
    }
}
```

可能你已经发现了问题，随着其他的公司不断的接入，Sensor的类会变得更加臃肿不看。或许我们应该好好区分一下整个过程中的角色问题。

![](/uploads/2018/03/Adapter-300x235.png)

1.  目标(Target)角色：智能家居公司和智能音箱公司的接口或者方法，使我们需要实现的目标(target)，因为外接需要调用这两个方法才行
2.  源(Adaptee)角色：Sensor的温湿度查询，是我们需要适配方法，也就是，我们可以通过这两个方法，进行温湿度查询。
3.  适配器(Adapter)角色：我们需要通过这个，来协调目标角色和源角色。

定义Sensor类

```php
class Sensor {

    public function geThumidity() {
        echo "the thumidity is 1234... \n";
    }

    public function getTemperature() {
        echo "the temperature is 45678... \n";
    }

}
```

定义智能家居和智能音箱的接口

```php
interface Household {
    public function geThumidityByRemote();
    public function getTemperatureByRemote();
}
```

```php
interface Spearkers {
    public function getByAnswer($code);
}
```

定义适配器

```php
class HouseholdTarget implements Household {

    private $_sensor = NULL;

    public function __construct(Sensor $sensor) {
        $this->_sensor = $sensor;
    }

    public function geThumidityByRemote() {
        $this->_sensor->geThumidity();
    }
    public function getTemperatureByRemote() {
        $this->_sensor->getTemperature();
    }

}
```

```php
class SpearkersTargat implements Spearkers {

    private $_sensor = NULL;

    public function __construct(Sensor $sensor) {
        $this->_sensor = $sensor;
    }

    public function getByAnswer($code) {
        if ($code == 0) {
            $this->_sensor->getTemperature();
        } else if ($code == 1) {
            $this->_sensor->geThumidity();
        }
    }

}
```

客户端进行调用

```php
function __autoload($class_name) {
    if (file_exists(__DIR__ . "/{$class_name}.php")) {
        require_once __DIR__ . "/{$class_name}.php";
    }
}

$sensor = new Sensor();
$HouseholdTarget = new HouseholdTarget($sensor);
$HouseholdTarget->geThumidityByRemote();
$HouseholdTarget->getTemperatureByRemote();

$SpearkersTargat = new SpearkersTargat($sensor);
$SpearkersTargat->getByAnswer(0);
$SpearkersTargat->getByAnswer(1);
```

输出结果 the thumidity is 1234... the temperature is 45678... the temperature is 45678... the thumidity is 1234...

#### 结论

使用适配器模式之后，能够更加方便对其进行拓展，如果有再多的公司进行接入，也不用担心影响Sensor类。

#### 使用场景

1．系统需要使用现有的类，而此类的接口不符合系统的需要。

2．想要建立一个可以重复使用的类，用于与一些彼此之间没有太大关联的一些类，包括一些可能在将来引进的类一起工作。这些源类不一定有很复杂的接口。

3.（对组合适配器而言）在设计里，需要改变多个已有子类的接口，如果使用类的适配器模式，就要针对每一个子类做一个适配器，而这不太实际。