---
layout:     post
title:      "java设计模式之适配器模式"
subtitle:   " java设计模式 "
date:       2018-07-28 14:15:00
author:     "Aliyang"
header-img: "img/post-bg-javaDesign-Adapter.jpg"
tags:
    - java设计模式
---

适配器模式的目的是让两个不兼容的接口能够一起工作。为了达到这个目的，适配器模式的做法是在中间创建一个适配器，从而可以使得两个接口一起工作。打个比方，最新的mac电脑都只有type-c接口，usb想要和电脑连接的话要用转接器，那么这里的usb和type-c就是两个不兼容的接口，而转接器就是我们的适配器。

适配器模式一共有三种模式：**类适配器，对象适配器和接口适配器**。

## 类适配器模式

原理：有接口A和接口B，类C实现接口A，适配器D继承类C实现接口B，在D实现接B的方法，并在该方法中调用类C的方法。

还是上边的例子，现在有两个接口，一个是type-c一个是usb，现在我们实现了usb接口的类，那么我们在适配器(即转接器)里继承usb类，然后实现type-c接口的方法，在该方法中调用usb类的方法(也可以说usb里的文件)，这样的话相当于将两个接口连接起来了。

``` java
//USB接口
public interface USB {
    void sendFile();
}
//Type-c接口
public interface Type_c {
    void recerveFile();
}
//USB接口实现类
public class USBImpl implements USB{
    @Override
    public void sendFile() {
        System.out.println("usb开始发送文件!");
    }
}
//适配器类
public class ClassAdapter extends USBImpl implements Type_c {
    @Override
    public void recerveFile() {
        sendFile();
        System.out.println("type-c开始接受文件!");
    }
}
//测试类
public class ClassAdapterTest {
    public static void main(String[] args){
        ClassAdapter classAdapter=new ClassAdapter();
        classAdapter.recerveFile();
    }
}
```

## 对象适配器模式

原理：还是USB和Type-c两个接口，但是不同在于，此时的适配器类并不是继承的USB实现类，而是只实现Type-c接口，USB实现类对象通过构造函数的方式传入适配器类，然后在实现的Type-c接口的方法中调用USB类对象的方法。

``` java
//USB接口和Type-c接口以及USB实现类和上边相同
//适配器
public class ObjectAdapter implements Type_c{
    private USB usb;
    public ObjectAdapter(USB usb){
        this.usb=usb;
    }
    @Override
    public void recerveFile() {
        usb.sendFile();
        System.out.println("type-c开始接收文件!");
    }
}
//测试类
public class ObjectAdapterTest {
    public static void main(String[] args){
        USB usb=new USBImpl();
        ObjectAdapter objectAdapter=new ObjectAdapter(usb);
        objectAdapter.recerveFile();
    }
}
```

## 接口适配器模式

原理：某个接口A有很多方法，现在我想用到其中的某几个方法，但是如果实现接口的话就需要将所有的方法都实现，很麻烦。这个时候先创建一个抽象类实现所有的接口方法(也可以实现一部分不常用方法，其他几个通用方法标为抽象方法)，然后通过继承的方式实现抽象方法和重写方法。

``` java
//方法接口
public interface MethodInterface {
    void method1();
    void method2();
    void method3();
}
//抽象方法类
public abstract class AbstractMethod implements MethodInterface{
    @Override
    public void method1() {
        System.out.println("方法1.");
    }

    @Override
    public void method2() {
        System.out.println("方法2.");
    }

    @Override
    public void method3() {
        System.out.println("方法3.");
    }
}
//抽象接口适配器
public class AbstractInterfaceAdapter extends AbstractMethod {
    @Override
    public void method1() {
        System.out.println("重写方法1.");
    }
}
//测试类
public class AbstractInterfaceAdapterTest {
    public static void main(String[] args){
        AbstractInterfaceAdapter abstractInterfaceAdapter=new AbstractInterfaceAdapter();
        abstractInterfaceAdapter.method1();
    }
}
```

适配器模式主要是为了防止对原来的类进行修改，通过创建新的类将原来的类关联在一起。

## Reference

* [[Java设计模式之《适配器模式》及应用场景](https://www.cnblogs.com/V1haoge/p/6479118.html)]