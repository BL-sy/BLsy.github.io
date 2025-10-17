---
title: 日志
published: 2025-10-14
description: "基于cherno Hazel引擎教学"
tags: ["学习", "游戏", "引擎"]
category: 学习
draft: false
parentCollection: "game-engine"
---


This blog template is built with [Astro](https://astro.build/). For the things that are not mentioned in this guide, you may find the answers in the [Astro Docs](https://docs.astro.build/).


# 日志系统

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
