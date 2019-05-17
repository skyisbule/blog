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

# 级联回滚

# CRC冗余校验
不要问我为什么混入了这种奇怪的东西，问学校。
# AQS
