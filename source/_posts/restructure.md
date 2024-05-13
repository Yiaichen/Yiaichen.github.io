---
title: 重构 (附源码)
date: 2018-08-02 16:46:00
tags: [Java,重构]
categories: Java
---

### 定义

**个人理解**：*计算机理解代码很简单,因为不管你怎么写终究会编译成字节码,所以重构的定义简单点来说就是让你写的代码能让其他人看懂。*

**定义（《重构》作者）**：*对软件内部结构的一种调整,目的是在不改变软件可观察行为的前提下,提高其可理解性,降低其修改成本。*


### 案例

代码大家都知道怎么写,多余的也不多说了,直接上干(dao)货(ban)实例非常简单,这是一个影片出租店用的程序,计算每一位顾客的消费金额并打印详单。
操作者告诉程序：租客租了那些影片、租期多长,程序便根据租赁时间和影片类型来算出费用。

影片（Movie）分三类：常规片、儿童片、新片,出了计算费用还需要为常客计算积分,积分会根据租的影片种类是否为新片会有所不同。


```java
/**
 * 影片
 *
 * @author vayi
 * @date 2018/7/30
 * @since 0.0.1
 */
public class Movie {

    public static final int regular = 0; //常规片
    public static final int new_release = 1; //新片
    public static final int childrens = 2; //儿童片

    private String title;
    private int priceCode;

    public Movie(String title, int priceCode) {
        super();
        this.title = title;
        this.priceCode = priceCode;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public int getPriceCode() {
        return priceCode;
    }

    public void setPriceCode(int priceCode) {
        this.priceCode = priceCode;
    }

}
```

```java
/**
 * 租赁
 *
 * @author vayi
 * @date 2018/7/30
 * @since 0.0.1
 */
public class Rental {
    
    private Movie movie; // 租的电影
    private int dayRented; // 租的时间

    public Rental(Movie movie, int dayRented) {
        super();
        this.movie = movie;
        this.dayRented = dayRented;
    }

    public Movie getMovie() {
        return movie;
    }

    public int getDayRented() {
        return dayRented;
    }

}
```

```java
/**
 * 消费者
 *
 * @author vayi
 * @date 2018/7/30
 * @since 0.0.1
 */
public class Customer {

    private Vector rentals = new Vector(); // 存租的影片
    private String name;

    public Customer(String name) {
        super();
        this.name = name;
    }

    public void addRental(Rental arg) {
        rentals.addElement(arg);
    }

    public String getName() {
        return name;
    }

    public String statement() {
        double totalAmount = 0;
        int frequentRenterPoints = 0;
        Enumeration rentalss = rentals.elements();
        String result = "RentalNew Record for" + " " + getName() + "\n";
        while (rentalss.hasMoreElements()) {
            double thisAmount = 0;
            Rental each = (Rental) rentalss.nextElement();

            switch (each.getMovie().getPriceCode()) {
                case Movie.regular:
                    thisAmount += 2;
                    if (each.getDayRented() > 2)
                        thisAmount += (each.getDayRented() - 2) * 1.5;
                    break;

                case Movie.new_release:
                    thisAmount += each.getDayRented() * 3;
                    break;

                case Movie.childrens:
                    thisAmount += 1.5;
                    if (each.getDayRented() > 3)
                        thisAmount += (each.getDayRented() - 3) * 1.5;
                    break;
            }

            //积分  每借一张加1个积分
            frequentRenterPoints++;
            //积分累加条件  新版本的片子,借的时间大于1天
            if ((each.getMovie().getPriceCode() == Movie.new_release) && each.getDayRented() > 1) {
                frequentRenterPoints++;
            }

            result += "\t" + each.getMovie().getTitle() + "\t"
                    + String.valueOf(thisAmount) + "\n";

            totalAmount += thisAmount;
        }

        result += "Amount owed is " + String.valueOf(totalAmount) + "\n";
        result += "You earned " + String.valueOf(frequentRenterPoints) + " "
                + "frequent renter points";
        return result;
    }
}
```

Customer里的statement()方法就是生成详单的函数,也是我们本次重构的入口。那重构之前我们先思考几个问题,你第一眼看到这个statement()方法的时候是怎么想的？
首先肯定会说它设计不好,因为没有面向对象实思想,并且后期维护起来也很费人力。
**假如我们的用户需求有修改,不是打印txt这种格式的文本,而是输出Json或者Html格式的内容的话**,你会发现很难修改,然后往往就是复制一份,接着改改改,编译通过,能跑就ok了
这样确实可以,**但如果计费的标准也发生变化了呢?**,那这个时候你得同时修改两个方法,
并且还要保证两处修改的一致性,如果后续还需要修改的话就会越积累越多,CV大法的问题就浮现出来了,所以这个时候就需要重构来拯救了。（越早重构越好,没事的时候就看看代码还能不能重构）


注：如果你发现自己需要为程序添加一个特性,而代码结构让你没法很方便的添加的时候,那么就先
    重构这个程序,使特性可以很方便的添加了再添加特性
    
#### 开始

首先开始之前给大家看看同一段代码重构前后的对比
<img src="/img/restructure/restruct.png">

代码量少了80%左右,并且结构更清晰了,可扩展性更好,耦合性更低了
  

#### 重构第一步
建立测试类

```java
/**
 * 测试类
 *
 * @author vayi
 * @date 2018/7/30
 * @since 0.0.1
 */
class test01 {

    public static void main(String[] args) {
        System.out.println("=========================重构前结果=========================");
        Movie mov = new Movie("xxx", 2);
        Rental ren = new Rental(mov, 8);
        Customer cus = new Customer("Cheng");
        cus.addRental(ren);

        System.out.println(cus.statement());
        System.out.println("=========================重构前结果=========================");

        System.out.println("=========================重构后结果=========================");
        MovieNew newMov = new MovieNew("xxx", 2);
        RentalNew newRen = new RentalNew(newMov, 8);
        CustomerNew cusNew = new CustomerNew("Cheng");
        cusNew.addRentalNew(newRen);

        System.out.println(cusNew.statement());
        System.out.println("=========================重构后结果=========================");
    }

}
```

切记：重构第一步先建立相应部分的测试类,一定不能影响原来结果的运行,并且重构一块的时候
     就需要测试一次,看结果是否一致。
     
#### 分解并重组statement()

首先我们需要找到重构部分的逻辑泥团,很明显statement()里面的逻辑泥团就是这个switch语句,那我们就把这个地方单独提出一个函数来进行计算,
首先找出这个函数内部的局部变量跟参数,我们可以找到一个是each一个是thisAmount,前者并没有被修改,后者会被修改,所以这里我们尽量把不用修改的当参数传递进去,
如果是会被修改的当参数就要认真考虑是否可行了,如果只有一个变量会被修改,那我们可以把它当作返回值

<img src="/img/restructure/amountFor.png">

直接用IDEA的话可以用 **CTRL+ALT+M** 组合键来提取选中的内容为方法
然后我们也把函数内的变量名修改下,增加可阅读性,同样是可以用快捷键来修改变量名 **SHIFT+F6**

<img src="/img/restructure/changeName.png">

每次重构完一部分,哪怕很小的一部分也要先测试一遍,只有编译测试通过了才可以进行下一步的重构
我们继续看这个提取出来的amountFor()函数,发现里面只用到了Rental类相关的操作,但是却并没有Customer类的操作,
所以我们怀疑这里是不是放错了位置,我们把amountFor移动到Rental里面,顺便方法名也改为getCharge(),同样的移动方法也有快捷键: **F6**

<img src="/img/restructure/getCharge.png">

这里就先对getCharge的操作到此为止了,我们再回到statement()函数来
这个时候我们已经把switch提取出来了,我们可以看到thisAmount这个变量接收一次getCharge的结果之后就没有改变了
那么我们完全可以直接用getCharge来替代它

<img src="/img/restructure/removeAmount.png">

同样的,重构完之后编译测试一次,保证自己没有破坏任何东西
然后回到我们的statement()函数,发现我们的积分计算也跟Rental有关,所以我们可以直接放到Rental类里面去
由于这里的积分变量frequentRenterPoints有了初始值，并且是用来统计的，所以我们不用当参数传递进去直接接收返回值进行累加就可以了

<img src="/img/restructure/getFre.png">
<img src="/img/restructure/getFrequentRenterPoints.png">



### 总结

[源码地址](https://github.com/Yiaichen/javaDemo/tree/master/javaDemo)
重构部分的源码在restructure目录下,还有一些其他demo,大家可自行学习


