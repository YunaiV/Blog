title: 【追光者系列】HikariCP 源码分析之 FastList
date: 2018-01-13
tags:
categories: HikariCP
permalink: HikariCP/zhazhawangzi/FastList
author: 渣渣王子
from_url: https://mp.weixin.qq.com/s/9zJ8eo5DisZyEVtfXLZAqw
wechat_url:

-------

摘要: 原创出处 https://mp.weixin.qq.com/s/9zJ8eo5DisZyEVtfXLZAqw 「渣渣王子」欢迎转载，保留摘要，谢谢！

- [概述](http://www.iocoder.cn/HikariCP/zhazhawangzi/FastList/)
- [源码分析](http://www.iocoder.cn/HikariCP/zhazhawangzi/FastList/)
- [参考资料](http://www.iocoder.cn/HikariCP/zhazhawangzi/FastList/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 概述

在Down-the-Rabbit-Hole中，作者提到了一些 Micro-optimizations，表达了作者追求极致的态度。

> HikariCP contains many micro-optimizations that individually are barely measurable, but together combine as a boost to overall performance. Some of these optimizations are measured in fractions of a millisecond amortized over millions of invocations.

其中第一条就提到了ArrayList的优化

> One non-trivial (performance-wise) optimization was eliminating the use of an ArrayList instance in the ConnectionProxy used to track open Statement instances. When a Statement is closed, it must be removed from this collection, and when the Connection is closed it must iterate the collection and close any open Statement instances, and finally must clear the collection. The Java ArrayList, wisely for general purpose use, performs a range check upon every get(int index) call. However, because we can provide guarantees about our ranges, this check is merely overhead.
>
> Additionally, the remove(Object) implementation performs a scan from head to tail, however common patterns in JDBC programming are to close Statements immediately after use, or in reverse order of opening. For these cases, a scan that starts at the tail will perform better. Therefore, ArrayList was replaced with a custom class FastList which eliminates range checking and performs removal scans from tail to head.

FastList是一个List接口的精简实现，只实现了接口中必要的几个方法。JDK ArrayList每次调用get()方法时都会进行rangeCheck检查索引是否越界，FastList的实现中去除了这一检查，只要保证索引合法那么rangeCheck就成为了不必要的计算开销(当然开销极小)。此外，HikariCP使用List来保存打开的Statement，当Statement关闭或Connection关闭时需要将对应的Statement从List中移除。通常情况下，同一个Connection创建了多个Statement时，后打开的Statement会先关闭。ArrayList的remove(Object)方法是从头开始遍历数组，而FastList是从数组的尾部开始遍历，因此更为高效。

简而言之就是 **自定义数组类型（FastList）代替ArrayList：避免每次get()调用都要进行range check，避免调用remove()时的从头到尾的扫描**

# 源码分析

java.util.ArrayList

```Java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

com.zaxxer.hikari.util.FastList

```Java
public final class FastList<T> implements List<T>, RandomAccess, Serializable
```

我们先看一下java.util.ArrayList的get方法

```Java
  /**
     * Returns the element at the specified position in this list.
     *
     * @param  index index of the element to return
     * @return the element at the specified position in this list
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E get(int index) {
        rangeCheck(index);
        return elementData(index);
    }
```

我们可以看到每次get的时候都会进行rangeCheck

```Java
    /**
     * Checks if the given index is in range.  If not, throws an appropriate
     * runtime exception.  This method does *not* check if the index is
     * negative: It is always used immediately prior to an array access,
     * which throws an ArrayIndexOutOfBoundsException if index is negative.
     */
    private void rangeCheck(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
```

我们对比一下com.zaxxer.hikari.util.FastList的get方法，可以看到取消了rangeCheck，一定程度下追求了极致

```Java
  /**
    * Get the element at the specified index.
    *
    * @param index the index of the element to get
    * @return the element, or ArrayIndexOutOfBounds is thrown if the index is invalid
    */
   @Override
   public T get(int index)
   {
      return elementData[index];
   }
```

我们再来看一下ArrayList的remove(Object)方法

```Java
    /**
     * Removes the first occurrence of the specified element from this list,
     * if it is present.  If the list does not contain the element, it is
     * unchanged.  More formally, removes the element with the lowest index
     * <tt>i</tt> such that
     * <tt>(o==null&nbsp;?&nbsp;get(i)==null&nbsp;:&nbsp;o.equals(get(i)))</tt>
     * (if such an element exists).  Returns <tt>true</tt> if this list
     * contained the specified element (or equivalently, if this list
     * changed as a result of the call).
     *
     * @param o element to be removed from this list, if present
     * @return <tt>true</tt> if this list contained the specified element
     */
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

再对比看一下FastList的remove(Object方法)

```Java
   /**
    * This remove method is most efficient when the element being removed
    * is the last element.  Equality is identity based, not equals() based.
    * Only the first matching element is removed.
    *
    * @param element the element to remove
    */
   @Override
   public boolean remove(Object element)
   {
      for (int index = size - 1; index >= 0; index--) {
         if (element == elementData[index]) {
            final int numMoved = size - index - 1;
            if (numMoved > 0) {
               System.arraycopy(elementData, index + 1, elementData, index, numMoved);
            }
            elementData[--size] = null;
            return true;
         }
      }
      return false;
   }
```

好了，今天的文章就是这么简短，最后附一下FastList的源码，内容真的是十分精简的。

```Java
/*
 * Copyright (C) 2013, 2014 Brett Wooldridge
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package com.zaxxer.hikari.util;
import java.io.Serializable;
import java.lang.reflect.Array;
import java.util.Collection;
import java.util.Comparator;
import java.util.Iterator;
import java.util.List;
import java.util.ListIterator;
import java.util.NoSuchElementException;
import java.util.RandomAccess;
import java.util.Spliterator;
import java.util.function.Consumer;
import java.util.function.Predicate;
import java.util.function.UnaryOperator;
/**
 * Fast list without range checking.
 *
 * @author Brett Wooldridge
 */
public final class FastList<T> implements List<T>, RandomAccess, Serializable
{
   private static final long serialVersionUID = -4598088075242913858L;
   private final Class<?> clazz;
   private T[] elementData;
   private int size;
   /**
    * Construct a FastList with a default size of 32.
    * @param clazz the Class stored in the collection
    */
   @SuppressWarnings("unchecked")
   public FastList(Class<?> clazz)
   {
      this.elementData = (T[]) Array.newInstance(clazz, 32);
      this.clazz = clazz;
   }
   /**
    * Construct a FastList with a specified size.
    * @param clazz the Class stored in the collection
    * @param capacity the initial size of the FastList
    */
   @SuppressWarnings("unchecked")
   public FastList(Class<?> clazz, int capacity)
   {
      this.elementData = (T[]) Array.newInstance(clazz, capacity);
      this.clazz = clazz;
   }
   /**
    * Add an element to the tail of the FastList.
    *
    * @param element the element to add
    */
   @Override
   public boolean add(T element)
   {
      if (size < elementData.length) {
         elementData[size++] = element;
      }
      else {
         // overflow-conscious code
         final int oldCapacity = elementData.length;
         final int newCapacity = oldCapacity << 1;
         @SuppressWarnings("unchecked")
         final T[] newElementData = (T[]) Array.newInstance(clazz, newCapacity);
         System.arraycopy(elementData, 0, newElementData, 0, oldCapacity);
         newElementData[size++] = element;
         elementData = newElementData;
      }
      return true;
   }
   /**
    * Get the element at the specified index.
    *
    * @param index the index of the element to get
    * @return the element, or ArrayIndexOutOfBounds is thrown if the index is invalid
    */
   @Override
   public T get(int index)
   {
      return elementData[index];
   }
   /**
    * Remove the last element from the list.  No bound check is performed, so if this
    * method is called on an empty list and ArrayIndexOutOfBounds exception will be
    * thrown.
    *
    * @return the last element of the list
    */
   public T removeLast()
   {
      T element = elementData[--size];
      elementData[size] = null;
      return element;
   }
   /**
    * This remove method is most efficient when the element being removed
    * is the last element.  Equality is identity based, not equals() based.
    * Only the first matching element is removed.
    *
    * @param element the element to remove
    */
   @Override
   public boolean remove(Object element)
   {
      for (int index = size - 1; index >= 0; index--) {
         if (element == elementData[index]) {
            final int numMoved = size - index - 1;
            if (numMoved > 0) {
               System.arraycopy(elementData, index + 1, elementData, index, numMoved);
            }
            elementData[--size] = null;
            return true;
         }
      }
      return false;
   }
   /**
    * Clear the FastList.
    */
   @Override
   public void clear()
   {
      for (int i = 0; i < size; i++) {
         elementData[i] = null;
      }
      size = 0;
   }
   /**
    * Get the current number of elements in the FastList.
    *
    * @return the number of current elements
    */
   @Override
   public int size()
   {
      return size;
   }
   /** {@inheritDoc} */
   @Override
   public boolean isEmpty()
   {
      return size == 0;
   }
   /** {@inheritDoc} */
   @Override
   public T set(int index, T element)
   {
      T old = elementData[index];
      elementData[index] = element;
      return old;
   }
   /** {@inheritDoc} */
   @Override
   public T remove(int index)
   {
      if (size == 0) {
         return null;
      }
      final T old = elementData[index];
      final int numMoved = size - index - 1;
      if (numMoved > 0) {
         System.arraycopy(elementData, index + 1, elementData, index, numMoved);
      }
      elementData[--size] = null;
      return old;
   }
   /** {@inheritDoc} */
   @Override
   public boolean contains(Object o)
   {
      throw new UnsupportedOperationException();
   }
   /** {@inheritDoc} */
   @Override
   public Iterator<T> iterator()
   {
      return new Iterator<T>() {
         private int index;
         @Override
         public boolean hasNext()
         {
            return index < size;
         }
         @Override
         public T next()
         {
            if (index < size) {
               return elementData[index++];
            }
            throw new NoSuchElementException("No more elements in FastList");
         }
      };
   }
   /** {@inheritDoc} */
   @Override
   public Object[] toArray()
   {
      throw new UnsupportedOperationException();
   }
   /** {@inheritDoc} */
   @Override
   public <E> E[] toArray(E[] a)
   {
      throw new UnsupportedOperationException();
   }
   /** {@inheritDoc} */
   @Override
   public boolean containsAll(Collection<?> c)
   {
      throw new UnsupportedOperationException();
   }
   /** {@inheritDoc} */
   @Override
   public boolean addAll(Collection<? extends T> c)
   {
      throw new UnsupportedOperationException();
   }
   /** {@inheritDoc} */
   @Override
   public boolean addAll(int index, Collection<? extends T> c)
   {
      throw new UnsupportedOperationException();
   }
   /** {@inheritDoc} */
   @Override
   public boolean removeAll(Collection<?> c)
   {
      throw new UnsupportedOperationException();
   }
   /** {@inheritDoc} */
   @Override
   public boolean retainAll(Collection<?> c)
   {
      throw new UnsupportedOperationException();
   }
   /** {@inheritDoc} */
   @Override
   public void add(int index, T element)
   {
      throw new UnsupportedOperationException();
   }
   /** {@inheritDoc} */
   @Override
   public int indexOf(Object o)
   {
      throw new UnsupportedOperationException();
   }
   /** {@inheritDoc} */
   @Override
   public int lastIndexOf(Object o)
   {
      throw new UnsupportedOperationException();
   }
   /** {@inheritDoc} */
   @Override
   public ListIterator<T> listIterator()
   {
      throw new UnsupportedOperationException();
   }
   /** {@inheritDoc} */
   @Override
   public ListIterator<T> listIterator(int index)
   {
      throw new UnsupportedOperationException();
   }
   /** {@inheritDoc} */
   @Override
   public List<T> subList(int fromIndex, int toIndex)
   {
      throw new UnsupportedOperationException();
   }
   /** {@inheritDoc} */
   @Override
   public Object clone()
   {
      throw new UnsupportedOperationException();
   }
   /** {@inheritDoc} */
   @Override
   public void forEach(Consumer<? super T> action)
   {
      throw new UnsupportedOperationException();
   }
   /** {@inheritDoc} */
   @Override
   public Spliterator<T> spliterator()
   {
      throw new UnsupportedOperationException();
   }
   /** {@inheritDoc} */
   @Override
   public boolean removeIf(Predicate<? super T> filter)
   {
      throw new UnsupportedOperationException();
   }
   /** {@inheritDoc} */
   @Override
   public void replaceAll(UnaryOperator<T> operator)
   {
      throw new UnsupportedOperationException();
   }
   /** {@inheritDoc} */
   @Override
   public void sort(Comparator<? super T> c)
   {
      throw new UnsupportedOperationException();
   }
}
```

# 参考资料

https://github.com/brettwooldridge/HikariCP/wiki/Down-the-Rabbit-Hole

# 666. 彩蛋

如果你对 HikariCP 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)