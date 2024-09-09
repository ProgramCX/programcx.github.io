---
title: Java中super关键字的理解
description: Java中super关键字的理解
date: 2024-09-09
image: java.jpg
slug: Java
categories:
    - Java
---

# Java中super关键字的理解

## 访问超类的成员函数和变量
举个例子：
```java
class Main {
    public static void main(String[] args) {
        Manager manager = new Manager();
        manager.setBonus(8000);
        System.out.println(manager.getSalary());
    }
}

class Employee{
    private int salary = 6000;
    public void setSalary(int salary){
        this.salary=salary;
    }
    public int getSalary(){
        return salary;
    }

}

class Manager extends Employee{
    private int bonus = 0;
    public void setBonus(int bonus){
        this.bonus=bonus;
    }
    public int getBonus(){
        return bonus;
    }
    //覆盖超类的函数
    @Override public int  getSalary(){
        return bonus + this.salary;
    }
}
```
这段代码声明了超类``Employee``，以及继承了这个超类的``Manager``类。定义了``Manager``类的``manager``对象，``manager``的奖金设置为8000元。最后打印出manager的总薪水。
**但是，这段代码有一个很严重的错误。由于在超类``Employee``中，``salary``为``private``类型的，所以``manager``无法访问``salary``的值。为了遵循数据的隐蔽性原则，我们不能将它定义为``public``类型。不过，我们可以借助``super``关键字来达到目的。**

以下是修改过的代码：

```java
class Main {
    public static void main(String[] args) {
        Manager manager = new Manager();
        manager.setBonus(8000);
        System.out.println(manager.getSalary());
    }
}

class Employee{
    private int salary = 6000;
    public void setSalary(int salary){
        this.salary = salary;
    }
    //获得salary的值
    public int getSalary(){
        return salary;
    }

}

class Manager extends Employee{
    private int bonus = 0;
    public void setBonus(int bonus){
        this.bonus = bonus;
    }
    public int getBonus(){
        return bonus;
    }
    @Override public int  getSalary(){
        return bonus + super.getSalary();	//改为 super.getSalary
    }
}

```

在这段代码中，增加了``getSalary()``函数，这个函数返回salary的值。由于可以使用关键字``super``访问超类的成员函数和变量，``super.getSalary()``就获得了Employee类的``salary``变量。

## 调用父类的构造方法
举个例子：

```java
class Main {
    public static void main(String[] args) {
       	Manager manager = new Manager(8000);
        manager.setBonus(800);
        System.out.println(manager.getSalary());
    }
}

class Employee{
    public Employee(int salary){
        this.salary = salary;
    }
    private int salary = 6000;
    public void setSalary(int salary){
        this.salary = salary;
    }
    public int getSalary(){
        return salary;
    }

}

class Manager extends Employee{
    
    private int bonus = 0;
    public Manager(int salary) {
        super(salary);	//调用超类的构造器
    }

    public void setBonus(int bonus){
        this.bonus = bonus;
    }
    public int getBonus(){
        return bonus;
    }
    @Override public int  getSalary(){
        return bonus + super.getSalary();
    }
}

```

```manger```无法访问超类的私有成员，但是可以通过超类的构造器来访问它们。

**注意：超类的构造器必须是子类的构造器里面的第一个语句！**

## 解决成员变量名或方法名冲突
举个例子：
```java
class Parent {  
    int x = 10;  
}  

class Child extends Parent {  
    int x = 20;  

    void display() {  
        System.out.println("Child x = " + x); // 访问 Child 类的 x  
        System.out.println("Parent x = " + super.x); // 使用 super 访问 Parent 类的 x  
    }  
}
```

当子类中的成员变量名或方法名与超类中的成员变量名或方法名相同时，子类中的方法或变量会隐藏超类中的方法或变量。此时，如果你想在子类中访问被隐藏的超类成员，就需要使用``super`` 关键字。