---
title: 一个栗子搞懂代理模式
date: 2018-11-24 22:12:30
tags: [设计模式]
categories: 设计模式
---

## 概念
> 代理模式，为其他对象提供一种代理以控制对这个对象的访问。在某些情况下，一个对象不适合或者不能直接引用另一个对象，而代理对象可以在客户端和目标对象之间起到中介的作用。

### UML
<img src="/img/proxy.png">

### 三种代理模式
- 静态代理
- JDK动态代理
- CGLib动态代理

## 举个栗子
> 假设我现在想买一台Switch，但是现在国内又买不到怎么办？那就只能找代购了，在代理模式的角度来看的话，这里的`我`就是`真实的对象(RealSubject)`，`代购`就是一个`代理对象(Proxy)`，`买Switch`就是抽象对象的行为(Subject)

### 静态代理
Subject：
``` java
/**
 * 抽象对象接口
 */
public interface Subject {

    /**
     * 声明需要被代理的方法
     * 让代理对象来帮我们买一台Switch
     */
    public void buySwitch();

}
```
RealSubject：
``` java
/**
 * 真实对象
 */
public class RealSubject implements Subject {

    /**
     * 我只想买一台Switch
     */
    @Override
    public void buySwitch() {
        System.out.println("我只想买一台Switch");
    }

}
```
Proxy：
``` java
**
 * 代理对象（代购）
 */
public class Proxy implements Subject {

    @Override
    public void buySwitch() {
        //引用并创建真实对象实例
        RealSubject realSubject = new RealSubject();

        //调用真实对象的方法，进行代理购买Switch
        realSubject.buySwitch();

        //代购进行一些额外的操作
        this.buyGameCard();
    }

    /**
     * 顺便买一些游戏卡
     */
    private void buyGameCard() {
        System.out.println("再顺便买一些游戏卡");
    }

}
```
Client：
``` java
/**
 * 客户端调用
 */
public class Run {
    public static void main(String[] args){
        Subject proxy = new Proxy();
        proxy.buySwitch();
    }
}
```
Result：
``` console
我只想买一台Switch
再顺便买一些游戏卡
```


优点：方便、快捷
缺点：每一个真实对象要有一个对应的代理类，并且行为多了会比较冗余，不满足`开闭原则`
> 开闭原则：对扩展开放，对修改关闭。

### 栗子延伸
> 假设这个时候买了Switch，又想玩其他的游戏。比如只狼啥的，那就得再买个PS4，按照我们刚刚的静态代理来看的话，那就得在`抽象行为对象(Subject)`里面再加一个`buyPS4()`的方法，并且对应的对象跟代理都需要进行实现这个方法，要是再多几个其他的也就太冗余了，这个时候就得考虑是不是可以`动态的进行代理`了？

### JDK动态代理 
真实对象跟抽象行为对象添加`buyPS4()`方法，修改抽象对象为DynamicHandler
DynamicHandler：
``` java
/**
 * JDK动态代理
 */
public class DynamicHandler implements InvocationHandler {
    // 真实对象
    private Object targetObject;

    // 创建代理对象 这段也可以不在此类，也可以放在客户端里面
    public Object createProxy(Object targetOjbect) {
        this.targetObject = targetOjbect;
        /*
         * 创建代理对象
         * Proxy.newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler h)
         * loader：代理类的类加载器
         * interfaces：指定代理类所实现的接口
         * h：动态代理对象在调用方法的时候，关联的InvocationHandler对象
         */
        return Proxy.newProxyInstance(targetOjbect.getClass().getClassLoader(),
                targetOjbect.getClass().getInterfaces(), this);
    }

    /**
     * InvocationHandler接口所定义的唯一的一个方法，该方法负责集中处理动态代理类上的所有方法的调用。
     * 调用处理器根据这三个参数进行预处理或分派到委托类实例上执行
     *
     * @param proxy  代理类的实例
     * @param method 代理类被调用的方法
     * @param args   调用方法的参数
     * @return Object
     * @throws Throwable
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        //触发真实对象之前或者之后可以做一些额外操作
        Object result = null;
        System.out.println("method:" + method.getName() + " proxy:" + proxy.getClass().getName());
        result = method.invoke(this.targetObject, args);//通过反射执行某个类的某方法
        this.buyGameCard();
        return result;
    }

    /**
     * 顺便买一些游戏卡
     */
    private void buyGameCard() {
        System.out.println("再顺便买一些游戏卡");
    }

}
```

Client：
``` java
/**
 * 客户端调用
 */
public class Run {
    public static void main(String[] args) throws Exception  {
        DynamicHandler dynamicHandler = new DynamicHandler();
        Subject subject = (Subject) dynamicHandler.createProxy(new RealSubject());
        subject.buySwitch();
        subject.buyPS4();
    }
}
```
Result：
``` console
我只想买一台Switch
再顺便买一些游戏卡
我想再买一台PS4
再顺便买一些游戏卡
```

> 疑问：为啥请求subject的对象会跑到DynamicHandler里面执行invoke()方法呢？


源码分析：
``` java
public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        Objects.requireNonNull(h);

        final Class<?>[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }

        /*
         * 生成指定的代理类
         */
        Class<?> cl = getProxyClass0(loader, intfs);

        /*
         * 调用代理类的构造函数
         */
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }

            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException|InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString(), e);
        }
    }
```

源码解析:
getProxyClass(loader, interfaces)创建代理类$Proxy0.$Proxy0类 实现了Subject接口,并继承了Proxy类. 

$Proxy0：
``` class
public final class $Proxy0 extends Proxy implements Subject {
    private static Method m1;
    private static Method m3;
    private static Method m2;
    private static Method m4;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void buySwitch() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void buyPS4() throws  {
        try {
            super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m3 = Class.forName("design.proxy.statics.Subject").getMethod("buySwitch");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m4 = Class.forName("design.proxy.statics.Subject").getMethod("buyPS4");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```
> 观察m3、m4这两个方法，然后看对应的方法里面都会有一个super.h.invoke()方法，这个`super.h`就是我们的`DynamicHandler代理对象`

步骤：
Proxy.newProxyInstance生成$Proxy0 -> $Proxy0调用bySwitch方法 -> DynamicHandler.invoke() -> 通过反射再调用真实对象请求的方法

### CGLib动态代理
``` java
public class DynamicHandler implements MethodInterceptor {

    private Object target;//业务类对象，供代理方法中进行真正的业务方法调用

    //相当于JDK动态代理中的绑定
    public Object getInstance(Object target) {
        this.target = target;  //给业务对象赋值
        Enhancer enhancer = new Enhancer(); //创建加强器，用来创建动态代理类
        enhancer.setSuperclass(this.target.getClass());  //为加强器指定要代理的业务类（即：为下面生成的代理类指定父类）
        //设置回调：对于代理类上所有方法的调用，都会调用CallBack，而Callback则需要实现intercept()方法进行拦
        enhancer.setCallback(this);
        // 创建动态代理类对象并返回
        return enhancer.create();
    }

    // 实现回调方法
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        proxy.invokeSuper(obj, args); //调用业务类（父类中）的方法
        this.buyGameCard();
        return null;
    }

    /**
     * 顺便买一些游戏卡
     */
    private void buyGameCard() {
        System.out.println("再顺便买一些游戏卡");
    }
}
```

Client：
``` java
/**
 * 客户端调用
 */
public class Run {
    public static void main(String[] args) throws Exception  {
        DynamicHandler handler = new DynamicHandler();
        RealSubject subject = (RealSubject) handler.getInstance(new RealSubject());
        subject.buySwitch();
        subject.buyPS4();
    }
}
```
Result：
``` console
我只想买一台Switch
再顺便买一些游戏卡
我想再买一台PS4
再顺便买一些游戏卡
```

原理跟JDK的类似，只不过不是继承Proxy了，而是通过Enhancer类操作节码生成代理对象来继承真实对象，然后进行方法拦截进入DynamicHandler的intercept()方法，因为是继承的真实对象所以真实的对象不能被final修饰。


### 区别
| · | 静态代理是 | JDK动态代理 | CGLib动态代理 |
| :---------:| :---------:| :---------: | :-------: |
| 优点| 方便、快，适合代理对象比较少的场景 | 易扩展，适合代理对象比较多得场景 | 易扩展，比反射调用方法快一点，没有太大的性能问题 |
| 缺点| 不好扩展，违反开闭原则 | 必须提供接口，并且因为通过反射来调用方法，消耗性能 | ASM操作生成类比较慢，真实对象不能为final |

## AOP中的应用
### AOP中的一些术语
- 1.通知(Advice):
通知定义了切面是什么以及何时使用。描述了切面要完成的工作和何时需要执行这个工作。
- 2.连接点(Joinpoint):
程序能够应用通知的一个“时机”，这些“时机”就是连接点，例如方法被调用时、异常被抛出时等等。
- 3.切入点(Pointcut)
通知定义了切面要发生的“故事”和时间，那么切入点就定义了“故事”发生的地点，例如某个类或方法的名称，spring中允许我们方便的用正则表达式来指定
- 4.切面(Aspect)
通知和切入点共同组成了切面：时间、地点和要发生的“故事”
- 5.引入(Introduction)
引入允许我们向现有的类添加新的方法和属性(Spring提供了一个方法注入的功能）
- 6.目标(Target)
即被通知的对象，如果没有AOP,那么它的逻辑将要交叉别的事务逻辑，有了AOP之后它可以只关注自己要做的事（AOP让他做爱做的事）
- 7.代理(proxy)
应用通知的对象，详细内容参见设计模式里面的代理模式
- 8.织入(Weaving)
把切面应用到目标对象来创建新的代理对象的过程，织入一般发生在如下几个时机:
(1)编译时：当一个类文件被编译时进行织入，这需要特殊的编译器才可以做的到，例如AspectJ的织入编译器
(2)类加载时：使用特殊的ClassLoader在目标类被加载到程序之前增强类的字节代码
(3)运行时：切面在运行的某个时刻被织入,SpringAOP就是以这种方式织入切面的，原理应该是使用了JDK的动态代理技术


### 拼多多版本AOP
``` java
/**
 * 前置增强
 */
public interface BeforeAdvice {
    public void before();
}
```
``` java
/**
 * 后置增强
 */
public interface AfterAdvice {
    public void after();
}
```
然后以JDK的动态代理为例子修改一下：
``` java
/**
 * JDK动态代理
 */
public class DynamicHandler implements InvocationHandler {
    // 真实对象
    private Object targetObject;
    //前值增强
    private BeforeAdvice beforeAdvice;
    //后置增强
    private AfterAdvice afterAdvice;

    public BeforeAdvice getBeforeAdvice() {
        return beforeAdvice;
    }

    public void setBeforeAdvice(BeforeAdvice beforeAdvice) {
        this.beforeAdvice = beforeAdvice;
    }

    public AfterAdvice getAfterAdvice() {
        return afterAdvice;
    }

    public void setAfterAdvice(AfterAdvice afterAdvice) {
        this.afterAdvice = afterAdvice;
    }

    // 创建代理对象 这段也可以不在此类，也可以放在客户端里面
    public Object createProxy(Object targetOjbect) {
        this.targetObject = targetOjbect;
        /*
         * 创建代理对象
         * Proxy.newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler h)
         * loader：代理类的类加载器
         * interfaces：指定代理类所实现的接口
         * h：动态代理对象在调用方法的时候，关联的InvocationHandler对象
         */
        return Proxy.newProxyInstance(targetOjbect.getClass().getClassLoader(),
                targetOjbect.getClass().getInterfaces(), this);
    }

    /**
     * InvocationHandler接口所定义的唯一的一个方法，该方法负责集中处理动态代理类上的所有方法的调用。
     * 调用处理器根据这三个参数进行预处理或分派到委托类实例上执行
     *
     * @param proxy  代理类的实例
     * @param method 代理类被调用的方法
     * @param args   调用方法的参数
     * @return Object
     * @throws Throwable
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        //触发真实对象之前或者之后可以做一些额外操作
        Object result = null;
        if(beforeAdvice != null) {
            beforeAdvice.before();
        }
        result = method.invoke(this.targetObject, args);//通过反射执行某个类的某方法
        if(afterAdvice != null) {
            afterAdvice.after();
        }
        return result;
    }

}
```
Client：
``` java
/**
 * 客户端调用
 */
public class Run {
    public static void main(String[] args) throws Exception  {
        DynamicHandler dynamicHandler = new DynamicHandler();
        Subject subject = (Subject) dynamicHandler.createProxy(new RealSubject());
        dynamicHandler.setBeforeAdvice(new BeforeAdvice() {
            @Override
            public void before() {
                System.out.println("听说任天堂出了一款不错的游戏，所以。。。");
            }
        });
        dynamicHandler.setAfterAdvice(new AfterAdvice() {
            @Override
            public void after() {
                System.out.println("再买点游戏卡");
            }
        });
        subject.buySwitch();
    }
}
```
Result：
``` console
听说任天堂出了一款不错的游戏，所以。。。
我只想买一台Switch
再买点游戏卡
```

这里我们只是简单的表明了一下前后通知配合动态代理的使用，真正的AOP涉及到切入点、切面什么时候织入等等。。想要了解的同学可以自行后续去研究，这里主要就是突出代理模式的应用。

## 总结
> 讲完这个栗子，总结一下，代理模式在Java中主要用于系统的解耦，如同中介机构，可以为目标类提供代理服务，以控制对对象的访问，目标类的任何方法在执行前都必须经过代理类，这样代理类就可以用来负责请求的预处理、过滤、将请求分派给目标类处理、以及目标类执行完请求后的后续处理。