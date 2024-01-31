---
title: "boost容器之bimap"
date: 2017-12-23 19:14:46
excerpt: "boost容器之bimap"
tags: boost
---

在使用mysql时我们经常有需求对不止一列建立索引，例如对于student表，既可能按照`id`查找，也可能按照`name`查找。

这种持久化存储的需求，在内存对象的使用中也经常遇到，现有的数据结构`boost::bimap`可以用于这个场景。

简单的讲`bimap`相当于两个不同方向的`std::map`，两个视图都是**key -> value**的数据结构，也称为左视图和右视图。

<!--more-->

## 1. 使用

```cpp
    //std::string <-> int
    typedef boost::bimap<std::string, int> bimap;
    bimap animals;

    animals.insert({"cat", 4});
    animals.insert({"shark", 0});
    animals.insert({"spider", 8});
    auto res = animals.insert({"dog", 4});
    std::cout << "insert res second:" << res.second << std::endl;//0
    std::cout << "insert res first.left:" << res.first->left << std::endl;//cat
    std::cout << "insert res first.right:" << res.first->right << std::endl;//4
     //error: no match for ‘ operator[]’  (operand types are ‘ bimap {aka boost::bimaps::bimap<std::basic_string<char>, int>}’  and ‘ std::string {aka std::basic_string<char>}’ )
    // animals[std::string("dog")] = 4;

    std::cout << animals.left.count("cat") << std::endl;//1
    std::cout << animals.right.count(8) << std::endl;//1

    for (const auto& animal : animals) {
        std::cout << animal.left << " " << animal.right << std::endl;
    }
```

可以看到`insert`的行为与`std::map`一致，当key存在的时候(这里是右视图的key)，`insert`失败。

`bimap`提供了`left right`接口，分别用于获取左右视图，对于左视图，是`string -> int`的`std::map`，例如

```
cat -> 4
shark -> 0
spider -> 8
```

对于右视图，是`int -> string`的`std::map`，例如

```
cat <- 4
shark <- 0
spider <- 8
```

从这里也可以看到`insert({"dog", 4})`为什么失败，因为对于右视图，key已经存在。

对于这种情况下，`operator[] at`不能使用，对于这一行会编译失败

```cpp
animals[std::string("dog")] = 4;
```

## 2. 值的集合类型

严格的讲，传入bimap的template类型，并不是真正的存储类型。bimap的left/right，实际上某种类型的container，支持手动指定。

默认类型为`boost::bimaps::set_of `，要求元素是unique的，看个指定container的例子

```cpp
    typedef boost::bimap<boost::bimaps::set_of<std::string>,
            boost::bimaps::multiset_of<int>> bimap;
    bimap animals;

    animals.insert({"cat", 4});
    animals.insert({"shark", 0});
    animals.insert({"dog", 4});

    for (const auto& animal : animals) {
        std::cout << animal.left << " " << animal.right << std::endl;
        // cat 4
        // dog 4
        // shark 0
    }
```

右视图修改为`boost::bimaps::multiset_of`后，`insert({"dog", 4})`可以正常插入数据。

bimap里container的类型很多

|type of container  |是否可以用作键值 |注释  |
|--|--|--|
|set_of  |Y  |有序且唯一，视图相当于map  |
|multiset_of  |Y  |有序，视图相当于multimap  |
|unordered_set_of  |Y  |无序且唯一，视图相当于unordered_map  |
|unordered_multiset_of  |Y  |无序，视图相当于unordered_multimap  |
|list_of  |N  |序列集合  |
|vector_of  |N  |随机访问集合  |
|unconstrained_set_of  |N  |无任何约束关系  |

## 3. std::map

```cpp
    typedef boost::bimap<std::string,
            boost::bimaps::unconstrained_set_of<int>> bimap;
    bimap animals;

    animals.insert({"cat", 4});
    animals.insert({"shark", 0});
    animals.insert({"dog", 4});
    for (const auto& animal : animals) {
        std::cout << animal.left << " " << animal.right << std::endl;
        // cat 4
        // dog 4
        // shark 0
    }

    // error: ‘ boost::bimaps::bimap<std::basic_string<char>, boost::bimaps::unconstrained_set_of<int> >::right_map’  has no member named ‘ find’
    //animals.right.find(0);

    auto it = animals.left.find("cat");
    animals.left.modify_key(it, boost::bimaps::_key = "elephant");
    for (const auto& animal : animals) {
        std::cout << animal.left << " " << animal.right << std::endl;
        // dog 4
        // elephant 4
        // shark 0
    }
```

右视图修改为`unconstrained_set_of`类型后，`bimap`与`std::map`行为非常类似，遍历结果可以看到是key有序。

同时右视图没有`find`接口，编译报错。

不过相比`std::map`，可以使用`modify_key`直接修改key。

## 4. tag

前面提到了左视图、右视图分别用`bimap.left` `bimap.right`来访问，为了提高易用性，boost提供了`bimaps::tagged`类给左右视图在语法层面上贴tag，在使用上可以更清晰的看到左右视图的含义。

```cpp
    boost::bimap<boost::bimaps::tagged<int, struct id>,
        boost::bimaps::tagged<std::string, struct name> > bm;
    //通过bimap::by接口可以更清晰的表名视图的含义
    bm.by<id>().insert(std::make_pair(1, "samus"));
    bm.by<id>().insert(std::make_pair(2, "adam"));

    bm.by<name>().insert(std::make_pair("link", 10));
    bm.by<name>().insert(std::make_pair("zelda", 11));

    for (const auto& iter : bm) {
        std::cout << iter.left << "->" << iter.right << std::endl;
        // 1->samus
        // 2->adam
        // 10->link
        // 11->zelda
    }
```

`boost::bimap`在引入不同类型的container后，组件变得非常复杂。在实际应用时，可以灵活使用，我使用的也并不全面，经验不多。本文大部分也是参考的[Boost程序库完全开发指南](https://book.douban.com/subject/26320630/)一文，有兴趣的读者建议深入阅读下。

## 5. 一个使用的例子

回到我们最开始的关于students表的使用场景，看下`bimap`如何有效的基于`id` or `name`为key的操作。

```cpp
    boost::bimap<boost::bimaps::tagged<int, struct id>,
        boost::bimaps::multiset_of<boost::bimaps::tagged< std::string, struct name> > > students;

    int student_id = 1;
    students.by<id>().insert(std::make_pair(student_id++, "Jeff"));
    students.by<id>().insert(std::make_pair(student_id++, "Tom"));
    students.by<name>().insert(std::make_pair("Ying", student_id++));
    students.by<name>().insert(std::make_pair("Shabby", student_id++));
    students.by<name>().insert(std::make_pair("Tom", student_id++));

    for (const auto& iter : students) {
        std::cout << iter.left << "->" << iter.right << std::endl;
    }

    std::cout << students.by<id>().find(3)->second << std::endl;//Ying
    std::cout << students.by<name>().count("Tom") << std::endl;//2
```

## 6. 参考

1. [Boost程序库完全开发指南](https://book.douban.com/subject/26320630/)  
2. [Boost.Bimap](https://theboostcpplibraries.com/boost.bimap)  
