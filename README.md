# 杂谈
姑且记录一下自己看过的想记下来的东西.
# bit map 
bitmap是一种数据压缩算法，用于大量数字排序、判重、快速查找、去重等领域，核心思想是用bit位代表数字是否存在，如12378可表示为11100011，这样存储相同的数据量的时候，所需要用到的内存就会少很多，具体实现为：new Byte[n/32 + 1],存储的数据为0~31为array[0],32~63为array[1]，以此类推。
# bloom filter
布隆过滤器，布隆过滤器是一种判重的算法，其核心思想是通过hash来确定输入元素是否存在在集合里，具体做法为，创建一个大的bit[](根据情况也可能会分片)，然后通过n个hash算法算出输入元素的n个hash值，再将bit数组里的相应的位数置为1，添加操作便完成了，判断元素是否存在时，就仍然通过那n个hash算法算出n个hash值，再判断相应的位数上的值是不是1，如果全都是1则元素可能存在在集合里（因为存在冲突），但如果只要有1个位不为1，那么该元素一定不存在在集合里。布隆过滤器只存特征而不存数据本身，所以内存占用会比hashset少非常非常多，这就是它最核心的优点。<br>
另外值得一说的是，布隆过滤器的精准度由bit数组的大小、hash算法的效率、hash算法的个数决定，需要根据场景的不同，选择合适的算法以及大小，甚至在必要的情况下，对bit数组进行分片处理。
# hash一致性
