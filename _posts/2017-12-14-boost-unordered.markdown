---
title: "boost容器之unordered"
date: 2017-12-14 23:21:45
excerpt: "boost容器之unordered"
tags: boost  unordered
---

boost里unordered容器由集合(unordered_set)与映射(unordered_map)组成。

**unordered**是相对于ordered而言，例如我们平时使用的`set map`是有序的

```
    std::map<std::string, int > url_weight;
    url_weight["www.baidu.com"] = 1;
    url_weight["www.sohu.com"] = 1;
    url_weight["www.sina.com"] = 1;
    url_weight["www.google.com"] = 1;

    for (const auto & uw : url_weight) {
        std::cout << uw.first << std::endl;
    }
    //www.baidu.com
    //www.google.com
    //www.sina.com
    //www.sohu.com
```

这种差别本质上是底层数据结构的不同：rb tree vs hash table.

本文主要介绍`boost::unordered_set boost::unordered_map`的常用接口，注意使用上与`std::unordered_set std::unordered_map`几乎完全一致。

<!--more-->

## 1. 使用

`boost::unordered_set`与`std::set`接口很像

```
    boost::unordered_set<std::string> zoo_set{"monkey", "wolf"};

    zoo_set.insert("elephant");
    zoo_set.insert("lion");
    zoo_set.insert("tiger");

    for (const std::string& s : zoo_set) {
        std::cout << s << std::endl;
    }

    std::cout << zoo_set.size() << std::endl;
    std::cout << zoo_set.max_size() << std::endl;

    std::cout << std::boolalpha << (zoo_set.find("panda") != zoo_set.end()) << std::endl;
    std::cout << zoo_set.count("lion") << std::endl;
```

`boost::unordered_map`与`std::map`接口很像，插入数据可以直接使用`std::pair`

```
    boost::unordered_map<std::string, int> scores;
    scores["chinese"] = 86;
    scores["english"] = 95;
    scores["maths"] = 100;

    for (const auto &p : scores) {
        std::cout << p.first << ":" << p.second << std::endl;
    }

    //std::pair<boost::unordered_map<std::string, int>::iterator, bool>;
    auto res = scores.insert(std::make_pair("maths", 60));
    std::cout << res.first->first << "\t" << res.second << std::endl;
    for (const auto &p : scores) {
        std::cout << p.first << ":" << p.second << std::endl;
    }

    std::cout << scores.size() << std::endl;
    std::cout << scores.max_size() << std::endl;

    std::cout << std::boolalpha << (scores.find("chinese") != scores.end()) << std::endl;
    std::cout << scores.count("maths") << std::endl;

    //unordered内部使用桶(bucket)来存储元素，散列值相同元素会放到同一个桶。
    std::cout << scores.bucket_count() << std::endl;//16
    for (size_t i = 0; i < scores.bucket_count(); ++i) {
        //1 0 0 0 0 1 0 0 0 0 0 0 0 0 1 0
        std::cout << scores.bucket_size(i) << " ";
    }
```

## 2. 自定义类型

因为底层使用的是hash_table，因此自定义类型需要实现`hash_value`与`operator==`

```
struct Link {
    std::string url;
    std::string schema;
};

bool operator==(const Link& lhs, const Link& rhs) {
    return lhs.url == rhs.url && lhs.schema == rhs.schema;
}

std::size_t hash_value(const Link& link) {
    std::size_t seed = 0;
    boost::hash_combine(seed, link.url);
    boost::hash_combine(seed, link.schema);

    return seed;
}

void test_user_defined_unordered() {
    boost::unordered_set<Link> links;
    links.insert({"www.baidu.com", "http"});
    links.insert({"www.baidu.com", "https"});
    links.insert({"www.google.com.cn", "https"});

    for (const auto& link : links) {
        std::cout << link.url << "\t" << link.schema << std::endl;;
    }
}
```

## 3. operator[] insert emplace的区别

operator[]返回的是一个对应value的引用，如果访问前key不存在，那么将创建一个默认的value，如果访问前key存在，那么可以修改。

insert只有在key不存在时才会成功，返回值是一个iterator。例如上面例子里res类型为

```
    //std::pair<boost::unordered_map<std::string, int>::iterator, bool>;
    auto res = scores.insert(std::make_pair("maths", 60));
```

其中first指向对应的iterator,second是一个bool，表名是否插入成功。

emplace实现了c++11里的move语义，具体看个例子

```
struct Student {
    Student() {
    }
    Student(const char* name_)
        : name(name_) {
              std::cout << "Con     this:" << this
                  << "\tname:" << name
                  << std::endl;
          }
    Student(const Student& s) {
        name = s.name;

        std::cout << "CopyCon this:" << this
            << "\tname:" << name
            << std::endl;
    }
    Student& operator=(const Student& s) {
        name = s.name;
        std::cout << "Oper=   this:" << this
            << "\tname:" << name
            << std::endl;
    }

    ~Student() {
        std::cout << "Decon  this:" << this
                  << "\tname:" << name
                  << std::endl;
    }

    std::string name;
};//Student

    boost::unordered_map<int, Student> students;
    students[1] = "Jeff";
    students.emplace(2, "Ying");
    students.insert(std::make_pair(3, "Tom"));
```

输出：

```
Con     this:0x7fff10871c00     name:Jeff
Oper=   this:0xe78118   name:Jeff
Decon  this:0x7fff10871c00      name:Jeff
Con     this:0xe78138   name:Ying
Con     this:0x7fff10871c18     name:Tom
CopyCon this:0xe78198   name:Tom
Decon  this:0x7fff10871c18      name:Tom
Decon  this:0xe78198    name:Tom
Decon  this:0xe78138    name:Ying
Decon  this:0xe78118    name:Jeff
```

可以看到emplace时只有一次构造函数，因此现代C++语法更推荐使用`emplace`，顺便吐槽下我厂的C++还停留在石器时代:sweat_smile:。

emplace返回值上跟insert行为一致。

emplace在使用上构造函数多参数时更复杂一些，这里不多做介绍。

## 4. 参考

[Boost程序库完全开发指南](https://book.douban.com/subject/26320630/)
