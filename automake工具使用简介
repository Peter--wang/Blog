[toc]
automake工具使用简介
=
编译内核
-
```
obj-m += hellomod.o
CURRENT_PATH := $(shell pwd)
LINUX_KERNEL := $(shell uname -r)
LINUX_KERNEL_PATH := /usr/src/kernels/$(LINUX_KERNEL)
all:
        make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) modules
clean:
        make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) clean
```

制作automake以及rpm包的过程


 
 
autoconf、automake以及rpm包的制作过程
 
第一步：
 
在root下建立目录hello-cxf-1.0,然后在该目录下新建子目录src和doc（doc几乎存放一些文档，但在这里暂时为空）。
``` 
    mkdir hello-cxf-1.0
    cd hello-cxf-1.0
    mkdir src
    mkdir doc
```

第二步：
在src目录下编辑文件main.c

```
#cd src
#vi main.c
#include <stdio.h>
 int main(void)
 {
         printf("this is hello-cxf-1.0 testing!\n");
         return 0;
 }
```
第三步：
 
    回到hello-cxf-1.0目录下，编辑configure.ac（或者叫做configure.in）和Makefile.am文件。
 
configure.ac的例子：
``` 
AC_PREREG(2.59)
 
#AC_INIT(FULL-PACKAGE-NAME, VERSION,BUG-REPORT-ADDRESS)
AC_INIT(hello-cxf,1.0)
AM_INIT_AUTOMAKE([foreign])
AC_CONFIG_SRCDIR([src/main.c])
AC_CONFIG_HEADER([src/config.h])
  
#Checks for programs.
AC_PROG_CC
#Checks for libraries.
 
#Checks for header files.
#Checks for typedefs,structures,and compiler characteristics.
#Checks for library functions.
 
AC_CONFIG_FILES([Makefile
                 src/Makefile
                 doc/Makefile
                ])
AC_OUTPUT
``` 
其中，
 
1.可以通过运行 autoscan（生成configure.ac的模板文件） , 自动创建两个文件： autoscan.log configure.scan，修改configure.scan的文件名为configure.in或者configure.ac，
 
2.最简单的configure.ac必须包含的AC_INIT (package, version, [bug-report], [tarname])、AC_CONFIG_FILES([files])、AC_OUTPUT三个宏，有用到automake的话，则必须添加AM_INIT_AUTOMAKE和AC_PROG_CC宏。automake一般是要用到的，因为生成的configure会根据Makefile.in生成Makefile文件，而Makefile.in可以有automake生成。
 
Makefile.am文件要在当前目录，子目录都个建一份，如下：
 
Hello-cxf-1.0目录下Makefile.am的例子：
``` 
AUTOMAKE_OPTIONS=foreign
 
SUBDIRS = src doc
``` 
Src目录下Makefile.am的例子：
``` 
bin_PROGRAMS = hello-cxf
 
hello_cxf_SOURCES = main.c
  
#hello_cxf_LDADD = -lpthread
 
#hello_cxf_debug_LDADD= -lpthread-debug
 
#SUBDIRS = lib

```
Doc目录下的Makefile.am是个空的。
 
注意：
 
Makefile.am就是一系列变量的定义。Makefile的语法：变量名 ＝ 变量值。
 
Makefile.am的编写
 
1.    需要生成Makefile的目录下都要编写一个Makefile.am
 
2.    顶层（项目）目录的Makefile.am,可能会包含AUTOMAKE_OPTIONS = foreign 1.4这样一句。
 
3.    如果当前目录有其它子目录，子目录中也要生成makefile，则要包含
SUBDIRS = 子目录…，例如：SUBDIRS = src doc
 
4.    对于不需要安装，但是需要一起打包发布的文件，则在当前目录的Makefile.am中可以添加一句：EXTRA_DIST = 发布的文件，例如：EXTRA_DIST = purge.conf purge
 
5.    Makefile.am中的变量名，有统一的命名机制，一般是：安装目录_主变量。
 
6.    安装目录。
有些标准的安装目录可以直接使用。例：bin_PROGRAMS = hello，表示hello将被安装在$(prefix)/bin目录下。但是这些标准安装目录可能不够使用，因此我们可以自己定义所需要的安装目录。安装目录一般是省略’dir’的，例如：
htmldir = $(prefix)/html
html_DATA = automake.html
就表示automake.html将被安装在$(prefix)/html目录下。这里DATA是主变量，稍后说明。
对于不安装的文件，变量名的目录变量可写为noinst。
 
7.    主变量。常见的主变量有：`PROGRAMS'、`LIBRARIES'、 `LISP'、`SCRIPTS'、`DATA'、 `HEADERS'、`MANS'和`TEXINFOS'。这些主变量说明了生成的对象类型。例如，主变量PROGRAMS 保存了需要被编译和连接的程序的列表。
 
8.    Makefile.am的例子。
目录结构：
```
  purge-1.2
   |―·doc
   |   |－purge.conf
   |   |－Makefile.am
   |－·src
   |   |－purge.h
   |   |－thread.h
   |   |－…
   |   |－list.h
   |   |－debug.c
   |   |－fun.c  
   |   |－list.c
   |   |－main.c
   |   |－map.c
   |   |－parseconf.c
   |   |－purge.c
   |   |－thread.c
   |   |－Makefile.am
   |－hello-0.1.0.spec
   |－configure.ac
   |－Makefile.am
```
各目录下的Makefile.am文件可以编写如下：
purge-1.2目录:vi Makefile.am
```
AUTOMAKE_OPTIONS = foreign 1.4
SUBDIRS = src doc
```
doc目录:vi Makefile.am
```
EXTRA_DIST = purge.conf
```
src目录:vi Makefile.am
```
bin_PROGRAMS = purge
purge_SOURCES = main.c purge.c fun.c list.c \
                parseconf.c thread.c debug.c map.c
purge_LDADD = -lpthread
```
说明：
 
a)      在Makefile.am也可以定义变量如：
ALL_SOURCE = main.c purge.c fun.c list.c parseconf.c thread.c debug.c map.c
purge_SOURCES = $( ALL_SOURCE)
 
b)      名字中除了字母、数字和下划线之外的所有字符都将用下划线代替。例如，如果你的程序被命名为sniff-glue，那么派生出的变量名将是sniff_glue_SOURCES，而不是sniff-glue_SOURCES；库liblob.a的’_SOURCES’变量对应的变量名为’liblob_a_SOURCES’，而不是’liblob.a_SOURCES’。
 
9.    注：共享库必须被安装，所以不允许使用 `noinst_LTLIBRARIES'和`check_LTLIBRARIES'。
 
10.  Makefile.am中configure输出变量的使用。
在configure.ac中通过宏AC_SUBST引出的变量在Makefile.am中就能被使用到。
例子：
在configure.ac中定义如下：
```
if test -z "$CACHE_ICP_PORT"; then
       CACHE_ICP_PORT="3130"
fi
AC_SUBST(CACHE_ICP_PORT)
```
在Makefile.am中通过@CACHE_ICP_PORT@格式就能引用变量CACHE_ICP_PORT。
例子：DEFAULT_ICP_PORT        = @CACHE_ICP_PORT@
 
第四步：
 
运行 aclocal, 它根据configure.ac或者configure.in生成一个“aclocal.m4”文件和一个缓冲文件夹autom4te.cache，该文件主要处理本地的宏定义。
 
在hello-cxf-1.0目录下运行aclocal。
```
#aclocal
```
第五步：
 
运行 autoconf, 根据configure.ac和aclocal.m4生成configure脚本。
```
#autoconf
```
第六步：
 
运行 autoheader，它负责生成config.h.in文件。该工具通常会从“acconfig.h”文件中复制用户附加的符号定义。即autoheader根据configure.ac，运行m4，生成config.h.in（该文件名由AC_CONFIG_HEADER([src/config.h])的定义而定）
``` 
#autoheader
```
第七步：
 
使用automake根据Makefile.am和aclocal.m4生成Makefile.in文件，在这里使用选项“—adding-missing”可以让automake自动添加有一些必需的脚本文件，如depcomp，install-sh, missing等。
``` 
#automake –adding-missing
``` 
第八步：
 
运行./configure根据makefile.in和config.h.in（如果有的话）生成makefile和config.h（如果有config.h.in）文件，及config.status,config.log用于记录检测到的一些状态。即通过运行自动配置设置文件configure，把Makefile.in变成了最终的Makefile。
``` 
#./configure
```
其中，autoreconf 相当于连续执行aclocal autoconf autoheader automake --add-missing。
 
第九步：
 
运行 make，对配置文件Makefile进行测试一下。
``` 
#make
#ls
#ls src/
```
可以看到，在src文件下面生成了main.c的输出文件hello-cxf。
 
第十步：
 
运行生成的文件hello-cxf
 
\#./src/hello-cxf
 
this is hello-cxf-1.0 testing!
 
**注意：**
 
Makefile文件
 
自动生成的Makefile文件常用几个命令如下：
make all - 生成程序、库、文档等，等同于make，根据Makefile编译原始码，连接，生成目标文件，可执行文件。
make install - 安装 
make install-strip -安装，去除调试符号 
make uninstall - 卸载 
make clean - 清除临时文件，make all 的反过程，即清除上次的make命令所产生的object文件（后缀为“.o”的文件）及可执行文件。
 
 make dist - 创建发布包：PACKAGE-VERSION.tar.gz.
 
 
 
 
 
rpm 打包过程
- 
make：根据Makefile编译原始码，连接，生成目标文件，可执行文件。
 
第一步：
 
创建发布包，运行make dist命令
```
#make dist
```
// 生成hello-cxf-1.0.tar.gz
 
注：make dist
　　产生发布软件包文件（即distribution package）。这个命令将会将可执行文件及相关文件打包成一个tar.gz压缩的文件用来作为发布软件的软件包。
　　他会在当前目录下生成一个名字类似“PACKAGE-VERSION.tar.gz”的文件。PACKAGE和VERSION，是我们在configure.in中定义的AM_INIT_AUTOMAKE(PACKAGE, VERSION)。
 
第二步：
```
#make distcheck
```
注意：
 
make distcheck-生成发布软件包并对其进行测试检查，以确定发布包的正确性。这个操作将自动把压缩包文件解开，然后执行configure命令，并且执行make，来确认编译不出现错误，最后提示你软件包已准备好，能发布了。
 
make distclean-类似make clean，但同时也将configure生成的文件全部删除掉，包括Makefile。
 
第三步：
 
我用的是as4.0，首先查看一下/usr/src/redhat/目录下面是否有BUILD  RPMS  SOURCES  SPECS  SRPMS这些子目录，如果没有，则创建。
 
把上一步生成的hello-cxf-1.0.tar.gz包复制到/usr/src/redhat/SOURCES/目录下：
```
#cp hello-cxf-1.0.tar.gz /usr/src/redhat/SOURCES/
```
第四步：
 
编辑将应用程序打包（package）必须的配置文件spec文件。
 
我的hello-cxf-1.0.spec配置文件为：
```
%define _name hello-cxf
%define _ver  1.0
%define _rel 1.0
Summary:It's a hello-cxf program
Name: %{_name}
Version: 1.0
Release: %{_rel}
License: Copyright
Group:Amusements/Games
Source: %{_name}-%{_ver}.tar.gz
BuildRoot: /var/tmp/hello-cxf-1.0-root
%description
print Hello-cxf world

%prep

%setup -q

#tar zxf %{_name}-%{_ver}.tar.gz 

%build
 
./configure
 
make

%install
 
rm -rf %{buildroot}
 
make DESTDIR="$RPM_BUILD_ROOT" install
%post
echo "OK, Hello is already installed for you!"
%postun
echo "OK, Hello is already uninstalled for you!"
echo "Thanks for using!"
%clean
rm -rf $RPM_DUILD_ROOT
%files
 
%defattr(-,root,root)
 
/usr/local/bin/hello-cxf

%changelog
```
第五步：
```
#rpmbuild -ba hello-cxf-1.0.spec
```
rpmbuild读入spec文件上的配置信息，自动生成rpm包。
 
第六步：
 
安装：rpm -ivh name-version-release.architecture.rpm (由刚rpmbuild -bb ×××.spec生成的rpm安装包)
 
可以先查看一下再/usr/src/redhat/RPMS/i386目录下是否已经生成了rpm包。
```
# cd /usr/src/redhat/RPMS/
 
```
 
athlon  i386  i486  i586  i686  noarch
```
# ls i386/
```
hello-cxf-1.0-1.0.i386.rpm  hello-cxf-debuginfo-1.0-1.0.i386.rpm
```
# rpm –ivh hello-cxf-1.0-1.0.i386.rpm
```
第七步：
 
运行#hello-cxf即可
```
# hello-cxf
 
this is hello-cxf-1.0 testing!
```

注意：
 
rpm文件命名规则：name-version-release.architecture.rpm
name指软件名，version软件版本，release发布版本，architecture表示该rpm包适用的平台（指cpu），典型的有：src, noarch, i386, i686,ppc64,x86_64,ia64,sparc64。其中src, noarch这两种适用各种平台，i386, i686（32位），x86_64(64位)，这三种比较常见。
 
rpm常用的命令：
安装：rpm -ivh name-version-release.architecture.rpm
升级：rpm –Uvh name-version-release.architecture.rpm
卸载：rpm -e name ；rpm -e name-version ；rpm -e name-version-release
版本查询：rpm -q name
其它命令：
    rpm -qi name  详细信息查询
 
    rpm -qpl name-version-release.architecture.rpm  rpm文件列表(该rpm包将安装的文件)
 
    rpm -qa 已经安装的所有rpm包。
 
rpm -ivh --nodeps name-version-release.architecture.rpm 中-nodeps选项可以忽略对其它rpm软件包的依赖。
    rpm -qf filename查看该文件属于哪个rpm包。
rpmbuid常用命令
 
    rpmbuild -bb|-bs|-ba ×××.spec 

rpmbuild -tb|-ts|-ta ×××.tar.gz        //spec文件在tar.gz中
 
-bb 和 -tb打包成二进制包
 
-bs 和 -ts 打包成源码包
 
-ba 和 -ta打包成二进制包和源码包
