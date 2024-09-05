---
title: Java 一种 java.util.ConcurrentModificationException 异常原因
description: Java 一种 java.util.ConcurrentModificationException 异常原因
date: 2024-09-05
image: java.jpg
slug: Java
categories:
    - Java
---

*该文章用来记录java学习过程中看似应该不会发生的异常*

我们有一个 Main.java 和 cn/pinsoftstd/Study.java，源代码如下：

Main.java:
```java
import cn.pinsoftstd.study.Study;
public class Main {
    public static void main(String[] args) {
        Study study=new Study();
        study.printHelloWorld();
        study.addProgramming("XiaoMing");
        study.addProgramming("XiaoHong");
        study.addProgramming("LiJun");
        study.removeProgramming("LiJun");	//发生异常
        study.printProgramingNames(); 	
    }
}
```

cn/pinsoftstd/Study.java
```java
package cn.pinsoftstd.study;
import java.util.ArrayList;
import java.util.Iterator;

/**
 * The class study provides an interface for developers to add and remove
 * the ones who are studying programming or not.
 * @author Program
 * @version 1.0
 */
public class Study {
    public Study(){

    }
    /** This prints "Hello World",which is the person who must print once
     * study a programming language.
     * @author Program
     */
    public void printHelloWorld(){
        System.out.println("Hello World");
    }
    /**
     * This method adds a person to the Study Programming category.
     */

    public Programming addProgramming(String name){
        Programming pr=new Programming(name);
        programmings.add(pr);
        return  pr;
    }
    /**
     * This method removes a person from the Study Programming category.
     */
    public  boolean removeProgramming(String name) {
       boolean removed = false;
        for (Programming pr : programmings) {
            if (pr.name.equals(name)) {
                programmings.remove(pr);	//发生异常
                removed = true;
            }
        }
        return removed;
    }
    /**
     * This method prints the names of people who are studying programming.
     */
    public void printProgramingNames(){
        for(Programming pr:programmings){
            System.out.println(pr.name);
        }
    }

    private ArrayList<Programming> programmings=new ArrayList<>();
    public  class Programming{
        private String name;
        public Programming(String name){
            this.name=name;
        }
        public String name(){
            return this.name;
        }
        public void printStudyingProgramming(){
            System.out.println("Study Programming:"+name);
        }
    }
}
```

自己写的时候感觉没有问题，结果运行的时候抛出了 ``java.ConcurrentModificationException ``异常。异常代码为Main.java中的`` study.printProgramingNames();``，Study.java中的``programmings.remove(pr);``。
![不科学](https://i-blog.csdnimg.cn/direct/2a0481043423416a859a7fb0c73e5ef0.png)


**其实，是因为在 Java 中增强型``for``循环使用了迭代器(iterator)遍历，如果在遍历的时候使用了`` study.printProgramingNames();``，迭代器会检测到数组列表的变化，从而抛出``java.util.ConcurrentModificationException ``异常。**

# 解决方法
- 方法一：使用迭代器遍历``programmings``数组列表，不使用增强型``for``循环遍历。下面是修改后的代码：
```java
 public  boolean removeProgramming(String name) {
        boolean removed=false;
        Iterator<Programming>it=programmings.iterator();
        while (it.hasNext()){	//使用迭代器遍历
            Programming pr=it.next();
            if(pr.name.equals(name)){
                it.remove();
                removed=true;
            }
        }
        return removed;
  }
```
方法二：通过遍历索引(index)来遍历数组列表``programmings``。下面是修改后的代码:
```java
public  boolean removeProgramming(String name) {
        boolean removed = false;
        for(int i=0;i<programmings.size();i++){
           if(programmings.get(i).name.equals(name)){
               programmings.remove(i);
               removed = true;
           }
       }
        return removed;
    }
```
