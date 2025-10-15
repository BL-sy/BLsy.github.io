---
title: 项目管理
published: 2025-10-13
description: "基于cherno Hazel引擎教学"
tags: ["学习", "游戏", "引擎","git","lua"]
category: 学习
draft: false
parentCollection: "game-engine"
---


This blog template is built with [Astro](https://astro.build/). For the things that are not mentioned in this guide, you may find the answers in the [Astro Docs](https://docs.astro.build/).


# 一.链接github远程仓库

- 在自己的github账号创建自己的仓库

- .gitignore

  在.git文件夹下新建.gitignore文件，可以声明一些不想提交到暂存区的文件

  ```txt
  # Binaries
  **/bin/
  bin-int/
  
  # Visual Studio files and folder
  .vs/
  **.sln
  **.vcxproj
  **.vcxproj.filters
  **.vcxproj.user
  **.csproj
  ```

  本地项目执行

  ```cmd
  git init
  git remote add origin <远程仓库URL>
  git add .
  git commit -m "first commit"
  git push origin main
  ```


- 一些git命令

  ```cmd
  git add *// 提交文件到暂存区
  git reset . // 将暂存区文件返回
  git status // 查看文件有无提交到暂存区状态
  git commit -m "注释"// 将暂存区的内容添加到仓库
  git push origin main // 将本地的分支版本上传到远程并合并
  git pull origin main // 拉取远程分支最新版本并合并
  git clone <远程仓库URL> <本地目录> // 克隆远程仓库
  ```

  ---------------------------------

# 二.Premake维护项目

## Premake是什么

Premake 是一个开源的**项目生成工具**，主要用于自动化生成各种开发环境的工程文件。
它通过编写 Lua 脚本（通常是 premake5.lua），描述项目结构、依赖、编译选项等内容，然后一键生成对应平台的项目文件。

## Why Premake

传统手动配置 Visual Studio 项目属性（如包含目录、宏定义、编译选项等）非常繁琐，尤其是多平台、多配置（Debug/Release/Dist）时。
Premake 通过 Lua 脚本一次性描述所有配置，能自动生成 VS 工程文件，大幅减少重复劳动,极大提升**开发效率**。

Premake 不仅能生成 Visual Studio 工程，还支持 Xcode、Makefile 等**多种平台**。只需维护一份脚本，就能适配不同开发环境，方便**团队协作和迁移**。

相比于Cmake，她更简单易用，语法更简洁。

## 创建项目的premake5

可以直接去[github](https://github.com/premake/premake-core)下载最新的premake5

将下载好的premake5文件放在根目录下的vendor/premake文件夹下,还有license

创建premake5.lua告诉premake5如何构建我们的项目

```lua
EngineName = "Hazel"

workspace (EngineName)
    architecture "x64" -- 平台
    startproject "Sandbox" -- 启动项目

    configurations {"Debug", "Release", "Dist"} -- 配置

outputdir = "%{cfg.buildcfg}-%{cfg.system}-%{cfg.architecture}"

project (EngineName)
    location (EngineName)
    kind "SharedLib" -- 配置类型
    language "C++"

    targetdir ("bin/" .. outputdir .. "/%{prj.name}") -- 输出目录
    objdir ("bin-int/" .. outputdir .. "/%{prj.name}") -- 中间文件目录

    files {
        "%{prj.name}/src/**.h",
        "%{prj.name}/src/**.cpp"
    }

    -- 包含目录
    includedirs {
        "%{prj.name}/vendor/spdlog/include",
        "%{prj.name}/src"
    }

    filter "system:windows"
        cppdialect "C++17"
        staticruntime "On"
        systemversion "latest"

        -- 预处理器
        defines {
            "HZ_PLATFORM_WINDOWS",
            "HZ_BUILD_DLL"
        }

        -- dll复制命令
        postbuildcommands {
            -- 复制DLL到Sandbox输出目录
            "{COPY} \"%{cfg.buildtarget.relpath}\" \"../bin/" .. outputdir .. "/Sandbox/\""
        }

    filter "configurations:Debug"
        defines "HZ_DEBUG"
        symbols "On"

    filter "configurations:Release"
        defines "HZ_RELEASE"
        optimize "On"

    filter "configurations:Dist"
        defines "HZ_DIST"
        optimize "On"

project "Sandbox"
    location "Sandbox"
    kind "ConsoleApp"
    language "C++"

    targetdir ("bin/" .. outputdir .. "/%{prj.name}") -- 输出目录
    objdir ("bin-int/" .. outputdir .. "/%{prj.name}") -- 中间文件目录

    files {
        "%{prj.name}/src/**.h",
        "%{prj.name}/src/**.cpp"
    }

    includedirs {
        "Hazel/src"
    }

    links { "Hazel" }

    filter "system:windows"
        cppdialect "C++17"
        staticruntime "On"
        systemversion "latest"
        defines { "HZ_PLATFORM_WINDOWS" }

    filter "configurations:Debug"
        defines "HZ_DEBUG"
        symbols "On"

    filter "configurations:Release"
        defines "HZ_RELEASE"
        optimize "On"

    filter "configurations:Dist"
        defines "HZ_DIST"
        optimize "On" 
```

我们再新建一个bat文件，这样就不需要每次打开控制台手动输入了

```bat
call vendor\bin\premake\premake5.exe vs2022
PAUSE
```

-----------------------------------------------------------------


# 三.预编译头

- **此节目的**

  由于项目中的头文件或者cpp文件都包含着c++的头文件，有些**重复**，可以将它们包含的c++头文件放在一个头文件内，这样不仅使代码**简洁**，而且预编译头可以**加快编译速度**。

### 如何实现

- src文件夹下创建hzpch类

  hzpch.h

  ```cpp
  #pragma once
  
  #include <iostream>
  #include <memory>
  #include <utility>
  #include <algorithm>
  #include <functional>
  
  #include <string>
  #include <sstream>
  #include <vector>
  #include <unordered_map>
  #include <unordered_set>
  
  #ifdef HZ_PLATFORM_WINDOWS
  #include <Windows.h>
  #endif
  ```

  hzpch.cpp

  ```cpp
  #include "hzpch.h"
  ```

- 修改premake

  ```lua
  	-- 预编译头 
  	pchheader "hzpch.h"
  	pchsource "Hazel/src/hzpch.cpp"
  ```

  **在每个cpp文件的顶部引入hzpch.h文件,不然会报错**

  重新生成后，premake预编译头设置的对应效果

  ![009](../assets/009.webp)

  ![010](../assets/010.webp)


# Tips

### Premake创建新项目

有了premake，我们可以写一个premake文件，以后每次需要新建项目，不需要每次都麻烦的配置属性了，只需要双击

```lua
projname = "NewProject"

workspace (projname)  -- 使用括号包裹变量
    architecture "x64"
    configurations { "Debug", "Release" }

outputdir = "%{cfg.buildcfg}-%{cfg.system}-%{cfg.architecture}"

project (projname) 
    location (projname) 
    kind "ConsoleApp"
    language "C++"
    
    -- 创建必要的目录结构
    os.mkdir(projname .. "/src")  -- 这里不能用%{prj.name}，不知道为什么
    
    targetdir ("bin/" .. outputdir .. "/%{prj.name}")
    objdir ("bin/intermediate/" .. outputdir .. "/%{prj.name}")

    files {
        "%{prj.name}/src/**.h",
        "%{prj.name}/src/**.cpp"
    }
    
    -- 如果src目录为空，创建默认main.cpp
    if #os.matchfiles("%{prj.name}/src/**.cpp") == 0 then
        local main_cpp = [[
#include <iostream>

int main() {
    std::cout << "Hello, NewProject!" << std::endl;
    return 0;
}
]]
        -- 使用直接路径
        local file = io.open(projname .. "/src/main.cpp", "w")
        file:write(main_cpp)
        file:close()
    end

    filter "system:windows"
        cppdialect "C++17"
        staticruntime "On"
        systemversion "10.0"
        defines { "WINDOWS_PLATFORM" }

    filter "configurations:Debug"
        defines { "DEBUG" }
        symbols "On"

    filter "configurations:Release"
        defines { "RELEASE" }
        optimize "On"
```

```bat
call premake5.exe vs2022
PAUSE
```

![004](../assets/004.webp)

放在同一级目录即可，保持简洁，点击bat文件就创建了新项目，可以在lua文件先修改项目名称

### C++知识：Function

1.Bind用法

```cpp
#include <iostream>
#include <functional>
using namespace std;
using namespace std::placeholders;// 占位符空间
void f(int a, int b, int c, int d, int e)
{
	cout << a << " " << b << " " << c << " " << d << " " << e << endl;
}
// _1 是在命名空间里的，bind可以翻转参数位置
int main(){
	int a = 1, b = 2, c = 3;
	auto g = bind(f, a, b, c, _2, _1);
	g(4, 5);	// 1 2 3 5 4
	return 0;
}
```

- 说明
  - bind可以用\_1,_2**预占位**，g可以理解是function<void(int a, int b, int c, int d, int e);function对象
  - auto g = bind(f, a, b, c, _2, _1);将f函数绑定到function对象g上，并定好第一、二、三个参数
  - g(4, 5)，将调用执行f函数，4将绑定到\_1上，5将绑定到\_2上，本来\_1实参会赋给f函数的d形参，_2实参给e形参，但由于bind时**改变**了对应位置
  - 于是\_1给e，_2给d，输出 1 2 3 5 4
