##概念
###强引用（Strong References）
最常见的Java对象引用，例如：<br>
```java  
StringBuffer buffer = new StringBuffer();
```
特点：当存在强引用时，对象无法被GC。
###弱引用（Weak References）
使用方式：
```java  
Book book=new Book();
WeakReference<Book> weakBook = new WeakReference<Book>(book);
Book b=weakBook.get();
```

特点：当只剩弱引用指向该对象时（上例中book=null之后），该引用会被GC回收（时间不确定，一说在下一轮GC时就会被回收）。
###软引用（Soft References）
使用方式：
```java
SoftReference<Book> softBook = new SoftReference<Book>(book);
Book b=weakBook.get();
```
特点：和弱引用类似，但比它略强。内存充足时不会被回收，只有当内存不足时才会被GC。

###幻象引用（Phantom references）
特点：与之前三种引用不同，它无论何时都是返回null。与Reference queue搭配使用，查看Reference queue就能知道该对象什么时候被GC，代替传统的finalize方法（可能导致对象复活）做对象的收尾操作。

##实际使用
####WeakHashmap
设想以下场景：我们需要关联Book的实例与它的author，但是由于种种原因无法扩展Book类，所以只能借助Map<Book,String>来保存Book和author之间的关系并做查询。当又由于种种原因某个book变成null了，此时Map中它对应的author也就无法再被使用，同时也无法被释放，这就造成了内存泄漏，WeakHashMap就是应对这种情况的。
####CacheBuilder
这是Google用SoftReference作为底层实现的缓存器，当我们需要保存一批数据在内存中快速读取时，会是适合的方案。

##参考文档
####Understanding Weak References Blog
https://community.oracle.com/blogs/enicholas/2006/05/04/understanding-weak-references
####Is there a SoftHashMap in Java?
http://stackoverflow.com/questions/264582/is-there-a-softhashmap-in-java
####When would you use a WeakHashMap or a WeakReference?
http://stackoverflow.com/questions/154724/when-would-you-use-a-weakhashmap-or-a-weakreference?noredirect=1&lq=1
####What is a WeakHashMap and when to use it?
http://stackoverflow.com/questions/5511279/what-is-a-weakhashmap-and-when-to-use-it
####Example of PhantomReference in Java
http://www.concretepage.com/java/example_phantomreference_java
####深入理解WeakHashmap
http://mikewang.blog.51cto.com/3826268/880775/
