[toc]



# 

## 字符串变成字符数组	

```java
String a ="adfsa";
char[] c =a.toCharArray();
```

## 字符数组变成字符串

String.valueOf(a)  或者 new String(a) //a是一个字符数组

## String的构造方法

```java
new String(字符数组，起始位置，长度)
```

## Arrays.copyOfRange()复制数组一部分

```java
Arrays.copyOfRange(原数组，起始坐标，结束坐标)  //左闭右开
```

## getOrDefault()

```java
map.getOrDefault(sub,0)  //是Map的方法，获取key为sub的值没有就返回0
```

## Deque是一个双端队列，但也实现了Stack的功能

## ArrayList的add方法中

```java
ans.add(1);  //正常的使用
ans.add(index,值);  // 可以插入到指定下标处
```

## 有序集合TreeSet的ceiling()和remove()方法

```java
//TreeSet是一个有序集合
TreeSet<Long> ans = new TreeSet<Long>();
ans.add(121);
ans.add(22);
ans.ceiling(23); //ceiling()方法返回有序集合中大于等于23的最小值，没有就返回null
ans.remove(121);
```

## DecimalFormat格式化

```java
DecimalFormat df = new DecimalFormat("0000");
return df.format(num);  //num原本是101，返回的就是“0101”
```

## 字节流与字符流

```java
//字节流转换成字符流都需要经过一个转换流，这里写出就是OutputStreamWriter
private PrintWriter writer = new PrintWriter(new OutputStreamWriter(new FileOutputStream(path),"UTF-8"));
//这是读取流
private BufferedReader reader =new BufferedReader(new InputStreamReader(new FileInputStream(path),"utf-8"));
```

## DecimalFormat将数据转换成字符串

```java
DecimalFormat df = new DecimalFormat(“格式”); //比如“0000”
return df.format(num);
```

## Arrays.asList将数组转换成list

## 字节流转变成字符流

![image-20210518145644276](/Users/ding/Library/Application Support/typora-user-images/image-20210518145644276.png)

## 整数变成字符串

```java
char[] strN = Integer.toString(N).toCharArray();
```





# 线程

```java
// start a thread to check and dynamic load functions
loadExternalFunctionExecutor = newSingleThreadScheduledExecutor(threadsNamed("load-external-functions-%s"));
loadExternalFunctionExecutor.scheduleWithFixedDelay(() -> {
    try {
        if (hiveFunctionsPlugin.isExternalFunctionsUpdate()) {
            installFunctionsPlugin(hiveFunctionsPlugin);
            hiveFunctionsPlugin.setExternalFunctionsUpdate(false);
        }
    }
    catch (Exception e) {
        log.error(e, "Error load external functions");
    }
}, 600, 300, TimeUnit.SECONDS);
```



## TreeMap

https://www.jianshu.com/p/2dcff3634326



# 不会的

new SetThreadName