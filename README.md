# 杂谈
姑且记录一下自己看过的想记下来的东西.
# bit map 
bitmap是一种数据压缩算法，用于大量数字排序、判重、快速查找、去重等领域，核心思想是用bit位代表数字是否存在，如12378可表示为11100011，这样存储相同的数据量的时候，所需要用到的内存就会少很多，具体实现为：new Byte[n/32 + 1],存储的数据为 0 ~ 31 为array[0],32~63为array[1]，以此类推。
# bloom filter
布隆过滤器，布隆过滤器是一种判重的算法，其核心思想是通过hash来确定输入元素是否存在在集合里，具体做法为，创建一个大的bit[](根据情况也可能会分片)，然后通过n个hash算法算出输入元素的n个hash值，再将bit数组里的相应的位数置为1，添加操作便完成了，判断元素是否存在时，就仍然通过那n个hash算法算出n个hash值，再判断相应的位数上的值是不是1，如果全都是1则元素可能存在在集合里（因为存在冲突），但如果只要有1个位不为1，那么该元素一定不存在在集合里。布隆过滤器只存特征而不存数据本身，所以内存占用会比hashset少非常非常多，这就是它最核心的优点。<br>
另外值得一说的是，布隆过滤器的精准度由bit数组的大小、hash算法的效率、hash算法的个数决定，需要根据场景的不同，选择合适的算法以及大小，甚至在必要的情况下，对bit数组进行分片处理。
# hash一致性
用于机器或库表的水平拓展的技术，让请求可以通过这个算法落到相应的机器（服务）上。比起普通的如：id跟8取余的分表方式，它能够很好的处理机器的增加和减少的情况。
# threadLocal
是一个普通的java类，但不同点在于，它更像是一个变量，（不同于类成员的全局变量、方法内的局部变量）它的作用域为线程，每个线程对它执行set后再get，只能看到自己set的值，于是，它便达到了线程安全的单例。
# 数据库mvcc机制
mvcc是一种多版本快照度的机制，用于提高并发事务下的**读效率**，innodb的做法是为每一行数据增加两个隐藏列，最后写事务id和rollbackptr，在一个事务开始时，事务会获取目前已提交的最大的事务id，作为自己的read view，即可视化id，之后在读取数据时，无论数据行有没有被加锁，都会去读，在读到后通过数据的事务id来判断此条数据是否是在本事务之后修改/创建的，如果发现rollbackptr不为空，那么证明次数据被修改过，那么就会通过这个指针去读undo日志，把旧值读出来，替换，返回给客户端，如果ptr为空，也就是说这条数据是新创建的，那么无视掉它就可。                  
这样既可在数据被加锁的时候仍然可以读取数据，从而提高系统吞吐量。但是它有两种不适合的场景：1.无事务场景2.每次事务会波及到大多数数据的场景
# undo log
undo log是如innodb等数据库保证数据强一致性的核心方法，简单来说undo log是一种日志，它存储了一个事务对表操作的原始记录，譬如我们begin一个事务，然后执行update table set a = '456' where id = 1; 其中a为表某列的名字，id为主键。         
那么数据库执行引擎就会先在日志里写上《事务id，begin》《事务id，操作了主键为1的记录，原先的值为123》（当然真实的数据库不是这么写的，这里只做一个示范），再然后我们直接关掉命令行或者执行rollback，引擎就会读取undo日志，将原先的值恢复到表里，从而保证了数据的一致性，推广一下，如果事务执行了很多语句，增删改查了很多记录，我们完全可以先写日志，写进日之后直接返回success，然后再让引擎继续执行表更新、索引更新操作，这样效率就会提高很多（因为单纯的写日志比计算更新-》改索引-》同步索引要快的多），而且因为写入了日志，所以即便事务回滚、事务执行失败，也不用担心数据产生不一致的问题。
# redo log
跟undo log一样，redo log存在的意义也是为了保证数据库的强一致性，但重做日志与撤销日志的区别在于，他是顺序记录事务的操作的，按顺序记录一个事务开始后，做了哪些事情、又做了哪些事情，那么这样做有什么好处呢？         
答案是可以在事务写入日志后直接向客户端返回成功，以此**提高效率**，原因也很简单，正常的事务流程是：获取io task，数据同步至内存，重写索引，重写表文件。   
那么现在就可以在获取io task后写入日志，然后直接向客户端返回success，因为写入日之后，无论后续任务有没有成功（数据库异常、操作系统异常），都无所谓，只需要下次启动数据库时，按顺序重新操作一遍即可。并且因为没有将数据立即影响到表文件，数据库还可以进行io合并，将多个事务的io合并为一个大io一次性提交至文件，这个操作叫做成组提交，对系统整体吞吐率有很大的提升。
# redis的hash结构
因为某些机缘巧合，大致看了redis hash这个数据结构的实现（4.0版本），没什么特殊的，只谈其中我认为最重要的一点，就是它的扩容机制。      
和jdk hashmap的rehash不同，redis采用了双table的设计，在扩容的时候，并非一次将所有的元素完成转移，而是新建一个table，一次移动一部分元素，移动的元素的量由rehash传的一个参数决定，然后这个参数又是由调用者决定的，主动rehash会传1，遇到空的bucket就会退出循环，而由定时函数执行则会传100或更大，rehash的执行时间会更长，处理元素会更多。多次执行rehash函数，当所有元素转移到另一个table里后，移除旧的table，扩容结束。      
哦对了，值得一提的是，这个函数的注释有说它在某些情况（空位较少）时会阻塞，然后通过代码分析，理由也很简单，这个函数的退出条件是在扩容遍历时遇到了了n个空的bucket，那么只要我的n足够大，然后我顺序遍历时bucket又都不是空的，而且链表还贼长，那么扩容的时候执行的迁移过程的次数就会非常的多，而redis又是单线程的，那自然会造成性能（阻塞）问题了。
# GC常见算法

# volatile关键字
volatile作为java中的关键词之一，用以声明变量的值可能随时会别的线程修改，使用volatile修饰的变量会强制将修改的值立即写入主存，主存中值的更新会使缓存中的值失效(非volatile变量不具备这样的特性，非volatile变量的值会被缓存，线程A更新了这个值，线程B读取这个变量的值时可能读到的并不是是线程A更新后的值)。volatile会禁止指令重排。

# 成组提交
在MYSQL 里面  INNDOB的REDO LOG和MYSQL 的BINLOG 是两个独立体，不像ORACLE是时间上的关系。因为MYSQL 里面可以包含多个存储引擎，每个引擎有自己的独立日志。BINLOG是处于MYSQL的服务层，而REDO LOG 是INNDODB存储引擎层。当一个事务涉及了多个存储引擎的时候，也就是跨了引擎。那么只有BINLOG记录的才是唯一正确的，而INNODB记录的只是事务修改了INNODB引擎的，而该事务修改别的引擎就无法记录了。所以在MYSQL里面一切以BINLOG为主。

# 级联回滚

# CRC冗余校验
不要问我为什么混入了这种奇怪的东西，问学校。
# AQS

# mysql最左匹配原则
在mysql中使用组合索引时，（如c1,c2,c3),那么innodb会为你创建（c1）,(c1,c2),(c1,c2,c3)三种索引，而不是一般人想到的类似于两两结合的那种方式，这种创建方式就叫最左匹配，故你的查询语句中的where后，必须要有c1，或c1,c2等才能命中索引，诸如c1,c3是无法命中索引的，另外值得一提的是，c1，c2的顺序并不重要，因为mysql的查询优化器会自动帮你调整顺序，转为c1,c2适应索引。

# change buffer

# ThreadLocal
至于为什么把这个东西又又又单独拉出来说是因为网上的很多文章是有问题和误导性的，故在此重新记录一下。所谓ThreadLocal其实更像是一个变量，让每个线程可以设置和获取值，然后这个操作是线程隔离的。它的核心是*Thread类里的ThreadLocalMap*对象，他会在第一次尝试插入或获取操作时初始化，*故它并不是在ThreadLocal里的东西，而在Thread里*，它是一个哈希表，key是threadLocal实例，value是你要设置的值，但由于key是一个弱引用对象，会在发生GC时直接死掉，而Map里的value又是个强引用对象，所以这就有可能造成内存泄漏（在线程存活时间过长的情况下）。

# linkedBlockingQueue
当添加元素超过队列最大长度时,add会抛出异常,put会阻塞，offer会直接返回false.当队列为空时,poll返回null,take会被阻塞,remove会抛出异常.
# 限流算法
令牌法，所有的请求都需要获取一个令牌才可继续执行请求，而另外的数量由后台定期生成，如果令牌池为空，则服务降级。漏斗法，控制一定时间内的请求数量，适量丢弃一些请求，就像往漏斗里加沙子一样。

# jvm的一些优化
1.逃逸分析2.锁消除3.锁粗化4.栈分配、标量替换5.TLAB      
这里主要谈一下4和5,所谓栈分配即并非所有对象都分配在堆上，如果我判断一个对象无法逃逸出它的方法，那么我就可以将其分配在栈上，因为它的生命周期即为方法的调用开始到结束，这样便可以大大的减轻gc压力。TLAB则是线程的一小块私有内存，在堆中的EDEN区，大小通常为EDEN的百分之1，对象会被优先分配到TLAB里，这样可以减少并发分配时，内存分配竞争所带来的开销，但坏处是可能会造成内存碎片。
# cms
CMS GC要决定是否在full GC时做压缩，会依赖以下几个条件：     
1.UseCMSCompactAtFullCollection 与 CMSFullGCsBeforeCompaction 是搭配使用的.默认是true，什么时候清理浮动垃圾（压缩整理）取决于后者。       
2.用户调用了System.gc()，而且DisableExplicitGC没有开启。        
3.young gen报告接下来如果做增量收集会失败；简单来说也就是young gen预计old gen没有足够空间来容纳下次young GC晋升的对象。      
上述三种条件的任意一种成立都会让CMS决定这次做full GC时要做压缩。       

CMSFullGCsBeforeCompaction 说的是，在上一次CMS并发GC执行过后，到底还要再执行多少次full GC才会做压缩（默认0）。也就是在默认配置下每次CMS GC顶不住了而要转入full GC的时候都会做压缩。       
把CMSFullGCsBeforeCompaction配置为n，就会让上面说的第一个条件变成每隔n次真正的full GC才做一次压缩，这会减少full GC压缩的次数，节省了gc时间，也就更容易使CMS的old gen受碎片化问题的困扰。 本来这个参数就是用来配置降低full GC压缩的频率，以期减少某些full GC的暂停时间。CMS回退到full GC时用的算法是mark-sweep-compact，但compaction是可选的，不做的话碎片化会严重些但这次full GC的暂停时间会短些；这是个取舍。

# redlock算法
分布式锁一般情况下有三种实现方案，数据库主键、redis、zk。这里主要说一下redis，所谓redlock算法简单点说就是客户端同时向redis集群里的每一个节点设置同一个key，和自己生成的v，并设置好过期时间。当超过半数的节点都设置成功后，就代表自己获取了锁，锁的有效时间即为过期时间-获取锁的时间。但它仍然是有问题的，比如客户端挂了，操作不当会引起双写不一致的问题。
