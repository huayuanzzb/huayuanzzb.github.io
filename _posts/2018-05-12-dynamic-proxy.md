---
layout: post
title:  "动态代理"
author: recaton
categories: [ java, spring ]
#image: assets/images/2.jpg
rating: 3
---

Spring AOP 使用JDK动态代理或者cglib创建动态代理，那么Spring是如何选择使用哪种机制来创建代理对象呢？
1. 当设置强制使用cglib时，spring使用cglib生成代理对象；
2. 当不强制使用cglib时，如果目标对象至少实现了一个interface，则使用JDK动态代理，目标对象实现的所有接口方法都会被代理；
3. 当不强制使用cglib时，如果目标对象没有实现任何接口，则使用cglib生成代理对象。

综上所述，在两者都能使用时，spring默认优先选择JDK动态代理。
### 原理

#### JDK 动态代理

关键的类是```java.lang.reflect.Proxy``` 和 ```java.lang.reflect.InvocationHandler```，Proxy负责生成代理对象，对代理对象的调用会被dispatch到InvocationHandler.invoke方法中，在invoke方法中可以完成代理操作。

#### cglib
cglib 是一个开源库，它可以在java运行时于内存中创建并加载class文件。
关键类是```net.sf.cglib.proxy.MethodInterceptor``` 和 ```net.sf.cglib.proxy.Enhancer```，Proxy负责生成代理对象，对代理对象的调用会被dispatch到MethodInterceptor.intercept方法中，在intercept方法中可以完成代理操作。
### 对比

#### JDK 动态代理
* JDK自带，不依赖于第三方库
* 只能基于接口生成代理对象

#### cglib
* 基于asm字节码技术，直接修改类的字节码，可以代理任何非final类，cglib的原理是生成一个目标类的子类，因此不能代理final类和final方法，因为final修饰的类或方法不能被覆写
* 代理对象的构造方法会被调用两次(spring文档中提到的，自己的示例代码并没有测试出来)。

### 示例
```java
import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;
import org.objectweb.asm.ClassWriter;

import java.io.FileOutputStream;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class TestDynamicProxy {

    interface Greeting {
        void greeting(String name);
    }

    enum ProxyType {
        JDK, CGLIB
    }

    public static void main(String[] args) throws Exception{
        Greeting proxy = getProxy(ProxyType.JDK);
        proxy.greeting("Tom");
        System.out.println();
        proxy = getProxy(ProxyType.CGLIB);
        proxy.greeting("Jack");

    }

    static Greeting getProxy(ProxyType type) throws Exception {
        switch (type){
            case JDK: return (Greeting) new TestDynamicProxy.JDKDynamic().newProxy((Greeting) name ->
                    System.out.println(String.format("Hey, %s, this is from JDK dynamic proxy", name)));
            default: {
                Enhancer enhancer = new Enhancer();
                enhancer.setSuperclass(DogGreeting.class);
                // 生成的子类是否实现Proxy接口
                enhancer.setUseFactory(false);
                enhancer.setCallback(new CglibDynamic());
                Greeting proxy = (Greeting) enhancer.create();
                //以下代码是为了输出cglib生成的类的class文件
//                ClassWriter cw = new ClassWriter(0);
//                enhancer.generateClass(cw);
//                byte[] klass = cw.toByteArray();
//                FileOutputStream fileOutputStream = new FileOutputStream(proxy.getClass().getName()+".class");
//                fileOutputStream.write(klass);
//                fileOutputStream.close();
                return proxy;
            }
        }
    }

    static class JDKDynamic implements InvocationHandler{

        private Object target;

        Object newProxy(Object target){
            this.target = target;
            return Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), this);
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("JDK proxy: here is before invoking");
            Object r = method.invoke(target, args);
            System.out.println("JDK proxy: here is before invoking");
            return r;
        }
    }

    static class CglibDynamic implements MethodInterceptor{

        @Override
        public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
            System.out.println("cgLib proxy: here is before invoking");
            Object r = methodProxy.invokeSuper(o, objects);
            System.out.println("cgLib proxy: here is before invoking");
            return r;
        }
    }

    static class DogGreeting implements Greeting{

        @Override
        public void greeting(String name) {
            System.out.println(String.format("Hey, %s, this is from Cglib dynamic proxy" , name));
        }
    }

}

```
