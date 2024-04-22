+++
title = 'C++STL之unordered_map用法简析,从造轮子到用轮子'
date = 2024-04-22T15:42:29+08:00
draft = false
+++

*原发布于2020年5月15日*
http://blog.gensou.cc/archives/346


unordered_map是C++11中加入的，以哈希表为索引方式的STL结构。与map不同，unordered_map寻找索引的值的理论时间复杂度仅为O(1)，而依靠红黑树的map是O(logn)。

要理解unordered_map的运作原理，首先来造个轮子，写个自己的哈希表寻址结构。

Hash值是每个数据某个数据（比如是key）通过运算得到的某个数值，将该数据放进整个结构（比如数组）中以这个Hash值为索引的地址中，就类似于把数据倒入桶中。当需要找到某个key值对应的数据时，也同样算出Hash值，然后直接在结构中寻找就可以。由于数组的地址连贯性，Hash值作为索引可以直接计算出数据所在的地址，因此时间复杂度仅为O(1)。

造个Hash Table试试
首先是用到的数据的结构Record：
```c++
class Record
{
private:
  int key;
  int value;
public:
  int the_key() { return key; }
  int the_value() { return value; }
  Record() { key = value = -1; }
  Record(int k, int v) : key(k), value(v) {}
  bool operator==(Record r) { return the_value() == r.the_value(); } //相等
  Record operator=(Record r); //赋值构造函数
  //...以及更多需要的函数
};
```
定义好了key和value，写好构造函数并且重载部分需要的操作符（具体略）。然后是利用Hash值寻址的结构hash_table。
```c++
enum Error_code { not_present, overflow, deplicate_error, success };
const int hash_size = 100;
class hash_table
{
private:
  Record table[hash_size];
  int hash(Record r)； //获得某个Record的Hash值
  bool is_empty(int index) { return table[index].the_key() == -1; } //查询table数组中index位上是否已经有数据
public:
  hash_table();
  hash_table(const hash_table& ht);
  //...以及更多构造函数
  Error_code insert(const Record& r); //插入某个Record的函数
  Error_code retrive(const int& key, Record& r); //查询表中是否存在某个key值，并且存在r中
  Error_code remove(const int& key); //删去该key值对应的Record
  Error_code find_key(int value, int& result); //寻找第一个value匹配key
  Error_code find_value(int key, int& result); //寻找key对应的value
  //...以及更多增删查改的操作
  hash_table operator=(const hash_table& ht); //赋值构造函数
  //...以及更多重载运算符 
};
```
其中，灵魂部分便是hash(Record r)，这个函数对r的key计算得到其Hash值，后面的函数中，增删查改等所有需要从key来寻找到地址的操作，都需要用到这个函数。

例：用取余运算作为Hash函数
//简单的采用取余得到的结果作为Hash值
```c++
int hash_table::hash(Record r) 
{ 
  return r.the_key() % hash_size; 
}
//每次insert一个Record，便先算出其hash值，然后根据索引存储数据
Error_code hash_table::insert(const Record& r)
{
  for (int index = hash(r), k = 0; k < hash_size; (index = (index == hash_size - 1) ? 0 : index + 1), k++)
  {
    if (is_empty(index))
    {
      table[index] = r;
      return success;
    }
  }
  return overflow;
}
//通过key寻找value也同样先算出Hash值，载根据索引查询数据
Error_code hash_table::find_value(int key, int& result)
{
  for (int index = hash(Record(key, 0)); index < hash_size; index++)
  {
    if (is_empty(index))
    {
      return not_present;
    }
    if (table[index].the_key() == key)
    {
      result = table[index].the_value();
      return success;
    }
  }
}
//...以及更多函数
```
这里需要注意的一点是，当数据发生冲突，即Hash值相等时，这里采用的办法是从这个地址开始不断向后寻找下一个可以存储数据的地址，然后存下，如果全满则返回overflow；取值的时候也类似。这种处理冲突的办法称为开放寻址法。

Unordered_map之用法
声明
像其他STL一样，unordered_map已经内置了很多方法；不过，它还可以传入更多的参数，先看看其定义
```c++
template < class Key,                  // unordered_map::key_type
           class T,                    // unordered_map::mapped_type
           class Hash = hash<Key>,     // unordered_map::hasher
           class Pred = equal_to<Key>, // unordered_map::key_equal
           class Alloc = allocator< pair<const Key,T> >  // unordered_map::allocator_type
           > class unordered_map;
```
第三到第五个参数都可以省略。第一个参数是Key值，第二个参数是存储的数据，第五个参数是空间配置器，没必要修改。这里聊聊第三第四两个参数。
其中，第三个参数需要一个包含取Hash值的函数，取的对象是第一个参数，即为数据的key值。这里依然以取余运算为Hash函数为例。
```c++
struct my_hash
{
  int operator()(const int& value) const
  {
    return value % hash_len;
  }
};
```
struct就是默认为public的class。为了简便，直接重载()运算符，这样，每个my_hash的对象都可以相当于一个函数名，用()传入参数，调用取Hash值的函数。
接下来是第四个参数，它需要包含一个判断key值是否相等的函数。
```c++
struct my_compare
{
  bool operator()(const int& x, const int& y) const
  {
    return x == y;
  }
};
```
也同样的道理，重载()运算符即可。
有了这样自定义好的两个类，便可以创建unordered_map了。
```c++
unordered_map<int, int, my_hash, my_compare> umap;
//声明了一个key值为int，数据也是int，取Hash值的方法的类为my_hash，判断key值是否相等的方法的类为my_compare的unordered_map
```
当然，这个例子中的my_hash和my_compare也是可以省略的，因为int型是可以直接比较的，默认的比较函数可以工作；unordered_map也内置了计算Hash值的方法。

方法
首先，键值对在unordered_map中的每个元素是pair<const Key, T>的模板类的实例化的对象，每一个pair中first属性是其key，second属性是其value。

另外，后一个insert或emplace入的的同key元素将被忽略，不会覆盖原有的value

- unordered_map::begin, end, cbegin, cend 头迭代器，超尾迭代器，const头迭代器，const超尾迭代器
- unordered_map::operator= 赋值构造函数，可以传入另一个unordered_map；移动构造函数，传入一个unordered_map右值；初始化列表，用直接量的列表赋值
- unordered_map::operator[] 其行为类似数组，以key为索引，可以读写值
- um3[“z”] = 5;
- um3[“y”] = um3[“z”];
- unordered_map::insert 插入一个或多个键值对
- um.insert(pair<string, int>{ “a”, 5 }); //使用直接量
- um.insert({ “a”, 5 }); //省略类型
- um2.insert(um.begin(), um.end()); //使用迭代器
- unordered_map::emplace 传入参数列表，自动创建一个键值对
- um.emplace( “c”, 10 ); //添加键值对pair<string, int>{ “c”, 10 }
- unordered_map::emplace_hint 传入一个迭代器和参数列表，自动（优先从该迭代器的位置）创建一个键值对，并且当成功插入时，返回指向插入元素的迭代器；因为key已存在而插入失败时，返回指向该key所在元素的迭代器（值不会覆盖）
- um.emplace_hint(um.begin(), “a”, 10); //返回指向{ “a”, 5 }的迭代器
- unordered_map::at 获得传入key对应的值
- um.at(“a”); //返回5
- unordered_map::find 寻找传入key对应的迭代器
- um.find(“a”); //返回指向pair<string, int>{ “a”, 5 }的迭代器
- um.find(“b”); //key不存在，返回um.end()
- unordered_map::erase 删去容器中的部分元素，可以传入key的值、某个迭代器、或者两个迭代器组成的范围
- um2.erase(um2.begin(), um2.end()); //这个例子相当于clear
- unordered_map::clear 清空容器
- unordered_map::swap 传入另一个unordered_map，交换两个容器的值
- unordered_map::empty 返回容器是否为空的布尔值
- unordered_map::count 计算传入的key在容器中存在的个数；事实上因为key是唯一的，只能得到0或1
- um.count(“a”); //返回1
- um.count(“b”); //返回0
- unordered_map :: hash_function 返回该容器使用的Hash函数的对象（可以当做函数指针使用）
- auto fn = um.hash_function();
- fn(“d”); //返回3775669363
- unordered_map :: key_eq 无参数，返回一个需要传入两个参数的函数，给该函数传入两个key，返回两个key是否Hash值相等的布尔值
- um.key_eq()(“a”, “A”); //返回false
- unordered_map::bucket 获得传入的key所在的桶的编号，传入的key不一定要在容器中已存在
- um.bucket(“a”); //返回4
- um.bucket(“b”); //返回5
- unordered_map::bucket_count 返回容器的桶的个数
- um.bucket_count(); //返回8
- unordered_map::bucket_size 传入一个unsigned int，获得这个编号的桶内的元素数
- um.bucket_size(4); //返回1
- um.bucket_size(5); //返回0
- unordered_map::size, max_size, load_factor, max_load_factor 该容器的元素个数，最大元素个数，已加载的桶的比例，最大加载的桶的比例，其中，load_factor = size / bucket_count
- unordered_map::reserve 传入一个unsigned int，将容器中的bucket_count设置为最适合的包含至少n个元素的大小
- unordered_map :: rehash 传入一个unsigned int，将容器中的bucket_count设置为n（或更多）
- unordered_map::equal_range 传入一个key，返回寻找到含有这个key的键值对所组成一个范围。返回值的first属性是头键值对，second属性是尾键值对。因为key是唯一的，返回的范围自然也只有一个元素，该方法在unordered_multimap中更常用。
```c++
auto range = um.equal_range(“a”);
for_each( range.first, range,second, [](auto x) {cout << x.first << x.second; } //输出a5
```
这里使用了for_each语句，给定头尾和执行的函数，并且用Lambda表达式作为匿名函数，简单实现将range中所有元素（只有一个）进行输出
其中x的类型是unordered_map::value_type
- unordered_map::get_allocator 返回allocator_type

本文参考了 http://www.cplusplus.com/reference/unordered_map/
