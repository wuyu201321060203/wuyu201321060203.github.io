---
layout: post
title:  Autotools入门使用教程
category: projects
tags: LIRS Cache algorithm c++
image:
    feature: head3.jpg
author:
    name:   WuYu
    avatar: bio-photo-alt.jpg
comments: true
share: true
---

#### 1. 什么是Autotools

The GNU build system, also known as the Autotools, is a suite of programming tools designed to assist in making source code packages portable to many Unix-like systems.------Wikipedia

#### 2. Autotools基本工作流程

Autotools的基本工作流程如下图所示：

![](/images/autotools1.png)

* 先autoscan扫描一下工程根目录，这一步会生成一个基本的配置脚本configure.scan，里面主要是一些m4(Macro)宏定义，这些定义都是告诉autoconf工具如何找到你需要的系统配置参数。
*  将configure.scan重命名为configure.ac，修改里面的内容，修改工程包名字，版本，bug报告邮箱，添加你想要的检查项（这些检查项都用预定义的m4宏表达即可），添加你工程需要的编译选项等等，具体可以查看示例。[^1]



*  运行aclocal命令，这一步生成一个aclocal.m4的文件，这个文件含有一些工具自动帮你生成的系统检查项
*  运行autoconf生成configure脚本
*  编写Makefile.am文件，它是一个makefile更高层次的抽象，描述了你想要生成工程需要编译的源文件，自定义头文件路径，需要链接的动态库，可执行文件生成到哪个路径下等
*  运行automake生成Makefile.in文件
*  有了configure脚本就可以利用Makefile.in生成makefile，之后相信程序员都知道该如何去编译和部署工程了。

#### 3. 一些有用的工具和示例

为了帮助大家在使用Autotools无需记住上述稍微有点繁琐的步骤，这里可以利用聂小文同学编写的小工具来帮你自动执行一些无脑的工作，但是最关键的修改configure.ac和修改Makefile.am操作还是需要身体力行，不过这里我将给出一个configure.ac和Makefile.am的示例写法足以你应付大多数情况。

#### 4. 小工具功能实现的主要脚本代码

{% highlight python %}

#!/usr/local/bin/python

##
# create a empty project in current directory
# with autoconf tools
#
import os
import sys
import re

## declaring of functions
def mkdir(dir):
	"""make a directory if not exists"""
	if not os.path.exists(dir):
		os.makedirs(dir)


# check the input arguments
if len(sys.argv) < 4:
	print 'Usage: createEmpytProject ProjectName Version Email.'
	exit(1)

project = sys.argv[1]
version = sys.argv[2]
email = sys.argv[3]

# make the directories of project
mkdir(project)
os.chdir(project)
mkdir('include')
mkdir('include/misc')
mkdir('src')
mkdir('src/misc')
mkdir('test')
mkdir('scripts')
mkdir('build')
mkdir('doc')
mkdir('m4')

# generate configure.ac
print("Generating configure.ac...")
os.system('autoscan')
file = open("configure.scan")
ac = file.read()
file.close()

append = "AC_INIT([" + project + "], [" + version + "], [" + email + "])\n\
AC_CONFIG_SRCDIR([])\n\
AM_INIT_AUTOMAKE(" + project + ", " + version + ")\n\
AC_CONFIG_HEADERS([config.h])\n\
AC_CONFIG_MACRO_DIR([m4])"
pattern = "AC_INIT\(\[FULL-PACKAGE-NAME\], \[VERSION\], \[BUG-REPORT-ADDRESS\]\)"
ac = re.sub(pattern, append, ac)

append = "# Checks for programs.\n\
AC_PROG_CC(clang llvm-gcc gcc)\n\
AC_PROG_CXX(clang++ llvm-g++ g++)\n\
AC_PROG_CPP\n\
AC_PROG_AWK\n\
AC_PROG_INSTALL\n\
AC_PROG_LN_S\n\
AC_PROG_MAKE_SET\n\
LT_INIT\n\
AC_PROG_LIBTOOL\n\
AC_ARG_ENABLE(debug, [  --enable-debug\tEnable DEBUG output. ],\n\
\t[ CXXFLAGS=\"-O0 -DDEBUG -Wall -Werror\" ],\n\
\t[ CXXFLAGS=\"-O3 -Wall -Werror\" ])"
ac = re.sub("# Checks for programs.", append, ac)

append = "AC_CONFIG_FILES([Makefile src/Makefile test/Makefile])\n\
AC_OUTPUT"
ac = re.sub("AC_OUTPUT", append, ac)

file = open("configure.ac", 'w')
file.write(ac)
file.close()
os.system('rm autoscan*.log')
os.system('rm configure.scan')
del file
del append

# generate automake.am
print("Generating Makefile.am...")
am = open('Makefile.am', 'w')
am.write('ACLOCAL_AMFLAGS = -I m4\nSUBDIRS = src test\nEXTRA_DIST = include doc')
am.close()

os.chdir('test')
am = open('Makefile.am', 'w')
am.write('INCLUDES = -I$(top_srcdir)/include -I$(top_srcdir)/src\n' +
		 'include_HEADERS = suites.h LogTest.h ExceptionTest.h\n' +
		 'noinst_PROGRAMS = ../bin/test\n' +
		 '___bin_test_SOURCES = test.cc LogTest.cc ExceptionTest.cc\n' +
		 '___bin_test_LDADD = ../src/lib' + project + '.la\n' +
		 '___bin_test_LDFLAGS = -L/usr/local/lib -lboost_unit_test_framework')
am.close()
os.system('cp ~/reposit/test.cc .')
os.system('cp ~/reposit/suites.h .')
os.system('cp ~/reposit/LogTest.h .')
os.system('cp ~/reposit/LogTest.cc .')
os.system('cp ~/reposit/ExceptionTest.h .')
os.system('cp ~/reposit/ExceptionTest.cc .')

os.chdir('../include/misc')
os.system('cp ~/reposit/Log.h .')
os.system('cp ~/reposit/Exception.h .')
os.chdir('../../src/misc')
os.system('cp ~/reposit/Log.cc .')

os.chdir('..')
am = open('Makefile.am', 'w')
am.write('INCLUDES = -I$(top_srcdir)/include\n' +
		 'include_HEADERS = \n' +
		 '\n#\n' +
		 '# lib' + project + '.la\n' +
		 '#\n' +
		 'lib_LTLIBRARIES = lib' + project + '.la\n' +
		 'lib' + project + '_la_SOURCES = ./misc/Log.cc')
am.close()
os.chdir('..')

# copy integrated test framework
print('Copying integrated test framework...')
os.chdir('scripts')
os.system('cp ~/reposit/itframe.py .')
os.system('cp ~/reposit/test.py .')
f = open('test.py', 'r')
kk = f.read()
f.close()
kkk = re.sub('\?\?\?', project, kk)
f = open('test.py', 'w')
f.write(str(kkk))
f.close()
os.chdir('..')
del f
del kk
del kkk

# copy document templates
print('Copying document templates...')
os.chdir('doc')
os.system('cp ~/reposit/requirements.dot .')
os.system('cp ~/reposit/architecture.dot .')
os.chdir('..')

# append files
print('Appending some files...')
os.system('touch NEWS')
os.system('touch README')
os.system('touch AUTHORS')
os.system('touch COPYING')
os.system('touch INSTALL')
os.system('touch ChangeLog')

# configure
print('Configure...')
os.system('libtoolize')
os.system('aclocal')
os.system('autoheader')
os.system('autoconf')
os.system('automake --add-missing')

# build
print('Building...')
os.chdir('build')
os.system('../configure --enable-debug')
os.system('make')

{% endhighlight %}

下面分别是示例configure.ac，Makefile.am,小工具使用教程的网址，它们都挂在了我的github的相应项目里。

* 示例configure.ac [^2]

* 示例Makefile.am [^3]

* 工具使用教程 [^4]

*
