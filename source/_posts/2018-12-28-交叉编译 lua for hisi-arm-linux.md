---
layout: post
title: 交叉编译 lua for hisi-arm-linux
date: 2018-12-28 09:32:24.000000000 +09:00
categories: lua
tags: [交叉编译, lua, 海思, arm]
abstract: 最近被借到其他部门做开发, 由于不可抗力因素，项目宿主机器由x64 Ubuntu Server转移到某arm 裸kernel上。为降低机器负载、统一技术栈和其他不可抗力因素后端调度器用C++11重构原有JAVA工程，由C++ with lua实现决策支持引擎。
---

<!-- toc -->

> 最近被借到其他部门做开发, 由于不可抗力因素，项目宿主机器由x64 Ubuntu Server转移到某arm 裸kernel上。为降低机器负载、统一技术栈和其他不可抗力因素后端调度器用C++11重构原有JAVA工程，由C++ with lua实现决策支持引擎。

## 二、**交叉编译lua5.3（ubunt18.10）**

下载安装交叉编译工具(一般没什么问题，如有问题查看对应交叉编译工具文档) 
    
    
    cd aarch64-himix100-linux
    sudo bash aarch64-himix100-linux.install


​    

下载lua5.3源代码,如官网描述，命令行执行下载解压缩 
    
    
    curl -R -O http://www.lua.org/ftp/lua-5.3.5.tar.gz
    tar zxf lua-5.3.5.tar.gz
    cd lua-5.3.5/src

Lua交互式环境中具备查阅历史命令功能，该部分功能由readline实现, 我们在arm仅作为C++扩展脚本使用，因此不需readline.h。 修改源代码屏蔽该库(不然还得交叉编译readline), 我这儿是添加了一个宏用来交叉编译时去掉readline.h, 在lauconf.h 60行添加如下代码 
    
    
    #if defined(LUA_USE_ARM)
    #define LUA_USE_POSIX
    #define LUA_USE_DLOPEN      /* needs an extra library: -ldl */
    // 去掉下面这句
    /*#define LUA_USE_READLINE*/    /* needs some extra libraries */
    #endif


修改src/Makefile, 在108（其他位置也行）行添加如下代码, -DLUA_USE_ARM表示lauconf.h这一段启动. 
    
    
    arm:
         $(MAKE) $(ALL) SYSCFLAGS="-DLUA_USE_ARM" SYSLIBS="-Wl,-E -ldl"


修改Makefile第9行，将gcc替换为hisi编译工具 
    
    
    #CC= gcc -std=gnu99
    修改为
    CC= aarch64-himix100-linux-gcc -std=gnu99


现在就可以进行编译了, 直接在src目录下执行命令. 
    
    
    make arm

如果出现loadlocale.c相关错误,添加环境变量后重新编译 
    
    
    export LC_ALL=C

根据根目录下Makefile可得，抽取部分核心文件:*.h头文件和liblua.a静态链接库 
    
    
    42 TO_BIN= lua luac
    43 TO_INC= lua.h luaconf.h lualib.h lauxlib.h lua.hpp
    44 TO_LIB= liblua.a
    45 TO_MAN= lua.1 luac.1

现在Lua交叉编译完成，可以把上述文件直接copy到hisi-arm-linux上运行使用。 

## 三、**C++ with lua test**

编写C++&&lua测试代码, 并使用交叉编译工具编译后传到hisi-arm-linux上运行. 
    
    
    #include <iostream>
    #include <string.h>  
    extern "C"
    {
      #include "lua-5.3.5/src/lua.h"  
      #include "lua-5.3.5/src/lauxlib.h"  
      #include "lua-5.3.5/src/lualib.h"  
    }	
    
    int main() {  
      int a, b;
      while (std::cin >> a >> b) {
        lua_State *L = luaL_newstate(); 
        luaopen_base(L);
        luaL_openlibs(L);
        luaL_dofile(L, "event.lua");
        lua_getglobal(L, "event");
        lua_pushnumber(L, a);
        lua_pushnumber(L, b);
        lua_call(L, 2, 0);
    
      }
        return 0; 
    }


​    
​    
​    aarch64-himix100-linux-g++  lua.cpp -L lua-5.3.5/src -static -llua -ldl -lm  -o lua_ext
​    
    -lm是链接math库（lua使用）
    -L指定链接库位置（交叉编译的arm平台lua库）


lua代码 
    
    
    function event(a, b)
      if a + b < 10 then
        print("welcome to C++ with lua regular")
      else	
        print("welcome to C++ with lua regular")
      end;
      print("a + b", a + b)
      print("a * b", a * b)
      print("a ^ b", a ^ b)
      print("a / b", a / b)
    end;

 



## 结束语

作为C++忠实粉丝，强烈抵制某些场景上直接上C++这种重炮, 改换语言能直接提高30%性能的都是自身设计问题. 

lua: 我是一直小小小小鸟～   

ref 
- [http://www.lua.org/download.html](http://www.lua.org/download.html)
- [https://blog.csdn.net/themagickeyjianan/article/details/75301960](https://blog.csdn.net/themagickeyjianan/article/details/75301960) 
- [https://www.cnblogs.com/wajika/p/6592659.html](https://www.cnblogs.com/wajika/p/6592659.html)

