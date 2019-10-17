## LruCache原理分析
#### 1.LRU(Least Recently Used) ,最近最少使用算法。
#### 2.使用方法   
当成一个Map使用就可以了，底层实现了LRU缓存策略。   
使用时需注意几点：   
1).需要提供一个缓存容量作为构造参数;    
2).重写 sizeOf 方法，自定义设计一条数据的容量计算，如果不重写就无法预知数据的容量，不能保证缓存容量在限定的最大容量以内;   
3).重写 entryRemoved 方法，可知道最少使用的缓存被清除时的数据（ evicted, key, oldValue, newVaule ）;      
4).LruCache是线程安全的，在内部的get、put、remove以及trimToSize都是安全的   
 
```
	LruCache<String, Bitmap> bitmapCache = new LruCache<String, Bitmap>(CACHE_SIZE){
		@Override
        protected int sizeOf(String key, Bitmap value) {
            return value.getByteCount();
        }
        
        @Override
        protected void entryRemoved(boolean evicted, String key, Bitmap oldValue, Bitmap newValue) {
            super.entryRemoved(evicted, key, oldValue, newValue);
        }
	};
```    

#### 3.源码分析及具体实现   
##### 3.1 LruCache 原理   
LruCache就是利用LinkedHashMap 的一个特性 （accessOrder=true 基于访问顺序）和对LinkedHashMap的数据操作上锁实现缓存策略。    
  
+ 1.LruCache构造函数中LinedHashMap 构造参数 accessOrder=true,实现了数据排序按照访问顺序；   
+ 2.每次 LruCache.get(K key) 方法中都会调用 LinkedHashMap.get(Object key);   
+ 3.LinkedHashMap 设置 accessOrder=true 后，每次 LinkedHashMap.get(Object key) 都会进行 afterNodeAccess 方法，Move node to last;   
+ 4.LinkedHashMap 是双向链表，通过afterNodeAccess,每次LruCache.get->LinkedHashMap.get 的数据都会被移动到最末尾；   
+ 5.在put 和 trimToSize 的方法执行时，如果发生数据移除，优先移除链表最前面的数据，即最近最少使用的数据。    

##### 3.2 LruCache 构造方法   

```
    /**
     * @param maxSize for caches that do not override {@link #sizeOf}, this is
     *     the maximum number of entries in the cache. For all other caches,
     *     this is the maximum sum of the sizes of the entries in this cache.
     */
    public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        /**
     	 * @param  initialCapacity the initial capacity,初始大小
         * @param  loadFactor      the load factor，负载因子 0.75
         * @param  accessOrder     the ordering mode - <tt>true</tt> 基于访问顺序, <tt>false</tt> 基于插入顺序
         **/
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }
```   
LruCache的实现主要是由LinkedHashMap实现，在LruCache的构造方法中主要是设置了最大缓存大小以及初始化LinkedHashMap。LinkedHashMap的构造方法中，第一个参数 initialCapacity 设置LinkedHashMap的初始大小，loadFactor 这个是HashMap里的构造参数，主要用于扩容问题，比如HashMap的最大容量是100，扩容因子为0.75，则HashMap在75容量的时候就会扩容。第三个参数accessOrder是最主要的一个参数，accessOrder=true时，LinkedHashMap数据排序就会基于数据的访问顺序，实现LruCache核心工作原理。   

##### 3.3 LruCache.get(K key)   

```
 	/**
     * Returns the value for {@code key} if it exists in the cache or can be
     * created by {@code #create}. If a value was returned, it is moved to the
     * head of the queue. This returns null if a value is not cached and cannot
     * be created.
     */
    public final V get(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V mapValue;
        synchronized (this) {
        	//LinkedHashMap每次get都会基于访问顺序来将取出的数据移动链表末尾
            mapValue = map.get(key);
            //计算命中次数
            if (mapValue != null) {
                hitCount++;
                return mapValue;
            }
            //计算丢失次数
            missCount++;
        }

        /*
         * Attempt to create a value. This may take a long time, and the map
         * may be different when create() returns. If a conflicting value was
         * added to the map while create() was working, we leave that value in
         * the map and release the created value.
         */

        V createdValue = create(key);
        if (createdValue == null) {
            return null;
        }

        /***
        *
        * 不重写create方法走不到下面，默认的create(key)方法返回null
        */

        synchronized (this) {
            createCount++;//创建次数
            mapValue = map.put(key, createdValue);//将自定义create创建的值，放入LinkedHashMap中，如果key已经存在，会返回之前相同的key的值

            if (mapValue != null) {
                // There was a conflict so undo that last put
                //有冲突，撤销刚才的操作，将之前相同key的值重新放回去
                map.put(key, mapValue);
            } else {
            	//拿到键值对，计算出容量中的相对长度，然后加上
                size += safeSizeOf(key, createdValue);
            }
        }


        if (mapValue != null) {
            entryRemoved(false, key, createdValue, mapValue);//create的值被删除，原来之前相同key的值被重新添加回去，通知自定义的entryRemoved方法
            return mapValue;
        } else {
            trimToSize(maxSize);//之前进行了size+ 操作，重整长度
            return createdValue;
        }
    }
```    
主要是在map.get(key);方法里，这里是调用LinkedHashMap.get方法，再加上LruCache构造里默认设置的LinkedHashMap 的accessOrder=true，将会按照访问顺序排序，将取出的值移动到链表尾端。   

##### 3.4 分析LinkedHashMap.get(Object key)方法   

```
    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)//按照访问顺序排序，则执行afterNodeAccess函数
            afterNodeAccess(e);
        return e.value;//通过hashkey取得节点不为空，返回节点的值
    }
```   
从源码中可以看到，判断accessOrder=true,则每次get都会执行afterNodeAccess(）   

##### 3.5 LinkedHashMap.afterNodeAccess(Node)    

```
    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMapEntry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMapEntry<K,V> p =
                (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)//前一个节点为空，即e为链表表头
                head = a;//将e的下一个节点置为链表表头
            else
                b.after = a;//将e的前一个节点的下一个节点指向a,
            if (a != null)//下一个节点不为空
                a.before = b;//下一个节点的前一个点指向b
            else
                last = b;//即e在链表表尾端，将b置为链表表尾
            if (last == null)
                head = p;//只有一个节点时，当前节点为链表表头
            else {
                p.before = last;//将当前节点的前一个节点指向链表表尾，即当前节点移动到链表末端
                last.after = p;//last的末端的下一个节点指向当前节点
            }
            tail = p;
            ++modCount;
        }
    }
```    
LinkedHashMap是双向链表，LruCache.get->LinkedHashMap.get 的数据就被移动到了链表的最尾端。以上是LruCache的核心工作原理。   

---   
##### 3.6LruCache.put(K key,V value)    

```
    /**
     * Caches {@code value} for {@code key}. The value is moved to the head of
     * the queue.
     *
     * @return the previous value mapped by {@code key}.
     */
    public final V put(K key, V value) {
        if (key == null || value == null) {
            throw new NullPointerException("key == null || value == null");
        }

        V previous;
        synchronized (this) {
            putCount++;//存入次数+1
            size += safeSizeOf(key, value);//统计当前总容量
            previous = map.put(key, value);//如存在相同的key，则返回之前的值
            if (previous != null) //已存在相同的key
            {
                size -= safeSizeOf(key, previous);//减掉相同的容量
            }
        }

        if (previous != null) {
            entryRemoved(false, key, previous, value);
        }

        trimToSize(maxSize);
        return previous;
    }
```






