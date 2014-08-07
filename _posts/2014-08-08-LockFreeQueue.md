---
layout: post
title:  无锁队列的分析与设计
category: articles
tags:  lockfree datastructure c++
image:
    feature: head1.jpg
author:
    name:   WuYu
    avatar: bio-photo-alt.jpg
comments: true
share: true
---

首先说明为什么要有无锁队列这样的数据结构。大家都知道当今一个提高应用性能的主要方式是采取并发编程的模式，而其中尤其以多线程编程方式为主。线程是共享其所属进程内存空间的独立执行实体，在linux系统里是没有所谓线程的，其实现方式就是标准进程，只不过这种进程能和其他某些进程共享一定的资源罢了。现如今的操作系统都具有切换线程上下文环境，允许线程并发执行的能力。这种能力的的优势是显而易见的，当某线程拥有慢速操作的时候（最主要是各种I/O操作），其他更需要CPU的线程可以“取而代之”执行，说白了就是不要因为一条鱼腥了一锅汤，让应用的整体的执行效率得到提高；但是多线程编程是一柄双刃剑，因为当多线程需要共享数据和通信的时候，事情就会变得非常麻烦，会发生诸如死锁，非控制的共享数据存取，内存动态分配与释放等等问题。即便你运气好到没有遇到这些问题，还是有诸如Cache失效率过高等应用性能的掣肘（假设你对自己程序的性能很在意），因此无锁数据结构就应运而生了。考虑到队列广泛的应用性，本文我就以无锁队列为引，和大家共同学习一下无锁数据结构的一点相关知识。当任务使用无锁队列的时候，它们不再需要竞争资源，因而不会发生阻塞，操作系统要将任务从阻塞队列调到执行队列等等费事的工作，毕竟可能应用只想插入或者删除个数据，不想牵扯出来一大堆额外的开销。无锁机制的实现是基于CPU提供的一种原子操作CAS（Compare and Swap），CAS是这样工作的，它会原子地将一块内存的值与一个给定的值进行比较，如果它们相等就用新值替换返回true，否则什么也不做直接返回false。如下图所示：

![](/images/LQ1.png)

下面给出一个用链表实现的无锁队列

{% highlight c++ %}

#include <iostream>
#include <pthread.h>
#include <stdlib.h>

#include "PrintError.h"

class CLMutex;

CLMutex* g_pMutex;

#define CAS(ptr , oldvalue , newvalue) __sync_bool_compare_and_swap(ptr , oldvalue , newvalue)

template<typename T>
struct Node
{
    T* data;
    Node<T>* next;
};

template<typename T>
struct Queue
{
    Node<T>* head;
    Node<T>* tail;
};

template<typename T>
Queue<T>*
queueNew(void)
{
    Node<T>* tmp = new Node<T>;
    Queue<T>* queue = new Queue<T>;
    queue->head = queue->tail = tmp;
    return queue;
}

Queue<int>* myqueue;

template<typename T>
void
queuePush(Queue<T>* queue , T* data)
{
    Node<T>* onenode= new Node<T>;
    onenode->data = data;
    onenode->next = NULL;
    Node<T>* tail;
    Node<T>* next;

    while(true)
    {
        tail = queue->tail;
        next = tail->next;

        if(tail != queue->tail)
            continue;
        if(next != NULL)
        {
            CAS(&queue->tail , tail , next);
            continue;
        }
        if(CAS(&tail->next , next , onenode))
            break;
    }
    CAS(&queue->tail , tail , onenode);
}

template<typename T>
T*
queuePop(Queue<T>* queue)
{
    Node<T>* head;
    Node<T>* tail;
    Node<T>* next;
    T* data = NULL;
    while(true)
    {
        head = queue->head;
        tail = queue->tail;
        next = head->next;

        if(head != queue->head)
            continue;
        if(next == NULL)
            return NULL;
        if(head == tail)//prevent head exceed tail
        {
            CAS(&queue->tail , tail , next);
            continue;
        }
        data = next->data;
        if(CAS(&queue->head , head , next))
            break;
    }
    delete head;
    return data;
}

{%  endhighlight %}

CLMutex是对互斥锁进行的简单封装，用于测试代码，我们主要关注除它以外的其它代码。我首先进行了个宏定义，将CAS操作定义为__sync_bool_compare_and_swap,它是自gcc4.1.0开始支持的一种原子操作，不需要引入任何头文件就可以使用。

先分析队列的插入操作，即上面的函数queuePush。有一个事实需要先交代一下，对于基于链表实现的队列来说，插入操作涉及两个有序子操作：生成新节点并将老队尾指针的next域设成此节点和将此节点作为新队尾指针，相信这个大家都很容易理解。无锁数据结构有一个一般性的实现原则就是先把“该准备的都准备好，然后公布结果”。对应于这里，我们要先把节点生成出来，该填的字段填好，然后再进行关键的“公布”操作。“公布”操作之所以复杂是因为对于非原子性操作我们不知道什么时候会有其它线程抢占CPU执行，我们需要采取措施来保证非原子操作在并发环境中的一致性和正确性。废话少说，直接上代码，进入while循环，我们获取队列尾指针，也获取此尾指针的next域，然后判断刚刚获取的队尾指针是不是最新值，如果不是最新的，说明有别的线程已经完成了插入操作，我们不能在“过时”的队尾后插入数据，否则就会覆盖掉人家已经插入的节点，所以continue下面的步骤，重新获取队尾指针及其next域的最新值来进行插入操作。如果获取的是最新的队尾指针，要去判断其next域是否为空，为空，走正常流程，不为空，就更新队尾指针并重新来过。这里为什么要判断队尾指针next域是否为空需要简单说明一下，其实上面已经给出了答案，因为插入操作是由更新老队尾指针next域和更新队尾指针两个有序子操作完成的，即便队尾指针是最新的，但是说明不了没有其它线程已经更新了next，如果人家更新了next，当前线程也不能进行插入操作，道理同上，转而“助人为乐”，帮其更新队尾指针。过了这两关，说明你有利用新节点更新队尾指针next域的资格了，但事情还没算完，如果更新next域成功，那线程才可以去更新队尾指针完成整个插入操作，否则只能说明当前线程”人品“不好，陪人家过了前两关但还是不能进行插入，只好再重新来过。这就是基于链表实现的无锁队列的插入操作，很烦吧，没办法，这个世界是公平的，你提升了性能，减少了运行时开销，就必然要增加实现的逻辑难度。

我们继续说无锁队列的删除操作。队列的删除都是基于头部进行的，所以我们获取队列头指针及其next域，但这里为什么还要获取队尾指针呢？我稍后再解释。我们出队列操作都是基于head->next，而不是head本身，这样考虑是由于一个边界条件：如果队列只有一个元素，head和tail指针可能指向同一个节点，这样插入（进队列）操作和删除（出队列）操作本身就需要互斥了，通过引入一个dummy头指针来解决这个问题。如下图所示：

![](/images/LQ2.png)

进入while循环首先还是判断我们获取的head指针是不是最新值，不是最新值那就不用出队列操作，因为队首元素已经被别的线程抢先一步消费（基于生产者-消费者模型术语）并完成了出队列操作，重新来过之。之后判断head的next域是否为空，如果为空说明此刻队列没有元素，就直接返回NULL 。接下来判断head是不是和tail指针相等，因为插入操作时先更新tail的next域，再更新tail，会出现这样一种情况：我们出队列时，head和tail相等，但tail得next域已经得到更新，如果不进行此步判断，很可能head指针要超过tail指针，造成生产者还未生产，但消费者却已经完成消费的逻辑错误，这也是我们为什么还要获取一下tail指针的原因。所以如果head和tail相等，那就帮tail指针更新一下，忽略后续操作重新来过。不相等就接着获取待消费元素，更新队列头指针。如果更新失败，重新来过，成功就退出循环，回收队首元素内存，返回对首元素地址给调用者消费。但此程序的第94行存在一个著名的隐患，即ABA问题。到底什么是ABA问题，维基百科给了一个生动的解释：

![](/images/LQ3.png)

我就上述的无锁队列来说说我对ABA问题的理解，考虑这样的场景：

1. 线程T1拿到了队首元素的值并正好停在了程序第91行 。
2. 线程T2抢占CPU，成功执行了程序第91行的CAS操作，并且成功移除了与线程T1获取值时相同的队首元素，释放其内存。
3. 线程T2或者一个新来的线程进行入队列操作，这个内存分配操作返回的地址和步骤（2）释放的相同。
4. 队首指针最新值被某某线程更新成了步骤（3）新分配的节点地址
5. 线程T1姗姗来迟，它进行CAS操作，发现队首指针最新值和自己获取head指针一样，以为自己拿到的head就是当前最新值，其实只是地址一样，地址的内容已经今非昔比了，线程T1并不知情，继续更新队首指针并回收head，返回值给调用者，队首元素还没被消费就被意外地删除了，这就是ABA问题。

为解决ABA为题，我们可以采用具有原子性的内存引用计数等等办法。利用循环数组实现无锁队列可以解决ABA问题，因为循环数组不涉及内存的动态分配和回收，所以规避了ABA。程序如下：

{%  highlight c++ %}

#include <iostream>
#include <pthread.h>
#include <stdint.h>
#include <stdlib.h>

#include "PrintError.h"

#define CAS(ptr , oldvalue , newvalue) __sync_bool_compare_and_swap(ptr , oldvalue , newvalue)
#define FULL false
#define EMPTY false
#define DEFAULTSIZE 100

template<typename T , uint32_t arraysize = DEFAULTSIZE>
class CLArrayLockedFree
{
public:

    CLArrayLockedFree();
    bool push(T);
    T pop();

private:

    T m_Queue[arraysize];
    uint32_t m_CurrentWriteIndex;
    uint32_t m_CurrentReadIndex;
    uint32_t m_MaxReadIndex;

    inline uint32_t countToIndex(uint32_t);
};

template<typename T , uint32_t arraysize>
CLArrayLockedFree<T , arraysize>::CLArrayLockedFree()
{
    m_CurrentWriteIndex = 0;
    m_CurrentReadIndex = 0;
    m_MaxReadIndex = 0;
}


template<typename T , uint32_t arraysize>
inline uint32_t
CLArrayLockedFree<T , arraysize>::countToIndex(uint32_t count)
{
    return (count%arraysize);
}

template<typename T , uint32_t arraysize>
bool
CLArrayLockedFree<T , arraysize>::push(T element)
{
    uint32_t CurrentWriteIndex;
    uint32_t CurrentReadIndex;

    do
    {
        CurrentReadIndex = m_CurrentReadIndex;
        CurrentWriteIndex = m_CurrentWriteIndex;


        if(countToIndex(CurrentWriteIndex + 1) == countToIndex(CurrentReadIndex))
            return FULL;

        if(!CAS(&m_CurrentWriteIndex , CurrentWriteIndex , CurrentWriteIndex + 1))
            continue;

        m_Queue[countToIndex(CurrentWriteIndex)] = element;
        break;

    }while(1);

    while(!CAS(&m_MaxReadIndex , CurrentWriteIndex , CurrentWriteIndex + 1))
    {
        sched_yield();
    }

    return true;
}

template<typename T , uint32_t arraysize>
T
CLArrayLockedFree<T , arraysize>::pop()
{
    uint32_t CurrentReadIndex;
    uint32_t result;

    do
    {
        CurrentReadIndex = m_CurrentReadIndex;

        if(countToIndex(CurrentReadIndex) == countToIndex(m_MaxReadIndex))
            return EMPTY;

        result = m_Queue[countToIndex(CurrentReadIndex)];

        if(!CAS(&m_CurrentReadIndex , CurrentReadIndex , CurrentReadIndex + 1))
            continue;

        return result;

    }while(1);
}

{% endhighlight %}

先解释一下三个指针：

* WriteIndex:一个新元素即将插入的位置，即生产者即将生产的位置。
* ReadIndex:下一个可读取元素位置，消费者即将进行消费的位置。
* MaxReadIndex:相当于一个“哨兵”，读取指针ReadIndex只能落后于（因为是循环数组，不能说小于）该指针。MaxReadIndex可能等于也可能落后于WriteIndex，这是由于队列的插入操作存在预取的性质，对应程序的第64行，如果预取成功则赋给那个位置新元素，然后更新MaxReadIndex位置，将新元素发布出去，完成整个插入操作。上面的程序很简单，唯一需要解释一下的就是第74行，sched_yield函数的作用是让当前线程主动让出处理器，这么做的原因是生产者需要预留生产位置进行生产，因此在更新MaxReadIndex时我们需要按顺序更新，否则前面的预留位置没赋给新元素（相当于生产者还没在这个位置生产），但后面的预留位置被赋值就会导致MaxReadIndex被更新，让消费者误认为MaxReadIndex之前元素都已经被生产好，因此如果发现更新不是按顺序发生就放弃处理器给别的线程。如果还没理解可以参照下面示意图：

####一个线程只插入一个元素：

* 插入前状态：

![](/images/LQ4.png)

* 预留一个位置，移动WriteIndex位置：

![](/images/LQ5.png)

* 插入一个元素，移动MaxReadIndex发布出去

![](/images/LQ6.png)

####再举个插入的例子，两个线程各插入一个元素：

* 插入前状态：

![](/images/LQ7.png)

* 两个线程都预留出了生产位置

![](/images/LQ8.png)

* 一个线程完成了插入操作后：

![](/images/LQ9.png)

* 另一个线程也完成了插入操作：

![](/images/LQ10.png)

####删除元素：

* 删除前状态：

![](/images/LQ11.png)

* 某线程消费一个元素后：

![](/images/LQ12.png)

* 某线程又消费了一个元素后：

![](/images/LQ13.png)

这时，ReadIndex和MaxReadIndex相等了，说明队列为空，不能继续消费。