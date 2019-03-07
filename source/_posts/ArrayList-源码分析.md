---
title: ArrayList 源码分析
categories:
  - 后端
tags:
  - Java编程
keywords: Java
date: 2019-01-09 20:21:09
---



> 以下源码分析使用的 Java 版本为 1.8 

#### 1. 概览

ArrayList 是基于数组实现的，继承 AbstractList， 实现了 List、RandomAccess、Cloneable、Serializable 接口，支持随机访问。

```
java.util public class ArrayList<E> extends AbstractList<E> 
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```


#### 2. Java Doc 关键点：

- 实现List接口的动态数组，容量大小为 capacity，默认的容量大小 10，会自动扩容
- 可包含空元素 null
- size, isEmpty, get, set, iterator, and listIterator 等操作的复杂度为 O(1)，The add operation runs in amortized constant time, that is, adding n elements requires O(n) time，其它操作为线性时间
- 非线程安全，多线程环境下必须在外部增加同步限制，或者使用包装对象 `List list = Collections.synchronizedList(new ArrayList(...));`
- 快速失败：在使用迭代器时，调用迭代器的添加、修改、删除方法，将抛出 `ConcurrentModificationException` 异常，但是快速失败行为不是硬保证的，只是尽最大努力


#### 3. 成员属性

当添加第一个元素时，`elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA` 的任何空ArrayList都将扩展为默认的capacity

```java
private static final int DEFAULT_CAPACITY = 10; // 默认容量大小
private static final Object[] EMPTY_ELEMENTDATA = {}; // ArrayList空实例共享的一个空数组
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {}; // ArrayList空实例共享的一个空数组，用于默认大小的空实例。与 EMPTY_ELEMENTDATA 分开，这样就可以了解当添加第一个元素时需要创建多大的空间
transient Object[] elementData; // 真正存储ArrayList中的元素的数组
private int size;   // 存储ArrayList的大小，注意不是elementData的长度
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8; // 数组的最大长度
protected transient int modCount = 0; //AbstractList类的，表示 elementData在结构上被修改的次数,每次add或者remove它的值都会加1
```


#### 4. 构造方法

```java
// 无参构造方法，默认初始容量10
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
// 提供初始容量的构造方法
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}
// 通过一个容器来初始化
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray(); 
    if ((size = elementData.length) != 0) { // c.toArray 返回的可能不是  Object[]
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        this.elementData = EMPTY_ELEMENTDATA; // replace with empty array.
    }
}
```



#### 5. 添加元素与扩容

**添加元素**时使用 ensureCapacityInternal() 方法来保证容量足够，`size + 1` 为最少需要的空间大小，如果elementData的长度不够时，需要使用 grow() 方法进行扩容

```java
// 添加一个元素
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
// 计算最少需要的容量
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) { 
        // 默认的空实例第一次添加元素时，使用默认的容量大小与minCapacity的最大值
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++; 
    if (minCapacity - elementData.length > 0) // 需要的容量大于elementData的长度
        grow(minCapacity);  // 进行扩容
}
```

扩容：当新容量小于等于 `MAX_ARRAY_SIZE` 时，新容量的大小为 `oldCapacity + (oldCapacity >> 1)` 与 `minCapacity` 之间的较大值 ，也就是旧容量的 1.5 倍与 `minCapacity` 之间的较大值


```
private void grow(int minCapacity) {
    int oldCapacity = elementData.length; // 原本的容量
    int newCapacity = oldCapacity + (oldCapacity >> 1); // 新的容量
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ? Integer.MAX_VALUE : MAX_ARRAY_SIZE;
}
```

最后调用 `Arrays.copyOf` 复制原数组，将 elementData 赋值为得到的新数组。由于数组复制代价较高，所以建议在创建 ArrayList 对象时就指定大概的容量大小，减少扩容操作的次数

```
public class Arrays {
    public static <T> T[] copyOf(T[] original, int newLength) {
        return (T[]) copyOf(original, newLength, original.getClass());
    }
    public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
        @SuppressWarnings("unchecked")
        T[] copy = ((Object)newType == (Object)Object[].class)
            ? (T[]) new Object[newLength] : (T[]) Array.newInstance(newType.getComponentType(), newLength);
        System.arraycopy(original, 0, copy, 0, Math.min(original.length, newLength));
        return copy;
    }
    //...
}
```

通过 addAll **添加一个集合中所有元素**时的扩容：至少需要的容量为两个集合的长度之和，同样是通过 ensureCapacityInternal() 来保证容量是足够的，然后调用 `System.arraycopy` 将要添加的集合中的元素复制到原集合已有元素的后面

```java
public boolean addAll(Collection<? extends E> c) {
    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount
    System.arraycopy(a, 0, elementData, size, numNew); // 复制元素到原数组尾部
    size += numNew;
    return numNew != 0;
}
```

#### 6. 删除元素
**删除指定下标的元素**时，如果下标没有越界，则取出下标对应的值，然后将数组中该下标后面的元素都往前挪1位，需要挪的元素数量是 `size - index - 1`，时间复杂度为 O(n)，所以删除元素的代价挺高

```java
public E remove(int index) {
    rangeCheck(index); // 检查下标是否在数组的长度范围内
    modCount++;
    E oldValue = elementData(index); // 下标为index的值
    int numMoved = size - index - 1; // 需要移动的元素数量
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index, numMoved);
    elementData[--size] = null; // clear to let GC do its work
    return oldValue;
}
private void rangeCheck(int index) {
    if (index >= size)  
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

**删除在指定集合中的所有元素** removeAll，**删除不在指定集合中的所有元素** retainAll

这两者都是通过 `batchRemove` 来批量删除

```
// 删除在指定集合中的所有元素
public boolean removeAll(Collection<?> c) {
    Objects.requireNonNull(c);  // c 不能为null
    return batchRemove(c, false);
}
// 删除不在指定集合中的所有元素，也就是只保留指定集合中的元素，其它的都删除掉
public boolean retainAll(Collection<?> c) {
    Objects.requireNonNull(c);
    return batchRemove(c, true);
}
// 批量删除
private boolean batchRemove(Collection<?> c, boolean complement) {
    final Object[] elementData = this.elementData;
    int r = 0, w = 0;   // r为当前下标，w为当前需要保留的元素的数量（或者说是下一个需保留元素的下标）
    boolean modified = false;
    try {
        for (; r < size; r++)
            if (c.contains(elementData[r]) == complement)   // 判断元素 elementData[r] 是否需要删除
                elementData[w++] = elementData[r];
    } finally {
        // r != size 的情况可能是 c.contains() 抛出了异常，将 r 之后的元素复制到 w 之后
        if (r != size) { 
            System.arraycopy(elementData, r, elementData, w, size - r);
            w += size - r;
        }
        if (w != size) {
            // w 之后的元素设置为 null 以让 GC 回收
            for (int i = w; i < size; i++) 
                elementData[i] = null;  
            modCount += size - w;
            size = w;
            modified = true;
        }
    }
    return modified;
}
```

**删除第一个值为指定值的元素** `remove(Object o)`，参数 o 可以为 null

`fastRemove(int index)` 与 `remove(int index)` 几乎一样，只不过不返回被删除的元素

```
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```

#### 7. 遍历

ArrayList 支持三种方式：

- for循环下标遍历
- 迭代器(Iterator和ListIterator)
- foreach 语句

**迭代器 Iterator 和 ListIterator 的主要区别：**：

> ArrayList 的迭代器 Iterator 和 ListIterator 在《[设计模式 | 迭代器模式及典型应用](https://mp.weixin.qq.com/s?__biz=MzI1NDU0MTE1NA==&mid=2247483752&idx=1&sn=7880679f18b5727ea64cd05c06817c35&chksm=e9c2ed65deb56473da688784c4562995c24daf4b13425d0d4d080208728b86525f6600127925&scene=0#rd)》这篇文章中有过详细介绍，这里只做一个小结


- ListIterator 有 add() 方法，可以向List中添加对象，而 Iterator 不能
- ListIterator 和 Iterator 都有 hasNext() 和 next() 方法，可以实现顺序向后遍历，但是 ListIterator 有 hasPrevious() 和 previous() 方法，可以实现逆向（顺序向前）遍历。Iterator 就不可以。
- ListIterator 可以定位当前的索引位置，nextIndex() 和 previousIndex() 可以实现。Iterator 没有此功能。
- 都可实现删除对象，但是 ListIterator 可以实现对象的修改，set() 方法可以实现。Iierator 仅能遍历，不能修改

**foreach 循环：**

foreach 循环涉及到一个 Consumer 接口，接收一个泛型的参数T，当调用 accept 方法时，Stream流中将对 accept 的参数做一系列的操作

```java
public void forEach(Consumer<? super E> action) {
    Objects.requireNonNull(action);
    final int expectedModCount = modCount;  // 记录当前的 modCount
    @SuppressWarnings("unchecked")
    final E[] elementData = (E[]) this.elementData;
    final int size = this.size;
    for (int i=0; modCount == expectedModCount && i < size; i++) {
        action.accept(elementData[i]);
    }
    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}
```

#### 8. 序列化

ArrayList 有两个属性被 `transient 关键字` 修饰，**transient 关键字** 的作用：让某些被修饰的成员属性变量不被序列化   

```java
transient Object[] elementData;
protected transient int modCount = 0;
```


**为什么最为重要的数组元素要用 transient 修饰呢？** 

跟Java的序列化机制有关，这里列出Java序列化机制的几个要点：

- 需要序列化的类必须实现java.io.Serializable接口，否则会抛出NotSerializableException异常
- 若没有显示地声明一个serialVersionUID变量，Java序列化机制会根据编译时的class自动生成一个serialVersionUID作为序列化版本比较（验证一致性），如果检测到反序列化后的类的serialVersionUID和对象二进制流的serialVersionUID不同，则会抛出异常
- Java的序列化会将一个类包含的引用中所有的成员变量保存下来（深度复制），所以里面的引用类型必须也要实现java.io.Serializable接口
- 当某个字段被声明为transient后，默认序列化机制就会忽略该字段，反序列化后自动获得0或者null值
- 静态成员不参与序列化
- 每个类可以实现readObject、writeObject方法实现自己的序列化策略，即使是transient修饰的成员变量也可以手动调用ObjectOutputStream的writeInt等方法将这个成员变量序列化。
- 任何一个readObject方法，不管是显式的还是默认的，它都会返回一个新建的实例，这个新建的实例不同于该类初始化时创建的实例
- 每个类可以实现private Object readResolve()方法，在调用readObject方法之后，如果存在readResolve方法则自动调用该方法，readResolve将对readObject的结果进行处理，而最终readResolve的处理结果将作为readObject的结果返回。readResolve的目的是保护性恢复对象，其最重要的应用就是保护性恢复单例、枚举类型的对象


所以问题的答案是：ArrayList 不想用Java序列化机制的默认处理来序列化 elementData 数组，而是通过 readObject、writeObject 方法自定义序列化和反序列化策略。

问题又来了，**为什么不用Java序列化机制的默认处理来序列化 elementData 数组呢**？

答案是因为效率问题，如果用默认处理来序列化的话，如果 elementData 的长度有100，但是实际只用了50，其实剩余的50是可以不用序列化的，这样可以提高序列化和反序列化的效率，节省空间。

现在来看 ArrayList 中自定义的序列化和反序列化策略

```java
private static final long serialVersionUID = 8683452581122892189L;

private void writeObject(java.io.ObjectOutputStream s) throws java.io.IOException{
    int expectedModCount = modCount;
    s.defaultWriteObject(); // 默认的序列化策略，序列化其它的字段
    s.writeInt(size);   // 实际用的长度，而不是容量

    for (int i=0; i<size; i++) {  // 只序列化数组的前 size 个对象
        s.writeObject(elementData[i]);
    }

    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}

private void readObject(java.io.ObjectInputStream s) throws java.io.IOException, ClassNotFoundException {
    elementData = EMPTY_ELEMENTDATA;
    // Read in size, and any hidden stuff
    s.defaultReadObject();
    s.readInt(); // ignored

    if (size > 0) {
        int capacity = calculateCapacity(elementData, size);
        SharedSecrets.getJavaOISAccess().checkArray(s, Object[].class, capacity);
        ensureCapacityInternal(size);

        Object[] a = elementData;
        for (int i=0; i<size; i++) {
            a[i] = s.readObject();
        }
    }
}

```

#### 9. 快速失败(fail-fast)

modCount 用来记录 ArrayList 结构发生变化的次数，如果一个动作前后 modCount 的值不相等，说明 ArrayList 被其它线程修改了

如果在创建迭代器之后的任何时候以任何方式修改了列表（增加、删除、修改），除了通过迭代器自己的remove 或 add方法，迭代器将抛出 `ConcurrentModificationException` 异常

需要注意的是：这里异常的抛出条件是检测到 `modCount != expectedmodCount`，如果并发场景下一个线程修改了modCount值时另一个线程又 "及时地" 修改了expectedmodCount值，则异常不会抛出。所以不能依赖于这个异常来检测程序的正确性。



```
private void writeObject(java.io.ObjectOutputStream s) throws java.io.IOException{
    int expectedModCount = modCount;    // 记录下当前的 modCount
    // 一些操作之后....
    if (modCount != expectedModCount) { // 比较现在与之前的 modCount，不相等表示在中间过程中被修改了
        throw new ConcurrentModificationException();
    }
}

private void writeObject(java.io.ObjectOutputStream s) throws java.io.IOException{
    int expectedModCount = modCount;
    // 一些操作之后....
    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}

public void forEach(Consumer<? super E> action) {
    final int expectedModCount = modCount;
    // 一些操作之后....
    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}

public boolean removeIf(Predicate<? super E> filter) {
    final int expectedModCount = modCount;
    // 一些操作之后....
    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}

public void replaceAll(UnaryOperator<E> operator) {
    final int expectedModCount = modCount;
    // 一些操作之后....
    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
    modCount++; // 修改了要加一
}

public void sort(Comparator<? super E> c) {
    final int expectedModCount = modCount;
    // 一些操作之后....
    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
    modCount++;
}

// 内部迭代器
private class Itr implements Iterator<E> {
    public void forEachRemaining(Consumer<? super E> consumer) {
        checkForComodification();
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```

### 后记

欢迎评论、转发、分享

更多内容可访问我的个人博客：http://laijianfeng.org

关注【小旋锋】微信公众号，及时接收博文推送



![关注_小旋锋_微信公众号](http://image.laijianfeng.org/20180913_001328.png)

