---
layout:     post
title:      "java设计模式之策略模式"
subtitle:   " java设计模式 "
date:       2018-07-28 16:30:00
author:     "Aliyang"
header-img: "img/post-bg-javaDesign-Strategy.jpg"
tags:
    - java设计模式
---

## 策略模式

在平时编码的时候，有时候会遇到很多if...else...条件判断语句，而且这些语句里边的代码功能类似，这样会造成代码冗余性很高，而策略模式就是解决这一问题的。

对于不同的if...else里边，功能的逻辑类似，但是有部分是固定的，有部分是会变化的，在这里，对于不变的部分，通过创建该部分的接口以及对应的各种实现类，然后将这些实现类通过参数传入，这样的话我们只需要修改具体的实现类，就可以控制if...else里的代码块的不同。这样的话，if...else的代码块和具体的实现就分离开，实现了松耦合。

借用网上的例子：

``` java
//武器接口
public interface IFight {
     void fight();
 }
//具体的不同武器的实现
public class FightUseAxe implements IFight {
    @Override
    public void fight() {
        System.out.println("使用斧子战斗");
    }
}
===============================================
public class FightUseBlade implements IFight {
    @Override
    public void fight() {
        System.out.println("使用剑战斗");
    }
}
===============================================
public class FightUseKnife implements IFight {
    @Override
    public void fight() {
        System.out.println("使用匕首战斗");
    }
}
//抽象角色类
public abstract class Role {
    
    private IFight weapon;
    
    public void fight() {
        weapon.fight();
    }
    
    public void setWeapon(IFight weapon) {
        this.weapon = weapon;
    }
    
    public abstract void display();
}
//具体角色类
public class King extends Role {
    @Override
    public void display() {
        System.out.println("显示国王的样子");
    }
    
    public static void main(String[] args) {
        Role role = new King();
        role.display();
        role.setWeapon(new FightUseAxe());
        role.fight();
        role.setWeapon(new FightUseBlade());
        role.fight();
    }
    /**
     * 运行结果：
     * 显示国王的样子
     * 使用斧子战斗
     * 使用剑战斗
     */
}
```

上述代码中，角色的武器部分被剥离出来，通过类似简单的接口模式实现，对于不同的角色的不同武器，通过传入不同的武器对象就可以。

## Reference

* [[设计模式之一策略模式](https://www.cnblogs.com/dotgua/p/strategy-pattern.html)]