---
layout: post
title:  Boyer-Moore字符串匹配算法分析与实现
category: articles
tags: string-match algorithm c++
image:
    feature: so-simple-sample-image-3.jpg
author:
    name:   WuYu
    avatar: bio-photo-alt.jpg
comments: true
share: true
---

各种文本编辑器的“查找”功能（Ctrl + F），大多采用Boyer-Moore字符串匹配算法，它不仅效率高，并且构思奇妙，容易理解。1977年，德克萨斯大学的Robert S. Boyer教授和J Strother Moore教授发明了这种算法。
首先，先阐述一下此算法的核心思想（算法讲解部分源自阮一峰的网络日志 [^1]

[^1]: <http://www.ruanyifeng.com/blog/2013/05/boyer-moore_string_search_algorithm.html>

1. [!All text](/images/BM1.png)
文本字符串为"HERE IS A SIMPLE EXAMPLE"，待搜索的字符串为"EXAMPLE"
2. [!All text](/images/BM2.png)
首先将待搜索字符串的头部与文本对齐，然后从尾部开始比较，发现尾部字符都不匹配（"S"和"E"），那么说明此时文本对应的字符串肯定不是我们想要的，可以直接将"EXAMPLE"字符串向后移动，那么移动的步数就是算法要解决的核心问题，这时可以按照所谓的“坏字符规则” ，比如上图中的情况，"S"是坏字符，它对应待搜索字符串中的位移是6（按照位移来算，"EXAMPLE"的 第一个字符"E"位移为0，以此类推），如果在待搜索字符串中坏字符"S"在前面还出现过，则移动位数为：坏字符位置-其在搜索串中的上一次出现位置。如果没有，则移动位数为：坏字符位置-（-1）。所以"坏字符"规则为：
      后移步数 = 坏字符位置 - 其在搜索串中上一次出现位置
      比如上图，"S"在"EXAMPLE"中没有出现，那么我们应该移动6-(-1)=7步。如下图：
3. [!All text](/images/BM3.png)
此时"P"是坏字符，按照坏字符规则，"EXAMPLE"向后移动6-4=2步
4. [!All text](/images/BM4.png)
依然从尾部开始匹配，发现"MPLE"都匹配成功，坏字符为"I"，按照坏字符规则搜索串应该向后移动2-（-1）=3步，但是此时有没有更好的移动方法呢？这就引出了所谓的“好后缀规则”。好后缀规则应该说包括3条规则：
   （1）“好后缀”的位置以最后一个字符为准。假定"ABCDEF"中"EF"是个好后缀，则其位置为"F"的位置，即为5
   （2）如果“好后缀”在搜索串中不只出现过一次，那么移动步数为：好后缀位置 - 其在搜索串中上一次出现的位置。
   （3）如果"好后缀"在搜索串中只出现过一次，并且除最长的那个好后缀外（本身除外），如果有“好后缀”的“好后缀”（本文叫好后缀子缀）出现在头部（注意必须只能出现在头部），则移动位数为：好后缀位置 -  其好后缀子缀位置。如果没有好后缀子缀，则移动位数为：好后缀位置 - （-1）
      所以如果按照上述的“好后缀”规则，则移动步数应该为6-0=6步，因为好后缀"PLE"的好后缀子缀"E"出现在了"EXAMPLE"头部。Boyer-Moore算法指出每次待搜索串移动步数应该为坏字符规则和好后缀规则得出的移动步数的较大值，这保证了此算法能够又快又准确地进行字符串匹配。样例中按照好后缀规则移动6步后如下图：
5. [!All text](/images/BM5.png)
继续匹配发现出现坏字符"P"，按照“坏字符”规则移动步数为6-4=2步，如下图：
6. [!All text](/images/BM6.png)
这样就完成了最终的匹配。应当指出，Boyer-Moore算法之所以得到普遍使用，最重要的原因是其搜索需要求解的移动步数只与待搜索字符串有关，这样我们就能通过提前建立一个“坏字符规则-步数映射表” 和“好后缀规则-步数映射表” ，当进行匹配的时候直接查表就可以获得结果。下面实现中就建立了“好后缀规则-步数映射表”，但没有建立“坏字符规则-步数映射表”，因为我觉得对于验证性的非工程项目所需的程序完全没有必要建立一个完整的“坏字符规则-步数映射表”，只需要根据坏字符每次求解一下移动步数返回给调用程序即可。

程序设计如下：

{% highlight c++ %}
/*main.cpp*/
#include "CLStrMatching.h"
#include "PrintError.h"

int main(int argc , char**argv)
{
    if(argc != 3)
    {
        PrintError(AT , "argc is invalid");
        exit(0);
    }
    CLStrMatching* temp = new CLStrMatching(argv[1] , argv[2]);
    temp->startMatch();
    delete temp;
    return 0;
}
/*CLStrMatching.h*/

#ifndef STRMATCHING_H
#define STRMATCHING_H

#include <string>

#include "CLWRuleProducer.h"

#define MAX(x , y) ((x>y)?(x):(y))

class CLStrMatching
{

public:

    CLStrMatching(std::string , std::string);
    ~CLStrMatching();
    int readFile();
    int startMatch();

private:

    std::string m_Context;
    std::string m_SearchStr;
    const std::string m_FilePath;
    const unsigned int m_SeaStrTailOffset;
    unsigned int m_StrCurOffset;//for search word
    unsigned int m_ContextCurOffset;
    const unsigned int m_SeaStrHeadOffset;

    CLWRuleProducer* m_pWRuleProducer;
};

#endif
/*CLStrMatching.cpp*/

#include <fstream>
#include <iostream>

#include "CLStrMatching.h"
#include "PrintError.h"

CLStrMatching::CLStrMatching(std::string search , std::string filepath):m_SearchStr(search)
, m_FilePath(filepath) , m_SeaStrTailOffset(search.length() - 1) ,
m_StrCurOffset(m_SeaStrTailOffset) ,m_ContextCurOffset(m_StrCurOffset)
, m_SeaStrHeadOffset(0) , m_pWRuleProducer(new CLWRuleProducer(m_SearchStr))
{

}

CLStrMatching::~CLStrMatching()
{
    if(m_pWRuleProducer != 0)
        delete m_pWRuleProducer;
}


int CLStrMatching::readFile()
{
    std::fstream filein;
    filein.open(m_FilePath.c_str());
    if(!filein.is_open())
    {
        PrintError(AT , "open file failed");
        filein.clear();
        exit(0);
    }
    std::getline(filein , m_Context);
    filein.close();
    return 0;
}

int CLStrMatching::startMatch()
{
    readFile();
    m_pWRuleProducer->init();

    int initial = -1;//just initialize initial var with an impossible value

    bool flag = true;//flag stands for if we have find an offset in context for
    //a given string or not . True:have found one , False:not have found

    while(m_ContextCurOffset <= m_Context.length() - 1)
    {
        initial = m_ContextCurOffset;
        while(m_StrCurOffset >= m_SeaStrHeadOffset)
        {
            if(m_SearchStr[m_StrCurOffset] == m_Context[m_ContextCurOffset])
            {
                if(m_StrCurOffset == 0)
                    break;
                --m_StrCurOffset;
                --m_ContextCurOffset;
            }
            else
            {
                int tmp = m_pWRuleProducer->pBadCharRule(m_Context[m_ContextCurOffset]
                                            , m_StrCurOffset);
                int tmp1 = -1;

                if(m_StrCurOffset < m_SeaStrTailOffset)//m_StrCurOffset + 1 is
                    //start of Good Suffix , so if m_StrCurOffset is equal to
                    //m_SeaStrTailOffset, it means we can only use bad char rule

                    tmp1 = (m_pWRuleProducer->m_GoodSuffixRule)[m_StrCurOffset + 1];
                int step = MAX(tmp , tmp1);
                m_ContextCurOffset = initial + step;
                m_StrCurOffset = m_SeaStrTailOffset;
                flag = false;
                break;
            }
        }
        if(flag)
        {
            std::cout << (m_ContextCurOffset + 1) << std::endl; //offset begins
            //with zero , bur offset in result needs to begin with 1 , so every
            //offset of result add by 1

            m_ContextCurOffset = (initial +
                        (m_pWRuleProducer->m_GoodSuffixRule)[m_StrCurOffset]);
            m_StrCurOffset = m_SeaStrTailOffset;
        }

        flag = true;
    }
    return 0;
}
/*CLWRuleProducer.h*/

#ifndef WORDRYLE
#define WORDRYLE

#include <string>
#include <utility>

#include <boost/unordered_map.hpp>

class CLStrMatching;

using boost::unordered_map;

class CLWRuleProducer
{

public:

    CLWRuleProducer(std::string word);
    void init();
    int pBadCharRule(char , unsigned int);
    int pGoodSuffixRule();

    friend class CLStrMatching;

private:

    std::string m_InputWord;
    const int m_DefalutMoveStep;
    const unsigned int m_WordLengthOffset;//search-word's length -1
    const unsigned int m_GoodSuffixTail; //search-word's length -1
    unordered_map<unsigned int , int> m_GoodSuffixRule;

    int hasSubSuffixAhead(unsigned int);
    int doHasSubSuffixAhead(unsigned int);
    int isNotOnlySuffix(unsigned int);

};

#endif
/*CLWRuleProducer.cpp*/

#include <iostream>

#include "CLWRuleProducer.h"
#include "PrintError.h"

CLWRuleProducer::CLWRuleProducer(std::string rhs):m_InputWord(rhs) , m_DefalutMoveStep(-1)
 , m_WordLengthOffset(rhs.length() - 1) , m_GoodSuffixTail(m_WordLengthOffset)
{}

void CLWRuleProducer::init()
{
    pGoodSuffixRule();
}


/*
 * @Function Name : pBadCharRule
 *
 * @Description : This function use bad char and the current search-word offset
 * to produce the step the search- word need to move forward and
 * continue to match
 *
 * @para1 : the bad char in the context which don't match the search-word
 *
 * @para2 : the current offset of the search-word
 *
 * @return : the step which search-word need to move forward
 *
 */
int CLWRuleProducer::pBadCharRule(char const badchar , unsigned int currentoffset)
{
    for(int ptr = currentoffset - 1 ; ptr >=0 ; --ptr)
    {
        if(m_InputWord[ptr] == badchar)
            return (currentoffset - ptr);
    }

    return (currentoffset - m_DefalutMoveStep);
}
/*
 * @Function Name : pGoodSuffixRule
 *
 * @Description : To produce complete Good Suffix Rule table
 *
 * @para : none
 *
 * @return : just a flag which stands for success , no meaningful
 *
 */
int CLWRuleProducer::pGoodSuffixRule()
{
    int tmp = -1;
    int tmp1 = -1;
    for(unsigned int ptr =0 ; ptr != m_WordLengthOffset + 1; ++ptr)
    {
        tmp = isNotOnlySuffix(ptr);
        if(tmp == -1)
        {
            tmp1 = hasSubSuffixAhead(ptr);

            m_GoodSuffixRule[ptr] = m_GoodSuffixTail - tmp1;
        }
        else
            m_GoodSuffixRule[ptr] = m_GoodSuffixTail - tmp;
    }
    return 0;
}

/*
 * @Function Name : hasSubSuffixAhead
 *
 * @Description : look for if there is a sub-goodsuffix of good suffix in the
 * beginning of the search-word
 *
 * @para1 : the beginning offset of good suffix
 *
 * @return : if there is , return the the sub-goodsuffix's last character's offset
 * in the search-word or return -1
 *
 */
int CLWRuleProducer::hasSubSuffixAhead(unsigned int suffixbegin)
{
    if(suffixbegin > m_WordLengthOffset)
    {
        PrintError(AT , "suffixbegin is too big in hasSubSuffixAhead function");
        exit(0);
    }

    unsigned int ptr = m_WordLengthOffset;
    int result = m_DefalutMoveStep;

    while(ptr > suffixbegin)
    {
        result = doHasSubSuffixAhead(ptr);
        if(result == -1)
            --ptr;
        else
            break;
    }
    return result;
}

int CLWRuleProducer::doHasSubSuffixAhead(unsigned int offset)//the excute function
    //for hasSubSuffixAhead function

{
    std::string search = m_InputWord.substr(offset , std::string::npos);
    int tmp = static_cast<int>(m_InputWord.find(search));
    if(tmp == 0)
        return (search.length() - 1);
    else
        return -1;
}

int CLWRuleProducer::isNotOnlySuffix(unsigned int suffixbegin)
{
    if(suffixbegin > m_WordLengthOffset)
    {
        PrintError(AT , "suffixbegin is invalid");
        exit(0);
    }
    std::string search = m_InputWord.substr(suffixbegin , std::string::npos);
    int start = -1;
    if(suffixbegin != 0)
        start = static_cast<int>(m_InputWord.rfind(search , m_GoodSuffixTail - search.length()));
    if(start == -1)
        return -1;
    else
        return (start + search.length() - 1);
}
/*PrintError.h*/

#ifndef ERROR_H_INCLUDED
#define ERROR_H_INCLUDED

#include <stdio.h>

#define STRINGIFY(x) #x
#define TOSTRING(x) STRINGIFY(x)
#define AT __FILE__ ":" TOSTRING(__LINE__)

void PrintError(const char* , const char*);

#endif
/*PrintError.cpp*/

#include "PrintError.h"

void PrintError(const char* location,const char* errormessage)
{
    fprintf(stdout,"Error at %s ; Error Discription:%s\n",location , errormessage);
}

{% endhighlight %}

测试如下：

(1)测试文本testfile内容：
[!All text](/images/BM7.png)
(2)待搜索字符串：
EXAMPLE
(3)测试结果：
[!All text](/images/BM8.png)