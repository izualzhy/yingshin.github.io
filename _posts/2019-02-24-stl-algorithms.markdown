---
title: "summary of <105 STL Algorithms in Less Than an Hour>"
date: 2019-02-24 12:02:56
tags: cpp
---

<video src="/assets/images/world map of C++ STL.mp4" width="80%" controls="controls">
your browser does not support the video tag
</video>

## 1. WHY?

1. STL algorithms can make code more expressive  
2. Avoid common mistakes  
3. Used by lots of people

## 2. THE WORLD OF C++ STL ALGORITHMS

### 2.1. LANDS of PERMUTATIONS

#### 2.1.1. PROVINCE OF HEAPS

+ make_heap  
+ push_heap  
+ pop_heap  

#### 2.1.2. SHORE OF SORTING

+ sort  
+ partial_sort  
+ nth_element  

```cpp
// Quick Sort
template<class FwdIt, class Compare = std::less<>>
void quickSort(FwdIt first, FwdIt last, Compare cmp = Compare{})
{
    auto const N = std::distance(first, last);
    if (N <= 1) return; 
    auto const pivot = std::next(first, N / 2);
    std::nth_element(first, pivot, last, cmp);
    quickSort(first, pivot, cmp); 
    quickSort(pivot, last, cmp); 
}
```

+ sort_heap   
+ inplace_merge   
+ partition  

```cpp
// Gather
template <typename BiIt, typename UnPred> 
auto gather(BiIt f, BiIt l, BiIt p, UnPred s) -> std::pair <BiIt, BiIt>
{
    return { stable_partition(f, p, not1(s)), 
             stable_partition(p, l, s) };
}
```
+ partition_point  
+ rotate  

```cpp
// Insertion Sort
for (auto i = start; i != end; ++i)
    std::rotate(std::upper_bound(start, i, *i), i, std::next(i));

// Slide
template <typename It> 
auto slide(It first, It last, It pos) -> std::pair<It, It>
{
    if (pos < first) return { pos, std::rotate(pos, frist, last) };
    if (last < pos) return { std::rotate(first, last, pos), pos };
    return { first, llast };
}
```
+ shuffle  
+ next_permutation  

```cpp
while (std::next_permutation(start, end));
```
+ prev_permutation  
+ reverse  

#### 2.1.3. SECRET RUNES

```cpp
stable_*  ->  stable_sort
              stable_partition
is_*      ->  is_sorted
              is_partitioned
              is_heap
is_*_until->  is_sorted_until
              is_partitioned_until
              is_heap_until
```

### 2.2. LANDS OF QUERIES

#### 2.2.1. PROVINCE OF VALUE QUERIES

+ count  
+ accumulate/(transform_)reduce  
+ partial_sum  
+ (transform_)inclusive_scan  
+ (transform_)exclusive_scan  
+ inner_product  
+ adjacent_difference  
+ sample  

#### 2.2.2. PROVINCE OF PROPERTY QUERIES

+ all_of  
+ any_of  
+ none_of  

#### 2.2.3. PROVINCE OF 2-RANGES PROPERTIES

+ equal  
+ lexicographical_compare  
+ mismatch  

#### 2.2.4. PROVINCE OF RESEARCH

search a value:  
not sorted:  
+ find  
+ adjacent_find  

sorted:  
+ equal_range  
+ lower_bound  
+ upper_bound  
+ binary_search  

search a range:  
+ search  
+ find_first_of  

search a relative value:  
+ min_element  
+ max_element  
+ minmax_element  

### 2.3. GLORIOUS COUNTRY OF ALGOS ON SETS

+ set_difference  
+ set_union  
+ includes  
+ set_intersection  
+ set_symmetric_difference  
+ merge  

### 2.4. TERRITORY OF MOVERS

+ copy  
+ move  
+ swap_ranges  
+ copy_backward  
+ move_backward  

### 2.5. LANFS OF MODIFIERS

+ fill  
+ generate  
+ iota  
+ replace  

### 2.6. ISLAND OF STRUCTURE CHANGERS

+ remove  
+ unique  


with runes:

```cpp
*_copy      -> remove_copy
               replace_copy
               reverse_copy
               rotate_copy
               unique_copy
               partition_copy
               partial_sort_copy

*_if        -> find_if
// Trim
std::string trim(const std::string &s) {
    return trimLeft(trimRight(s));
}

std::string trimLeft(const std::string &s) {
    auto temp = s;
    temp.erase(std::begin(temp), 
                std::find_if(std::begin(temp), std::end(temp), 
                    [](char c){return !std::isspace(c, std::locale()); 
                }));
    return temp;
}

std::string trimRight(const std::string &s) {
    auto temp = s;
    temp.erase(std::find_if(std::rbegin(temp), std::rend(temp), 
                [](char c){return !std::isspace(c, std::locale()); }).base(), 
                   std::end(temp));
    return temp;
}

               find_if_not
               count_if
               remove_if
               remove_copy_if
               replace_if
               replace_copy_if
               copy_if
```

### 2.7. LONELY ISLAND

+ transform  
+ for_each  

### 2.8. PENINSULA OF RAW MEMORY

+ uninitialized_fill  
+ uninitialized_copy  
+ uninitialized_move  
+ uninitialized_default_construct  
+ uninitialized_value_construct  
+ destory  

with runes

```cpp
*_n        -> copy_n
              fill_n
              generate_n
              search_n
              for_each_n
              uninitialized_copy_n
              uninitialized_fill_n
              uninitialized_move_n
              uninitialized_default_construct_n
              uninitialized_value_construct_n
              destroy_n
```

## 3. Refs

[The World Map of C++ STL Algorithms](https://www.fluentcpp.com/getthemap/)
