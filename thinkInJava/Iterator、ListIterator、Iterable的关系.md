[TOC]

#### **Iterator与Iterable的关系**

##### 在Iterable接口中的iterator方法，返回了一个Iterator对象！

```java
public interface Iterable<T> {
    Iterator<T> iterator();
    default void forEach(Consumer<? super T> action) {...}；
    default Spliterator<T> spliterator() {...}；   
}
public interface Iterator<E> {
    boolean hasNext();
    E next();
    default void remove() {
        throw new UnsupportedOperationException("remove");
    }
    default void forEachRemaining(Consumer<? super E> action) {
    	Objects.requireNonNull(action);
    	while (hasNext())
    	action.accept(next());
    }
}
```

#### **Iterator及Iterable在容器类中的应用（Iterator与ListIterator的关系）**

##### 1首先List接口继承了Collection接口，Collection接口继承了Iterable接口，所以所有的实现List的接口都要实现Iterable接口，如ArrayList和LinkedList。

```java
public interface List<E> extends Collection<E> {}
public interface Collection<E> extends Iterable<E> {}
//RandomAccess接口没有方法，只是表示继承这个接口的类，随机访问其中元素的时间复杂度为O(1)，和Cloneable、Serializable一样，都是起声明意思的作用
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable {};
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable {};

```

##### 2 List接口中有额外加了两个listIterator方法

```java
ListIterator<E> listIterator();
ListIterator<E> listIterator(int index);
```

##### 3 ListIterator继承了Iteratro，ArrayList内部类同时实现了Iterator和ListIterator这两个接口

```java
//其中ListItr和Itr分别是实现了ListIterator和Iterator接口的类，而ListIterator继承了Iterator接口！
public interface ListIterator<E> extends Iterator<E> {}

public ListIterator<E> listIterator(int index) {
    if (index < 0 || index > size)
        throw new IndexOutOfBoundsException("Index: "+index);
    return new ListItr(index);
}

public ListIterator<E> listIterator() {
    return new ListItr(0);
}

public Iterator<E> iterator() {
    return new Itr();
}
```

##### 4 LinkedList实现了listIterator方法，**并在Iterator方法中，直接调用listIterator方法**

```java
//LinkedList继承了AbstractSequentialList类，并实现了listIterator(int index)方法
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable {
    public ListIterator<E> listIterator(int index) {
   		checkPositionIndex(index);
    	return new ListItr(index);
	}
}
//在AbstractSequentialList类中实现了iterator方法
public abstract class AbstractSequentialList<E> extends AbstractList<E> {
    public Iterator<E> iterator() {
        return listIterator();//调用的是AbstractList类中的方法
    }
    public abstract ListIterator<E> listIterator(int index);
}
public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E> {
    public ListIterator<E> listIterator() {
        return listIterator(0);//调用LinkedList中的listIterator()！
    }
}
```

##### 5 set也继承了Collection接口，但是其**实现的iterator方法返回的是map中keySet类中的iterator方法**

```java
public interface Set<E> extends Collection<E> {}
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable {};

public Iterator<E> iterator() {
    return map.keySet().iterator();
}
```

##### 6 map没有实现Collection接口，hashmap在keySet、Values、EntrySet类中实现了Iterator方法，**关键是继承了AbstractSet、AbstractCollection**

```java
public interface Map<K,V> {}
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {}

final class KeySet extends AbstractSet<K> {
	public final Iterator<K> iterator()     { return new KeyIterator(); }
}

final class Values extends AbstractCollection<V> {
    public final Iterator<V> iterator()     { return new ValueIterator(); }
}

final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
    public final Iterator<Map.Entry<K,V>> iterator() {
        return new EntryIterator();
    }
}

//其中hasNext()和nextNode()方法在HashIterator中实现
final class KeyIterator extends HashIterator
        implements Iterator<K> {
        public final K next() { return nextNode().key; }
}

    final class ValueIterator extends HashIterator
        implements Iterator<V> {
        public final V next() { return nextNode().value; }
    }

    final class EntryIterator extends HashIterator
        implements Iterator<Map.Entry<K,V>> {
        public final Map.Entry<K,V> next() { return nextNode(); }
    }
```

##### 7 ListIterator与Iterator的关系

###### 1 Iterator只可以往下遍历，而ListIterator可以往上遍历

###### 2 ListIterator遍历的时候，可以改变当前的值，也可以在当前值后面插入一个值

```java
public interface ListIterator<E> extends Iterator<E> {
    boolean hasNext();
    E next();
    boolean hasPrevious();
    E previous();
    int nextIndex();
    int previousIndex();
    void set(E e);
    void add(E e);
    void remove();
}
```

#### **[Iterator实例ListIterator类和HashIterator解析](./Iterator实例ListIterator类和HashIterator解析.md)**

##### 1 ListIterator的关键就是：它是一个内部类，它持有了ArrayList的内部对象 `ArrayList.this.elementData`，通过`cursor`移动来遍历这个内部数组！

##### 2 遍历HashMap/Set的关键就是：`HashIterator`这个内部类，也持有了内部数组`table`， **通过遍历这个数组的所有桶中的所有节点，达到目的！**

#### **为什么在Iterable接口中返回一个Iterator对象，而不是直接用Iterator代替Iterable**

##### 1 在jdk 1.5以后，引入了Iterable，使用foreach语句（增强型for循环）必须使用Iterable类。 

##### 2 通过分析java容器中的用法，如下可知，当返回的是一个对象时，首先，我们后期可以很安全、方便的更新或添加不同的迭代器（如ListIterator）；并且多次调用迭代器，各个调用直接是不相关的，这种数据和操作分开的思想，大大提升了代码的扩展性。

```java
public final Iterator<Map.Entry<K,V>> iterator() {
    return new EntryIterator();
}
```

#### **自定义迭代器**

首先，能够使用迭代器的数据结构，一般是个容器类。CustomContainer

```java
//后期再修改，先将泛型弄明白
public class CustomContainer<T> implements Iterable<T>{

    private T[] data;
    public CustomContainer() {
        
    }

    @Override
    public Iterator<T> iterator() {
        return new MyIterator();
    }

    private final class MyIterator implements Iterator<T> {
        @Override
        public boolean hasNext() {
            return false;
        }

        @Override
        public T next() {
            return null;
        }
    }
}
```