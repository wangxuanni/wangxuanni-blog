---
title: javaSE（下）
date: 2018-10-10 21:00:50
categories: java
description: 关于java异常，IO，集合。
---
# 异常
## 当finally中有返回语句
当try和finally中都有有返回语句会执行哪一个？
请看下面代码，猜猜程序的运行结果是？
```
public class Test {
    public static int testFinally() {
        try {
            return 1;
        } catch (Exception ex) {
            return 2;
        } finally {
            System.out.println("execute finally");
            return 3;
        }
    }
    public static void main(String[] args) {
        int result = testFinally();
        System.out.println(result);
    }
}
```

程序在执行try中遇到return语句时，会先将返回值储存到一个指定位置，然后执行finally代码块。（除非碰到exit(0)函数，就直接结束不执行finally）
如果finally里面有返回语句将会覆盖其它返回语句，最终执行finally的return语句，储存中的return语句被回收。
如果finally 没有返回语句，则执行完finally按储存中的return来。
因此运行结果是:execute finally 3

## 常见的runtime exception(运行时异常)：
(不是必须进行try catch的异常 )
NullPointerException 空指针异常
ArithmeticException 算术异常，比如除数为零
ClassCastException 类型转换异常
ConcurrentModificationException 同步修改异常，遍历一个集合的时候，删除集合的元素，就会抛出该异常 
IndexOutOfBoundsException 数组下标越界异常
NegativeArraySizeException 为数组分配的空间是负数异常

## throw和throws的区别
举个栗子
throw是语句抛出一个异常  
```
    String s = "abc"; 
    if(s.equals("abc")) { 
      throw new NumberFormatException(); 
    }
```

throws是方法可能抛出异常的声明
`public static void function() throws NumberFormatException{ `


 
# 集合
## 基础概念
- Java容器类类库的用途是"保存对象"，并将其划分为两个不同的概念：
1) Collection
一组"对立"的元素，通常这些元素都服从某种规则
　　1.1) List必须保持元素特定的顺序
　　1.2) Set不能有重复元素
　　1.3) Queue保持一个队列(先进先出)的顺序
2) Map
一组成对的"键值对"对象


- Collection与Collections
Collection是所有集合类的根接口；
Collections是提供集合操作的工具类；常用方法如下
reverse	反转	
shuffle	混淆
sort	排序	
swap	交换（交换0和5下标的数据后）` Collections.swap(numbers,0,5);`
rotate	滚动(把集合中的数据向右滚动2个单位)	 `Collections.rotate(numbers,2);`
synchronizedList	线程安全化

## 常用集合
### ArrayList
代码展示ArrayList常用方法
```
public static void main(String[] args) {
    List<String> animal = new ArrayList<String>();
    //增
    animal.add("松鼠");
    animal.add(1, "花猪");
    //删
    animal.remove(0);
    animal.remove("花猪");
    //查
    System.out.println(animal.get(1));//根据位置获取对象
    System.out.println(animal.indexOf("松鼠"));//查对象的位置
    //改
    animal.set(0, "猫");
    
    
    //获取大小
    System.out.println(animal.size());
    
    //判断是否包含
    System.out.println(animal.contains("小狗"));
    
    //把另一个容器所有对象都加进来
    ArrayList human = new ArrayList();
    animal.addAll(human);
    
    //转为数组，类型要是一样的哦
    String[] array = new String[animal.size()];
    animal.toArray(array);
}
```

# IO流（待完善）
