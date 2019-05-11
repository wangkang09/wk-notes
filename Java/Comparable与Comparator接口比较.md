## Comparable接口

```java
public interface Comparable<T> {
	//负数：表示自身小；0：表示相等；正数：表示自身大
	//上述只是一个约定俗成的规则，因为真正别的代码用到ComparaTo时，默认返回值为负时，就认为本身比比较的小
    public int compareTo(T o);
}
//以下式JDK1.8中一个mergeSort的例子中的一段
// Insertion sort on smallest arrays
if (length < INSERTIONSORT_THRESHOLD) {
    for (int i=low; i<high; i++)
        //默认当调用compareTo小于0时，dest[j-1]<dest[j]
        for (int j=i; j>low &&
             ((Comparable) dest[j-1]).compareTo(dest[j])>0; j--)
            swap(dest, j, j-1);
    return;
}
```

- comparable接口是**类内比较器**，有一个compareTo抽象方法
- 当某个类实现comparable接口时，就可以被集合或数组直接排序（也可以自己调用compareTo接口来自定义集合中的排序）
- 约定的规则：负数：表示自身小；0：表示相等；正数：表示自身大



## Comparator

- Comparator接口是类外比较器，有一个抽象方法compare。当一个类需要排序，但没有实现类内比较器或需要另一种比较规则时，就可以实现一个类外比较器

```java
public IntegerComparator implements Comparator<Integer> {
    public int compare(Integer i1, Integer i2) {
        return i2-i1;
    }
}
Collections.sort(list<Integer>,IntegerComparator)；//就实现了list的逆序排序
```

- 有四个比较重要的default方法
  - reversed()：返回一个镜像比较器
  - thenComparing(Comparator<? super T> other)：在自身的比较器基础上，再加一个比较器，基础比较器相等的时候，用额外的比较器再次比较
  - thenComparing(Function<? super T, ? extends U> keyExtractor,Comparator<? super U> keyComparator)：也是再加一个额外比较器，只是这个额外比较器生成思路：首先通过keyExtractor将两个待比较的参数生成一个新的keyComparator对应的类型实例，再将新的参数使用keyComparator作比较
  - thenComparing(Function<? super T, ? extends U> keyExtractor)：思想相同。额外的比较器生成思路：通过keyExtractor将原始比较参数转换成另一种实现了Comparator接口的参数，这个比较器就是使用转换参数的 compartTo方法作比较的
- 有四个比较重要的static方法
  - reverseOrder()：返回一个镜像比较器（反规则，o2-o1）
  - naturalOrder()：返回一个自然比较器（符合规则,o1-o2）
  - nullsFirst()：返回一个参数可以是null的比较器，其第一个参数为null，返回-1
  - nullsLast()：返回一个参数可以是null的比较器，其第2个参数为null，返回-1
  - comparing(Function<? super T, ? extends U> keyExtractor,  Comparator<? super U> keyComparator)：生成一个比较器，生成思路上面已讲
  - comparing(Function<? super T, ? extends U> keyExtractor)：生成一个比较器，生成思路上面已讲

```java
public interface Comparator<T> {
    //和CompareToy一个规则，返回小于0，第一个参数小
    int compare(T o1, T o2);
    boolean equals(Object obj);
    //返回一个该比较器的镜像比较，即，返回的比较器的compare(o2,o1)，参数位置变了
    default Comparator<T> reversed()
    //返回一个新的比较器（字典序比较器（多个比较器顺序执行））
    //当自身比较器比较为相等时，才调用other比较器，在做一次判断
    //这样可以使用这个方法，多层嵌套，形成多个相等判断比较器
    default Comparator<T> thenComparing(Comparator<? super T> other)
    //keyExtractor必须返回2个int值，新的比较器是一个integer比较器，形成字典序比较器    
    default Comparator<T> thenComparingInt(ToIntFunction<? super T> keyExtractor) {
        return thenComparing(comparingInt(keyExtractor));
    }
    public static <T> Comparator<T> comparingInt(ToIntFunction<? super T> keyExtractor) {
        return (Comparator<T> & Serializable)
            //关键的就是keyExtractor必须返回一个int
            (c1, c2) -> Integer.compare(keyExtractor.applyAsInt(c1), 		 keyExtractor.applyAsInt(c2));
    }
    //同理
    default Comparator<T> thenComparingLong(ToLongFunction<? super T> keyExtractor)
    default Comparator<T> thenComparingDouble(ToDoubleFunction<? super T> keyExtractor)
    
    default <U extends Comparable<? super U>> Comparator<T> thenComparing(
        Function<? super T, ? extends U> keyExtractor) {
        //comparing返回一个比较器，再形成字典比较器
        //关键就是comparing(KeyExtractor)怎么返回的
        //通过KeyExtractor生成的两个比较类得到的比较器，形成字典序比较器
        return thenComparing(comparing(KeyExtractor));
    }
   //返回一个Compartor对象，关键是KeyExtractor生成了两个继承了Comparable的对象，并计较     
   public static <T, U extends Comparable<? super U>> Comparator<T> comparing(
            Function<? super T, ? extends U> keyExtractor)
    {
        Objects.requireNonNull(keyExtractor);
        return (Comparator<T> & Serializable)
            //（c1,c2）满足后面的函数式！
            (c1, c2) -> keyExtractor.apply(c1).compareTo(keyExtractor.apply(c2));
    }
    //通过一个T实例，使用keyExtractor得到一个新的T实例
    //得到的2个新的T实例，使用KeyComparator作比较
    default <U> Comparator<T> thenComparing(
            Function<? super T, ? extends U> keyExtractor,
        Comparator<? super U> keyComparator) {
        //通过KeyExtractor生成新的类，作用于keyComparator得到的新比较器，再形成字典序比较器
        return thenComparing(comparing(keyExtractor, keyComparator));
    }     
    //返回一个Compartor对象，关键是通过KeyExtractor返回了一个新的比较对象
    public static <T, U> Comparator<T> comparing(
            Function<? super T, ? extends U> keyExtractor,
            Comparator<? super U> keyComparator)
    {
        Objects.requireNonNull(keyExtractor);
        Objects.requireNonNull(keyComparator);
        //显然，首先是通过keyExtractor形成一个新的类（即新的key（这个key只能是KeyComparator中compare中的参数类型））
        return (Comparator<T> & Serializable)
            (c1, c2) -> keyComparator.compare(keyExtractor.apply(c1),
                                              keyExtractor.apply(c2));
    }
    
    //--------------以上是default方法
    //--------------以下是剩余的static方法
    
    //返回一个镜像比较器，这个比较器的compare参数必须实现Comparable接口
    public static <T extends Comparable<? super T>> Comparator<T> reverseOrder() {
        return Collections.reverseOrder();
        //返回比较器的compare方法如下，可以
        public int compare(Comparable<Object> c1, Comparable<Object> c2) {
            return c2.compareTo(c1);//这里调用compareTo方法，完美的结合了comparator接口和comparable接口
        }
        //一般是compare是这样的
        public int compare(MyComparable o1, MyComparable o2) {
            return o1.getValue()-o2.getValue();
        }
    }
    
    //返回一个继承了Comparable接口的比较器
    public static <T extends Comparable<? super T>> Comparator<T> naturalOrder() {
        return (Comparator<T>) Comparators.NaturalOrderComparator.INSTANCE;
        //关键，返回的是这种比较器
        public int compare(Comparable<Object> c1, Comparable<Object> c2) {
            return c1.compareTo(c2);
        }
    }    
    
    //返回一个参数可以为null的比较器
	//两种情况1：a==null,b!=null ->  ture:-1,false:1
    //2.a!=null,b==null -> ture:1,false:-1
    //nullsFirst：表示第一个为null，返回-1
    //nullsLast:表示第一个为null，返回1
    public static <T> Comparator<T> nullsFirst(Comparator<? super T> comparator) {
        return new Comparators.NullComparator<>(true, comparator);
        //关键就是这个，      
        public int compare(T a, T b) {
            if (a == null) {
                return (b == null) ? 0 : (nullFirst ? -1 : 1);
            } else if (b == null) {
                return nullFirst ? 1: -1;
            } else {
                return (real == null) ? 0 : real.compare(a, b);
            }
        }
    }        
}
```

```java
public class ComparableTest {
    public static void main(String[] args) {
        testComparable();
        testComparator();

    }
    private static void testComparable() {
        MyComparable my = new MyComparable(1);
        MyComparable my1 = new MyComparable(2);

        List<MyComparable> list = Arrays.asList(my,my1);

        Collections.sort(list);

        for (MyComparable myComparable : list) {
            System.out.println(myComparable.getValue());
        }
        System.out.println(my.compareTo(new MyComparable(2)));
    }
    private static void testComparator() {
        MyComparator my = new MyComparator();

        my.reversed();//返回一个MyComparator比较器的镜像比较器
        //返回一个新的比较器，当自身比较器比较为相等时，才调用参数里的比较器，在做一次判断
        //这样可以使用这个方法，多层嵌套，形成多个相等判断比较器
        my.thenComparing(new MyComparator());

        //通过一个T实例，使用keyExtractor得到一个新的T实例
        //得到的2个新的T实例，使用外比较器KeyComparator作比较
        my.thenComparing((k)->{
            String name = k.getName();
            int len = name.length();
            return new MyComparable(len);
        },new MyComparator());

        //和上个的区别在于，这个直接使用两个返回值的内比较器的compareTo作比较
        my.thenComparing((k)->{
            String name = k.getName();
            int len = name.length();
            return new MyComparable(len);
        });
    }
}

class MyComparable implements Comparable<MyComparable> {
    private int value;
    private String name;
    public MyComparable(int value) {
        this.value = value;
    }
    public MyComparable(int value, String name) {
        this.value = value;
        this.name = name;
    }
    public String getName() {
        return name;
    }
    public int getValue() {
        return value;
    }
    @Override
    public int compareTo(MyComparable o) {
        return -this.value + o.value;
    }
}
class MyComparator implements Comparator<MyComparable> {
    @Override
    public int compare(MyComparable o1, MyComparable o2) {
        return o1.getValue()-o2.getValue();
    }
}
```