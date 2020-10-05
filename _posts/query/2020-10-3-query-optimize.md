---
layout: post
title: 多关联查询的优化
---


想象一下，你有一个淘宝店铺，然后你想看一下这个月的交易量，销售额，应该怎么做？假设目前数据库中有一个表示交易的表，比如`goods_order(id
, money, create_at)`。

| id | money | create_at |  
| -- | ----- | -------- |  
| 1 | 105 | 2020-10-01 |      
| 2 | 67 | 2020-10-01 |  
| 3 | 76 | 2020-10-02 |  

那么我们的查询语句可能是：
```sql
select count(1), sum(money) from goods_order where create_at >= '2020-10-01
' and create_at < '2020-11-01'
```
现在看起来似乎没什么问题，但是我们看的是总的交易量和销售额，我们希望看到每个商品的不同的销量和销售额怎么办呢？我们数据库中在增加一个商品表，比如`goods(id, name)`, 同时再给交易表增加一个`goods_id`字段, 改为`goods_order(id, goods_id, money, create_at)`，用来表示每笔交易的商品。现在我们数据库里的表如下：  

- 商品表(goods)

| id | name |  
| -- | ---- |  
| 1 | 商品A |  
| 2 | 商品B |  

- 交易表(goods_order)
  
| id | money | create_at |    
| -- | ----- | -------- |    
| 1 | 105 | 2020-10-01 |        
| 2 | 67 | 2020-10-01 |    
| 3 | 76 | 2020-10-02 |   

然后我们的查询语句如下:
```sql
select any(goods.name), count(1), sum(money) from goods_order join goods on
 goods_id
 = goods
.id group by goods_id
 where create_at
 >= '2020-10-01
' and create_at < '2020-11-01'
```
这样我们就需要用到join
, 看上去也没有什么问题，但是如果你的商品很多，那么是需要分页的。好，那么我们再把分页加上，做分页的话必须指明一个排序，我们这里就按照id来排序。  
```sql
select any(goods.name), count(1), sum(money) from goods_order join goods on
 goods_id
 = goods
.id group by goods_id
 where create_at
 >= '2020-10-01
' and create_at < '2020-11-01' order by goods_id asc limit 10
```
似乎也还好，但是，商品还有分类，我们再加上一个商品分类表`goods_category(id, name
)`。商品还有来源，代表从不同的进货渠道，再加上`goods_channel`。商品地址`goods_city(id, name
)`, 商品店铺`goods_shop(id, name)`, 商品经营者`shop_operator(id, name
)`。过多就不再举例了。好，接下来，我们想下面这个需求，同样是求不同商品的销量和销售额。但是我们还需要商品的分类，商品的来源，商品的店铺，商品的经营者，并且，店铺为小米，城市在北京，时间为10月。好了，想一下如何改写sql来完成上面的下需求。
```sql
select any(goods.name), 
    any(goods_category.name),
    any(goods_channel.name),
    any(goods_city.name),
    any(goods_shop.name),
    any(shop_operator.name),
    count(1),
    sum(money),
from goods_order
join goods on goods_id = goods.id
join goods_category on goods.category_id = goods_category.id
join goods_channel on goods.channel_id = goods_channel.id
join goods_city on goods.city_id = goods_city.id
join goods_shop on goods.shop_id = goods_shop.id
join shop_operator on goods_shop.operator_id = shop_operator.id
group by goods_order.goods_id
where goods_order.create_at > '2020-10-01' and goods_order.create_at <= '2020-11-01'
and goods_shop.name = '小米'
and goods_city.name = '北京'
order by goods_order.goods_id asc limit 10
```
好了，现在看起来似乎会有问题了，假设你的数据量还很大，那么估计会天天报慢查询。现在来想一下，有什么办法来优化呢。首先我们想一下，慢的原因是什么呢？太多的关联操作。那么这些关联操作都是必须的吗？有没有什么地方，资源被浪费掉了呢？  
当然是有的，首先，让我们回忆一下sql中不同运算的执行顺序。

1. 获取数据(from)
2. 连接(join)
4. 筛选(where)
5. 分组(group)
6. 聚合运算(sum, count)
7. 聚合筛选(having)
8. 选取(select)
9. 去重(distinct)
10. 排序(order by)
11. 分页(limit)

可以看到，是先执行from和join
的部分，那么如果我们表数量很大，那么就会花费很多时间在这些上面。但是我们想要查询的数据其实不多，是十月，店铺为小米，城市在北京，前十条的商品销量和销售额。所以最开始的join操作其实连接的数据大部分并没有被用到，但是缺仍然参与了开始的运算，浪费了资源。那么优化的思路就很明显了，延迟关联，先查询出前十10的销量和销售额了。再通过关联键去查询商品的名称，分类、渠道等信息。这样就避免了无效的关联操作。
大概的操作如下。
1. 首先查询十月店铺为小米、城市在北京，前十条的商品销量和销售额
```sql
select goods_id,
    count(1),
    sum(money) 
from goods_order
group by goods_id
where goods_order.create_at > '2020-10-01' and goods_order.create_at <= '2020-11-01'
and goods_id in (
    select id from goods 
    where shop_id in (select id from goods_shop where goods_shop.name = '小米')
    and city_id in (select id from goods_city where goods_city.name = '北京')
    )
order by goods_id asc limit 10
```
2. 接着根据结果中的goods_id查询商品的名称，分裂，渠道等信息, 查询完后再拼接起来。
```sql
select goods.name,
    city_id,
    category_id,
    channel_id,
    shop_id,
from goods 
where id in (?);
select id, name from goods_city where id in (?);
select id, name from goods_category where id in (?);
select id, name from goods_cahnnel where id in (?);
select id, name, operator_id from goods_shop where id in (?);
select id, name from shop_operator where id in (?)
```
你可能会有疑问，你这做的不是数据库里的查询优化器该做的吗？没错，其实这个叫做延迟关联，但是大多数据库都没这个东西，更不要说很多数据库的查询优化非常糟糕, 而且在多关联的情况下，由于太复杂，数据库可能会直接放弃优化。并且数据库处于比较低的层次，其实也很难对特定场景做优化。

## 延迟关联
好了，现在大家应该都明白了我们现在要谈的延迟关联是什么了。我们在将一个多关联查询转化为延迟关联的时候，有以下几个方面需要注意。分别是：
- 查询条件
- 排序和分页
- 关联的类型


### 查询条件
可以看到条件被转化成了子查询的形式，有人可能会担心子查询的效率问题，以及子查询创建临时表的损耗，不过一般来说忽略不及，不必在意这部分。还有人会建议先根据条件查询出id，再根据关联键过滤。不过这样有个缺点，因为sql的长度是有限制的，参数过多可能会不支持。网络传输也浪费了时间。在转换的时候，我们是根据两个表的关联键来转成一个in的子查询形式。在转换店铺和城市的条件的时候，关联的路径分别是`goods_order->goods->goods_shop`和`goods_order->goods->goods_city`。因此转换为了一个嵌套的子查询的形式。
```sql
goods_id in (
    select id from goods 
    where shop_id in (select id from goods_shop where goods_shop.name = '小米')
    and city_id in (select id from goods_city where goods_city.name = '北京')
    )
```

### 排序和分页
这里主要有一个问题，那就是我们第一次查询的时候仅仅是在goods_order上的, 并没有获取到关联表上的数据。想一下前面的排序商品的id
，由于`goods_order`中有一个关联键`goods_id
`，所以直接用这个来排序了。如果要求按照商品的创建时间，商品的名称来排序，该如何处理呢？这里就无法使用延迟关联了，不过只是这一个表join
,也不会对性能有太大影响。

### 关联的类型
在那么现在再来谈一下关联的类型，我们使用的都是join
, 也就是内关联。我们想一下，两个转换前后的查询是否是等价的呢。比如说，有一件商品并没有分类，那么join
 的时候就会失败，这样在转换之前的查询就没有这个商品的信息，而转换后的查询因为没有关联操作，所以这个商品的信息会留下。你可能会问，怎么会有这种数据呢，我业务上保证每个商品都会有一个分类不就行了，当然可以。不过在实际开发过程中，并不是一下子就把一个完整的系统出来的，很多功能是后期慢慢添加上去的，这就会导致这种情况发生，当然如果在添加每一个新功能的时候，都能考虑到全局的影响，已有的业务数据如何兼容，也不会有这种问题。但是一般实际上并不都是这么理想。一般来说，我们都是希望保留这个商品信息的，所以实际上关联类型是left join。  
 还有一个问题，就是我们这里交易和商品，商品和商品类型、商品渠道等等之间都是多对一的关系，如果是多对多，比如说一个交易包含多个商品，或者一个商品有多个分类，那也就不等价了。这种情况下，只能将多对多拆开，比如将转化为包含多个商品的大交易转成包含一个商品的小交易。或者将一边转为一，比如商品的分类，可以使用一个数组来保存商品的分类。

## 感悟
在整个过程中获得的感悟就是，在设计一个系统的时候，一定要确定好需求是什么，应该应对需求来设计。并且一定要明确自己的系统在上下游处于什么地位，应该承担什么样的角色和功能。在不同的地方应该要处理不同的东西，比如上面的优化，在特定的业务中做特定的优化是非常简单的，不应一味的丢给数据库解决。而且很多的问题也只能比如关联的类型，其他一些问题也只能在业务层解决。比如在同一个数据库中，查询条件转成子查询，以及排序分页丢到数据库里是最好的，但是如果存在不同数据库，或者还要微服务中查询，就只能在业务内部做这些东西。而你在设计这些的时候，一定要明白边界在那里，把任务安排给最适合做这些的地方，而不是为了大而全，把一些已有的东西再做一遍，这也是unix的思想。
