如果我们的hashMap源码单纯的用数组实现，那么它们的增加和查找的时间复杂度为o(n)，因为在增加的时候要遍历一次查询key是否存在，在查找的时候要遍历一次查找key。**但是HashMap使用了散列表来存储数据，可以使得增加和查找的时间复杂度为o(1)。**
**散列表**
![散列表示意图](https://upload-images.jianshu.io/upload_images/7730759-e8425739ffd1c13a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
数组的各个下标相当于一个个桶，通过hashCode(key)就可以拿到每个桶的下标，桶里面的数据用链表存储，**因此如果此时桶里只有一个链节点就只要o(1),有两个节点就只要o(2)，一个桶装的链表节点太多还可以转换成红黑树**。这就是散列表的好处。

###源码分析
如下是我自己写的简易版的hashMap:
  ```
public class HashMapSample<K,V> {

    /**
     * 散列表(桶)
     */
    public MapEntry[] tables;

    /**
     *存放的个数
     */
    transient int size;

    /**
     * 扩容阈值
     */
    int threshold = 12;

    /**
     * 加载因子
     */
    final float loadFactor = 0.75f;

    public class  MapEntry<K,V>{
        K key;
        V value;
        MapEntry<K,V> next;  //链表
        int hash;
        public MapEntry(int hash,K key,V value,MapEntry<K,V> next){
            this.hash=hash;
            this.key=key;
            this.value=value;
            this.next=next;
        }
    }


    private int getIndext(int hash,int length) {
        //大小为2的幂次方,因为&操作符是:两个1为1，其他为0。2的幂次方-1的二进制全是1，可避免过度哈希冲突
        return hash&length-1;
    }

    //哈希code得到哈希值
    private int hash(K key) {
        int h = 0;
        return key==null?0:(h=key.hashCode()^(h>>>16));
        //两对象hash相等，两个对象可能相等；两个对象hash不等，则俩对象一定不等
        //原理: value>16,是对象的32位地址向右移了16位，剩下的16位可能相等
    }


    /**
     * Get区域
     * @param key
     * @return
     */
    public V get(K key){

        if(key == null){
            return null;
        }

        //判断有没有该key
        MapEntry<K,V> mapEntry =getEntry(key);
        return mapEntry==null?null:mapEntry.value;
    }

    private MapEntry<K,V> getEntry(K key) {
        //判断同一个桶是否有这个key
        int hash=hash(key);
        int indext = getIndext(hash,tables.length);

        for(MapEntry<K,V> mapEntry = tables[indext];mapEntry!=null;mapEntry=mapEntry.next){
            Object k;
            if(mapEntry.hash == hash && ((k=mapEntry.key)==key || key.equals(k))){
                return mapEntry;
            }
        }
        return null;
    }
    
    
    /**
     * Put区域
     * @param key
     * @param value
     * @return
     */
    public V put(K key,V value){
        if(tables == null){
            tables=new MapEntry[16];
        }

        if(key==null){
            return null;
        }

        //1.找到table的位置
        int hash = hash(key);
        int index = getIndext(hash,tables.length);

        //2.判断有没有重复key
        for(MapEntry<K,V> table = tables[index];table!=null;table=table.next){
            Object k;
            if(table.hash == hash && ((k=table.key)==key || key.equals(k))){
                V oldValue = table.value;
                table.value=value;
                return oldValue;
            }
        }

        //3.添加一个新的mapEntry
        addEntry(hash,key,value,index);

        return  null;
    }

    /**
     * 添加一个新的entry
     * @param hash
     * @param key
     * @param value
     * @param index
     */
    private void addEntry(int hash,K key,V value,int index){
        //判断要不要扩容
        if(size>=threshold && tables[index] != null){
            resize(size<<1);
            //跟新添加的index
            index = getIndext(hash,tables.length);
        }

        //真正的执行添加
        createEntry(hash,key,value,index);

    }

    private void createEntry(int hash, K key, V value, int index) {
        MapEntry<K,V> newMapEntry = new MapEntry<>(hash,key,value,tables[index]);
        newMapEntry.next=tables[index];
        tables[index]=newMapEntry;
    }

    private void resize(int newCapacity) {
        MapEntry<K,V>[] newTable = new MapEntry[newCapacity];
        //重新计算index
        transform(newTable);
        tables = newTable;
        threshold =(int)(newCapacity*loadFactor);
    }

    private void transform(MapEntry<K,V>[] newTable) {
        //重新计算index
        int newCapacity = newTable.length;
        for (MapEntry<K,V> e:tables){
           //这里的节点会变成逆序排序，如1-2-3变成3-2-1
            //在多线程执行时会造成死循环，如两个对象的next相互调用
            while (null != e){

                MapEntry<K,V> next = e.next;

                int index = getIndext(e.hash,newCapacity);

                //先把newTable[index]的值存到e.next
                e.next= newTable[index];
                //把e存到第一个newTable[index]
                newTable[index] = e;
                //给e继续遍历旧table的链节点
                e=next;
            }
        }
    }
}
  ```

**加载因子：为什么是0.75？不是1**
因为：如果加载因子太小，就会导致太快的扩容(扩容要进行遍历一次重新计算index)。如果加载因子太大，比如100，就是导致散列表的每个桶存放太多的节点。为什么不是1，是因为用一个空间换时间的概念。

**为什么hashMap的大小是2的幂次方**
因为使用**hash&length-1**这个公式把得到的hash值&(求%)时2的幂次方-1的二进制全都是1，对哈希值的影响小。可避免过度的哈希碰撞。

**hashcode的计算原理**
利用对象的地址(32位)，然后右移16位，即取后面的16位的二进制作为hash值。所以hash值相等，两对象可能不等；hash值不相等，两对象一定不相等。

**初始化大小**
hashMap的初始化的大小为16，因为扩容对性能比较消耗，所以我们在知道数据大小的情况下要尽量不扩容，我们可以手动初始化大小为:count*0.75+1

**多线程下在hashMap中put导致的问题**
数据丢失，死循环：每个子线程都有自己的工作内存，通过**拷贝需要的部分在主内存的资源到子线程的工作内存，子线程修改好后就将资源更新到主内存。**hashMap就在主内存，所以会造成子线程的获取资源的不同步。
1.死循环，在**扩容时旧表的元素重新计算index时，多线程情况下，链节点可能会形成环**，从而造成死循环。
2.多线程同时对同一链表增加数据，更新后可能会导致数据丢失，前面存储的数据被覆盖。不同链表的增加数据，更新后不会影响。
**解决方案：**1.hashTable、2.Collections.synchronizedMap()  3.ConcurrentHashMap  :前两个方法都是锁住方法，conCurrentHashMap利用：分段锁 Lock , 分段锁 （synchronized，CAS）。
当put方法synchronized时，其他线程也不可以get()操作。
![image.png](https://upload-images.jianshu.io/upload_images/7730759-91060953e25901fc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**hashTable、ConcurrentHashMap的分析：**
1.hashTable是将整个方法加上synchronized，整个方法锁住比较消耗性能，因为整个hash表都被锁住了，其他线程只能阻塞等待重新去抢资源。一个线程在 put 的时候，另外一个线程不能再 put 和 get 必须进入等待状态。
2.concurrentHashMap则是把需要改变的链表锁住，而不是把整个hash表锁住，其他线程的就可以访问hash表里未被锁住的链表。这样多线程就不会同时改动同一个链表造成数据丢失和死循环，也提高了性能。
