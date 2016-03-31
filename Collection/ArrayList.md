#ArrayList源码分析
###简介
#####类型：
类 |extends AbstractList|implements  List<E>, RandomAccess, Cloneable, java.io.Serializable
#####梗概：
- ArrayList是一个大小可变的数组，由于其实现是基于数组，所以其用于数组所特有的属性，对于查找与修改操作，其时间复杂度可以认为是O(1),但是对于增加和删除操作，其时间复杂度则可认为是O(n)，所以一般情况下，对于频繁执行查找和修改操作时，我们优先使用ArrayList；但是对于频繁执行删除和增加操作时，我们会建议优先使用LinkedList；
- ArrayList的实现是不同步的，如果你的程序是针对于多线程并发的情况下，要么不使用ArrayList，要么就自己处理同步过程所产生的不一致的问题。但是JDK中提供了一些方法，可以直接的帮你处理这些问题。通过Collections.synchronizedList()会返回一个支持同步的List。
- ArrayList扩容之后，扩容后的容量大于原容量的一半，且ArrayList的可变数组特性，是通过新建数组并将旧数组的数据复制到新数组而实现的。

####源码解析部分
#####结构：
- 成员变量：
 - DEFAULT_CAPACITY ：默认的初始容量。
 - EMPTY_ELEMENTDATA：
 - DEFAULTCAPACITY_EMPTY_ELEMENTDATA：一个空的数组，当使用无参构造函数时，将此函数赋值给elementData
 - elementData：保存ArrayList的数据数组
 - size：ArrayList的大小（所包含元素的个数）
- 构造函数：
 - public ArrayList(int initialCapacity)：使用指定大小容量来创建一个空的list
 -  public ArrayList()：使用默认容量构造一个空的list（默认大小为10）
 -  public ArrayList(Collection<? exends E> c):构造一个包含指定 collection 的元素的列表，这些元素是按照该 collection 的迭代器返回它们的顺序排列的。
- 方法：
 - add(E e)：将指定的元素添加到此列表的尾部。
 - add(int index, E element)：将指定的元素插入此列表中的指定位置。

#####源码分析：
- add(E e)：
```
public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
```
 从上面的方法调用栈我们可以看到，当执行添加操作的时候，首先会调用方法ensureCapacityInternal(int minCapacity)，该方法的作用是判断ArrayList中是否有minCapacity个容量。从上面的结构部分我们知道，size变量实际上就是当前ArrayList的大小，而参数为size+1就是确认当前的容量是否有size+1个，因为我们要加入一个新的数据。
 下面我们来看看ensureCapacityInternal方法是如何确认容量并且当容量不够时又是如何处理的？
```
 private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }
```
首先判断elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA是否成立，当成立的话，说明构造ArrayList是通过无参构造函数产生的，那么ArrayList的默认容量就是DEFAULT_CAPACITY(10)，通过求DEFAULT_CAPACITY和minCapacity的最大值来指定此次操作的最小容量。下面你会看到ensureExplicitCapacity这个方法，从名字来看，可以译为“确保明确的容量”，实际是在这个方法中确认容量，也就是说如果当前的容量合格的话，就不管了，如果溢出的话，就会执行扩容操作。让我们看一下方法的具体实现。
```
 private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```
如果minCapacity - elementData.length>0的话，那就说明当前的数组容量是不能满足我再次添加一个元素的，那么就会执行扩容操作grow(int minCapacity)。
```
private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```
上面就是扩容的真实面目，其实扩容的方法是很简单的，首先会计算出扩容后的新容量，新容量为旧容量的大小加上其大小的一半，也就是说ArrayList的扩容的成半扩容的。然后将旧的数组的数据拷贝到新的数组当中，并返回给elementData的引用，这也就完成了ArrayList的扩容，是不是很神奇？
好了，我们已经解释了由ensureCapacityInternal()方法所迁出的所有线，总结一句，就是在执行add方法之前，判断一下保存ArrayList的缓存数组的容量是否满足我再添加一个数据的要求，如果满足就不说了，不然的话，就会重新创建一个比当前数组容量大一倍的数组来实现扩容。下面就简单了。
```
 elementData[size++] = e;
```
将要添加的数据加入到数组当中即可。
注意：其实对于扩容操作，还有一个细节我刚刚没有说明，就是扩容过程是不是无论如何都能成功呢？当然不是，对于grow()方法中，有一句代码我刚没有分析：
```
 MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8；
  if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
```
实际上，新建的数组容量是有界的，值是不能大于Integer.MAX_VALUE - 8；
- add(int index, E element)：
```
 public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
```
首先会通过rangeCheckForAdd(index)方法来判断index的有效性，因为index <= size && index >= 0。然后和add(E e)方法一样，会判断一个当前容量的有效性。然后修改指定下标下的数据。这个方法与add(E e)大部分类似，就不多讲解。
- addAll(Collection<? extends E> c)：
```
 public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }
```
首先会将一个集合转化成一个数组a，然后再通过方法System.arraycopy()将数组a复制到elementData上，然后重置size大小。
- remove(int index) ：
```
public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
```
首先通过 rangeCheck(index)方法来判断index的有效性，然后通过一个公式size - index - 1来计算移动元素的个数，当移动的个数大于0个时，通过数组拷贝函数System.arraycopy()，将数组从index+1开始到最后的元素部分拷贝到从index开始的位置，这也就实现了从index+1到结束的数组部分的所有元素向前移动了以为，从而实现了删除第index个元素。elementData[--size] = null这一步的想法很好，在C/C++中我们可以通过free函数释放掉一块内存，但是在JAVA中我们一般不会主动去释放一块内存，而是通过将对象的引用赋值为null，当GC发现一块没有被引用的内存时，会自动回收内存。

- remove(Object o)：
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
```
对于删除指定对象的方法，逻辑很简单，主要的逻辑就是计算采用顺序查询的方法找到要删除对象的下标，然后调用与remove(int index)功能相同的fastRemove方法，其实两个方法基本一致，只是在fastRemove方法中不需要判断值的有效性，并且不需要返回要删的数据。因为对象判断相同是通过equals方法，所以为了考虑null的情况，要单独处理为null的情况。
- set(int index, E element)：

#####更多细节
- 在多线程中，如果一个线程在遍历的一个ArrayList的时，另一个线程修改了这个ArrayList，此时会报ConcurrentModificationException，为什么呢？
首先，我们要知道一个变量： 
```
protected transient int modCount = 0;
```
这个变量定义在AbstractList中，其记录着一个List被结构性修改的次数。好了，暂且你知道有这么个东西。
在ArrayList中有三个迭代器的实现，我们就研究其中的一个源码。
```
 private class Itr implements Iterator<E> {
 		....
        int expectedModCount = modCount;
        @SuppressWarnings("unchecked")
        public E next() {
            checkForComodification();
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }
         final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
        ....
```
我们只需看这个迭代器中的next方法即可了解其原理。
我们发现，在这个next()函数内部，第一行，会调用一个方法checkForComodification();在这个方法中会判断modCount != expectedModCount，对于expectedModCount这个变量，是创建一个迭代器对象的时候就会被初始化，其值是在创建一个迭代器之时就确定了的，在创建之后，如果再发生结构性的修改的时候就使得modCount的值发生改变，则modCount != expectedModCount就会成立，从而抛出ConcurrentModificationException。
