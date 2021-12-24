# Java范型中 ? extends T 和 ? super T

前言：向上转型是安全的，向下转型是不安全的，除非你知道List中的真实类型，否则向下转型就会报错。

`<? extends T>` 表示类型的上界，表示参数化类型的可能是T 或是 T的子类;

`<? super T> `表示类型下界（Java Core中叫超类型限定），表示参数化类型是此类型的超类型，直至Object;

### 1、上界 <? extends T> 只取不存

`List<? extends Number> numbers`意味着下面的赋值语句都是合法的：

```java
List<? extends Number> numbers = new ArrayList<Number>();  // Number "extends" Number (in this context)
List<? extends Number> numbers = new ArrayList<Integer>(); // Integer extends Number
List<? extends Number> numbers = new ArrayList<Double>();  // Double extends Number
```

#### 1.1 读取

给定上述可能的赋值语句，能保证你从`List numbers`中取出什么样类型的对象？

- 你可以读取一个`Number`对象，因为上面任意一个list都包含`Number`对象或者`Number`子类的对象（上面的Number、Integer、Double都可以转型成Number，并且是安全的，所以读取总是可以的）。如下代码就不会报错：

```java
List<? extends Number> numbers = new ArrayList<Integer>();
Number number = numbers.get(0);
```

- 你不能读取一个`Integer`对象，因为numbers可能指向的是`List<Double>`（与其运行时发现Double转成Integer报错，不如编译时就不让从numbers中取`Integer`对象）。如下代码编译时会报`Incompatible types`错的：

```java
List<? extends Number> numbers = new ArrayList<Integer>();
Integer number = number.get(0);
```

因为编译的时候编译器只知道numbers引用是一个List<? extends Number>，要到运行时才会绑定到new ArrayList<Integer>()，所以编译的时候是无法判断numbers指向的List中到底是什么类型，唯一能确定的就是这个类型是Number的子类（或者就是Number类）。

- 你也不能读取一个`Double`对象，因为numbers可能指向的是`List<Integer>`。

#### 1.2 写入

给定上述可能的赋值语句，你能往`List numbers`中添加什么类型的对象从而保证它对于所有可能的`ArrayList`都是合法的呢？

- 你不能添加一个`Integer`对象，因为`numbers`可能指向的是`List<Double>`。如下代码是会编译报错的：

```java
List<? extends Number> numbers = new ArrayList<Integer>();
numbers.add(new Integer(1));
```

原因编译期间是无法知道numbers指向的`ArrayList`中到底放的是什么类型，List和ArrayList两者之间并没有绑定只有到运行时才知道（就是Java所谓的晚绑定或运行时绑定）。与其到运行时发现往一个`ArrayList<Double>`中add一个`Integer`导致抛出类型转换异常，倒不如编译时就报错，即使`ArrayList`中放的就是`Integer`类型。

- 你不能添加一个`Double`对象，因为numbers可能指向的是`List<Integer>`。
- 你不能添加一个`Number`对象，因为numbers可能指向的是`List<Integer>`。

**总结一下**：你不能往`List<? extends T>`中添加任何对象，因为你不能保证`List`真正指向哪个类型，所以不能确定添加的对象就是`List`所能接受的类型。能保证的，仅仅是你可以从`List`中读取的时候，你获得的肯定是一个`T`类型的对象（即使是`T`类型的子类对象也是`T`类型的）。

### 2、下界 <? super T> 只存不取

现在考虑`List<? super T>`
包含通配符的声明`List<? super Integer> numbers`意味着下面任何一个赋值语句都是合法的：

```java
List<? super Integer> numbers = new ArrayList<Integer>();  // Integer is a "superclass" of Integer (in this context)
List<? super Integer> numbers = new ArrayList<Number>();   // Number is a superclass of Integer
List<? super Integer> numbers = new ArrayList<Object>();   // Object is a superclass of Integer
```

#### 2.1 读取

给定上述可能的赋值语句，当读取`List numbers`中的元素的时候，你能保证接收到什么类型的对象呢？

- 你不能保证是一个`Integer`对象，因为numbers可能指向一个`List<Number>`或者`List<Object>`。
- 你不能保证是一个`Number`对象，因为numbers可能指向一个`List<Object>`。
- 你能保证的仅仅是它一定是一个`Object`类的实例或者`Object`子类的实例（但是你不知道到底是哪个子类）。

#### 2.2 写入

给定上述可能的赋值语句，你能往`List numbers`中添加什么类型的对象从而保证它对于所有可能的`ArrayList`都是合法的呢？

- 你可以添加一个`Integer`实例，因为`Integer`类型对于上述所有的list都是合法的。
- 你可以添加任何`Integer`子类的实例，因为一个`Integer`子类的实例都可以向上转型成上面列表中的元素类型。
- 你不可以添加`Double`类型，因为numbers可能指向的是`ArrayList<Integer>`。
- 你不可以添加`Number`类型，因为numbers可能指向的是`ArrayList<Integer>`。
- 你不可以添加`Object`类型，因为numbers可能指向的是`ArrayList<Integer>`。

### 3、PECS

PECS是"Producer Extends,Consumer Super"（生产者用Extends，消费者用Super）的缩写。

- "Producer Extends"的意思是，如果你需要一个`List`去生产`T`类型values（也就是说你需要去list中读取`T`类型实例），你需要声明这个`List`中的元素为`? extends T`，例如`List<? extends Integer>`，但是你不能往里面添加元素。
- "Consumer Super"的意思是，如果你需要一个`List`去消费`T`类型values（也就是说你需要往list中添加`T`类型实例），你需要声明这个`List`中的元素为`? super T`，例如`List<? super Integer>`。但是不能保证你从这个list中读取出来对象类型。
- 如果你既需要往list中写，也需要从list中读，那么你就不能用通配符`?`，必须用精确的类型，比如`List<Integer>`。
- 可以参考JDK源码中的Collections类的copy方法，来理解PECS，源码在文末有。

```java
/**
     * Copies all of the elements from one list into another.  After the
     * operation, the index of each copied element in the destination list
     * will be identical to its index in the source list.  The destination
     * list must be at least as long as the source list.  If it is longer, the
     * remaining elements in the destination list are unaffected. <p>
     *
     * This method runs in linear time.
     *
     * @param  <T> the class of the objects in the lists
     * @param  dest The destination list.
     * @param  src The source list.
     * @throws IndexOutOfBoundsException if the destination list is too small
     *         to contain the entire source List.
     * @throws UnsupportedOperationException if the destination list's
     *         list-iterator does not support the <tt>set</tt> operation.
     */
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    int srcSize = src.size();
    if (srcSize > dest.size())
        throw new IndexOutOfBoundsException("Source does not fit in dest");

    if (srcSize < COPY_THRESHOLD ||
        (src instanceof RandomAccess && dest instanceof RandomAccess)) {
        for (int i=0; i<srcSize; i++)
            dest.set(i, src.get(i));
    } else {
        ListIterator<? super T> di=dest.listIterator();
        ListIterator<? extends T> si=src.listIterator();
        for (int i=0; i<srcSize; i++) {
            di.next();
            di.set(si.next());
        }
    }
}

```

