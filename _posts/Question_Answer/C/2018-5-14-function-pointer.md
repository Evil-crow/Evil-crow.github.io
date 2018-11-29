---
layout: post
comments: true
title: 从链表排序说起
date: May 14, 2018 5:01 PM
excerpt: 链表排序只是一个小的需求,但是我们如何排好序,如何写出代码复用度高的代码,就是我们水平的问题了,在这里还是要感谢hepangda的指点
categories:
- Q&A
toc: true
---

*如何使得自己的代码写的华丽而优雅, 如何提高代码复用程度, 如何提高我们的设计能力,需要长期的磨练*

*这里抛砖引玉,说说我自己的学习经历*

首先, 这个小Demo基本上4个多月前写的,当时hepangda看过之后就说过: 

你链表的排序可以借用qsort()的实现思想,来时代码写的更优雅一点,之前一直也没管, 这两天才想起来

那么改一改吧!

**首先,是需求,要求完成地址升序,地址降序,尺寸升序,尺寸降序这四种排序需求**

**不假思索的,我们可以将address,size封装成成员, 然后直接写四个排序函数, 但是这样有很大的弊端**

**四个函数,基本上只有一句话不同,这样的代码有什么意思,丑的一批(虽然之前为了省事,我可喜欢这样写了)**

所以,初版的代码如下:

```cpp
void bubble_sort_ascending_address(Node *block)
{
    block_node *temp, *cur, *loop;
    temp = block->head;
    if(temp->next == NULL) {
        printf("Error,the block is NULL!\n");
        exit(0);
    }

    for(int i = 0; i < block->node_num; i++) {
        for(temp = block->head; temp->next->next != NULL; temp = temp->next) {
            if(temp->next->block_address > temp->next->next->block_address) {
                cur = temp->next;
                loop = temp->next->next;
                cur->next = loop->next;
                temp->next = loop;
                loop->next = cur;
            }
        }
    }
}

void bubble_sort_ascending_size(Node *block)
{
    block_node *temp, *cur, *loop;
    temp = block->head;
    if(temp->next == NULL) {
        printf("Error,the block is NULL!\n");
        exit(0);
    }

    for(int i = 0; i < block->node_num; i++) {
        for(temp = block->head; temp->next->next != NULL; temp = temp->next) {
            if(temp->next->block_size > temp->next->next->block_size) {
                cur = temp->next;
                loop = temp->next->next;
                cur->next = loop->next;
                temp->next = loop;
                loop->next = cur;
            }
        }
    }
}

void bubble_sort_descending_address(Node *block)
{
    block_node *temp, *cur, *loop;
    temp = block->head;
    if(temp->next == NULL) {
        printf("Error,the block is NULL!\n");
        exit(0);
    }

    for(int i = 0; i < block->node_num; i++) {
        for(temp = block->head; temp->next->next != NULL; temp = temp->next) {
            if(temp->next->block_address < temp->next->next->block_address) {
                cur = temp->next;
                loop = temp->next->next;
                cur->next = loop->next;
                temp->next = loop;
                loop->next = cur;
            }
        }
    }
}

void bubble_sort_descending_size(Node *block)
{
    block_node *temp, *cur, *loop;
    temp = block->head;
    if(temp->next == NULL) {
        printf("Error,the block is NULL!\n");
        exit(0);
    }

    for(int i = 0; i < block->node_num; i++) {
        for(temp = block->head; temp->next->next != NULL; temp = temp->next) {
            if(temp->next->block_size < temp->next->next->block_size) {
                cur = temp->next;
                loop = temp->next->next;
                cur->next = loop->next;
                temp->next = loop;
                loop->next = cur;
            }
        }
    }
}

// 这四段代码,我写的时候,就是复制粘贴的,好丢人...
```

但是,听了hepangda的建议后,我们可以修改代码,这样写.

```cpp
void exchange(block_node **temp, block_node **cur, block_node **loop)
{
    *cur = (*temp)->next;
    *loop = (*temp)->next->next;
    (*cur)->next = (*loop)->next;
    (*temp)->next = *loop;
    (*loop)->next = *cur;
}

void bubble_sort(Node *block, bool (*compare)(block_node *, block_node *))
{
    block_node *temp, *cur, *loop;
    temp = block->head;
    if (temp->next == NULL) {
        printf("Error,the block is NULL!\n");
        return ;
    }

    for (int i = 0; i < block->node_num; ++i)
        for (temp = block->head; temp->next->next != NULL; temp = temp->next)
            if (compare(temp->next, temp->next->next))
                exchange(&temp, &cur, &loop);
}

bool address_ascend(block_node *a, block_node *b)         // 地址升序
{
    return a->block_address > b->block_address ? true : false;
}

bool address_descend(block_node *a, block_node *b)        // 地址降序
{
    return a->block_address < b->block_address ? true : false;
}

bool size_ascend(block_node *a, block_node *b)            // 尺寸升序
{
    return a->block_size > b->block_size ? true : false;
}

bool size_descend(block_node *a, block_node *b)           // 尺寸降序
{
    return a->block_size < b->block_size ? true : false;
}
```

*此时,上面的写法就相对优雅许多了,不是那么的无脑且垃圾.*

*主要使用函数指针的方式,将比较那一步抽出来,写了几个比较函数,然后bubble_sort()变成为通用函数*

**其中 bool类型时 #include < stdbool.h > 实现的,是为假实现,而_Bool则是真正意义上的boolen类型**

**不过,_Bool都是C11的特性了,话又说回来,如果写C11,那么这几个比较函数,直接写到.h文件**

**而且是inline函数呦**

如下:

```cpp
file: block_node.h

inline bool address_ascend(block_node *a, block_node *b)
{
    return a->block_address > b->block_address ? true : false;
}

inline bool address_descend(block_node *a, block_node *b)
{
    return a->block_address < b->block_address ? true : false;
}

inline bool size_ascend(block_node *a, block_node *b)
{
    return a->block_size > b->block_size ? true : false;
}

inline bool size_descend(block_node *a, block_node *b)
{
    return a->block_size < b->block_size ? true : false;
}
```

**另外,其中注意的是,C使用函数指针,函数原型中,返回类型直接写,不需要括号**

**C++中,一般就使用function object了,函数指针效率一般,当然C中就使用函数指针了**

### 总结 :

关于程序的设计,实际上就是一门艺术,我们如何设计好的程序,如何写出优雅的代码,都是值得思考的问题

就目前而言,多写,多练,无疑是一个重要的手段,是加上多看别人好的实现代码,意义更大

**比如,此处的qsort()思想,还是很有意义的**

多写,多练.所以说,写写C with STL 还是很有必要的.
