## 上源码

~~~java
package java.util.concurrent;

import java.lang.invoke.VarHandle;
import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collection;
import java.util.Collections;
import java.util.Comparator;
import java.util.ConcurrentModificationException;
import java.util.Iterator;
import java.util.List;
import java.util.ListIterator;
import java.util.NoSuchElementException;
import java.util.Objects;
import java.util.RandomAccess;
import java.util.Spliterator;
import java.util.Spliterators;
import java.util.function.Consumer;
import java.util.function.IntFunction;
import java.util.function.Predicate;
import java.util.function.UnaryOperator;
import java.util.stream.Stream;
import java.util.stream.StreamSupport;
import jdk.internal.access.SharedSecrets;
import jdk.internal.util.ArraysSupport;


/**
 * @Classname CopyOnWriteArrayList
 *
 * @Description TODO 源码解析
 * @Author Elon.Zhang
 * @Date 2024/6/26
 */
public class CopyOnWriteArrayList<E>
		implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
	private static final long serialVersionUID = 8673264195747942595L;

	/**
	 * 相比于使用ReentrantLock，现在使用synchronized效率更高些（Open JDK 21）
	 */
	final transient Object lock = new Object();

	/** The array, accessed only via getArray/setArray. */
	private transient volatile Object[] array;

	/**
	 * 获取数组
	 * @return
	 */
	final Object[] getArray() {
		return array;
	}

	/**
	 * 将array指向数组a
	 * @param a
	 */
	final void setArray(Object[] a) {
		array = a;
	}

	/**
	 * 创建一个空的CopyOnWriteArrayList对象
	 */
	public CopyOnWriteArrayList() {
		setArray(new Object[0]);
	}

	/**
	 * 创建一个带集合参数的CopyOnWriteArrayList
	 * 如果参数集合就是CopyOnWriteArrayList，那就直接将array指向集合的数组
	 * 如果参数集合不是CopyOnWriteArrayList，那么需要将原集合中的对象转存到新的数组中，然后将array指向新的数组
	 *
	 * 还有一点，CopyOnWriteArrayList提供的构造参数可以体现出设计者对于CopyOnWriteArrayList使用的一个情景：
	 * 设计者希望使用CopyOnWriteArrayList时，尽量使用批量插入的方式。如果是单个插入的话，会带来高额的系统负担，每一个插入操作都会创建一个新的数组。
	 * 内存中会有大量的垃圾内存需要回收处理。
	 *
	 * @param c
	 * @throws NullPointerException 如果c为空，会抛出空指针异常
	 */
	public CopyOnWriteArrayList(Collection<? extends E> c) {
		Object[] es;
		// 首先判断传入的集合是不是CopyOnWriteArrayList，如果是的话，直接将array指向c的数组
		if (c.getClass() == CopyOnWriteArrayList.class)
			es = ((CopyOnWriteArrayList<?>)c).getArray();
		else {
			es = c.toArray();
			if (c.getClass() != java.util.ArrayList.class)
				es = Arrays.copyOf(es, es.length, Object[].class);
		}
		setArray(es);
	}

	/**
	 * 创建一个带有给定数组参数的CopyOnWriteArrayList
	 * 直接将数组复制，然后将array指向新数组
	 *
	 * @param toCopyIn
	 * @throws NullPointerException 如果toCopyIn数组为空，则抛出空指针异常
	 */
	public CopyOnWriteArrayList(E[] toCopyIn) {
		// 直接将toCopyIn复制到一个新数组，然后将CopyOnWriteArrayList的array指向新数组
		setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
	}

	/**
	 * 返回数组/集合长度
	 *
	 * @return 数组/集合长度
	 */
	public int size() {
		return getArray().length;
	}

	/**
	 * 判断集合是否为空
	 *
	 * @return {@code true} if this list contains no elements
	 */
	public boolean isEmpty() {
		return size() == 0;
	}

	/**
	 * 静态版的indexOf方法，不需要实例化CopyOnWriteArrayList，直接调用即可。
	 *
	 * @param o 需要寻找的元素
	 * @param es 当前CopyOnWriteArrayList的内部数组
	 * @param from 开始下标
	 * @param to 结束为止
	 * @return 如果找到，返回对应下标，如果没有返回-1
	 */
	private static int indexOfRange(Object o, Object[] es, int from, int to) {
		// CopyOnWriteArrayList是允许null值的
		if (o == null) {
			for (int i = from; i < to; i++)
				if (es[i] == null)
					return i;
		} else {
			for (int i = from; i < to; i++)
				if (o.equals(es[i]))
					return i;
		}
		// 如果遍历之后，还是没有（即使是null值），那就返回-1
		return -1;
	}

	/**
	 * 静态版的lastIndexOf，无需实例化CopyOnWriteArrayList
	 *
	 * @param o 需要寻找的元素
	 * @param es 当前CopyOnWriteArrayList的内部数组
	 * @param from 开始下标，从后往前找
	 * @param to 结束下标，数组的开头位置
	 * @return 如果找到，返回对应下标，如果没有返回-1
	 */
	private static int lastIndexOfRange(Object o, Object[] es, int from, int to) {
		// 和indexOfRange方法逻辑相同，不过是使用倒序遍历查找
		if (o == null) {
			for (int i = to - 1; i >= from; i--)
				if (es[i] == null)
					return i;
		} else {
			for (int i = to - 1; i >= from; i--)
				if (o.equals(es[i]))
					return i;
		}
		return -1;
	}

	/**
	 * 判断对应值是否存在
	 *
	 * @param o element whose presence in this list is to be tested
	 * @return {@code true} if this list contains the specified element
	 */
	public boolean contains(Object o) {
		return indexOf(o) >= 0;
	}

	/**
	 * {@inheritDoc}
	 */
	public int indexOf(Object o) {
		Object[] es = getArray();
		return indexOfRange(o, es, 0, es.length);
	}

	/**
	 * 返回第一个等于e的元素的位置，如果没有找到的话，返回-1
	 *
	 * @param e 寻找的目标值
	 * @param index 开始寻找的下标位置
	 * @return the index of the first occurrence of the element in
	 *         this list at position {@code index} or later in the list;
	 *         {@code -1} if the element is not found.
	 * @throws IndexOutOfBoundsException if the specified index is negative
	 */
	public int indexOf(E e, int index) {
		Object[] es = getArray();
		return indexOfRange(e, es, index, es.length);
	}

	/**
	 * {@inheritDoc}
	 */
	public int lastIndexOf(Object o) {
		Object[] es = getArray();
		return lastIndexOfRange(o, es, 0, es.length);
	}

	/**
	 * 和indexOf一样，不过是倒序查找，从后往前找第一个等于e的元素
	 *
	 * @param e 寻找的目标值
	 * @param index 寻找的起始位置
	 * @return the index of the last occurrence of the element at position
	 *         less than or equal to {@code index} in this list;
	 *         -1 if the element is not found.
	 * @throws IndexOutOfBoundsException if the specified index is greater
	 *         than or equal to the current size of this list
	 */
	public int lastIndexOf(E e, int index) {
		Object[] es = getArray();
		return lastIndexOfRange(e, es, 0, index + 1);
	}

	/**
	 * 浅拷贝，内存地址不变
	 *
	 * @return a clone of this list
	 */
	public Object clone() {
		try {
			@SuppressWarnings("unchecked")
			CopyOnWriteArrayList<E> clone =
					(CopyOnWriteArrayList<E>) super.clone();
			clone.resetLock();
			// Unlike in readObject, here we cannot visibility-piggyback on the
			// volatile write in setArray().
			VarHandle.releaseFence();
			return clone;
		} catch (CloneNotSupportedException e) {
			// this shouldn't happen, since we are Cloneable
			throw new InternalError();
		}
	}

	/**
	 * 返回一个包含此列表中所有元素的数组
	 * 按正确的顺序（从第一个元素到最后一个元素）。
	 *
	 * 并且，此时返回的array是一个安全的数组，CopyOnWriteArrayList本身的array指向的对象不变，也不提供当前的array。
	 * 直接提供一个当前array的克隆
	 *
	 * @return an array containing all the elements in this list
	 */
	public Object[] toArray() {
		return getArray().clone();
	}

	/**
	 * Returns an array containing all of the elements in this list in
	 * proper sequence (from first to last element); the runtime type of
	 * the returned array is that of the specified array.  If the list fits
	 * in the specified array, it is returned therein.  Otherwise, a new
	 * array is allocated with the runtime type of the specified array and
	 * the size of this list.
	 *
	 * <p>If this list fits in the specified array with room to spare
	 * (i.e., the array has more elements than this list), the element in
	 * the array immediately following the end of the list is set to
	 * {@code null}.  (This is useful in determining the length of this
	 * list <i>only</i> if the caller knows that this list does not contain
	 * any null elements.)
	 *
	 * <p>Like the {@link #toArray()} method, this method acts as bridge between
	 * array-based and collection-based APIs.  Further, this method allows
	 * precise control over the runtime type of the output array, and may,
	 * under certain circumstances, be used to save allocation costs.
	 *
	 * <p>Suppose {@code x} is a list known to contain only strings.
	 * The following code can be used to dump the list into a newly
	 * allocated array of {@code String}:
	 *
	 * <pre> {@code String[] y = x.toArray(new String[0]);}</pre>
	 *
	 * Note that {@code toArray(new Object[0])} is identical in function to
	 * {@code toArray()}.
	 *
	 * @param a the array into which the elements of the list are to
	 *          be stored, if it is big enough; otherwise, a new array of the
	 *          same runtime type is allocated for this purpose.
	 * @return an array containing all the elements in this list
	 * @throws ArrayStoreException if the runtime type of the specified array
	 *         is not a supertype of the runtime type of every element in
	 *         this list
	 * @throws NullPointerException if the specified array is null
	 */
	@SuppressWarnings("unchecked")
	public <T> T[] toArray(T[] a) {
		Object[] es = getArray();
		int len = es.length;
		if (a.length < len)
			return (T[]) Arrays.copyOf(es, len, a.getClass());
		else {
			System.arraycopy(es, 0, a, 0, len);
			if (a.length > len)
				a[len] = null;
			return a;
		}
	}

	// 下标访问操作
	@SuppressWarnings("unchecked")
	static <E> E elementAt(Object[] a, int index) {
		return (E) a[index];
	}

	static String outOfBounds(int index, int size) {
		return "Index: " + index + ", Size: " + size;
	}

	/**
	 * {@inheritDoc}
	 *
	 * @throws IndexOutOfBoundsException {@inheritDoc}
	 */
	public E get(int index) {
		return elementAt(getArray(), index);
	}

	/**
	 * {@inheritDoc}
	 *
	 * @throws NoSuchElementException {@inheritDoc}
	 * @since 21
	 */
	public E getFirst() {
		Object[] es = getArray();
		if (es.length == 0)
			throw new NoSuchElementException();
		else
			return elementAt(es, 0);
	}

	/**
	 * {@inheritDoc}
	 *
	 * @throws NoSuchElementException {@inheritDoc}
	 * @since 21
	 */
	public E getLast() {
		Object[] es = getArray();
		if (es.length == 0)
			throw new NoSuchElementException();
		else
			return elementAt(es, es.length - 1);
	}

	/**
	 * 将指定index位置的元素值进行修改
	 * @param index 修改元素的位置
	 * @param element 修改之后的值
	 * @throws IndexOutOfBoundsException {@inheritDoc}
	 */
	public E set(int index, E element) {
		synchronized (lock) {
			// 获取当前array，并且将es指向当前array
			Object[] es = getArray();
			// 先获取指定index的值
			E oldValue = elementAt(es, index);

			// 判断指定index的值和修改的值进行比较
			if (oldValue != element) {
				// 如果指定index的值和要修改的值不同，那么就将es指向一个array的克隆，对es进行修改
				es = es.clone();
				es[index] = element;
			}
			/**
			 * 无论是否对值进行修改，都将array指向es的新数组。
			 * 因为要考虑到CAS操作的ABA问题
			 */
			setArray(es);
			return oldValue;
		}
	}

	/**
	 * 使用尾插法，在数组最后插入元素。
	 * 如果有大量数据的情况下，不推荐使用add()方法，可以使用addAllAbsent()方法。
	 * 大量数据循环单次插入的情况，会导致创建大量的垃圾数组，对内存不友好。
	 *
	 * @param e element to be appended to this list
	 * @return {@code true} (as specified by {@link Collection#add})
	 */
	public boolean add(E e) {
		synchronized (lock) {
			Object[] es = getArray();
			int len = es.length;
			/**
			 * 最主要的，还是写操作会新建一个数组，对新建的数组进行操作。
			 * 还有一点，每次进行写操作的时候，数组的长度只是+1，而不像ArrayList那样直接扩容1.5倍。
			 * 其实还是推荐往CopyOnWriteArrayList进行大量数据写入的时候，通过集合或者数组的方式进行写入。
			 */
			es = Arrays.copyOf(es, len + 1);
			es[len] = e;
			// 操作对新数组操作完成之后，将array指向新数组。
			setArray(es);
			return true;
		}
	}

	/**
	 * 向指定位置插入元素
	 * 这个方法可以分为两种情况
	 * 1.在末尾插入
	 * 2.在中间插入
	 *
	 * @param index 插入元素的位置
	 * @param element 插入的元素值
	 * @throws IndexOutOfBoundsException {@inheritDoc}
	 */
	public void add(int index, E element) {
		synchronized (lock) {
			Object[] es = getArray();
			int len = es.length;
			// 判断插入的位置是否合理
			if (index > len || index < 0)
				throw new IndexOutOfBoundsException(outOfBounds(index, len));
			Object[] newElements;
			// 找到数组从后往前的插入位置
			int numMoved = len - index;
			if (numMoved == 0)
				/**
				 * 此时，在末尾进行插入
				 * 那么就克隆一个新数组，且数组长度相较于原数组+1
				 */
				newElements = Arrays.copyOf(es, len + 1);
			else {
				/**
				 * 此时，在数组的中间插入。
				 * 操作步骤为：
				 * 1.创建一个新数组，数组长度为原数组+1
				 * 2.将插入位置之前的原数组的值，插入到新数组中
				 * 3.将插入位置之后的原数组的值，插入到新数组中
				 */
				newElements = new Object[len + 1];
				System.arraycopy(es, 0, newElements, 0, index);
				System.arraycopy(es, index, newElements, index + 1,
						numMoved);
			}
			// 在克隆出来的新数组的位置插入元素
			newElements[index] = element;
			// 将array指向新数组
			setArray(newElements);
		}
	}

	/**
	 * {@inheritDoc}
	 * 头插
	 *
	 * @since 21
	 */
	public void addFirst(E e) {
		add(0, e);
	}

	/**
	 * {@inheritDoc}
	 * 尾插
	 *
	 * @since 21
	 */
	public void addLast(E e) {
		synchronized (lock) {
			add(getArray().length, e);
		}
	}

	/**
	 * 移除指定位置的元素
	 *
	 * @throws IndexOutOfBoundsException {@inheritDoc}
	 */
	public E remove(int index) {
		synchronized (lock) {
			Object[] es = getArray();
			int len = es.length;
			E oldValue = elementAt(es, index);
			int numMoved = len - index - 1;
			// 还是对新数组进行操作
			Object[] newElements;
			if (numMoved == 0)
				// 从尾部删除就很简单，直接将es拷贝的长度-1即可
				newElements = Arrays.copyOf(es, len - 1);
			else {
				// 从中间删除的操作和插入的操作类似，还是分段进行复制。
				newElements = new Object[len - 1];
				System.arraycopy(es, 0, newElements, 0, index);
				System.arraycopy(es, index + 1, newElements, index,
						numMoved);
			}
			// 将array指向新数组即可
			setArray(newElements);
			return oldValue;
		}
	}

	/**
	 * {@inheritDoc}
	 *
	 * @throws NoSuchElementException {@inheritDoc}
	 * @since 21
	 */
	public E removeFirst() {
		synchronized (lock) {
			if (getArray().length == 0)
				throw new NoSuchElementException();
			else
				return remove(0);
		}
	}

	/**
	 * {@inheritDoc}
	 *
	 * @throws NoSuchElementException {@inheritDoc}
	 * @since 21
	 */
	public E removeLast() {
		synchronized (lock) {
			int size = getArray().length;
			if (size == 0)
				throw new NoSuchElementException();
			else
				return remove(size - 1);
		}
	}

	/**
	 * 移除元素的操作
	 * 首先根据元素找到对应元素的下标
	 * 然后如果有对应下标，那就调用remove(Object o, Object[] snapshot, int index)方法进行删除操作
	 *
	 * @param o element to be removed from this list, if present
	 * @return {@code true} if this list contained the specified element
	 */
	public boolean remove(Object o) {
		Object[] snapshot = getArray();
		// 找到移除元素的下标
		int index = indexOfRange(o, snapshot, 0, snapshot.length);
		return index >= 0 && remove(o, snapshot, index);
	}

	/**
	 * A version of remove(Object) using the strong hint that given
	 * recent snapshot contains o at the given index.
	 */
	private boolean remove(Object o, Object[] snapshot, int index) {
		synchronized (lock) {
			Object[] current = getArray();
			int len = current.length;
			if (snapshot != current) findIndex: {
				int prefix = Math.min(index, len);
				for (int i = 0; i < prefix; i++) {
					if (current[i] != snapshot[i]
							&& Objects.equals(o, current[i])) {
						index = i;
						break findIndex;
					}
				}
				if (index >= len)
					return false;
				if (current[index] == o)
					break findIndex;
				index = indexOfRange(o, current, index, len);
				if (index < 0)
					return false;
			}
			Object[] newElements = new Object[len - 1];
			System.arraycopy(current, 0, newElements, 0, index);
			System.arraycopy(current, index + 1,
					newElements, index,
					len - index - 1);
			setArray(newElements);
			return true;
		}
	}

	/**
	 * Removes from this list all of the elements whose index is between
	 * {@code fromIndex}, inclusive, and {@code toIndex}, exclusive.
	 * Shifts any succeeding elements to the left (reduces their index).
	 * This call shortens the list by {@code (toIndex - fromIndex)} elements.
	 * (If {@code toIndex==fromIndex}, this operation has no effect.)
	 *
	 * @param fromIndex index of first element to be removed
	 * @param toIndex index after last element to be removed
	 * @throws IndexOutOfBoundsException if fromIndex or toIndex out of range
	 *         ({@code fromIndex < 0 || toIndex > size() || toIndex < fromIndex})
	 */
	void removeRange(int fromIndex, int toIndex) {
		synchronized (lock) {
			Object[] es = getArray();
			int len = es.length;

			if (fromIndex < 0 || toIndex > len || toIndex < fromIndex)
				throw new IndexOutOfBoundsException();
			int newlen = len - (toIndex - fromIndex);
			int numMoved = len - toIndex;
			if (numMoved == 0)
				setArray(Arrays.copyOf(es, newlen));
			else {
				Object[] newElements = new Object[newlen];
				System.arraycopy(es, 0, newElements, 0, fromIndex);
				System.arraycopy(es, toIndex, newElements,
						fromIndex, numMoved);
				setArray(newElements);
			}
		}
	}

	/**
	 * Appends the element, if not present.
	 *
	 * @param e element to be added to this list, if absent
	 * @return {@code true} if the element was added
	 */
	public boolean addIfAbsent(E e) {
		Object[] snapshot = getArray();
		return indexOfRange(e, snapshot, 0, snapshot.length) < 0
				&& addIfAbsent(e, snapshot);
	}

	/**
	 * A version of addIfAbsent using the strong hint that given
	 * recent snapshot does not contain e.
	 */
	private boolean addIfAbsent(E e, Object[] snapshot) {
		synchronized (lock) {
			Object[] current = getArray();
			int len = current.length;
			if (snapshot != current) {
				// Optimize for lost race to another addXXX operation
				int common = Math.min(snapshot.length, len);
				for (int i = 0; i < common; i++)
					if (current[i] != snapshot[i]
							&& Objects.equals(e, current[i]))
						return false;
				if (indexOfRange(e, current, common, len) >= 0)
					return false;
			}
			Object[] newElements = Arrays.copyOf(current, len + 1);
			newElements[len] = e;
			setArray(newElements);
			return true;
		}
	}

	/**
	 * Returns {@code true} if this list contains all of the elements of the
	 * specified collection.
	 *
	 * @param c collection to be checked for containment in this list
	 * @return {@code true} if this list contains all of the elements of the
	 *         specified collection
	 * @throws NullPointerException if the specified collection is null
	 * @see #contains(Object)
	 */
	public boolean containsAll(Collection<?> c) {
		Object[] es = getArray();
		int len = es.length;
		for (Object e : c) {
			if (indexOfRange(e, es, 0, len) < 0)
				return false;
		}
		return true;
	}

	/**
	 * Removes from this list all of its elements that are contained in
	 * the specified collection. This is a particularly expensive operation
	 * in this class because of the need for an internal temporary array.
	 *
	 * @param c collection containing elements to be removed from this list
	 * @return {@code true} if this list changed as a result of the call
	 * @throws ClassCastException if the class of an element of this list
	 *         is incompatible with the specified collection
	 * (<a href="{@docRoot}/java.base/java/util/Collection.html#optional-restrictions">optional</a>)
	 * @throws NullPointerException if this list contains a null element and the
	 *         specified collection does not permit null elements
	 * (<a href="{@docRoot}/java.base/java/util/Collection.html#optional-restrictions">optional</a>),
	 *         or if the specified collection is null
	 * @see #remove(Object)
	 */
	public boolean removeAll(Collection<?> c) {
		Objects.requireNonNull(c);
		return bulkRemove(e -> c.contains(e));
	}

	/**
	 * Retains only the elements in this list that are contained in the
	 * specified collection.  In other words, removes from this list all of
	 * its elements that are not contained in the specified collection.
	 *
	 * @param c collection containing elements to be retained in this list
	 * @return {@code true} if this list changed as a result of the call
	 * @throws ClassCastException if the class of an element of this list
	 *         is incompatible with the specified collection
	 * (<a href="{@docRoot}/java.base/java/util/Collection.html#optional-restrictions">optional</a>)
	 * @throws NullPointerException if this list contains a null element and the
	 *         specified collection does not permit null elements
	 * (<a href="{@docRoot}/java.base/java/util/Collection.html#optional-restrictions">optional</a>),
	 *         or if the specified collection is null
	 * @see #remove(Object)
	 */
	public boolean retainAll(Collection<?> c) {
		Objects.requireNonNull(c);
		return bulkRemove(e -> !c.contains(e));
	}

	/**
	 * Appends all of the elements in the specified collection that
	 * are not already contained in this list, to the end of
	 * this list, in the order that they are returned by the
	 * specified collection's iterator.
	 *
	 * @param c collection containing elements to be added to this list
	 * @return the number of elements added
	 * @throws NullPointerException if the specified collection is null
	 * @see #addIfAbsent(Object)
	 */
	public int addAllAbsent(Collection<? extends E> c) {
		Object[] cs = c.toArray();
		if (c.getClass() != ArrayList.class) {
			cs = cs.clone();
		}
		if (cs.length == 0)
			return 0;
		synchronized (lock) {
			Object[] es = getArray();
			int len = es.length;
			int added = 0;
			// uniquify and compact elements in cs
			for (int i = 0; i < cs.length; ++i) {
				Object e = cs[i];
				if (indexOfRange(e, es, 0, len) < 0 &&
						indexOfRange(e, cs, 0, added) < 0)
					cs[added++] = e;
			}
			if (added > 0) {
				Object[] newElements = Arrays.copyOf(es, len + added);
				System.arraycopy(cs, 0, newElements, len, added);
				setArray(newElements);
			}
			return added;
		}
	}

	/**
	 * Removes all of the elements from this list.
	 * The list will be empty after this call returns.
	 */
	public void clear() {
		synchronized (lock) {
			setArray(new Object[0]);
		}
	}

	/**
	 * Appends all of the elements in the specified collection to the end
	 * of this list, in the order that they are returned by the specified
	 * collection's iterator.
	 *
	 * @param c collection containing elements to be added to this list
	 * @return {@code true} if this list changed as a result of the call
	 * @throws NullPointerException if the specified collection is null
	 * @see #add(Object)
	 */
	public boolean addAll(Collection<? extends E> c) {
		Object[] cs = (c.getClass() == CopyOnWriteArrayList.class) ?
				((CopyOnWriteArrayList<?>)c).getArray() : c.toArray();
		if (cs.length == 0)
			return false;
		synchronized (lock) {
			Object[] es = getArray();
			int len = es.length;
			Object[] newElements;
			if (len == 0 && (c.getClass() == CopyOnWriteArrayList.class ||
					c.getClass() == ArrayList.class)) {
				newElements = cs;
			} else {
				newElements = Arrays.copyOf(es, len + cs.length);
				System.arraycopy(cs, 0, newElements, len, cs.length);
			}
			setArray(newElements);
			return true;
		}
	}

	/**
	 * Inserts all of the elements in the specified collection into this
	 * list, starting at the specified position.  Shifts the element
	 * currently at that position (if any) and any subsequent elements to
	 * the right (increases their indices).  The new elements will appear
	 * in this list in the order that they are returned by the
	 * specified collection's iterator.
	 *
	 * @param index index at which to insert the first element
	 *        from the specified collection
	 * @param c collection containing elements to be added to this list
	 * @return {@code true} if this list changed as a result of the call
	 * @throws IndexOutOfBoundsException {@inheritDoc}
	 * @throws NullPointerException if the specified collection is null
	 * @see #add(int,Object)
	 */
	public boolean addAll(int index, Collection<? extends E> c) {
		Object[] cs = c.toArray();
		synchronized (lock) {
			Object[] es = getArray();
			int len = es.length;
			if (index > len || index < 0)
				throw new IndexOutOfBoundsException(outOfBounds(index, len));
			if (cs.length == 0)
				return false;
			int numMoved = len - index;
			Object[] newElements;
			if (numMoved == 0)
				newElements = Arrays.copyOf(es, len + cs.length);
			else {
				newElements = new Object[len + cs.length];
				System.arraycopy(es, 0, newElements, 0, index);
				System.arraycopy(es, index,
						newElements, index + cs.length,
						numMoved);
			}
			System.arraycopy(cs, 0, newElements, index, cs.length);
			setArray(newElements);
			return true;
		}
	}

	/**
	 * @throws NullPointerException {@inheritDoc}
	 */
	public void forEach(Consumer<? super E> action) {
		Objects.requireNonNull(action);
		for (Object x : getArray()) {
			@SuppressWarnings("unchecked") E e = (E) x;
			action.accept(e);
		}
	}

	/**
	 * @throws NullPointerException {@inheritDoc}
	 */
	public boolean removeIf(Predicate<? super E> filter) {
		Objects.requireNonNull(filter);
		return bulkRemove(filter);
	}

	// A tiny bit set implementation

	private static long[] nBits(int n) {
		return new long[((n - 1) >> 6) + 1];
	}
	private static void setBit(long[] bits, int i) {
		bits[i >> 6] |= 1L << i;
	}
	private static boolean isClear(long[] bits, int i) {
		return (bits[i >> 6] & (1L << i)) == 0;
	}

	private boolean bulkRemove(Predicate<? super E> filter) {
		synchronized (lock) {
			return bulkRemove(filter, 0, getArray().length);
		}
	}

	boolean bulkRemove(Predicate<? super E> filter, int i, int end) {
		// assert Thread.holdsLock(lock);
		final Object[] es = getArray();
		// Optimize for initial run of survivors
		for (; i < end && !filter.test(elementAt(es, i)); i++)
			;
		if (i < end) {
			final int beg = i;
			final long[] deathRow = nBits(end - beg);
			int deleted = 1;
			deathRow[0] = 1L;   // set bit 0
			for (i = beg + 1; i < end; i++)
				if (filter.test(elementAt(es, i))) {
					setBit(deathRow, i - beg);
					deleted++;
				}
			// Did filter reentrantly modify the list?
			if (es != getArray())
				throw new ConcurrentModificationException();
			final Object[] newElts = Arrays.copyOf(es, es.length - deleted);
			int w = beg;
			for (i = beg; i < end; i++)
				if (isClear(deathRow, i - beg))
					newElts[w++] = es[i];
			System.arraycopy(es, i, newElts, w, es.length - i);
			setArray(newElts);
			return true;
		} else {
			if (es != getArray())
				throw new ConcurrentModificationException();
			return false;
		}
	}

	public void replaceAll(UnaryOperator<E> operator) {
		synchronized (lock) {
			replaceAllRange(operator, 0, getArray().length);
		}
	}

	void replaceAllRange(UnaryOperator<E> operator, int i, int end) {
		// assert Thread.holdsLock(lock);
		Objects.requireNonNull(operator);
		final Object[] es = getArray().clone();
		for (; i < end; i++)
			es[i] = operator.apply(elementAt(es, i));
		setArray(es);
	}

	public void sort(Comparator<? super E> c) {
		synchronized (lock) {
			sortRange(c, 0, getArray().length);
		}
	}

	@SuppressWarnings("unchecked")
	void sortRange(Comparator<? super E> c, int i, int end) {
		// assert Thread.holdsLock(lock);
		final Object[] es = getArray().clone();
		Arrays.sort(es, i, end, (Comparator<Object>)c);
		setArray(es);
	}

	/**
	 * Saves this list to a stream (that is, serializes it).
	 *
	 * @param s the stream
	 * @throws java.io.IOException if an I/O error occurs
	 * @serialData The length of the array backing the list is emitted
	 *               (int), followed by all of its elements (each an Object)
	 *               in the proper order.
	 */
	private void writeObject(java.io.ObjectOutputStream s)
			throws java.io.IOException {

		s.defaultWriteObject();

		Object[] es = getArray();
		// Write out array length
		s.writeInt(es.length);

		// Write out all elements in the proper order.
		for (Object element : es)
			s.writeObject(element);
	}

	/**
	 * Reconstitutes this list from a stream (that is, deserializes it).
	 * @param s the stream
	 * @throws ClassNotFoundException if the class of a serialized object
	 *         could not be found
	 * @throws java.io.IOException if an I/O error occurs
	 */
	private void readObject(java.io.ObjectInputStream s)
			throws java.io.IOException, ClassNotFoundException {

		s.defaultReadObject();

		// bind to new lock
		resetLock();

		// Read in array length and allocate array
		int len = s.readInt();
		SharedSecrets.getJavaObjectInputStreamAccess().checkArray(s, Object[].class, len);
		Object[] es = new Object[len];

		// Read in all elements in the proper order.
		for (int i = 0; i < len; i++)
			es[i] = s.readObject();
		setArray(es);
	}

	/**
	 * Returns a string representation of this list.  The string
	 * representation consists of the string representations of the list's
	 * elements in the order they are returned by its iterator, enclosed in
	 * square brackets ({@code "[]"}).  Adjacent elements are separated by
	 * the characters {@code ", "} (comma and space).  Elements are
	 * converted to strings as by {@link String#valueOf(Object)}.
	 *
	 * @return a string representation of this list
	 */
	public String toString() {
		return Arrays.toString(getArray());
	}

	/**
	 * Compares the specified object with this list for equality.
	 * Returns {@code true} if the specified object is the same object
	 * as this object, or if it is also a {@link List} and the sequence
	 * of elements returned by an {@linkplain List#iterator() iterator}
	 * over the specified list is the same as the sequence returned by
	 * an iterator over this list.  The two sequences are considered to
	 * be the same if they have the same length and corresponding
	 * elements at the same position in the sequence are <em>equal</em>.
	 * Two elements {@code e1} and {@code e2} are considered
	 * <em>equal</em> if {@code Objects.equals(e1, e2)}.
	 *
	 * @param o the object to be compared for equality with this list
	 * @return {@code true} if the specified object is equal to this list
	 */
	public boolean equals(Object o) {
		if (o == this)
			return true;
		if (!(o instanceof List))
			return false;

		List<?> list = (List<?>)o;
		Iterator<?> it = list.iterator();
		for (Object element : getArray())
			if (!it.hasNext() || !Objects.equals(element, it.next()))
				return false;
		return !it.hasNext();
	}

	private static int hashCodeOfRange(Object[] es, int from, int to) {
		int hashCode = 1;
		for (int i = from; i < to; i++) {
			Object x = es[i];
			hashCode = 31 * hashCode + (x == null ? 0 : x.hashCode());
		}
		return hashCode;
	}

	/**
	 * Returns the hash code value for this list.
	 *
	 * <p>This implementation uses the definition in {@link List#hashCode}.
	 *
	 * @return the hash code value for this list
	 */
	public int hashCode() {
		Object[] es = getArray();
		return hashCodeOfRange(es, 0, es.length);
	}

	/**
	 * 这个就是为什么可以实现读写分离的原因，这里的迭代器是创建的COWIterator，这个迭代器中引用的array是CopyOnWriteArrayList的array。
	 * 但是在写操作的过程中，还没有将array指向写操作中新创建的数组，所以这是弱数据一致性。
	 *
	 * <p>The returned iterator provides a snapshot of the state of the list
	 * when the iterator was constructed. No synchronization is needed while
	 * traversing the iterator. The iterator does <em>NOT</em> support the
	 * {@code remove} method.
	 *
	 * @return an iterator over the elements in this list in proper sequence
	 */
	public Iterator<E> iterator() {
		return new COWIterator<E>(getArray(), 0);
	}

	/**
	 * {@inheritDoc}
	 *
	 * <p>The returned iterator provides a snapshot of the state of the list
	 * when the iterator was constructed. No synchronization is needed while
	 * traversing the iterator. The iterator does <em>NOT</em> support the
	 * {@code remove}, {@code set} or {@code add} methods.
	 */
	public ListIterator<E> listIterator() {
		return new COWIterator<E>(getArray(), 0);
	}

	/**
	 * {@inheritDoc}
	 *
	 * <p>The returned iterator provides a snapshot of the state of the list
	 * when the iterator was constructed. No synchronization is needed while
	 * traversing the iterator. The iterator does <em>NOT</em> support the
	 * {@code remove}, {@code set} or {@code add} methods.
	 *
	 * @throws IndexOutOfBoundsException {@inheritDoc}
	 */
	public ListIterator<E> listIterator(int index) {
		Object[] es = getArray();
		int len = es.length;
		if (index < 0 || index > len)
			throw new IndexOutOfBoundsException(outOfBounds(index, len));

		return new COWIterator<E>(es, index);
	}

	/**
	 * Returns a {@link Spliterator} over the elements in this list.
	 *
	 * <p>The {@code Spliterator} reports {@link Spliterator#IMMUTABLE},
	 * {@link Spliterator#ORDERED}, {@link Spliterator#SIZED}, and
	 * {@link Spliterator#SUBSIZED}.
	 *
	 * <p>The spliterator provides a snapshot of the state of the list
	 * when the spliterator was constructed. No synchronization is needed while
	 * operating on the spliterator.
	 *
	 * @return a {@code Spliterator} over the elements in this list
	 * @since 1.8
	 */
	public Spliterator<E> spliterator() {
		return Spliterators.spliterator
				(getArray(), Spliterator.IMMUTABLE | Spliterator.ORDERED);
	}

	static final class COWIterator<E> implements ListIterator<E> {
		/** Snapshot of the array */
		private final Object[] snapshot;
		/** Index of element to be returned by subsequent call to next.  */
		private int cursor;

		COWIterator(Object[] es, int initialCursor) {
			cursor = initialCursor;
			snapshot = es;
		}

		public boolean hasNext() {
			return cursor < snapshot.length;
		}

		public boolean hasPrevious() {
			return cursor > 0;
		}

		@SuppressWarnings("unchecked")
		public E next() {
			if (! hasNext())
				throw new NoSuchElementException();
			return (E) snapshot[cursor++];
		}

		@SuppressWarnings("unchecked")
		public E previous() {
			if (! hasPrevious())
				throw new NoSuchElementException();
			return (E) snapshot[--cursor];
		}

		public int nextIndex() {
			return cursor;
		}

		public int previousIndex() {
			return cursor - 1;
		}

		/**
		 * Not supported. Always throws UnsupportedOperationException.
		 * @throws UnsupportedOperationException always; {@code remove}
		 *         is not supported by this iterator.
		 */
		public void remove() {
			throw new UnsupportedOperationException();
		}

		/**
		 * Not supported. Always throws UnsupportedOperationException.
		 * @throws UnsupportedOperationException always; {@code set}
		 *         is not supported by this iterator.
		 */
		public void set(E e) {
			throw new UnsupportedOperationException();
		}

		/**
		 * Not supported. Always throws UnsupportedOperationException.
		 * @throws UnsupportedOperationException always; {@code add}
		 *         is not supported by this iterator.
		 */
		public void add(E e) {
			throw new UnsupportedOperationException();
		}

		@Override
		public void forEachRemaining(Consumer<? super E> action) {
			Objects.requireNonNull(action);
			final int size = snapshot.length;
			int i = cursor;
			cursor = size;
			for (; i < size; i++)
				action.accept(elementAt(snapshot, i));
		}
	}

	/**
	 * Returns a view of the portion of this list between
	 * {@code fromIndex}, inclusive, and {@code toIndex}, exclusive.
	 * The returned list is backed by this list, so changes in the
	 * returned list are reflected in this list.
	 *
	 * <p>The semantics of the list returned by this method become
	 * undefined if the backing list (i.e., this list) is modified in
	 * any way other than via the returned list.
	 *
	 * @param fromIndex low endpoint (inclusive) of the subList
	 * @param toIndex high endpoint (exclusive) of the subList
	 * @return a view of the specified range within this list
	 * @throws IndexOutOfBoundsException {@inheritDoc}
	 */
	public List<E> subList(int fromIndex, int toIndex) {
		synchronized (lock) {
			Object[] es = getArray();
			int len = es.length;
			int size = toIndex - fromIndex;
			if (fromIndex < 0 || toIndex > len || size < 0)
				throw new IndexOutOfBoundsException();
			return new COWSubList(es, fromIndex, size);
		}
	}

	/**
	 * Sublist for CopyOnWriteArrayList.
	 */
	private class COWSubList implements List<E>, RandomAccess {
		private final int offset;
		private int size;
		private Object[] expectedArray;

		COWSubList(Object[] es, int offset, int size) {
			// assert Thread.holdsLock(lock);
			expectedArray = es;
			this.offset = offset;
			this.size = size;
		}

		private void checkForComodification() {
			// assert Thread.holdsLock(lock);
			if (getArray() != expectedArray)
				throw new ConcurrentModificationException();
		}

		private Object[] getArrayChecked() {
			// assert Thread.holdsLock(lock);
			Object[] a = getArray();
			if (a != expectedArray)
				throw new ConcurrentModificationException();
			return a;
		}

		private void rangeCheck(int index) {
			// assert Thread.holdsLock(lock);
			if (index < 0 || index >= size)
				throw new IndexOutOfBoundsException(outOfBounds(index, size));
		}

		private void rangeCheckForAdd(int index) {
			// assert Thread.holdsLock(lock);
			if (index < 0 || index > size)
				throw new IndexOutOfBoundsException(outOfBounds(index, size));
		}

		public Object[] toArray() {
			final Object[] es;
			final int offset;
			final int size;
			synchronized (lock) {
				es = getArrayChecked();
				offset = this.offset;
				size = this.size;
			}
			return Arrays.copyOfRange(es, offset, offset + size);
		}

		@SuppressWarnings("unchecked")
		public <T> T[] toArray(T[] a) {
			final Object[] es;
			final int offset;
			final int size;
			synchronized (lock) {
				es = getArrayChecked();
				offset = this.offset;
				size = this.size;
			}
			if (a.length < size)
				return (T[]) Arrays.copyOfRange(
						es, offset, offset + size, a.getClass());
			else {
				System.arraycopy(es, offset, a, 0, size);
				if (a.length > size)
					a[size] = null;
				return a;
			}
		}

		public int indexOf(Object o) {
			final Object[] es;
			final int offset;
			final int size;
			synchronized (lock) {
				es = getArrayChecked();
				offset = this.offset;
				size = this.size;
			}
			int i = indexOfRange(o, es, offset, offset + size);
			return (i == -1) ? -1 : i - offset;
		}

		public int lastIndexOf(Object o) {
			final Object[] es;
			final int offset;
			final int size;
			synchronized (lock) {
				es = getArrayChecked();
				offset = this.offset;
				size = this.size;
			}
			int i = lastIndexOfRange(o, es, offset, offset + size);
			return (i == -1) ? -1 : i - offset;
		}

		public boolean contains(Object o) {
			return indexOf(o) >= 0;
		}

		public boolean containsAll(Collection<?> c) {
			final Object[] es;
			final int offset;
			final int size;
			synchronized (lock) {
				es = getArrayChecked();
				offset = this.offset;
				size = this.size;
			}
			for (Object o : c)
				if (indexOfRange(o, es, offset, offset + size) < 0)
					return false;
			return true;
		}

		public boolean isEmpty() {
			return size() == 0;
		}

		public String toString() {
			return Arrays.toString(toArray());
		}

		public int hashCode() {
			final Object[] es;
			final int offset;
			final int size;
			synchronized (lock) {
				es = getArrayChecked();
				offset = this.offset;
				size = this.size;
			}
			return hashCodeOfRange(es, offset, offset + size);
		}

		public boolean equals(Object o) {
			if (o == this)
				return true;
			if (!(o instanceof List))
				return false;
			Iterator<?> it = ((List<?>)o).iterator();

			final Object[] es;
			final int offset;
			final int size;
			synchronized (lock) {
				es = getArrayChecked();
				offset = this.offset;
				size = this.size;
			}

			for (int i = offset, end = offset + size; i < end; i++)
				if (!it.hasNext() || !Objects.equals(es[i], it.next()))
					return false;
			return !it.hasNext();
		}

		public E set(int index, E element) {
			synchronized (lock) {
				rangeCheck(index);
				checkForComodification();
				E x = CopyOnWriteArrayList.this.set(offset + index, element);
				expectedArray = getArray();
				return x;
			}
		}

		public E get(int index) {
			synchronized (lock) {
				rangeCheck(index);
				checkForComodification();
				return CopyOnWriteArrayList.this.get(offset + index);
			}
		}

		public E getFirst() {
			synchronized (lock) {
				if (size == 0)
					throw new NoSuchElementException();
				else
					return get(0);
			}
		}

		public E getLast() {
			synchronized (lock) {
				if (size == 0)
					throw new NoSuchElementException();
				else
					return get(size - 1);
			}
		}

		public int size() {
			synchronized (lock) {
				checkForComodification();
				return size;
			}
		}

		public boolean add(E element) {
			synchronized (lock) {
				checkForComodification();
				CopyOnWriteArrayList.this.add(offset + size, element);
				expectedArray = getArray();
				size++;
			}
			return true;
		}

		public void add(int index, E element) {
			synchronized (lock) {
				checkForComodification();
				rangeCheckForAdd(index);
				CopyOnWriteArrayList.this.add(offset + index, element);
				expectedArray = getArray();
				size++;
			}
		}

		public void addFirst(E e) {
			add(0, e);
		}

		public void addLast(E e) {
			synchronized (lock) {
				add(size, e);
			}
		}

		public boolean addAll(Collection<? extends E> c) {
			synchronized (lock) {
				final Object[] oldArray = getArrayChecked();
				boolean modified =
						CopyOnWriteArrayList.this.addAll(offset + size, c);
				size += (expectedArray = getArray()).length - oldArray.length;
				return modified;
			}
		}

		public boolean addAll(int index, Collection<? extends E> c) {
			synchronized (lock) {
				rangeCheckForAdd(index);
				final Object[] oldArray = getArrayChecked();
				boolean modified =
						CopyOnWriteArrayList.this.addAll(offset + index, c);
				size += (expectedArray = getArray()).length - oldArray.length;
				return modified;
			}
		}

		public void clear() {
			synchronized (lock) {
				checkForComodification();
				removeRange(offset, offset + size);
				expectedArray = getArray();
				size = 0;
			}
		}

		public E remove(int index) {
			synchronized (lock) {
				rangeCheck(index);
				checkForComodification();
				E result = CopyOnWriteArrayList.this.remove(offset + index);
				expectedArray = getArray();
				size--;
				return result;
			}
		}

		public E removeFirst() {
			synchronized (lock) {
				if (size == 0)
					throw new NoSuchElementException();
				else
					return remove(0);
			}
		}

		public E removeLast() {
			synchronized (lock) {
				if (size == 0)
					throw new NoSuchElementException();
				else
					return remove(size - 1);
			}
		}

		public boolean remove(Object o) {
			synchronized (lock) {
				checkForComodification();
				int index = indexOf(o);
				if (index == -1)
					return false;
				remove(index);
				return true;
			}
		}

		public Iterator<E> iterator() {
			return listIterator(0);
		}

		public ListIterator<E> listIterator() {
			return listIterator(0);
		}

		public ListIterator<E> listIterator(int index) {
			synchronized (lock) {
				checkForComodification();
				rangeCheckForAdd(index);
				return new COWSubListIterator<E>(
						CopyOnWriteArrayList.this, index, offset, size);
			}
		}

		public List<E> subList(int fromIndex, int toIndex) {
			synchronized (lock) {
				checkForComodification();
				if (fromIndex < 0 || toIndex > size || fromIndex > toIndex)
					throw new IndexOutOfBoundsException();
				return new COWSubList(expectedArray, fromIndex + offset, toIndex - fromIndex);
			}
		}

		public void forEach(Consumer<? super E> action) {
			Objects.requireNonNull(action);
			int i, end; final Object[] es;
			synchronized (lock) {
				es = getArrayChecked();
				i = offset;
				end = i + size;
			}
			for (; i < end; i++)
				action.accept(elementAt(es, i));
		}

		public void replaceAll(UnaryOperator<E> operator) {
			synchronized (lock) {
				checkForComodification();
				replaceAllRange(operator, offset, offset + size);
				expectedArray = getArray();
			}
		}

		public void sort(Comparator<? super E> c) {
			synchronized (lock) {
				checkForComodification();
				sortRange(c, offset, offset + size);
				expectedArray = getArray();
			}
		}

		public boolean removeAll(Collection<?> c) {
			Objects.requireNonNull(c);
			return bulkRemove(e -> c.contains(e));
		}

		public boolean retainAll(Collection<?> c) {
			Objects.requireNonNull(c);
			return bulkRemove(e -> !c.contains(e));
		}

		public boolean removeIf(Predicate<? super E> filter) {
			Objects.requireNonNull(filter);
			return bulkRemove(filter);
		}

		private boolean bulkRemove(Predicate<? super E> filter) {
			synchronized (lock) {
				final Object[] oldArray = getArrayChecked();
				boolean modified = CopyOnWriteArrayList.this.bulkRemove(
						filter, offset, offset + size);
				size += (expectedArray = getArray()).length - oldArray.length;
				return modified;
			}
		}

		public Spliterator<E> spliterator() {
			synchronized (lock) {
				return Spliterators.spliterator(
						getArrayChecked(), offset, offset + size,
						Spliterator.IMMUTABLE | Spliterator.ORDERED);
			}
		}

		public List<E> reversed() {
			return new Reversed<>(this, lock);
		}
	}

	private static class COWSubListIterator<E> implements ListIterator<E> {
		private final ListIterator<E> it;
		private final int offset;
		private final int size;

		COWSubListIterator(List<E> l, int index, int offset, int size) {
			this.offset = offset;
			this.size = size;
			it = l.listIterator(index + offset);
		}

		public boolean hasNext() {
			return nextIndex() < size;
		}

		public E next() {
			if (hasNext())
				return it.next();
			else
				throw new NoSuchElementException();
		}

		public boolean hasPrevious() {
			return previousIndex() >= 0;
		}

		public E previous() {
			if (hasPrevious())
				return it.previous();
			else
				throw new NoSuchElementException();
		}

		public int nextIndex() {
			return it.nextIndex() - offset;
		}

		public int previousIndex() {
			return it.previousIndex() - offset;
		}

		public void remove() {
			throw new UnsupportedOperationException();
		}

		public void set(E e) {
			throw new UnsupportedOperationException();
		}

		public void add(E e) {
			throw new UnsupportedOperationException();
		}

		@Override
		@SuppressWarnings("unchecked")
		public void forEachRemaining(Consumer<? super E> action) {
			Objects.requireNonNull(action);
			while (hasNext()) {
				action.accept(it.next());
			}
		}
	}

	/**
	 * {@inheritDoc}
	 * <p>
	 * Modifications to the reversed view are permitted and will be propagated
	 * to this list. In addition, modifications to this list will be visible
	 * in the reversed view. Sublists and iterators of the reversed view have
	 * the same restrictions as those of this list.
	 *
	 * @since 21
	 */
	public List<E> reversed() {
		return new Reversed<>(this, lock);
	}

	/**
	 * Reversed view for CopyOnWriteArrayList and its sublists.
	 */
	private static class Reversed<E> implements List<E>, RandomAccess {
		final List<E> base;
		final Object lock;

		Reversed(List<E> base, Object lock) {
			this.base = base;
			this.lock = lock;
		}

		class DescendingIterator implements Iterator<E> {
			final ListIterator<E> it;
			DescendingIterator() {
				synchronized (lock) {
					it = base.listIterator(base.size());
				}
			}
			public boolean hasNext() { return it.hasPrevious(); }
			public E next() { return it.previous(); }
			public void remove() { it.remove(); }
		}

		class DescendingListIterator implements ListIterator<E> {
			final ListIterator<E> it;
			final int size; // iterator holds a snapshot of the array so this is constant

			DescendingListIterator(int pos) {
				synchronized (lock) {
					size = base.size();
					if (pos < 0 || pos > size)
						throw new IndexOutOfBoundsException();
					it = base.listIterator(size - pos);
				}
			}

			public boolean hasNext() {
				return it.hasPrevious();
			}

			public E next() {
				return it.previous();
			}

			public boolean hasPrevious() {
				return it.hasNext();
			}

			public E previous() {
				return it.next();
			}

			public int nextIndex() {
				return size - it.nextIndex();
			}

			public int previousIndex() {
				return nextIndex() - 1;
			}

			public void remove() {
				throw new UnsupportedOperationException();
			}

			public void set(E e) {
				throw new UnsupportedOperationException();
			}

			public void add(E e) {
				throw new UnsupportedOperationException();
			}
		}

		// ========== Iterable ==========

		public void forEach(Consumer<? super E> action) {
			for (E e : this)
				action.accept(e);
		}

		public Iterator<E> iterator() {
			return new DescendingIterator();
		}

		public Spliterator<E> spliterator() {
			return Spliterators.spliterator(this, Spliterator.ORDERED);
		}

		// ========== Collection ==========

		public boolean add(E e) {
			base.add(0, e);
			return true;
		}

		public boolean addAll(Collection<? extends E> c) {
			@SuppressWarnings("unchecked")
			E[] es = (E[]) c.toArray();
			if (es.length > 0) {
				ArraysSupport.reverse(es);
				base.addAll(0, Arrays.asList(es));
				return true;
			} else {
				return false;
			}
		}

		public void clear() {
			base.clear();
		}

		public boolean contains(Object o) {
			return base.contains(o);
		}

		public boolean containsAll(Collection<?> c) {
			return base.containsAll(c);
		}

		// copied from AbstractList
		public boolean equals(Object o) {
			if (o == this)
				return true;
			if (!(o instanceof List))
				return false;

			ListIterator<E> e1 = listIterator();
			ListIterator<?> e2 = ((List<?>) o).listIterator();
			while (e1.hasNext() && e2.hasNext()) {
				E o1 = e1.next();
				Object o2 = e2.next();
				if (!(o1==null ? o2==null : o1.equals(o2)))
					return false;
			}
			return !(e1.hasNext() || e2.hasNext());
		}

		// copied from AbstractList
		public int hashCode() {
			int hashCode = 1;
			for (E e : this)
				hashCode = 31*hashCode + (e==null ? 0 : e.hashCode());
			return hashCode;
		}

		public boolean isEmpty() {
			return base.isEmpty();
		}

		public Stream<E> parallelStream() {
			return StreamSupport.stream(spliterator(), true);
		}

		public boolean remove(Object o) {
			synchronized (lock) {
				int index = indexOf(o);
				if (index == -1)
					return false;
				remove(index);
				return true;
			}
		}

		public boolean removeAll(Collection<?> c) {
			return base.removeAll(c);
		}

		public boolean retainAll(Collection<?> c) {
			return base.retainAll(c);
		}

		public int size() {
			return base.size();
		}

		public Stream<E> stream() {
			return StreamSupport.stream(spliterator(), false);
		}

		public Object[] toArray() {
			return ArraysSupport.reverse(base.toArray());
		}

		@SuppressWarnings("unchecked")
		public <T> T[] toArray(T[] a) {
			// TODO optimize this
			return toArray(i -> (T[]) java.lang.reflect.Array.newInstance(a.getClass().getComponentType(), i));
		}

		public <T> T[] toArray(IntFunction<T[]> generator) {
			return ArraysSupport.reverse(base.toArray(generator));
		}

		// copied from AbstractCollection
		public String toString() {
			Iterator<E> it = iterator();
			if (! it.hasNext())
				return "[]";

			StringBuilder sb = new StringBuilder();
			sb.append('[');
			for (;;) {
				E e = it.next();
				sb.append(e == this ? "(this Collection)" : e);
				if (! it.hasNext())
					return sb.append(']').toString();
				sb.append(',').append(' ');
			}
		}

		// ========== List ==========

		public void add(int index, E element) {
			synchronized (lock) {
				base.add(base.size() - index, element);
			}
		}

		public void addFirst(E e) {
			base.add(e);
		}

		public void addLast(E e) {
			base.add(0, e);
		}

		public boolean addAll(int index, Collection<? extends E> c) {
			@SuppressWarnings("unchecked")
			E[] es = (E[]) c.toArray();
			if (es.length > 0) {
				ArraysSupport.reverse(es);
				synchronized (lock) {
					base.addAll(base.size() - index, Arrays.asList(es));
				}
				return true;
			} else {
				return false;
			}
		}

		public E get(int i) {
			synchronized (lock) {
				return base.get(base.size() - i - 1);
			}
		}

		public E getFirst() {
			synchronized (lock) {
				int size = base.size();
				if (size == 0)
					throw new NoSuchElementException();
				else
					return base.get(size - 1);
			}
		}

		public E getLast() {
			synchronized (lock) {
				if (base.size() == 0)
					throw new NoSuchElementException();
				else
					return base.get(0);
			}
		}

		public int indexOf(Object o) {
			synchronized (lock) {
				int i = base.lastIndexOf(o);
				return i == -1 ? -1 : base.size() - i - 1;
			}
		}

		public int lastIndexOf(Object o) {
			synchronized (lock) {
				int i = base.indexOf(o);
				return i == -1 ? -1 : base.size() - i - 1;
			}
		}

		public ListIterator<E> listIterator() {
			return new DescendingListIterator(0);
		}

		public ListIterator<E> listIterator(int index) {
			return new DescendingListIterator(index);
		}

		public E remove(int index) {
			synchronized (lock) {
				return base.remove(base.size() - index - 1);
			}
		}

		public E removeFirst() {
			synchronized (lock) {
				int size = base.size();
				if (size == 0)
					throw new NoSuchElementException();
				else
					return base.remove(size - 1);
			}
		}

		public E removeLast() {
			synchronized (lock) {
				if (base.size() == 0)
					throw new NoSuchElementException();
				else
					return base.remove(0);
			}
		}

		public boolean removeIf(Predicate<? super E> filter) {
			return base.removeIf(filter);
		}

		public void replaceAll(UnaryOperator<E> operator) {
			base.replaceAll(operator);
		}

		public void sort(Comparator<? super E> c) {
			base.sort(Collections.reverseOrder(c));
		}

		public E set(int index, E element) {
			synchronized (lock) {
				return base.set(base.size() - index - 1, element);
			}
		}

		public List<E> subList(int fromIndex, int toIndex) {
			synchronized (lock) {
				int size = base.size();
				var sub = base.subList(size - toIndex, size - fromIndex);
				return new Reversed<>(sub, lock);
			}
		}

		public List<E> reversed() {
			return base;
		}
	}

	/** Initializes the lock; for use when deserializing or cloning. */
	private void resetLock() {
		@SuppressWarnings("removal")
		Field lockField = java.security.AccessController.doPrivileged(
				(java.security.PrivilegedAction<Field>) () -> {
					try {
						Field f = CopyOnWriteArrayList.class
								.getDeclaredField("lock");
						f.setAccessible(true);
						return f;
					} catch (ReflectiveOperationException e) {
						throw new Error(e);
					}});
		try {
			lockField.set(this, new Object());
		} catch (IllegalAccessException e) {
			throw new Error(e);
		}
	}
}

~~~

