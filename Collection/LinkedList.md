###LinkedList源码解析


[TOC]





####------简介
- LinkedList底层是使用双线链表来实现的，将数组添加到这个集合或者是从集合删除其实都是对双向链表增加节点和删除及节点的操作。
- LinkedList实现的类AbstractSequentialList<E>中定义的modCount属性使得继承自它的集合不能够异步的进行合集的增加删除等等操作，即操作是线程不安全的。


####------结构
LinkedList继承自AbstractSequentialList<E>，实现了List<E>, Deque<E>, Cloneable, java.io.Serializable接口。
下面是LinkedList中的内部类以及重要的属性：
- 内部类
 - private class ListItr implements ListIterator<E>
 - private static class Entry<E>
 - private class DescendingIterator implements Iterator
- 主要属性
 - private transient Entry<E> header = new Entry<E>(null, null, null);
 - private transient int size = 0;
- 构造方法
 -  public LinkedList()
 -   public LinkedList(Collection<? extends E> c)
- 主要讲的方法
 -  public boolean add(E e)
 -  public boolean remove(Object o)
 -  public void push(E e)
 -  public E pop()

####------内部类讲解
- **静态内部类Entry**
在以上说过，LinkedList是使用双向链表实现的，那么这个类就定义了双向链表的数据结构。不懂得双向链表的就赶紧看看数据结构吧。

```
private static class Entry<E> {
	E element;
	Entry<E> next;
	Entry<E> previous;
}//双向链表，所以定义了前后指针
```
- **内部类listItr**
看简介的说明，我们会发现此类实现了ListIterator<E>接口，因此LinkedList集合可以迭代其中的元素，他通过复写及口中的hasNext()等方法找到链表中的下一个元素，直到链表为空。
 - 属性
  private Entry<E> lastReturned = header;//记录上一个遍历节点
  private Entry<E> next;//用来遍历元素
  private int nextIndex;//记录当前元素的下标
  private int expectedModCount = modCount;（A-最后一起讲）

 - 构造方法
ListItr(int index)
此类的构造方法很重要。它的主要作用是找到index下标的元素，这样就可以向此元素之前或者此元素之后遍历所有了。
实现原理：如果传进的索引值index小于0或者大于LinkedList的size，那么我们就会看到经常会出现的异常IndexOutOfBoundsException。如果index小于数组大小(size)的一半的话，将头指针附给next，然后从数组的第0个遍历到index，找到下标为index的元素。如果index的值大于等于数组值的一半，那么，将尾指针付给next，然后从后向前遍历集合知道找到index对应的元素，这样一来，我们每一次遍历都不会超过数组大小的一半，大大的提高了效率。
下面来看他的实现：

```
	ListItr(int index) {
	    if (index < 0 || index > size)
		throw new IndexOutOfBoundsException("Index: "+index
        + ", Size: "+size);
	    if (index < (size >> 1)) {
		next = header.next;
		for (nextIndex=0; nextIndex<index; nextIndex++)
		    next = next.next;
	    } else {
		next = header;
		for (nextIndex=size; nextIndex>index; nextIndex--)
		    next = next.previous;
	    }
	}

```

可以看到它计算数组的一半的方法：（size >> 1）是将size右移一位，在计算机中，数字以二进制的形式表示的，右移一位表示将size除以2，相反的左移一位代表乘以2.
 - 实现ListIterator<E>的方法
由于此内部类实现了ListIterator<E>接口所以复写了其中的方法，我们现在主要讲三个方法：
**1.**public void add(E e)
此方法的作用是在集合中添加一个元素。添加的方法使用的是尾插法，所以lastReturned先被附为header（这说明LinkedList使用双向链表时将头节点位置放在链表的尾部），然后调用方法addBefore将元素e插入链表尾部，将当前元素的位置加1.

```
public void add(E e) {
	    checkForComodification();//B-随后一起讲
	    lastReturned = header;
	    addBefore(e, next);
	    nextIndex++;
	    expectedModCount++;//D-随后一起讲
	}
    看看addBefore的代码：
    
    private Entry<E> addBefore(E e, Entry<E> entry) {
        Entry<E> newEntry = new Entry<E>(e, entry, entry.previous);
        newEntry.previous.next = newEntry;
        newEntry.next.previous = newEntry;
        size++;
        modCount++;//C-最后一期讲
        return newEntry;
    }

```

**2.**private E remove(Entry<E> e)
删除元素的方法很简单，就是双向链表的删除操作，但一定要注意顺序，删除的那四句代码的顺序不可以改变。删除以后，将集合的大小减去一。

```
private E remove(Entry<E> e) {
	if (e == header)
	    throw new NoSuchElementException();

    E result = e.element;
	e.previous.next = e.next;
	e.next.previous = e.previous;
    e.next = e.previous = null;
    e.element = null;
	size--;
	modCount++;
        return result;
    }

```

**3.**public E previous()
此方法主要是可以逆序找到上一个元素。如果表示当前元素下标的属性nextIndex等于0，表示链表为空，所以会抛出异常NoSuchElementException；若果不为空的话，将记录上一个元素的属性lastReturned标记为要寻找的元素（lastReturned = next = next.previous;）当然，由于是逆序遍历，当前元素的下表也是要减的。

```
	public E previous() {
	    if (nextIndex == 0)
		throw new NoSuchElementException();

	    lastReturned = next = next.previous;
	    nextIndex--;
	    checkForComodification();
	    return lastReturned.element;
	}

```

同样的，checkForComodification();也到最后在一起讲
- **内部类DescendingIterator**
这个内部类是实现集合的减序遍历的，实现了Interator接口。同样的，他遍历的时候就使用上一个内部类ListItr复写的方法previous就可以得到上一个元素了。

```
 private class DescendingIterator implements Iterator {
        final ListItr itr = new ListItr(size());
		public boolean hasNext() {
	    	return itr.hasPrevious();
		}
		public E next() {
   	         return itr.previous();
        }
		public void remove() {
            itr.remove();
        }
    }

```

####------属性expectedModCount属性的讲解



- A处的expectedModCount=modCount
modCount是LinkedList继承自他的父类的属性。这是一个很简单的赋值语句，看AbstractList的源码我们可以知道属性modCount的类型是pretected类型的，他就是用来给子类继承的。
- B处的CheckForComodification()；方法的讲解
在内部类ListItr中的add方法中，首先调用了这个方法，我们来看看这个方法是干什么的

```
private void checkForComodification() {
        if (this.modCount != l.modCount)
            throw new ConcurrentModificationException();
    }

```

我么可以看到，这个类是检查子类中的modCount是否与他的父类中的modCount是否一致，如果不一致就会抛出异常ConcurrentModificationException；他在这里的作用我们讲完C和D时再来分析。
- C处的modCount++的讲解
这句代码在add方法调用的addBefore中出现，我们可以看到，每当集合中增加一个元素的时候，调用addBefore方法都会现将父类AbstractList中的modCount加上一。这个作用我们接下来讲。
- D处的ecpectedModCount++的讲解
这句代码同样出现在add方法中，我么可以看到在addBefore中将父类的modCount加上一以后再讲子类LinkedList中的expectedModCount的值再加上一。

**现在我们来讲一讲这样作的好处**
当同步操作或者单线程时，集合的操作是没有问题的，那如果在多线程并发的时候呢？如果一个操作集合的一个线程在不停的删除元素，一个集合在不停的添加元素，那么这个集合还有什么意义呢？为了使这种情况不发生，JDK的设计者要求使用时不可以异步使用，如果你非得异步使用，那么就会抛出异常。那JDK是怎样阻止使用者不能异步使用的呢？就是上面的ABCD的作用了。当用户在添加一个元素时，先检查父类的modCount与子类的expectedModcount是否相同（调用CheckForComodification()方法），如果不相同，说明有多个线程在操作集合，那么就抛出异常，如果相同，将父类的modCount加上一，再将子类的expectedModCount再加上一，这中间也有执行的代码。正因为这样，如果进行异步造作的话，就有可能导致程序没有按照预想的顺序执行，那么就很有可能导致弗雷德modCount与子类的expectedModCount不一致，这样的话在添加时就会抛出异常。


####------主要方法讲解

其实在LinkedList中，只要掌握了他的数据结构以及内部类，那么接下来LinkedList中的对元素的操作都是围绕着这些内部类来实现的。
下面我们来看看LinkedList中的方法
-  public boolean add(E e)
此方法时添加一个元素到集合中。我们可以看到他的实现代码就是调用内部类中的方法来实现他的功能的。
```
	public boolean add(E e) {
		addBefore(e, header);
        return true;
    }

如果前面的看懂了，那应该没有任何难度了。
-  public E remove(int index)
此方法从集合中删除指定的元素。

```
public E remove(int index) {
        return remove(entry(index));
    }

```


也是调用前面内部类中的方法。这里就不详细讲解了，又不理解的看看内部类中的讲解。

####------实现的接口的讲解
- **List接口**
有序的 collection（也称为序列）。此接口的用户可以对列表中每个元素的插入位置进行精确地控制。用户可以根据元素的整数索引（在列表中的位置）访问元素，并搜索列表中的元素。
与 set 不同，列表通常允许重复的元素。更正式地说，列表通常允许满足 e1.equals(e2) 的元素对 e1 和 e2，并且如果列表本身允许 null 元素的话，通常它们允许多个 null 元素。（来自JDK）
- **Deque接口**
一个线性 collection，支持在两端插入和移除元素。相当于一个队列，的确LinkedList中有对队列操作的方法，即，你可以将一个LinkedList当成一个队列来使用。
队列的方法主要有以下：
**1.**public void push(E e)
和队列相同，如果需要插入一个元素的话就在链表的尾部插入一个元素，同样的调用的时候仍然是使用上面内部类中的操作。

```
	public void addFirst(E e) {
		addBefore(e, header.next);//上面讲过
    }

```
我们浏览内部类的讲解可以看到addBefore是在链表的尾部插入元素，正好符合队列的要求。
**2.** public E pop()
如果需要删除一个元素，那么就要在链表的头部删除。
```
	public E pop() {
        return removeFirst();
    }
    //我们来看一下removeFist的实现
     public E removeFirst() {
		return remove(header.next);
    }

```
我们可以看到又再次调用了内部类中的remove方法，但是我们注意remove中的参数是header。next，上面说过，LinkedList中的双向链表是将头指针指向尾部的，所以header.next就是头部，将头节点传入就删除了头部。实现了队列的思想。

- **Cloneable接口**

此类实现了 Cloneable 接口，以指示 Object.clone() 方法可以合法地对该类实例进行按字段复制。
如果在没有实现 Cloneable 接口的实例上调用 Object 的 clone 方法，则会导致抛出 CloneNotSupportedException 异常。 
按照惯例，实现此接口的类应该使用公共方法重写 Object.clone（它是受保护的）。

LinkedList集合的clone是浅克隆,一般来说clone时是递归的先克隆对象本身，然后逐层克隆对象的属性，方法等等。但是，在LinkedList中，克隆时只克隆LinkedList本身，而它里面的元素不进行递归克隆，所以说LinkedList是浅克隆的。

- **java.io.Serializable**
使用Serializable接口可以使此集合序列化，当内存不够时，可以讲对象写成二进制的文件放进硬盘中，需要时再从硬盘中读出，然后恢复。这样可以节省内存。集合属于比较大的对象，如果此对象已被加载到虚拟机中，但是又不经常使用，由于他也是一个强引用，垃圾回收器不能将它回收，这时，如果他存活时间过长，被垃圾回收期已经转移到老年代时，长时间的存活会引起GC，GC是一个耗时的过程，我们不愿意遇到这样的情况，所以将它序列化，然后需要的时候再将它使用IO从硬盘中读取出来。


**更多java源码解析：**
[https://github.com/ccqy66/JDK-s-source-parser](http://)


<!--如有不妥，欢迎指正。-->

