# Optional的使用

JDK8中新增了`java.util.Optional`类，用起来消除NPE是非常爽的



## 方法列表

| 方法名                                               | 返回值      | 作用                                                         |
| ---------------------------------------------------- | ----------- | ------------------------------------------------------------ |
| empty()                                              | Optional<T> | 返回一个空的Optional                                         |
| of(T value)                                          | Optional<T> | 使用Optional封装一个不可空对象                               |
| ofNullable(T value)                                  | Optional<T> | 使用Optional封装一个可空对象                                 |
| get()                                                | T           | 获取封装的对象，如空则抛异常                                 |
| isPresent()                                          | boolean     | 判断封装的对象是否为空                                       |
| ifPresent(Consumer<? super T> consumer)              | void        | 如封装对象不为空，则调用传入的消费函数                       |
| filter(Predicate<? super T> predicate)               | Optional<T> | 根据指定过滤函数过滤封装对象                                 |
| map(Function<? super T, ? extends U> mapper)         | Optional<U> | 将当前对象转换成另一个Optional对象                           |
| flatMap(Function<? super T, Optional<U>> mapper)     | Optional<U> | 将当前对象转换成另一个Optional对象，转换函数的返回值是一个Optional对象 |
| orElse(T other)                                      | T           | 如当前封装对象不为空，返回当前封装对象，如为空，则返回指定值 |
| orElseGet(Supplier<? extends T> other)               | T           | 如当前封装对象不为空，返回当前封装对象，如为空，则调用给定获取函数获取返回值 |
| orElseThrow(Supplier<? extends X> exceptionSupplier) | T           | 如当前封装对象不为空，返回当前封装对象，如为空，则抛出给定异常 |

## 使用示例

### 准备对象

```java
//模拟函数
HashMap<Object, Object> map = Maps.newHashMap();
Object obj = map.get("test");
Object obj2 = map.get("test2");
```



### 获取封装对象

```java
//获取封装值, 此处如obj为null, 则会报NPE
Object result = Optional.ofNullable(obj).get();
```



### 获取封装对象，如为空则取其他值

```java
//获取封装值, 此处如obj为null, 则返回"is null"
Object result = Optional.ofNullable(obj).orElse("is null");
```



## 获取封装对象，多重判空取其他值

```java
//获取封装值, 此处如obj为null, 则判断obj2, 如obj2不为null, 则返回obj2, 如obj2也为null, 则返回"all is null"
Optional.ofNullable(obj)
        .orElse(Optional.ofNullable(obj2)
                        .orElse("all is null"));
```



### 处理集合empty情况下设置为null

```java
HashMap<Object, Object> result4 = Optional.ofNullable(map)
                .filter(Map::isEmpty)
                .orElse(null);
```



### 转换对象

```java
Object result5 = Optional.ofNullable(map)
                .map(m -> m.get("test"))
                .orElse(null);
```



### 消费对象

```java
//如obj不为null, 打印对象
Optional.ofNullable(obj).ifPresent(System.out::println);
```



### 复杂应用

```java
//1.封装map
//2.转换成key为"aaa"对应的value
//3.过滤value是否为null
//4.如果过滤后为null, 返回map.get("bbb")
Object result7 = Optional.ofNullable(map)
                .map(m -> m.get("aaa"))
                .filter(Objects::isNull)
                .orElseGet(() -> map.get("bbb"));
```



## 扩展

> 由于Optional是final标记的，不能使用继承，只能新建一个类

### 增加方法

```java
//当前封装对象为null时使用other封装成一个Optional对象
public Optional2<T> orElseOptional(T other){
	return value !=null ? this: Optional2.ofNullable(other);
}

//当前封装对象为null时使用other获取值并封装成一个Optional对象
public Optional2<T> orElseOptionalGet(Supplier<? extends T> other){
	return value !=null ? this: Optional2.ofNullable(other.get());
}
```



### 示例

```java
//当前值为null, 依次取以下值
Object result8 = Optional2.ofNullable(obj)
                .orElseOptional(obj2)
                .orElseOptional(obj)
                .orElseOptional(obj2)
                .orElse(null);

//当前值为null, 依次调用方法获取以下值
Object result9 = Optional2.ofNullable(obj)
                .orElseOptionalGet(()-> map.get("aaa"))
                .orElseOptionalGet(()-> map.get("bbb"))
                .orElseOptionalGet(()-> map.get("ccc"))
                .orElse(null);
```

