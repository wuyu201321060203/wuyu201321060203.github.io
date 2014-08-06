---
layout: post
title: C++11特性学习之华容道求解
category: articles
tags: c++11
image:
    feature: so-simple-sample-image-5.jpg
comments: true
share: true
---
华容道游戏相信大家都有所耳闻，一种起始布局如下图1所示，游戏规则很简单：每次移动一个棋子，不能重叠也不能越过其它棋子，直至把曹操移到下图2中的位置。本文将使用程序并结合C++11的一些新特性去获得华容道问题解的最小值，即最少步数。

华容道问题求解算法很容易理解，就是一个广度优先搜索，步骤如下：

1. 将初始棋盘状态设为Initial，插入队列。
2. 判断队列是否为空，如果为空，说明无解并退出程序，否则从队首弹出一个棋局状态。如果此状态中曹操到达图2的位置，跳到步骤（4），否则对每一个棋子的每一个可以移动的方向移动棋子以产生新的棋局状态（注意此时可能产生多个新状态）。
3. 如果此状态与队列中已有的棋局状态等效（比如图1中的赵云和张飞换位置），则忽略此状态；否则将此状态插入队列，转向步骤（2）。
4. 问题得以求解，打印结果，程序结束。

说多无益，程序及注释如下图所示：
```
#include <assert.h>
#include <strings.h>
#include <cstring>
#include <type_traits>
#include <iostream>
#include <stdlib.h>
#include <list>
#include <stddef.h>
#include <string>
#include <vector>
#include <tuple>

#include <boost/unordered_set.hpp>
#include <boost/tr1/functional.hpp>
#include <boost/lambda/lambda.hpp>
#include <boost/functional/hash/hash.hpp>

const unsigned int g_cloumns = 4;
const unsigned int g_rows = 5;
const unsigned int g_BlockNum = 10;

enum class Shape
{
    Invalid,
    Single,
    Horizon,
    Vertical,
    Square,

};

enum class Direction
{
    Invalid,
    Up,
    Down,
    Left,
    Right,
};

static std::string const g_dirpresentation[5] =
{"Invalid" , "Up" , "Down" , "Left" , "Right"};

/*Mask的引入是为了去掉等效状态*/
struct Mask
{
    Mask();

    void generateMask(Shape , unsigned int , unsigned int);

    bool operator == (Mask const& rhs) const
    {
        return (bcmp(m_Box , rhs.m_Box , sizeof(m_Box)) == 0);
    }

    bool empty(unsigned int x , unsigned int y) const;

    size_t hashValue() const;

    unsigned int m_Box[g_rows][g_cloumns];
};

Mask::Mask()
{
    bzero((void*)&m_Box[0][0] , sizeof(m_Box));
}

void
Mask::generateMask(Shape shape , unsigned int top , unsigned int left)
{
    unsigned int value = static_cast<unsigned int>(shape);
    m_Box[top][left] = value;

    switch(shape)
    {

        case Shape::Horizon:

            m_Box[top][left + 1] = value;
            break;

        case Shape::Vertical:

            m_Box[top + 1][left] = value;
            break;

        case Shape::Square:

            m_Box[top][left + 1] = value;
            m_Box[top + 1][left] = value;
            m_Box[top + 1][left + 1] = value;
            break;

        default:

            break;
    }
}

bool
Mask::empty(unsigned int x , unsigned int y) const
{
    if(x >= 0 && x < g_rows && y >= 0 && y < g_cloumns)
        return (m_Box[x][y] == 0);

    else
        return false;
}

/*Mask的hasValue成员函数和特化版本的hash函数对象是为了能把Mask对象放入unordered_set而设计的*/
size_t
Mask::hashValue() const
{
    unsigned int const* begin = m_Box[0];
    return boost::hash_range(begin , begin + sizeof(m_Box));
}

namespace boost
{
    template<>
    struct hash<Mask>
    {
        size_t operator()(Mask const& rhs) const
        {
            return rhs.hashValue();
        }
    };
}

struct Block
{
    Block();
    Block(Shape , std::string , unsigned int , unsigned int);

    unsigned int bottom() const;
    unsigned int right() const;

    void complementMask(Mask* mask) const;

    Shape m_Shape;
    std::string m_Name;
    unsigned int m_Top;
    unsigned int m_Left;
};

Block::Block():m_Shape(Shape::Invalid) , m_Name("") , m_Top(-1) , m_Left(-1)
{}

Block::Block(Shape shape , std::string name , unsigned int top , unsigned int left)
    :m_Shape(shape) , m_Name(name) , m_Top(top) , m_Left(left)
{
    assert(m_Shape != Shape::Invalid);
    assert(m_Name != "");
    assert(m_Top >= 0 && m_Top < g_rows);
    assert(m_Left >= 0 && m_Left < g_cloumns);
}

unsigned int
Block::bottom() const
{
    static int const factor[5] = {0 , 0 , 0 , 1 , 1};
    assert(m_Shape != Shape::Invalid);
    return m_Top + factor[static_cast<int>(m_Shape)];
}

unsigned int
Block::right() const
{
    static int const factor[5] = {0 , 0 , 1 , 0 , 1};
    assert(m_Shape != Shape::Invalid);
    return m_Left + factor[static_cast<unsigned int>(m_Shape)];
}

void
Block::complementMask(Mask* mask) const
{
    mask->generateMask(m_Shape , m_Top , m_Left);
}

struct State
{

    State();

    Mask toMask() const;

    bool isSolved() const;

    template<typename FUNC>
    void move(FUNC const& func) const;

    Block m_BlockArray[g_BlockNum];

    unsigned int m_Step;

    mutable std::vector<std::tuple<std::string , Direction> > m_StepDiscription;
};

State::State():m_Step(0)
{}

Mask
State::toMask() const
{
    Mask tmp;

    for(int i = 0 ; i != g_BlockNum; ++ i)
    {
        m_BlockArray[i].complementMask(&tmp);
    }

    return tmp;
}

bool
State::isSolved() const
{
    Block tmp = m_BlockArray[1];
    assert(tmp.m_Shape == Shape::Square);

    return (tmp.m_Top == 3 && tmp.m_Left == 1);
}

template<typename FUNC>
void
State::move(FUNC const& func) const
{
    static_assert(std::is_convertible<FUNC ,
    boost::function<void (State const&)> >::value , "func is invalid");

    Mask mask = toMask();

    for(int i = 0 ; i != g_BlockNum ; ++i)
    {
        Block tmp = m_BlockArray[i];

        //move up
        if(tmp.m_Top > 0 && mask.empty(tmp.m_Top - 1 , tmp.m_Left)
            && mask.empty(tmp.m_Top - 1 , tmp.right()))
        {
            State next = *this;
            --next.m_BlockArray[i].m_Top;

            next.m_StepDiscription.push_back(
            std::make_tuple((next.m_BlockArray[i]).m_Name , Direction::Up));

            ++next.m_Step;
            func(next);
        }
        //move down
        if(tmp.bottom() < g_rows - 1 && mask.empty(tmp.bottom() + 1 , tmp.m_Left)
            && mask.empty(tmp.bottom() + 1 , tmp.right()))
        {
            State next = *this;
            ++next.m_BlockArray[i].m_Top;

            next.m_StepDiscription.push_back(
            std::make_tuple((next.m_BlockArray[i]).m_Name , Direction::Down));

            ++next.m_Step;
            func(next);
        }
        //move left
        if(tmp.m_Left > 0 && mask.empty(tmp.m_Top , tmp.m_Left - 1)
            && mask.empty(tmp.bottom() , tmp.m_Left - 1))
        {
            State next = *this;
            --next.m_BlockArray[i].m_Left;

            next.m_StepDiscription.push_back(
            std::make_tuple((next.m_BlockArray[i]).m_Name , Direction::Left));

            ++next.m_Step;
            func(next);
        }
        //move right
        if(tmp.right() < g_cloumns - 1 && mask.empty(tmp.m_Top , tmp.right() + 1)
            && mask.empty(tmp.bottom() , tmp.right() + 1))
        {
            State next = *this;
            ++next.m_BlockArray[i].m_Left;

            next.m_StepDiscription.push_back(
            std::make_tuple((next.m_BlockArray[i]).m_Name , Direction::Right));

            ++next.m_Step;
            func(next);
        }
    }
}

int main(void)
{
    boost::unordered_set<Mask> seen;
    std::list<State> queue;

    State initial;
    initial.m_BlockArray[0] = Block(Shape::Vertical , "ZhaoYun" , 0 , 0);
    initial.m_BlockArray[1] = Block(Shape::Square , "Caocao" , 2 , 1);
    initial.m_BlockArray[2] = Block(Shape::Vertical , "ZhangFei" , 0 , 1);
    initial.m_BlockArray[3] = Block(Shape::Vertical , "MaChao" , 2 , 0);
    initial.m_BlockArray[4] = Block(Shape::Horizon , "GuanYu" , 0 , 2);
    initial.m_BlockArray[5] = Block(Shape::Vertical , "HuangZhong" , 2 , 3);
    initial.m_BlockArray[6] = Block(Shape::Single , "Zu1" , 1 , 2);
    initial.m_BlockArray[7] = Block(Shape::Single , "Zu2" , 1 , 3);
    initial.m_BlockArray[8] = Block(Shape::Single , "Zu3" , 4 , 2);
    initial.m_BlockArray[9] = Block(Shape::Single , "Zu4" , 4 , 0);

    queue.push_back(initial);
    seen.insert(initial.toMask());

    while(!queue.empty())
    {
        State cur = queue.front();
        queue.pop_front();

        if(cur.isSolved())
        {
            std::cout << "Bingo" << std::endl;
            std::cout <<"step: " << cur.m_Step << std::endl;

            for (auto i : cur.m_StepDiscription)
            {
                std::string name;
                Direction direct;

                std::tie(name , direct) = i;
                std::cout << "<" << name << " , " <<
                        g_dirpresentation[static_cast<unsigned int>(direct)] << ">" << std::endl;
            }

            break;
        }

        else if(cur.m_Step >= 200)
        {
            std::cout << "too many steps" << std::endl;
            break;
        }

        cur.move([&seen , &queue](State const& rhs){

            auto result = seen.insert(rhs.toMask());

            if(result.second)
                queue.push_back(rhs);
        });
    }

    return 0;
}
```