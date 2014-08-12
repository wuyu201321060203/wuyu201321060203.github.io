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

{% endhighlight %}

下面分别是示例configure.ac，Makefile.am,小工具使用教程的网址，它们都挂在了我的github的相应项目里。

* 示例configure.ac [^2]

* 示例Makefile.am [^3]

* 工具使用教程 [^4]

* 
