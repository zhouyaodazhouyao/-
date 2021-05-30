```mysql
CREATE TABLE `t` ( 
    `id` int(11) NOT NULL, 
    `city` varchar(16) NOT NULL, 
    `name` varchar(16) NOT NULL, 
    `age` int(11) NOT NULL, 
    `addr` varchar(128) DEFAULT NULL,
    PRIMARY KEY (`id`), KEY `city` (`city`)
) ENGINE=InnoDB;
```

```mysql
select city,name,age from t where city='杭州' order by name limit 1000;
```

## 排序时MySQL会给每个线程分配一块内存用于排序，称为sort_buffer.

#### 全字段排序

执行流程如下

1. 初始化sort_buffer，确定放入name、city、age三个字段；
2. 从索引city找到第一个满足city=‘杭州’条件的主键id；
3. 到主键id索引取出整行，取name、city、age三个字段的值，存入sort_buffer中；
4. 从city索引中取下一个记录的主键id；
5. 重复步骤3、4直到city的值不满足查询条件；
6. 对sort_buffer中的数据按照字段name做**快速排序**；
7. 按照排序结果取前1000行返回给客户端

其中排序这个动作，根据sort_buffer_size和待排数据量的大小，会在不同的地方完成。如果数据量小于sort_buffer的大小，排序就在**内存**中完成，这时候使用的是**快速排序**；如果数据量太大，内存放不下，就不得不利用磁盘**临时文件**辅助排序，这时候使用的是**归并排序**。

**问题：如果取出的字段过多，当行长度太大，很容易超过sort_buffer的大小，从而变成临时文件的归并排序，效率变差**

####  rowid排序

MySQL中max_length_for_sort_data是专门用于控制排序行长度的一个参数，如果单行长度超过这个值，MySQL就会换算法。

执行流程如下

1. 初始化sort_buffer，确定放入name、id两个字段；
2. 从索引city找到第一个满足city=‘杭州’条件的主键id；
3. 到主键id索引取出整行，取name、id两个字段的值，存入sort_buffer中；
4. 从city索引中取下一个记录的主键id；
5. 重复步骤3、4直到city的值不满足查询条件；
6. 对sort_buffer中的数据按照字段name进行排序；
7. 遍历排序结果，取前1000行，并**按照id的值回到原表中取出city、name和age三个字段**返回给客户端

#### 全字段排序 vs rowid排序

MySQL的一个设计思想就是：**如果内存够，就多利用内存，尽量减少磁盘访问。**所以如果内存够用的话，尽量使用全字段排序，避免rowid排序最后的回表操作导致磁盘访问。

#### 排序优化

前面这条语句之所以用到了排序，根本原因在于city索引中**name是无序的**，也就是说，如果我们建立**联合索引（city，name）**，那么就能保证在访问联合索引的阶段，name也是有序的，从而省去了排序的步骤。

更进一步的优化方式就是，整个过程中，还有回主键索引取数的操作，如果我们利用**覆盖索引**，即建立**联合索引（city，name，age）**，就能保证排序返回所需字段都在这个索引中，从而省去了回表的步骤。

当然，索引的维护和建立也是耗费资源的，二者之间还需要根据业务进行取舍。

#### Order by rand()

随机排序需要使用**内存临时表**，当然，也需要排序。上面提到，对于**InnoDB表**来说，rowid排序和全字段排序相比，多了一次回表操作，即多了一次磁盘访问，所以全字段排序在条件允许的情况下一般是首选。不过order by rand()使用的是内存临时表，回表操作对性能影响极小，所以这种情况下，一边rowid排序是首选。

```mysql
select word from words order by rand() limit 3;
```

执行流程如下

1. 创建一个临时表。这个临时表使用的是memory引擎，表里有两个字段，第一个是double类型，记为R，第二个字段是varchar(64)类型，也就是word字段，记为W。**这个表没有索引**
2. 从表中，按主键顺序取出所有word值，对于每一个word，调用rand()函数生成一个大于0小于1的随机小数，把这个随机小数和word分别存入临时表的R和W字段中
3. 临时表内按照字段R排序
4. 初始化**sort_buffer**，sort_buffer中有两个字段，一个是double类型，另一个是整型。
5. 从内存临时表中逐行取出R值和位置信息***[注]***，分别存入sort_buffer中的两个字段里。这个过程需要对内存临时表做全表扫描
6. 在sort_buffer中根据R的值进行排序
7. 排序完成后，取出前三个结果的位置信息，到内存表中取出word值，返回给客户端。

[^位置信息]: 1、对于有主键的InnoDB表来说，是主键id；2、对于没有主键的InnoDB表来说，是系统生成的6字节rowid；3、内存表相当于一个数组，位置信息是数组下标

