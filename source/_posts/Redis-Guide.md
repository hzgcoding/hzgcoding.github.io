---
title: 'Redis Guide 使用手册'
date: 2020-07-10 18:28:44
tags: Redis
categories: Redis
---

# Redis 数据结构与应用

## 普通字符串

### SET 命令

1.  SET KEY VALUE 设置
2.  NX 不存在才设置  可用于实现分布式锁
3.  XX 存在才设置

### GET 命令

1. GET KEY 返回 VALUE 值  
2. GETSET 返回旧 OLD_VALUE 值 , 并且设置新 NEW_VALUE

### MSET 命令

1. MSET KEY1 VALUE1 [ KEY2 VALUE2 ...] 一次设置多个键值对

### MGET 命令

1. MGET KEY1 [ KEY2 ...] 一次设置多个键值对
   返回VALUE列表, 这在批量获取多个键值对的时候非常实用, 业务层不用做封装

### MSETNX 命令

1.  MSETNX KEY VALUE [ KEY2 VALUE2 ...] 
   设置多个键值对, 这是一个原子操作, 当且仅当所有KEY都不存在的时候才会成功

### STRLEN 命令

1. STRLEN KEY 获取值的字节长度

    ![](https://i.loli.net/2020/07/14/Wwyzn7lEbvIJxpF.png)

### GETRANGE 命令

1. GETRANGE KEY start end 
   根据指定索引范围设置值，相当于取值的子串

### SETRANGE 命令

1. SETRANGE KEY startIndex substitute 
   根据指定开始索引，开始替换新串substitute; 当startIndex大于值长度(len - 1), redis会自动扩展值长度，并且填充空字节(\x00)

### APPEND 命令

1. APPEND KEY suffix 对值进行追加操作, 并返回新串长度
   当KEY不存在, 那么APPEND命令等价于SET操作

### INCRBY DECRBY 命令

1. INCRBY/DECRBY KEY incrment 当存储的值能够被解释为整数时， 可以使用这两个命令来对值进行加或减
   当KEY不存在，那么会被初始化为0，然后再进行操作

### INCR DECR 命令

1. INCR/DECR KEY incrment 当存储的值能够被解释为整数时， 可以使用这两个命令来对值进行加或减1
   当KEY不存在，那么会被初始化为0，然后再进行操作

### INCRBYFLOAT 命令

1. INCRBYFLOAT KEY incrment， 命令与INCRBY无差别，主要是针对的是浮点数
   但是没有DECRBYFLOAT命令，可以通过设置incrment为负数来实现减法
   INCRBYFLOAT 命令对值的格式更加宽松，那么值是整数的时候，命令与INCRBY等同，存储的值也会是整数
   注意Redis在处理浮点数的时候，小数位长度是有限制，最长为17位，对于大部分应用是足够的


## 散列

### 结构

|   KEY      |       FIELD       |         VALUE        |
|  :----:    |      :----:       |      :----:          |
|article     |     title         |     "Hello World"    |
|            |     content       |     "son of bitch"   |
|            |     author        |     genge            |
|            |     created_at    |     2010-10-10       |

### HSET 命令

1. HSET HASHKEY FIELD VALUE 
   如果key或者field不存在, 那么会创建，返回值为1
   如果field存在, 那么会更新，返回值为0

### HSETNX 命令

1. HSETNX HASH FIELD VALUE
   只在字段不存在的时候设置值，返回值为1，设置失败返回0

   