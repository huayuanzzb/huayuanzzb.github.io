---
layout: post
title:  代理机制
date:   2019-01-23 08:25:26 +0800
nav_order: 3
parent: spring
categories: spring
---

Spring AOP 使用JDK动态代理或者cglib创建动态代理，那么Spring是如何选择使用哪种机制来创建代理对象呢？
1. 当设置强制使用cglib时，spring使用cglib生成代理对象；
2. 当不强制使用cglib时，如果目标对象至少实现了一个interface，则使用JDK动态代理，目标对象实现的所有接口方法都会被代理；
3. 当不强制使用cglib时，如果目标对象没有实现任何接口，则使用cglib生成代理对象。

综上所述，在两者都能使用时，spring默认优先选择JDK动态代理。

### 对比
#### JDK 动态代理
* JDK自带，不依赖于第三方库
* 只能基于接口生成代理对象
#### cglib
* 基于asm字节码技术，直接修改类的字节码，可以代理任何非final类，cglib的原理是生成一个目标类的子类，因此不能代理final类和final方法，因为final修饰的类或方法不能被覆写
* 代理对象的构造方法会被调用两次。




