# undo log与redo log原理分析

数据库通常借助日志来实现事务，常见的有undo log、redo log，undo/redo log都能保证事务特性，这里主要是原子性和持久性，即事务相关的操作，要么全做，要么不做，并且修改的数据能得到持久化。

>  假设数据库在操作时，按如下约定记录日志：

1. 事务开始时，记录START T
2. 事务修改时，记录（T，x，v），说明事务T操作对象x，x的值为v
3. 事务结束时，记录COMMIT T

## undo log原理

undo log是把所有没有COMMIT的事务回滚到事务开始前的状态，系统崩溃时，可能有些事务还没有COMMIT，在系统恢复时，这些没有COMMIT的事务就需要借助undo log来进行回滚。

>  使用undo log时，要求：

1. 记录修改日志时，(T，x，v）中v为x修改前的值，这样才能借助这条日志来回滚；

2. 事务提交后，必须在事务的所有修改（包括记录的修改日志）都持久化后才能写COMMIT T日志；这样才能保证，宕机恢复时，已经COMMIT的事务的所有修改都已经持久化，不需要回滚。 

>  使用undo log时事务执行顺序:

1. 记录START T
2. 记录需要修改的记录的旧值
3. 持久化undo log
4. 根据事务的需要更新数据库
5. 记录COMMIT T (持久化)

>  使用undo log进行宕机回滚:

1. 扫描日志，找出所有已经START,还没有COMMIT的事务。

2. 针对所有未COMMIT的日志，根据undo log来进行回滚。 

> 如果数据库访问很多，日志量也会很大，宕机恢复时，回滚的工作量也就很大，为了加快回滚，可以通过checkpoint机制来加速回滚:

1. 在日志中记录checkpoint_start (T1,T2…Tn) (Tx代表做checkpoint时，正在进行还未COMMIT的事务）

2. 等待所有正在进行的事务（T1~Tn）COMMIT

3. 在日志中记录checkpoint_end

>  借助checkpoint来进行回滚:

1. 从后往前，扫描undo log

2. 如果先遇到checkpoint_start, 则将checkpoint_start之后的所有未提交的事务进行回滚；(回滚T1,...Tn和在checkpoint_start之后开始而未提交的事务)

3. 如果先遇到checkpoint_end, 则将前一个checkpoint_start之后所有未提交的事务进行回滚；（在checkpoint的过程中，可能有很多新的事务START或者COMMIT). (回滚checkpoint_start之后开始而未提交的事务)

使用undo log，在写COMMIT日志之前，要求undo log以及事务的所有修改都必须已经持久化(落盘)，这种做法通常很影响性能。

## redo log原理

redo log是指在回放日志的时候把已经COMMIT的事务重做一遍，对于没有commit的事务按照abort处理，不进行任何操作。

> 使用redo log时, 要求:

1. 记录redo log时，(T,x，v）中的v必须是x修改后的值，否则不能通过redo log来恢复已经COMMIT的事务。

2. 写COMMIT T日志之前，事务的修改不能进行持久化，否则恢复时，对于未COMMIT的操作，可能有数据已经修改，但重放redo log不会对该事务做任何处理，从而不能保证事务的原子性。 

> 使用redo log时事务执行顺序:

1. 记录START T
2. 记录事务需要修改记录的新值
3. 持久化redo log
4. 记录COMMIT T（持久化）
5. 将事务相关的修改写入数据库

> 使用redo log重做事务:

1. 扫描日志，找到所有已经COMMIT的事务；
2. 对于已经COMMIT的事务，根据redo log重做事务；

> 在日志中使用checkpoint:

1. 在日志中记录checkpoint_start (T1,T2...Tn) (Tx代表做checkpoint时，正在进行还未COMMIT的日志）
2. 将所有已提交的事务的更改进行持久化；
3. 在日志中记录checkpoint_end

> 根据checkpoint来加速恢复:

1. 从后往前，扫描redo log

2. 如果先遇到checkpoint_start, 则把T1~Tn以及checkpoint_start之后的所有已经COMMIT的事务进行重做；

3. 如果先遇到checkpoint_end, 则T1~Tn以及前一个checkpoint_start之后所有已经COMMIT的事务进行重做； 

与undo log类似，redo log在使用时对持久化以及事务操作顺序的要求都比较高，可以将两者结合起来使用. 

## undo/redo log

在恢复时，对于已经COMMIT的事务使用redo log进行重做，对于没有COMMIT的事务，使用undo log进行回滚。

redo/undo log结合起来使用时，要求同时记录操作修改前和修改后的值，如（T，x，v，w），v为x修改前的值，w为x修改后的值，具体操作顺序为：

1. 记录START T
2. 记录修改日志（T，x，v，w）
3. 持久化日志, 其中v用于undo, w用于redo. 
4. 更新数据库
5. 记录 COMMIT T

4和5的操作顺序没有严格要求，并且都不要求持久化；因为如果宕机时4已经持久化，则恢复时可通过redo log来重做；如果宕机时4未持久化，则恢复时可通过undo log来回滚；在处理checkpoint时，可采用与redo/undo log相对应的处理方式。
