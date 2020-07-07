---
title: 'ifelse解决方案:策略模式+表驱动'
date: 2020-07-07 22:30:16
tags:
	- Java
	- 设计模式
---

问题来源：写脚本的时候有用到不同的key对应不同的操作的情况，问同事，同事说用表驱动的方法

写完Python，查了一下Java中表驱动，有了下边的学习记录

<!--more-->

[设计完美的策略模式，消除if else](https://www.cnblogs.com/jiujiduilie/p/9191629.html)

代码是js代码：
![](https://upload-images.jianshu.io/upload_images/19092361-4da72bddb100edbb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
改成 Java：

```
public class Client {
    public static void main(String param) throws Exception {
        Context context;
        if (param == "A") {
            context = new Context(new ConcreteStrategyA());
        }else if (param == "B"){
            context = new Context(new ConcreteStrategyB());
        }else if (param == "C"){
            context = new Context(new ConcreteStrategyC());
        }else {
            throw new Exception("没有可用的策略");
        }
        context.ContextFunc();
    }
}
```


## 用策略模式设计商场vip收费规则
```
策略接口
package strategy;

public interface Strategy {
    int CalcPrice(int originalPrice);
}
```
要实现的几种策略：
```
package strategy;

public class VipNone implements Strategy{

    @Override
    public int CalcPrice(int originalPrice) {
        return originalPrice;
    }
}

package strategy;

public class VipOrdinary implements Strategy{
    @Override
    public int CalcPrice(int originalPrice) {
        return (int)(originalPrice*0.9+0.5);
    }
}

package strategy;

public class VipGold implements Strategy{
    @Override
    public int CalcPrice(int originalPrice) {
        return (int)(originalPrice*0.8+0.5);
    }
}

package strategy;

public class VipDiamond implements Strategy{
    @Override
    public int CalcPrice(int originalPrice) {
        return (int)(originalPrice*0.7+0.5);
    }
}

```

然后是服务端：
```
package strategy;

public enum Vip {
    ORDINARY,GOLD,DIAMOND
}

package strategy;

public class Customer {

    public int _totalAmount = 0;
    // 这个地方如果是实际的话可以从数据库查询，获取会员卡id，得到totalAmount
    public Vip vip = null;
    public Strategy vip_algorithm = null;
    public void Buy(int originalPriceM){
        if (_totalAmount>=500*10){
            vip = Vip.DIAMOND;
            vip_algorithm = new VipDiamond();
        }else if (_totalAmount>=300*10){
            vip = Vip.GOLD;
            vip_algorithm = new VipGold();
        }else if (_totalAmount>=100*10){
            vip = Vip.ORDINARY;
            vip_algorithm = new VipOrdinary();
        }else{
            vip = null;
            vip_algorithm = new VipNone();
        }

        int finalPrice = vip_algorithm.CalcPrice(originalPriceM);
        System.out.println("您在本店购买商品原价"+originalPriceM+"需支付"+finalPrice+"元");
        _totalAmount +=originalPriceM;
        // 这个地方如果是实际的话可以从数据库查询，根据会员卡id，写入totalAmount
    }
}
```
客户端进行调用：
```
package strategy;

public class Client {
    public static void main(String[] args) {
        Customer customer = new Customer();
        customer.Buy(1000);
        customer.Buy(1000);
        customer.Buy(1000);
        customer.Buy(1000);
        customer.Buy(1000);
        customer.Buy(1000);
        customer.Buy(1000);
        customer.Buy(1000);
        customer.Buy(1000);
        customer.Buy(1000);
        customer.Buy(1000);
        customer.Buy(1000);
    }
}
```
结果输出：
```
您在本店购买商品原价1000需支付1000元
您在本店购买商品原价1000需支付900元
您在本店购买商品原价1000需支付900元
您在本店购买商品原价1000需支付800元
您在本店购买商品原价1000需支付800元
您在本店购买商品原价1000需支付700元
您在本店购买商品原价1000需支付700元
您在本店购买商品原价1000需支付700元
您在本店购买商品原价1000需支付700元
您在本店购买商品原价1000需支付700元
您在本店购买商品原价1000需支付700元
您在本店购买商品原价1000需支付700元
```

>疑问：这里还是在用很多if else 处理业务，只是把业务内容交给了具体的策略方法，增加了代码的可读性。

## 进阶：抽离customer的等级判断和策略方法，集成简单工厂模式返回 特定customer的 vip算法：
简单工厂类：
```
package strategy;

public class VipAlgorithmFactory {
    private VipAlgorithmFactory(){}
    public static Strategy getVipAlgorithm(Customer customer){
        if (customer._totalAmount>=500*10){
            customer.vip_level = Vip.DIAMOND;
            return new VipDiamond();
        }else if (customer._totalAmount>=300*10){
            customer.vip_level = Vip.GOLD;
            return new VipGold();
        }else if (customer._totalAmount>=100*10){
            customer.vip_level = Vip.ORDINARY;
            return  new VipOrdinary();
        }else{
            customer.vip_level = null;
            return new VipNone();
        }
    }
}
```
服务端调用工厂类设定customer的策略：
```
public void Buy(int originalPriceM){
        vip_algorithm = VipAlgorithmFactory.getVipAlgorithm(this);
```
其他代码不用修改。
>疑问：工厂模式里还是很多if else，加入不同的vip等级还是要修改工厂类的代码

## 再进阶：表驱动法修改工厂模式
本文开头的博客给出的是基于注解的方式，鉴于注解我还不熟……改成表驱动：
```
public static Strategy getVipAlgorithm(Customer customer){
        Strategy vip_algorithm = null;
        Map<Integer,Strategy> vip_map = Maps.newLinkedHashMap();
        vip_map.put(Integer.MAX_VALUE,new VipDiamond());
        vip_map.put(500*10,new VipGold());
        vip_map.put(300*10,new VipOrdinary());
        vip_map.put(100*10,new VipNone());
        for (Map.Entry<Integer, Strategy> entry : vip_map.entrySet()) {
            if (customer._totalAmount>=entry.getKey()){
                vip_algorithm = entry.getValue();
                break;
            }
        }
        return vip_algorithm;
    }
```
主要碰到的问题：
- 这是阶梯式表驱动的一种，如何确定相应的上下限
上下限决定后边的if判断逻辑
- 如何顺序访问Map
最常用的HashMap是随机访问的，如何访问Map中所有的key得到对应的value成为一个问题，用LinkedHashMap解决了。（~~其实也可以用HashMap,只要访问完所有的key就可以~~，实际鉴定不可以QAQ，而且要按照阶梯顺序将Integer跟Method()放入Map，要不然后边的逻辑也受影响）。

最后结果：
![](https://upload-images.jianshu.io/upload_images/19092361-51fb507b2bbeeeec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
就可以把customer的vip_level这个属性删掉了

## 问题：
1.为什么HashMap是随机访问key？
2.lindkedHashMap为什么可以按插入顺序访问？

P.s不用工厂，直接用策略模式＋表驱动是不是就可。QAQ