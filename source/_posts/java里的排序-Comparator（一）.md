---
title: java里的排序-Comparator（一）
date: 2017-12-02 21:29:36
tags: [java]
categories: 排序算法
---
### 前言
在排序算法上进阶之后，我们来看看这些算法的实践，最近工作上涉及一些list对象的排序，用到了Collections.sort。一直对Collections.sort里实现的Comparator到底怎么实现的升序和降序。研究了一下源码，Comparator底层实现是用来归并算法。归并算法在上一篇已经做了介绍的铺垫。今天我们来说一下这个sort的用法和实现原理。

### 实现Comparator
java8之前，我们对list排队多半是实现一个Comparator接口，先定义一个简单的实体类:
```
public class Person {

    private String name;

    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```
<!--more-->
然后我们根据name对一个person列表进行简单的排序：
```
public class ComparatorService extends BaseTestService{

    private List<Person> personList;

    @Before
    public void initPersonList(){
        personList = Lists.newArrayList(new Person("f",20),
                new Person("c",21),new Person("g",18),new Person("e",18));

    }

    @After
    public void printPersonList(){
        personList.stream().forEach(person -> System.err.println(person.getName()));
    }

    //比较
    @Test
    public void compareWithInnerClass(){
        Collections.sort(personList, new Comparator<Person>() {
            @Override
            public int compare(Person o1, Person o2) {
                int i= o1.getName().compareTo(o2.getName());
                return i;
            }
        });
    }
}
```
o1.getName().compareTo(o2.getName())这样写的时候是升序的，o2.getName().compareTo(o2.getName())就是降序的。这里Collections.sort实际还是调用了java.util.List的sort方法。
### 使用comparing
java8为Comparator提供了新的静态方法，例子：
    @Test
    public void compareWithLamda(){
        Collections.sort(personList,Comparator.comparing(Person::getName));
    }
结果为默认是升序的，如果我们要反序呢，很简单，有个Comparator有一个反序的方法：reversed（）
```
    @Test
    public void compareWithLamdaReversed(){
        Collections.sort(personList,Comparator.comparing(Person::getName).reversed());
    }
```
扩展：Comparator还提供了支持多个比较条件的方法：
```
    @Test
    public void compareWithLamdaMore(){
        Collections.sort(personList,Comparator.comparing(Person::getName).thenComparing(Person::getAge));
    }
```
### 使用Lambda表达式
java8在java.util.List里新增了支持lambda的sort方法可以直接使用,代码更简洁：
```
    @Test
    public void compareWithNewSort(){
       personList.sort((p1,p2)-> p1.getName().compareTo(p2.getName()));
    }
```
当然这里也支持多条件排序的，我们可以在Person里自定义一个排序的静态方法：
```
    public static int compareWithNameAndAge(Person p1, Person p2) {
        if (p1.getName().equals(p2.getName())) {
            return p1.getAge() - p2.getAge();
        }
        return p1.getName().compareTo(p2.getName());
    }

    @Test
    public void compareWithLamdaMethod() {
        personList.sort(Person::compareWithNameAndAge);
    }
```
总之java8排序方法形式有很多，但万变不离其宗都是使用了List.sort()方法，下一个笔记，我们来分析一下源码。