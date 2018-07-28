---
layout:     post
title:      "java设计模式之构造者模式"
subtitle:   " java设计模式 "
date:       2018-07-28 15:10:00
author:     "Aliyang"
header-img: "img/post-bg-javaDesign-Builder.jpg"
tags:
    - java设计模式
---

## 构造者模式

为什么会有构造者模式？当我们要创建一个PC类的时候，要传入很多零件对象到构造函数，这样的话会使得方法的参数很多，构造者模式就是为了解决参数过多的问题，或者说，是针对组件很多的情况。

构造者模式和工厂模式很像，却别在于，工厂模式的工厂类对不同组件或者组件进行创建，构造者模式中，有一个新的角色，那就是“指挥者”角色，“指挥者”可以与建造者类之间交互，让建造者建造不同的组件，然后在"指挥者"类里边进行组装，最后将组装好的产品交给客户端。也就是说，在工厂模式之上多了一个“指挥者”角色。

构造者模式的角色：

* 抽象构造者，为各个组件提供创建的接口。
* 具体构造者：实现抽象构造者接口，实现具体的构造和装配方法。
* 产品：被构建出来的最终复杂对象，包含各个组件。
* 指挥者：和构造者交互，负责指挥构造的次序和构造哪些对象，客户端的交互对象。

``` java
//指挥者类
public class PCBuild {
//产品所需要的组件
    private final int money;
    private final int size;
    private final String cpu;
    private final String memory;
    private final String graphics;
//构造者类
    public static class Builder{
        //必要参数
        private final int money;
        private final int size;
        //可选组件
        private  String mainBoard="华硕";
        private  String cpu="intel i5";
        private  String memory="DDR4 1600";
        private  String graphics="GTX 1080";

        public Builder(int money,int size){
            this.money=money;
            this.size=size;
        }

        public Builder cpu(String cpu){
            this.cpu=cpu;
            return this;
        }

        public Builder memory(String memory){
            this.memory=memory;
            return this;
        }

        public Builder graphics(String graphics){
            this.graphics=graphics;
            return this;
        }
        //构造方法，创建最终产品，将构造者创建的组件传入其构造函数
        public PCBuild build(){
            return new PCBuild(this);
        }
    }

    private PCBuild(Builder builder){
        money=builder.money;
        size=builder.size;
        cpu=builder.cpu;
        memory=builder.memory;
        graphics=builder.graphics;
    }

    public static void main(String[] args){
        PCBuild pcBuild=new Builder(100,10).cpu("i7").memory("三星").graphics("1070").build();
    }
}
```

一般来说，如果产品是一个系列的话，将“指挥者”类和该产品的构造者类放到一起。由上面代码可以知道，其实所有的构造过程都是在构造者类里边完成的，在指挥者类里边有一个静态构造者类，然后构造者类创建完需要的组件后，将其自身作为指挥这类的构造函数参数传过去，生成最终的产品，在指挥者里边其实只有对应的组件列表，没有创建过程。