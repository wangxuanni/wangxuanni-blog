---
title: javaSE（上）
date: 2018-10-6 21:00:50
categories: java
description: 不要小瞧基础呀!
top: true
---

# 封装、继承、多态
## 封装
访问修饰权限：
private（私有）, default（同一包）, public（共有）, protected（同一包和子孙类）
如果不写修饰符默认用default（package/friendly）修饰。
**default和protected区别：protected能访问子孙类，default不能。**

<u>什么情况用什么修饰符？</u>
1. 属性通常使用private封装起来
2. 方法一般使用public用于被调用
3.**会被子类继承的方法，通常使用protected**

## 继承

接口和抽象类的主要区别？
从概念上来说
抽象类是一种对**事物的抽象**，而接口就像是一种约定，是一种对**行为**的抽象；
抽象类是对整个类整体进行抽象，包括属性、行为，但是接口却是对类局部（行为）进行抽象。
抽象类是一种模板式设计，而接口是一种**行为规范**，是一种辐射式设计。

模板式设计：如果B和C都使用了公共模板A，如果他们的公共部分需要改动，那么只改动A就可以了；
辐射式设计：如果B和C都实现了公共接口A，如果现在要向A中添加新的方法，那么B和C都必须进行相应改动。

区别1：
**一个类只能继承一个父类。
一个类可以实现多个接口**

区别2：
抽象类可以定义
public,protected,package,private、静态和非静态属性、final和非final属性
但是接口中声明的属性，只能是public、静态、final的

问题：Override(重写)和Overload(重载)的区别?Overload能改变返回值类型吗?

子类**重写**父类方法：子类的方法名与父类的一样，但是参数类型不一样。
重载：**本类**中出现的方法名一样，参数列表不同的方法。
与返回值类型无关。

思考：如果没有重写这样的机制，会发生什么？
答：一旦继承了父类，**所有方法都不能修改了**。另外，对象调用方法的时候，先找子类本身的方法，再找父类。(就近原则)

隐藏，就是子类覆盖父类的**类方法**。（重写是子类覆盖父类的对象方法 ）


为什么Java语言不支持c++所有的多重继承?
多重继承有它的弊端。
1)多重继承存在**二义性**。比如，类C同时继承类A和类B,如果类A和类B中都有方法f,那么调用类C的的f方法时,无法确定是调用类A还是类B的方法,将会产生二义性。但是Java语言却可以通过实现多个接口的方式间接地支持多重继承,由于接口只有方法体,没有方法实现,假设类C实现了接口A和接口B，即使AB都有f方法，但接口只有定义没有实现，在C中才有一个方法的实现，也就不存在二义性了。
2）多重继承会使得类型转换，构造方法的调用顺序变得非常复杂。








## 多态
**操作符**的多态 
加号+可以作为算数运算，也可以作为字符串连接。不同情境下，具备不同的作用
如果+号两侧都是整型，那么+代表数字**相加**
如果+号两侧，任意一个是字符串，那么+代表**字符串连接**

**类**的多态 比如**父类的引用指向子类的对象**、重写。


<u>下面程序的运行结果是？</u>
```
class Base
{
    int num = 1;
    public Base(){
        this.print();
        num=2;
    }
    public void print(){
        System.out.println("Base.num="+num);
    }
}
class Sub extends Base{
    int num= 3;
    public Sub(){
        this.print();
        num=4;
    }
    public void print(){
        System.out.println("Sub.num="+num);
   }
}
public class Test1 {
  public static void main(String[] args)
  {
      Base b = new Sub();
      System.out.println(b.num);
  }
}
```

在执行语句 Base b= new Sub时,会首先调用父类的构造方法,而在父类构造方法中调用了print()方法。
根**据多态的特性,此时实例化的是sub类的对象,因此,会调用Sub类的print()方法。由于此时Sub类中的初始化代码 Int num=3还没有执行,因此,num的默认值为0,所以,输出为 Sub num=0。接着下一条语句父类num初始化为2。**
然后会调用子类的构造方法,根据初始化的顺序可知在调用子类构造方法时,非静态的变量会先执行初始化动作,所以,此时子类Sub的mum值为3,因此,调用 print方法会输出 Sub num=3。
接着输出b.num,由于b的类型为Base,**而属性没有多态的概念**因此,此时会输出父类中的mm值:2
程序的运行结果如下：
Sub.num=0
Sub.num=3
2
题目总结：当父类的引用指向子类的对象时，会先初始化父类。如果子类重写某方法，不管父类子类都是调用子类方法。另外属性没有多态的概念。



## 其他
### java初始化原则
### 对象属性初始化方法有3种
1. 声明该属性的时候初始化 
2. 构造方法中初始化
3. 初始化块

如果同时初始化同一变量，则优先级是：构造方法中初始化>初始化块> 声明该属性的时候初始化 

静态成员变量>成员变量>构造方法。
1静态变量优先与非静态变量
2父类优先于子类
3按照成员变量定义的顺序。
父子类的初始化执行顺序如下：
父类静态变量，父类静态代码块
子类静态变量，子类静态代码块
父类非静态变量，父类非静态代码块，父类构造方法
子类非静态变量，子类非静态代码块和子类构造函数。

    
### this关键字
 this关键字代表自身实例
```
 //参数名和属性名一样
    //在方法体中，只能访问到参数name
    public void setName1(String name){
        name = name;
    }
     
    //为了避免setName1中的问题，参数名不得不使用其他变量名
    public void setName2(String heroName){
        name = heroName;
    }
     
    //通过this访问属性
    public void setName3(String name){
        //name代表的是参数name
        //this.name代表的是属性name
        this.name = name;
    }
```
### 实参与行参的问题
<u>猜一猜程序运行结果</u>
```
public class Test {

    public void change(int j, StringBuffer ss1) {
        j = 100;
        ss1.append("world");
    }

    public static void main(String[] args) {
        int i = 1;
        StringBuffer s1 = new StringBuffer("hello ");
        Test t = new Test();
        t.change(i, s1);
        System.out.println(i);//1处
        System.out.println(s1);//2处
    }
}
```


由于i是基本类型，因此参数是**按值**传递。**会创建一个i的副本**。把这个副本作为参数赋值给j。既然j是i的副本，那么对副本的任何修改都不会对i有影响。因此1处输出1。
由于StringBuffer是一个类，因此是按**引用**传递。当ss1修改的时候。由于实参s1和形参ss1指向的是同一块储存空间，因此ss1修改了值之后，s1指向的字符串也被修改了。因此2处输出hello world。
那么，下面程序运行结果又是什么
```

public class Test {

    public void change(StringBuffer ss1) {
       ss1 = new StringBuffer("world");
    }

   public static void main(String[] args) {
   StringBuffer s1 = new StringBuffer("hello ");
        Test t = new Test();
        t.change(s1);
        System.out.println(s1);
    }}
```


由于StringBuffer是一个类，因此是按引用传递。**但是**这里的change方法不是对"hello "进行修改，而是使形参ss1的指向另一个字符串 “world”。而对形参ss1的改变对实参s1没有影响，实参s1仍然指向 "hello "。

<u>引用就是指针吗？</u>
不是，二者不能等同。虽然java引用在底层是通过指针实现的，但指针可以执行比较运算和整数的加减运算，而引用却不行。





### 关键字 final
修饰变量时，用以定义常量；
修饰方法时，方法不能被重写（Override）；
修饰类时，类不能被继承。


# 变量、数组、循环
## 变量
变量的范围
1.变量声明在类下，叫做**字段**或者**属性**或者**成员变量**
2.变量声明在一个方法上的，就叫做**参数**或者**局部变量**
 
数据类型    
八种基本类型(二进制位数)： 
 整型 4种： byte（8位）、short（16位）、int（32位）、long（64位）
 浮点型 2种： float（32位）、double （64位）
 字符型 1种： char（16位）
 布尔型 1种：boolean（1位）
 
 现在问题来了
 


<u>问：int和integer的区别？</u>
答：1)int默认值为0。而integer默认值为null。由此可见,证int无法区分未赋值与赋值为0的情况,而integer却可以区分这两种情况。
2)int是是值传递。而integer是引用传递
3)int只能用来运算,而integer提供了很多有用的方法
4)当需要往容器(例如List)里存放整数时,无法直接存放int,因为List里面放的都是对象,所以,在这种情况下只能使用 Integer

<u>问：char型变量中能不能存贮一个中文汉字?为什么?</u>
答：char是16位的，占两个字节
汉字通常使用GBK或者UNICODE编码，也是使用两个字节，可以正常存放汉字。如果是utf-8编码，一个中文占三个字节，编译不会报错。但运行会报error“未结束的字符文字”


<u>问：请解释这三条语句的输出</u>
```
  System.out.println((byte)127);//127
System.out.println((byte)128);//-128
System.out.println((byte)129);//-127
```
答：byte的取值是{-128,127},如果把128强制转换成byte已结超出了byte范围，此时会溢出，相当于最小的负数-128.而129强转后就是-127


<u>问：请解释这条语句的输出</u>
```
System.out.println(Math.min(Double.MIN_VALUE,0.0));//输出0.0
```
答：对于Double来说MIN_VALUE并不是取值范围的最小数，而是正数范围的最小数，也就是最接近于0的正数。最接近于0的正数和0比起来，当然是0小。



<u>问：在java里调用什么方法能把二进制数转化为十进制？</u>
答：
```
System.out.println(Integer.valueOf("11101",2));
```


### 类型转换
#### 强制转换   
```
int i1 = 10;
byte b = (byte) i1;
```


#### 自动装箱和拆箱
把基本数据类型和对应的包装类之间转换。比如int和Integer。
```
Integer i = 100;  //自动装箱，编译器执行Integer.valueOf(100)
int j = i;        //自动拆箱，编译器执行i.intValue()
```



#### ==与equals()
==比较的是两个对象的引用是否相同，或者是比较原始数据类型是否相等；
equals()比较的是两个对象的内容是否相同。

### 关于变量的题目

<u> 题目一：解释为何行3编译错误，而行4编译正确</u>
```
        byte b1 = 3;
        byte b2 = 4;
        byte b3 = b1 + b2;                //编译错误
        byte b4 = 3 + 4;                //编译正确
//（1）变量相加，首先首先进行类型提升，之后再进行计算，计算后将结果赋值；
//（2）常量相加，首先进行计算，之后判断是否在接受类型的范围，在则赋值。
```
 


<u>
题目二：判断下列代码是否有误，并指出错误</u>
``` 
        short s = 1;
        s = s + 1;                //错误，s在参加运算时会自动提示类型为int。int类型值无法直接赋值于short类型
        
        short z = 1;
        z += 1;                //正确,扩展赋值运算符包含强制类型转换。等价于 z = (short)(z + 1);
//还有，-128~127的Integer值可以从缓存中取得。其他情况要重新创建
```



<u>题目三，int i = 1;i+=++i;的运算结果？</u>
i+=++i,其中先算++i,得到2
由于++i并未进行赋值，所以i还是1
1+=2结果为3


## 数组
一维数组的3种创建方式
```
int[] arr1 = {1,2,3,4};             //正确
int[] arr2 = new int[4];            //正确
int[] arr3 = new int[]{1,2,3,4};    //正确
int[] arr4 = new int[4]{1,2,3,4};  //错误，编译不通过
```

二维数组的3种声明方式
```
int arr1[][];
int [][]arr2;
int []arr3[];
```
与C/C++不同的是，java的二维数组允许第二维的长度可以不同。
## 循环
### for循环
<u>写出下面程序运行结果</u>
```
public class Test {

   static boolean p(char c) {
       System.out.print(c);
       return true;
    }

    public static void main(String[] args) {
       int i=0;
        for (p('a'); p('b')&&i<2; p('c')) {//for(表达式1;表达式2;表达式3){循环体]
            p('d');
            i++; } }}
```

因为，1.初始化只会执行一次2.先执行循环体后执行for循环的表达式3
所以答案是：abdcbdcb

### switch
```
 switch(day){
            case 1:
                System.out.println("星期一");
                break;
            case 2:
                System.out.println("星期二");
                break;
            case 3:
                System.out.println("星期三");
                break;
            case 4:
                System.out.println("星期四");
                break;
            case 5:
                System.out.println("星期五");
                break;
            case 6:
                System.out.println("星期六");
                break;
            case 7:
                System.out.println("星期天");
                break;
            default:
                System.out.println("输入有误");
        }
```
**使用switch特别注意，必须在case语句后加break **   

### 枚举
```
public enum Season {//是枚举enum不是类class
            SPRING, SUMMER, AUTUMN, WINTER;//直接这样写

            public static void main(String[] args) {
                Season season = Season.SPRING;
                switch (season) {
                    case SPRING:
                        System.out.println("春天");
                        break;
                    case SUMMER:
                        System.out.println("夏天");
                        break;
                    case AUTUMN:
                        System.out.println("秋天");
                        break;
                    case WINTER:
                        System.out.println("冬天");
                        break;
                }}}
```

#### 如何跳出多重循环？
在外部循环的前一行，加上自定义标签，比如 out:
在break的时候使用该标签。break out;




# 内部类

## 非静态内部类
可以被看作外部类的一个成员（与类的属性和方法类似）
1可以自由的引用外部类的属性和方法。2外部类被实例化之后，内部类才能被实例化。3不能有静态成员。

```
public class Hero {//英雄类
    private String name; 
    float hp; 
    // 非静态内部类，只有一个外部类对象存在的时候，才有意义
    // 比如战斗成绩只有在一个英雄对象存在的时候才有意义
    
    class BattleScore {//战斗成绩类
        int kill;

        public void legendary() {
            if (kill >= 8)
                System.out.println(name + "超神！");
            else
                System.out.println(name + "尚未超神！");
        }
    }
 
    public static void main(String[] args) {
        Hero garen = new Hero();
        garen.name = "盖伦";
        
        // BattleScore对象只有在一个英雄对象存在的时候才有意义
        BattleScore score = garen.new BattleScore();// 所以其实例化必须建立在一个外部类对象的基础之上
                                                    
        score.kill = 9;
        score.legendary();
    }
 
}
```

## 静态内部类。
1不能访问外部类普通成员，只能访问外部内中静态成员和静态方法。
2可以不依赖于外部类实例化而实例化。
3不可以与外部类同名。
## 局部内部类
是指定义在一个代码块内的类，不可以被修饰符修饰。
## 匿名内部类
是一种没有类名的内部类，不使用关键字class、extends、implement，没有构造方法，必须继承类或其他接口。一般用于gui编程中事件处理。

```
public abstract class Hero{

    public abstract void attack();

    public static void main(String[] args) {
        Hero h = new Hero(){
            //当场实现attack方法
            public void attack() {
                System.out.println("新的进攻手段");
            }
        };
        h.attack();
        //通过打印h，可以看到h这个对象属于Hero$1这么一个系统自动分配的类名
        System.out.println(h);
    }}


```


# 字符串



## 格式化
%s 表示字符串
%d 表示数字
%n 表示换行
```
        String name ="盖伦";
        int kill = 8;
        String title="超神";
         
        String sentenceFormat ="%s 在进行了连续 %d 次击杀后，获得了 %s 的称号%n";
        //使用printf格式化输出
        System.out.printf(sentenceFormat,name,kill,title);//第一个是原字符串
```

## StringBuffer追加 删除 插入 反转
```
String str1 = "let there ";
StringBuffer sb = new StringBuffer(str1); //根据str1创建一个StringBuffer对象
sb.append("be light"); //在最后追加
sb.delete(4, 10);//删除4-10之间的字符
sb.insert(4, "there ");//在4这个位置插入 there
sb.reverse(); //反转
```

## 字符串的转化

### 数字与字符串
- 数字转字符串
方法1： 使用String类的静态方法valueOf 
String str = String.valueOf(i);
方法2： 先把基本类型装箱为对象，然后调用对象的toString
Integer it = i;
String str2 = it.toString();


- 字符串转数字
String str = "999";
int i= Integer.parseInt(str);

### 字符串与字符串数组
- 字符数组 转 字符串
char[] data={'a','b','c'};
String s=new String(data);

字符数组转换成字符串
```
char[]   data={'a','b','c'};   
String  s=new   String(data);
```


s1.charAt(0)=c1;是不行的
s1.charAt(0);是可以的

## 字符串是否相等的提问
问：1和2处分别输出什么?
```
        String str1 = "the light";
        String str2 = new String(str1);
        System.out.println( str1  ==  str2);//1
       String str4 = new String("the light");//2

        String str3 = "the light";
        String str4 = "the light";
        System.out.println( str4  ==  str3);//3
```
答：1输出false。因为new String会为str2开辟一个新的区域.
2输出true。因为str3创建了一个新的字符串"the light"，在str4编译器发现已经存在现成的"the light"，那么就直接拿来使用，而没有进行重复创建
# 时间
## 格式化时间
时间（注意月和小时是大写的喔）

y 代表年
M 代表月
d 代表日
H 代表24进制的小时
h 代表12进制的小时
m 代表分钟
s 代表秒
S 代表毫秒
```
Date d = new Date();
System.out.println(d);//输出当前时间
SimpleDateFormat sdf = new SimpleDateFormat("yyyy年MM月dd日,HH时mm分ss秒");//设置样式
String s = sdf.format(d);//把d这个时间实例格式化，并且赋值给字符串s
System.out.println(s);//输出的字符串s就是格式化好的时间啦
```


## 日历（可以做“查看明年的今天是几号”之类的事）
```
Calendar c = Calendar.getInstance();//Calendar用单例模式创造实例
Date now = c.getTime();

c.setTime(now);
//先翻到下下个月
c.add(Calendar.MONTH,2);
//设置到月初
c.set(Calendar.DATE,1);
//再往回翻3天
c.add(Calendar.DATE,-3);

System.out.println("下个月的倒数第3天是哪天"+s);
```

# 一些基础但不重要的知识
- 变量名起名规则
    1. 字母 数字 $ _ 组成
    2. 变量第一个字符，不能使用数字。
    3. 不可以使用关键字。
为什么不重要？
起名的时候选择有意义描述性的词比如“toString”"getFlow"，不要用关键字。起名规则无需死记硬背。