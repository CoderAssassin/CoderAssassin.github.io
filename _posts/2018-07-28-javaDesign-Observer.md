---
layout:     post
title:      "java设计模式之观察者模式"
subtitle:   " java设计模式 "
date:       2018-07-28 15:30:00
author:     "Aliyang"
header-img: "img/post-bg-javaDesign-Observer.jpg"
tags:
    - java设计模式
---

## 观察者模式

观察者模式，顾名思义，就是有观察者和被观察者。其实就像是发布订阅模式，发布者发布消息，订阅者接收消息。

观察者模式的角色：被观察者接口，被观察者实现类，观察者接口，观察者实现类。

被观察者保存有观察者对象列表，并且有方法添加删除观察者，并且有发送通知的方法；观察者有接收消息的方法。

``` java
//发布者接口
public interface Observered {
    void addObserver(Observer observer);
    void removeObserver(Observer observer);
    void sendMessage();
}
//观察者接口
public interface Observer {
    void readMessage(String message);
}
//发布者实现类
public class ObserveredImpl implements Observered{
    List<Observer> observers;
    private String message;

    public ObserveredImpl(){
        observers=new ArrayList<>();
    }
    @Override
    public void addObserver(Observer observer) {
        this.observers.add(observer);
    }

    @Override
    public void removeObserver(Observer observer) {
        if (!observers.isEmpty()){
            observers.add(observer);
        }
    }

    @Override
    public void sendMessage() {
        for (Observer observer:observers){
            observer.readMessage(message);
        }
    }

    public void setMessage(String message){
        this.message=message;
        sendMessage();
    }
}
//观察者实现类
public class ObserverImpl implements Observer {
    private String name;
    private String message;
    public ObserverImpl(String name){
        this.name =name;
    }
    @Override
    public void readMessage(String message) {
        this.message=message;
    }

    public void showMessage(){
        System.out.println(name+" receive message:"+message);
    }
}
//测试类
public class ObserverPatternTest {
    public static void main(String[] args){
        ObserveredImpl observered=new ObserveredImpl();

        Observer observer1=new ObserverImpl("observer1");
        Observer observer2=new ObserverImpl("observer2");
        Observer observer3=new ObserverImpl("observer3");

        observered.addObserver(observer1);
        observered.addObserver(observer2);
        observered.addObserver(observer3);

        observered.setMessage("这是观察者模式!");
        observered.sendMessage();

        ((ObserverImpl) observer1).showMessage();
        ((ObserverImpl) observer2).showMessage();
        ((ObserverImpl) observer3).showMessage();
    }
}
```

从上面代码可以看出，发布者向观察者列表的每个对象都一一发布信息，通过调用观察者的方法将信息传给观察者，然后观察者再读出来。

