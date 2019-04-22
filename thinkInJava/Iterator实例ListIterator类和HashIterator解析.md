ListIterator的关键就是：它是一个内部类，它持有了ArrayList的内部对象 `ArrayList.this.elementData`，通过`cursor`移动来遍历这个内部数组！

遍历HashMap/Set的关键就是：`HashIterator`这个内部类，也持有了内部数组`table`， **通过遍历这个数组的所有桶中的所有节点，达到目的！**

```java
public ListIterator<E> listIterator(int index) {
    if (index < 0 || index > size)
        throw new IndexOutOfBoundsException("Index: "+index);
    return new ListItr(index);//返回一个Iterator对象
}

private class ListItr extends Itr implements ListIterator<E> {
    ListItr(int index) {
        super();//Itr中默认当前位置为0，没有index属性！
        cursor = index;//next()返回值的地址
    }

    public boolean hasPrevious() {
        return cursor != 0;//当cursor为0是表示没有前面元素了
    }

    public int nextIndex() {
        return cursor;
    }

    public int previousIndex() {
        return cursor - 1;
    }

    @SuppressWarnings("unchecked")
    public E previous() {
        checkForComodification();
        int i = cursor - 1; //进入这里表示cursor>=1的
        if (i < 0)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;//
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i;//游标前移
        return (E) elementData[lastRet = i];//lastRet
    }

    public void set(E e) {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();

        try {
            ArrayList.this.set(lastRet, e);//set()操作也必须在next()或previous()后，将返回值地址的值改变成e
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }

    public void add(E e) {
        checkForComodification();

        try {
            int i = cursor;//add()操作是独立的
            ArrayList.this.add(i, e);设置下一个位置的值，和next()，previous()效果一样，都是往后走
            cursor = i + 1;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }
}


private class Itr implements Iterator<E> {
    int cursor;       // index of next element to return
    int lastRet = -1; // index of last element returned; -1 if no such
    int expectedModCount = modCount;

    Itr() {}

    public boolean hasNext() {
        return cursor != size;//当cursor为size时，已经到末端了
    }

    @SuppressWarnings("unchecked")
    public E next() {
        checkForComodification();
        int i = cursor;
        if (i >= size)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;//每次next都重新赋值，只是i变了
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i + 1;//下一个地址+1
        //最后返回值的下标变为i
        return (E) elementData[lastRet = i];//返回的就是cursor(下一个地址的值)
    }

    //删除最后一个返回的元素，必须有返回的值，即至少调用next()一次
    public void remove() {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();

        try {
            ArrayList.this.remove(lastRet);
            cursor = lastRet;//cursor减1了
            lastRet = -1;//每次remove之前，必须有next()操作
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }
```

```java
abstract class HashIterator {
    Node<K,V> next;        // next entry to return
    Node<K,V> current;     // current entry
    int expectedModCount;  // for fast-fail
    int index;             // current slot

    HashIterator() {
        expectedModCount = modCount;
        Node<K,V>[] t = table;
        current = next = null;
        index = 0;
        if (t != null && size > 0) { // advance to first entry
            do {} while (index < t.length && (next = t[index++]) == null);//第一个不为null的头指针，退出循环
        }
    }

    public final boolean hasNext() {
        return next != null;
    }

    //返回的是头节点的Entry
    final Node<K,V> nextNode() {
        Node<K,V>[] t;
        Node<K,V> e = next;
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (e == null)
            throw new NoSuchElementException();
        if ((next = (current = e).next) == null && (t = table) != null) {
            do {} while (index < t.length && (next = t[index++]) == null);
        }
        return e;
    }

    public final void remove() {
        Node<K,V> p = current;
        if (p == null)
            throw new IllegalStateException();
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        current = null;
        K key = p.key;
        removeNode(hash(key), key, null, false, false);
        expectedModCount = modCount;
    }
}
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next; //遍历的时候必须要确保，next==null，不然会少遍历元素！

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
}
final class EntryIterator extends HashIterator
        implements Iterator<Map.Entry<K,V>> {
    public final Map.Entry<K,V> next() { return nextNode(); }
}
    final class KeyIterator extends HashIterator
        implements Iterator<K> {
        public final K next() { return nextNode().key; }
    }

    final class ValueIterator extends HashIterator
        implements Iterator<V> {
        public final V next() { return nextNode().value; }
    }

```