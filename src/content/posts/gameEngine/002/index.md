---
title: 入口点
published: 2025-10-13
description: "基于cherno Hazel引擎教学"
tags: ["学习", "游戏", "引擎"]
category: 学习
draft: false
parentCollection: "game-engine"
---


This blog template is built with [Astro](https://astro.build/). For the things that are not mentioned in this guide, you may find the answers in the [Astro Docs](https://docs.astro.build/).



# 什么是入口点

在编写应用程序时，应用程序总是会有一个入口点，本质上的意思就是应用程序要从**哪里开始运行**，其实就是main()函数。

我们已经在Sandbox应用程序项目中写了一个入口点：

```cpp
void main() {
	Hazel::Print();
}
```

但在Game中写入口点其实并不是很好的做法，应用程序虽然说依赖于引擎，但是它对引擎的启动细节并不感兴趣。
而且引擎的启动细节还与平台有关。所以引擎的入口点最好还是由引擎负责定义，由引擎负责提供给应用程序。

在Hazel中建立引擎Application

```cpp
$Hazel/Application.h

namespace Hazel {
	class Application
	{
		Application();
		virtual ~Application();

		void Run();
	};
}
```

```cpp
$Hazel/Application.cpp

#include "Application.h"
namespace Hazel
{
	Application::Application()
	{
	}

	Application::~Application()
	{
	}

	void Application::Run()
	{
		while (true);
	}
}
```


# 动态链接库DLL(Hazel)的导入导出

这里使用到 **__declspec(dllimport)** 和 **__declspec(dllexport)**

### 为什么要使用这两个关键字呢？

在C++编程中，__declspec(dllimport) 是一个特殊的关键字，用于从动态链接库（DLL）中导入类、函数和数据，用于**声明哪些函数和变量是从dll导出的**。
与之相对的是 __declspec(dllexport)，它用于导出DLL中的相应项，告诉编译器这个符号在外部dll定义。
这两个关键字在创建DLL和使用DLL时都非常重要，它们确保了函数和数据可以在不同的模块之间共享。

### 宏定义处理

我们先在Hazel中新建一个Hazel文件夹，存放核心文件，在这个文件夹中新建一个Core.h。
我们将在Core中宏定义一个**HAZEL_API**，用来区分是导入还是导出

- 在Core.h中

  ```cpp
  #pragma once
  #ifdef HZ_PLATFORM_WINDOWS
	#ifdef HZ_BUILD_DLL
  		#define HAZEL_API __declspec(dllexport)
	#else
  		#define HAZEL_API __declspec(dllimport)
	#endif
  #else
	#error Hazel only supports Windows!
  #endif
  ```

  根据条件编译定义**HAZEL_API**，Hazel项目将是__declspec(dllexport)**(导出作为dll的引擎Hazel)**，Sandbox项目是__declspec(dllimport)**(导入引擎dll)**

  去Hazel项目预处理器中定义**HZ_BUILD_DLL**宏,Sandbox项目中不定义;当然还要定义**HZ_PLATFORM_WINDOWS**宏

- Application.h中,引用一下Core.h
  ```cpp
  #include "Core.h"

  namespace Hazel {
	  class HAZEL_API Application
	  {
		  Application();
		  virtual ~Application();

		  void Run();
	  };
  }
  ```

现在我们可以在游戏应用Sandbox中使用Hazel::Application了，但是#include"Hazel/Application.h"让人不知所云，我们希望直接包含Hazel.h
在Hazel中新建一个Hazel.h，作为只被游戏项目包含的头文件

```cpp
$Hazel.h

// For use by Hazel applications
#include "Hazel/Application.h"
```

为了在游戏文件SandboxApp直接应用Hazel.h,我们在Sandbox项目的属性中添加$(SolutionDir)Hazel/src的包含目录

我们来写一个实际的应用程序

  ```cpp
  #include <Hazel.h>
  class Sandbox : public Hazel::Application
  {
  public:
  	Sandbox(){}
  	~Sandbox(){}
  };

  void main()
  {
	Hazel::Application* app = new Sandbox();
	app->Run();
	delete app;
  }
  ```

# 建立入口点

我们在Hazel中建立一个入口点EntryPoint.h，将SandboxApp的main函数转移

Hazel 引擎本身不知道你的具体应用类型（比如 Sandbox），所以通过声明一个 CreateApplication 工厂函数，让**客户端去实现它**，返回自己的 Application 派生类实例。
在 Hazel 的入口文件(EntryPoint.h)中，只需调用 CreateApplication()，无需关心具体实现细节。这样 Hazel 引擎可以通用，客户端只需实现自己的 CreateApplication。

我们需要一个创建应用程序的方法，建立一个函数CreateApplication()，这个函数由外部应用程序实现，所以要使用关键字extern

```cpp
$Hazel/EntryPoint.h

#ifdef HZ_PLATFORM_WINDOWS

extern Hazel::Application* Hazel::CreateApplication();
int main(int argc, char** argv) {
	auto app = Hazel::CreateApplication();
	app->Run();
	delete app;
}

#endif
```

argc 和 argv 让你的程序可以接收命令行参数，方便后续开发和维护。

在Application声明CreateApplication方法，在Sandbox中实现

```cpp
$Hazel/Application.h

namespace Hazel {
	// To be defined in CLIENT
	Application* CreateApplication();
}
```
```cpp
$Sandbox/SandboxApp.cpp

Hazel::Application* Hazel::CreateApplication() 
{
	return new Sandbox();
}
```

在Hazel.h中包含EntryPoint.h
```cpp
$Hazel.h

#pragma once
// For use by Hazel applications

#include "Hazel/Application.h"

// -----EntryPoint----------------------
#include "Hazel/EntryPoint.h"
// ---------------------------------------
```
