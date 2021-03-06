# chain-tree
链表树-复合数据结构应用实例

我们清楚：数据库设计中，表结构设计的好坏，直接影响程序的复杂度。所以，本文就无限级分类（目录）树与链表的复合在表设计中的应用进行探讨。当然，什么是树，什么是链表，这里不作介绍。有兴趣可以去看相关的教材。

需求简介：

经常遇到这样的需求，我们希望能将保存在数据库中的树结构能够按确定的顺序读出来。比如，多级菜单、组织结构、商品分类。更具体的，我们希望某个二级菜单在这一级别中就是第一个。虽然它是最后插入的。或者，我们希望这个商品小类在其父类下也是第一个，以使可以网上促销……

实现方式：

关于在数据库中保存树型结构并不复杂，一般，我们只要保存当前记录在谁下面，所以，给定一个parent_id(父节点ID)字段就可以了。 

以mysql为例，我们只要使用inner join，就能一次性查出无限级的树的数据。
 
一般而言，我们在设计中会增加以下一些字段，以增强数据的可操作性：

path: 以逗号分隔的ID字串，标明了从顶级到本节点的path,
 
比如：3,34,257 ：那说明，本节点的parent_id是257,257的parent_id是34，34的parent_id是3。 

使用path的好处：可以使用在sql语句中使用like直接查出某节点下的所有子孙节点。

level：层级，表明此节点是第几层节点。 

当然，我们完全可以从path中的逗号看出此节点是第几层，但有了层级这个数据，将会给程序操作树偍供很大的方便。

is_leaf: 是否叶节点， 

这也是为程序渲染界面，以及程序中控制所用的。

sort_no: 排序编号。 

这是我们经常需要的，并且也是最难维护的。它的目的是维护树中每一节点的正常顺序。有多种方案。一般有，直接使用整 数的方案，以及对树进行编码的方案。 

1) 直接使用整数，好处是排序速度快。问题在于，维护连续整数的sort_no较为复杂。 

比如，当我们插入一个节点，我们要将其节点后的所有节点的sort_no全部加1。 

当我们移动一个节点，我们则要判断是前移，还是后移。 

如果前移，则要将新位置到当前位置的节点的sort_no全部加1，如果是后移，则要将新位置到当前位置的节点的sort_no全部减1。

2) 对树进行编码的方案，好处是，每次只对局部操作。编码可以不连续。比如一个节点从父节点3移动父节点7下面，则只要操作父节点7下面的记录。如果在新的父节点下只是追加，则不需要变更所有sort_no。但是，基于编码的解决方案虽有这些优点，也有明显的缺点。那就是，我们有时无法预知树的层级数量，以及每一层的节点数。 

比如，如果我们预估每一层是100个节点，那最多是两位编码足够了，但一旦节点数超标，则编码方案就得调整。 

链表算法：

3) 针对这一问题，最完美的解决方案是引入链表算法。即sort_no仍使用整数。同时为每一个节点增加链表的before指针，或next指针，或者before、next指针二者均加上。 

理论上，单向链表即可以支持插入操作了。所以，本例中，我们只增加before指针，指明，它的前一个节点是谁。

但是，我们无法能够根据before指针更新其排序码。所以，我们需要根据排序码来更新before和next。所以，我们要对排序码算法作进一步优化。 

遥遥相对取的方式是：使用奇数序列或偶数序列作为排序码。那么，我们即可以在任一两节点间插入。插入后再作更新。

总结下来，我们的表结构如下：

id:           记录的id(可使用mysql自增字段）

parent_id：   父节点ID

path：        基于逗号分隔的ID完整路径

level：       层级，标明节点是第几层

is_leaf:      是否叶节点 （默认为1, 即叶节点）

sort_no:      排序编号

before_id     前一节点的id

假如此表是一个商品分类表，那么，表名称是prd_catagory 

我们增加一个业务数据字段：cat_name 分类名称。

接下来，就看我们sql算法是不是非常简单了。 

无论插入，更新（移动或不移动）还是删除，我们可用以下SQL重新更新排序码：

SET @var =0;UPDATE t1 a,
(
 SELECT t1.id, t1.sort_no,@var:=@var+2 AS new_sort_no
 FROM t1
 ORDER BY t1.sort_no
) b
SET a.sort_no=b.new_sort_no
WHERE a.id = b.id;

由此，我们通过链表结构维护了树结构中的基于公差为2的整数序列的排序码。 

有一点比较好的是：我们根据其排序码，可以相当方便地维护before和next. 只要再加一两条SQL语句即可。 

所以，当我们要完整示树结构时，我们的SQL查询只要是

select id, parent_id, path, level, is_leaf, sort_no, before_id from prd_catagory orderby sort_no

即可，非常简单吧。你的程序中只要用单层for循环顺序读取记录集即可。

查出某个节点下的所有子孙节点

select id, parent_id, path, level, is_leaf  from prd_catagory where path like :path orderby sort_no.

查出某个节点下的所有叶节点或非叶点

select id, parent_id, path, level, is_leaf from prd_catagory where parent_id = :parent_id and is_leaf=:is_leaf orderby sort_no.

查出所有同一级别的叶节点或非叶点

select id, parent_id, path, level, is_leaf from prd_catagory where level = :level and is_leaf=:is_leaf orderby sort_no.

好了。这个基于链表算法的无限级分类树的设计已经完成了。不知你还有什么改进意见。
