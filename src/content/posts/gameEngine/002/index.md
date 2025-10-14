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


# 入口点


我们希望在**游戏中而不是引擎中**决定建立的sandbox，并在**引擎中**实现

```cpp
Hazel::Application* Hazel::CreateApplication()
{
    return new Sandbox();
}
```

```cpp
#ifdef HZ_PLATFORM_WINDOW

extern Hazel::Application* Hazel::CreateApplication();
```

将CreateApplication函数声明为**extern**，表示此函数会在Hazel外部定义，接下来使用的这函数时将使用在外部定义的CreateApplication



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

  根据条件编译定义**HAZEL_API**是dll导入还是导出，可知Hazel项目将是__declspec(dllexport)**(导出作为dll的引擎Hazel)**，Sandbox项目是__declspec(dllimport)**(导入引擎dll)**

- 由于Sandbox#include <Hazel.h>，而Hazel项目的Hazel.h**包含**了Application.h，Application.h又包含了Core.h文件，


  ```cpp
  #include <Hazel.h>
  class Sandbox : public Hazel::Application
  {
  public:
  	Sandbox(){}
  	~Sandbox(){}
  };
  ```

  所以Sandbox项目也有**HAZEL_API**宏定义，且**是__declspec(dllimport)**
