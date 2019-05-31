---
layout: post
title:  "java 中的 equals 和 hashcode"
author: recaton
categories: [ java ]
#image: assets/images/15.jpg
---

在java中，重写一些类的的```equals```方法一定要同时重写```hashcode```方法，之前一直是一知半解，最新闲了下来，是时候弄弄清楚，充实一下自己了。

## 为什么要重写equals
```equals``` 和 ```hashcode``` 方法都是 ```Object``` 对象中提供的方法。
* ```Object``` 类中 ```equals``` 实现为```return (this == obj)```，只有两个内存地址相同的对象才认为是相等的
* ```Object``` 类中 ```hashcode``` 是 ```native``` 方法，返回一个基于内存地址计算出来的整数值

如果需要在业务上判断两个对象是否相当，那么默认的 ```equals``` 方法做不到了。

例如，存在以下 ```User``` 对象，只要 ```firstName``` 和 ```lastName``` 相等就认为 ```User``` 是相等的
```java
public class User {
    private firstName;
    private lastName;
}
```

此时我们就必须重写 ```equals``` 方法。

## 重写equals
重写 ```User``` 也很简单
```java
public class User {
    private firstName;
    private lastName;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        User that = (User) o;
        if (!Objects.equals(firstName, that.firstName)) return false;
        return Objects.equals(lastName, that.lastName);

    }
}
```

## 新的需求
截止目前，```User``` 可以很好地工作。

直到有一天，我们遇到了新的需求：

每个 ```User``` 对应一个 ```String``` 类型的 address，我们需要将这个对应关系维护在 ```HashMap``` 中。
```java
public HashMap<User, String> buildUser() {
    HashMap<User, String> map = new HashMap<>();
    map.put(new User("Jack", "Ma"), "Hangzhou");
    map.put(new User("Jack", "Ma"), "Shanghai");
    return map;
}
```

因为我们重写了 ```User``` 的 ```equals```方法，所以上述 ```map``` 中应该只有第二次 put 的值。

现在我们测试一下：
```java
System.out.println(buildUser());
// 以下是输出
// {recaton.study.saop.User@11028347=Shanghai, recaton.study.saop.User@707f7052=Hangzhou}
```

可以发现，结果并不是我们预期的。那么原因是什么呢？

## HashMap 不工作的原因
```HashMap``` 内部有一个 ```hash``` 函数，根据 key 的 ```hashcode``` 计算出一个整数值，进而确定 key 应该存储在哪个 bucket 中。
```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

```buildUser``` 先后将两个新创建出来的 ```User``` 对象 put 到 map 中，而 ```User``` 的 ```hashcode``` 方法没有被重写，他们的 ```hashcode``` 肯定不一样（因为内存地址不一样），进而 ```HashMap``` 内部存储时找到的 bucket 也不一样，所以 ```HashMap``` 中有两个元素。

现在测试一下重写 ```hashcode```
```java
    @Override
    public int hashCode() {
        int result = firstName != null ? firstName.hashCode() : 0;
        result = 31 * result + (lastName != null ? lastName.hashCode() : 0);
        return result;
    }
```
```java
System.out.println(buildUser());
// 以下是输出
// {recaton.study.saop.User@4406d95=Shanghai}
```
可以发现，结果已经是我们预期的。

## 为什么一定要重写hashcode
综上，可以看出，如果在重写 ```equals``` 时不重写 ```hashcode``` 方法，可能会造成 ```HashMap``` 工作异常（当类作为key时），推而广之，其他以 ```hashcode``` 为基础实现的集合类（如 ```HashTable```）也可能无法工作，所以在重写 ```euqals``` 时一定要重写 ```hashcode``` 方法。

## 后续思考
回过头来总结一下，其实是一定要保证 ```equals``` 和 ```hashcode``` 的"判断标准"一致，就像 ```Object``` 中都使用内存地址作为判断标准，```User``` 中都使用属性值作为判断标准。