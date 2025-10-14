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
  ```

# 二.Premake维护项目

由于之前配置VS项目各项属性需要根据不同平台**手动**一个一个设置，很麻烦，缺乏灵活性。

用[lua脚本](https://so.csdn.net/so/search?q=lua脚本&spm=1001.2101.3001.7020)配置项目属性，使用premake运行程序**一键生成**VS项目及属性，更灵活简便

```lua
EngineName = "Hazel" -- 引擎名称

workspace (EngineName)
    architecture "x64"
    startproject "Sandbox"  -- 启动项目

    configurations 
    {
        "Debug",
        "Release",
        "Dist"  -- 发行版本
    }

outputdir = "%{cfg.buildcfg}-%{cfg.system}-%{cfg.architecture}"

project (EngineName)
    location (EngineName)
    kind "SharedLib"  -- Dll
    language "C++"

    -- 创建必要的目录结构
    os.mkdir(EngineName .. "/src/Hazel")

    targetdir ("bin/" .. outputdir .. "/%{prj.name}")
    objdir ("bin/intermediate/" .. outputdir .. "/%{prj.name}")

    files {
        EngineName .. "/src/**.h",
        EngineName .. "/src/**.cpp"
    }

    includedirs 
    {
        "%{prj.name}/vendor/spdlog/include"
    }

    filter "system:windows"
        cppdialect "C++17"
        staticruntime "On"
        systemversion "10.0"

        defines -- 宏
        {
            "HZ_PLATFORM_WINDOWS",
            "HZ_BUILD_DLL"
        }
        buildoptions   -- 命令行
        { 
            "/utf-8" 
        }

        -- 使用绝对路径和正斜杠
        postbuildcommands  -- 后处理
        {
            "{COPY} %{cfg.buildtarget.relpath} \"" .. "%{wks.location}bin/" .. outputdir .. "/Sandbox" .. "\""
        }

    filter "configurations:Debug"
        defines "HZ_DEBUG"
        symbols "On"
        -- 禁用 LTO 并启用增量链接
        linktimeoptimization "Off"
        flags { "MultiProcessorCompile" }
        disablewarnings { "4503" }  -- 禁用装饰名过长警告

    filter "configurations:Release"
        defines "HZ_RELEASE"
        optimize "Speed"  -- 明确优化速度
        -- 启用 LTO 和优化
        linktimeoptimization "On"
        linkoptions { "/OPT:REF", "/OPT:ICF" }

    filter "configurations:Dist"
        defines "HZ_DIST"
        optimize "Full"
        linktimeoptimization "On"


project "Sandbox"
    location "Sandbox"
    kind "ConsoleApp"
    language "C++"

    -- 创建必要的目录结构
    os.mkdir("Sandbox/src")

    targetdir ("bin/" .. outputdir .. "/%{prj.name}")
    objdir ("bin/intermediate/" .. outputdir .. "/%{prj.name}")

    files 
    {
        "%{prj.name}/src/**.h",
        "%{prj.name}/src/**.cpp"
    }

    includedirs 
    {
        "Hazel/vendor/spdlog/include",
        "Hazel/src"
    }

    links 
    {
        "Hazel"
    }

    filter "system:windows"
        cppdialect "C++17"
        staticruntime "On"
        systemversion "10.0"
        defines 
        {
            "HZ_PLATFORM_WINDOWS"
        }
        buildoptions { "/utf-8" }

    filter "configurations:Debug"
        defines "HZ_DEBUG"
        symbols "On"
        -- 启用调试构建优化
        flags { "MultiProcessorCompile" }

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

# 三.日志系统

这里暂时使用spdlog

https://github.com/gabime/spdlog.git

### 本地添加spdlog

这里直接使用cherno远程git仓库的文件,在希望的文件目录下打开控制台,输入

```cmd
  git submodule add https://github.com/gabime/spdlog Hazel/vendor/spdlog
```

添加附加目录,引用

我们这里进行一下封装,日后可以写自己的日志记录系统

```cpp
$log.h
#include <memory>
#include "Core.h"
#include spdlog/spdlog.h"
namespace Hazel {
	class HAZEL_API Log
	{
	public:
		static void Init();

		inline static std::shared_ptr<spdlog::logger>& GetCoreLogger() { return s_CoreLogger; }
		inline static std::shared_ptr<spdlog::logger>& GetClientLogger() { return s_ClientLogger; }

	private:
		//两种日志记录器
		//引擎日志和客户端日志
		static std::shared_ptr<spdlog::logger> s_CoreLogger;
		static std::shared_ptr<spdlog::logger> s_ClientLogger;
	};
}
```

```cpp
$log.cpp
#include "Log.h"
#include "spdlog/sinks/stdout_color_sinks.h"

namespace Hazel {
	std::shared_ptr<spdlog::logger> Log::s_CoreLogger;
	std::shared_ptr<spdlog::logger> Log::s_ClientLogger;

	void Log::Init()
	{
		spdlog::set_pattern("%^[%T] %n: %v%$");//日志格式 --- [时间戳]日志名称:内容

		s_CoreLogger = spdlog::stdout_color_mt("HAZEL");//命名
		s_CoreLogger->set_level(spdlog::level::trace);

		s_ClientLogger = spdlog::stdout_color_mt("APP");
		s_ClientLogger->set_level(spdlog::level::trace);
	}
}
```

**注**：这里直接使用会报错，需要在属性的命令行界面添加/utf-8

![005](../assets/005.webp)

日志信息的宏定义，方便记录和观察

```cpp
$log.h
// Core log macros
#define HZ_CORE_TRACE(...)		::Hazel::Log::GetCoreLogger()->trace(__VA_ARGS__)
#define HZ_CORE_INFO(...)		::Hazel::Log::GetCoreLogger()->info(__VA_ARGS__)
#define HZ_CORE_WARN(...)		::Hazel::Log::GetCoreLogger()->warn(__VA_ARGS__)
#define HZ_CORE_ERROR(...)		::Hazel::Log::GetCoreLogger()->error(__VA_ARGS__)
#define HZ_CORE_FATAL(...)		::Hazel::Log::GetCoreLogger()->fatal(__VA_ARGS__)
								
//Client log macros				
#define HZ_TRACE(...)			::Hazel::Log::GetClientLogger()->trace(__VA_ARGS__)
#define HZ_INFO(...)			::Hazel::Log::GetClientLogger()->info(__VA_ARGS__)
#define HZ_WARN(...)			::Hazel::Log::GetClientLogger()->warn(__VA_ARGS__)
#define HZ_ERROR(...)			::Hazel::Log::GetClientLogger()->error(__VA_ARGS__)
#define HZ_FATAL(...)			::Hazel::Log::GetClientLogger()->fatal(__VA_ARGS__)
  ```

我们的入口点现在：

```cpp
#ifdef HZ_PLATFORM_WINDOWS
extern Hazel::Application* Hazel::CreateApplication();

int main(int argc,char** argv)
{
    Hazel::Log::Init();
    HZ_CORE_WARN(Initialized Log!);
    int a = 5;
    HZ_INFO(“Hello!Var={0}",a);

    auto app = Hazel:CreateApplication();
    app->Run()
    delete app;
}
#endif
```
运行结果：

![006](../assets/006.webp)

不同级别的报告有不同的颜色.Done!(for now)


# 四.预编译头

- **此节目的**

  由于项目中的[头文件](https://so.csdn.net/so/search?q=头文件&spm=1001.2101.3001.7020)或者cpp文件都包含着c++的头文件，有些**重复**，可以将它们包含的c++头文件放在一个头文件内，这样不仅使代码**简洁**，而且[预编译](https://so.csdn.net/so/search?q=预编译&spm=1001.2101.3001.7020)头可以**加快编译速度**。

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
  project "Hazel"		--Hazel项目
  	location "Hazel"--在sln所属文件夹下的Hazel文件夹
  	kind "SharedLib"--dll动态库
  	language "C++"
  	targetdir ("bin/" .. outputdir .. "/%{prj.name}") -- 输出目录
  	objdir ("bin-int/" .. outputdir .. "/%{prj.name}")-- 中间目录
  
  	-- 预编译头 
  	pchheader "hzpch.h"
  	pchsource "Hazel/src/hzpch.cpp"
  ```

- 在每个cpp文件的顶部引入hzpch.h文件,不然会报错

- 重新生成后，premake预编译头设置的对应效果

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
