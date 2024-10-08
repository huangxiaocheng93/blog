Redis的持久化为什么要放在和主从同步一起写呢？因为这两者就是一件事。

# 持久化

Redis的持久化有两种方式，一种是快照，也就是**RDB**（Redis DataBase），一种是**AOF**（Append Only File）日志。

## **快照（RDB）**

快照是一次性的全量备份，将某一时刻的全量数据以二进制序列化的形式存储，在空间上非常紧凑，能大大缩小存储所用的空间。

Redis通过fork操作产生一个子线程来完成快照持久化的操作（所以Redis的单线程只是网络请求是单线程处理）。这里利用了linux的COW（copy on write）机制，子进程和父进程共享内存数据，父进程如果修改数据会把对应的页copy出去。子进程并不会修改数据，所以保存的时候相当于保存了进程fork出来一瞬间的数据，所以这也是这种模式叫快照的原因。

快照的优势是备份出来的文件比较小，和内存中的数据一样大；

快照的劣势是需要额外的线程处理，并且耗时较长；

## AOF日志

AOF日志是连续性的增量备份，记录的是修改内存数据的指令记录文本。这样就可以通过对一个空的Redis实例顺序执行记录的命令，也就是重放，来复原实例。Redis在收到修改指令后，会先进行校验，如果没问题，会首先把指令追加记录磁盘上的AOF日志中，然后再执行指令，这样即使突发宕机，重放时也能重放到这个指令。

AOF的优势是不需要额外的操作，主线程执行的过程中就完成了；

AOF的劣势是AOF文件会随时间膨胀，后续想要用这个AOF文件重放也越来越难；

### AOF重写

上面说到AOF文件会随时间膨胀越来越大，所以需要瘦身，所以每隔一段时间就需要重写一次AOF文件。重写的原理是fork一个子进程，将当前内存的数据转换为指令写入新的AOF文件（是不是很像RDB），然后追加这段时间产生的新的AOF指令，最后替换旧的AOF文件。

### fsync

上面讲到AOF文件是通过主进程追加的，如果每次都要把文件写到磁盘那么Redis的速度优势就不复存在了。但是如果是写道缓存，就没有办法保证AOF的安全性。

Redis4.0的做法是通过专门的异步线程去出去fsync的事项。

## Redis4.0的混合持久化

Redis重启时，很少使用RDB来恢复数据，因为会丢失最后一次快照之后的数据。但是使用AOF日志重放，效率上又会慢很多。因此Redis4.0提供了混合持久化的策略，就是RDB和AOF同时使用。RDB正常持久化，而AOF不在记录全量指令，而是记录每次RDB快照之后的增量AOF，这样Redis重启时就可以先加载RDB的内容，然后再重放AOF日志，效率大大提升。

# 主从同步

主从同步能够在master挂掉之后，让从节点接管，从而快速恢复。避免因重启所需要的时间较长而影响业务。

## **最终一致**

Redis是AP系统，会保证最终一致性，不保证强一致性。

## **快照同步**

主节点进行一次bgsave（就是做一次RDB）,然后将RDB文件传输给从节点，从节点清空内存，按照主节点传过来的RDB文件完成一次全量加载。此过程中，增量同步继续进行，从节点按照RDB重加载之后，就走增量同步路子进行追赶。但是如果快照同步时间过长或者buffer过小，就会导致从节点加载完RDB后发现又有覆盖情况了，继续执行快照同步，成了死循环。  因此需要合理设置buffer的大小避免这种情况发生。

当一个新节点加入集群之后，必须先完成一次快照同步，完成之后才能继续走增量同步。

## **增量同步**

主节点将修改性指令记录在本地的buffer中，然后异步同步给从节点，从节点读取buffer中的指令进行同步，并向主节点反馈同步到的位置（偏移量）。

Redis的同步buffer是一个定长的环形数组，大小是有限的，如果数组满了就会覆盖前面的内容从头开始。如果出现网络分区，那么恢复后的buffer可能已经有指令被覆盖掉了，丛节点无法获得这些指令就会出现数据差异，此时就需要再次使用**快照同步**。

## 无盘复制

无盘复制是Redis 2.8.18开始支持的。主要原因是快照同步需要将RDB文件写入磁盘，这是一个很重的IO操作，对系统负载的影响比较大，对主节点的服务效率产生影响。无盘复制就是不再生成RDB文件，而是直接将快照内容发送给从节点，也就是主节点一边遍历内容，一边将序列化的内容发送给从节点，从节点将接收到的内容写到磁盘上，接收完毕后再一次性加载。 总的来说就是将磁盘写入的压力由主节点转移到了从节点上面。

## wait指令

Redis是异步同步的，一致性也是保持的最终一致性。wait指令则是尝试完成一次强一致同步。有两个参数，第一个参数是从节点数量，第二个几点是最大等待时间，单位是ms：

`wait nodeNum waitTime`: 就是完成nodeNum个从节点的强一致同步，完成同步则结束阻塞，否则阻塞到最大等待时间waitTime，如果超过waitTime还没完成就不管了。如果waitTime设置为0，则表示无限等待，此时如果出现网络分区导致无法同步，主节点的就会一直阻塞，失去可用性。

## CP还是AP？

Redis是典型的AP系统，主从不同步的情况下仍然可以正常提供服务。