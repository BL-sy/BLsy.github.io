---
title: 窗口抽象与GLFW窗口
published: 2025-10-20
description: "基于cherno Hazel引擎教学"
tags: ["学习", "游戏", "引擎"]
category: 学习
draft: false
---


This blog template is built with [Astro](https://astro.build/). For the things that are not mentioned in this guide, you may find the answers in the [Astro Docs](https://docs.astro.build/).


# GLFW

我们的游戏引擎需要一个窗口，这里我们使用GLFW库

```cmd
git submodule add https://github.com/TheCherno/glfw.git Hazel/vendor/GLFW
```

# 项目设置(Premake5)

游戏引擎需要链接到GLFW库，这里直接在Hazel项目的premake5.lua中添加GLFW库的引用

```lua
workspace Hazel

-- 包含相对于根目录的其他项目
IncludeDir = {}
IncludeDir["GLFW"] = "Hazel/vendor/GLFW/include"

-- 将GLFW premake5.lua包含进来,相当于把内容拷贝在这里面
include "Hazel/vendor/GLFW"

project Hazel
    -- 链接库      
    links {
        "GLFW",
        "opengl32.lib"
    }
```

--------------------------------

# 窗口类

我们的游戏引擎有一个窗口，
Application类可以调用创建窗口函数，而窗口类使用glfw库创建窗口。窗口类检测glfw窗口的事件，并**回调**给Application的处理事件函数。

我们希望为**不同的平台**做独立的窗口适配，因此我们需要一个**窗口基类**，不同平台的窗口类继承自这个基类

窗口基类

```cpp
$Window.h

#include "hzpch.h"

#include "Hazel/Core.h"
#include "Hazel/Event/Event.h"

namespace Hazel {
	// 窗口属性结构体
	struct WindowProps
	{
		std::string Title;
		unsigned int Width;
		unsigned int Height;

		WindowProps(const std::string& title = "Hazel Engine",
					unsigned int width = 1280,
					unsigned int height = 720)
			: Title(title), Width(width), Height(height)
		{}
	};
	// 基于平台的窗口抽象类
	class HAZEL_API Window
	{
	public:
		// 事件回调函数类型定义
		// 使用std::function封装一个接受Event引用的函数
		using EventCallbackFn = std::function<void(Event&)>;

		virtual ~Window() {}
		virtual void OnUpdate() = 0;

		virtual unsigned int GetWidth() const = 0;
		virtual unsigned int GetHeight() const = 0;

		// 设置事件回调函数
		// 窗口属性
		virtual void SetEventCallback(const EventCallbackFn& callback) = 0;
		virtual void SetVSync(bool enabled) = 0;
		virtual bool IsVSync() const = 0;

		static Window* Create(const WindowProps& props = WindowProps());
	};
}
```

我们为Windows平台实现一个窗口类，继承自Window基类

```cpp
$Platform/WindowsWindow

#include "hzpch.h"

#include "Hazel/Core.h"
#include "Hazel/Event/Event.h"

namespace Hazel {
	// 窗口属性结构体
	struct WindowProps
	{
		std::string Title;
		unsigned int Width;
		unsigned int Height;

		WindowProps(const std::string& title = "Hazel Engine",
					unsigned int width = 1280,
					unsigned int height = 720)
			: Title(title), Width(width), Height(height)
		{}
	};
	// 基于平台的窗口抽象类
	class HAZEL_API Window
	{
	public:
		// 事件回调函数类型定义
		// 使用std::function封装一个接受Event引用的函数
		using EventCallbackFn = std::function<void(Event&)>;

		virtual ~Window() {}
		virtual void OnUpdate() = 0;

		virtual unsigned int GetWidth() const = 0;
		virtual unsigned int GetHeight() const = 0;

		// 设置事件回调函数
		// 窗口属性
		virtual void SetEventCallback(const EventCallbackFn& callback) = 0;
		virtual void SetVSync(bool enabled) = 0;
		virtual bool IsVSync() const = 0;

		static Window* Create(const WindowProps& props = WindowProps());
	};
}
```


```cpp
$WindowsWindow.cpp

#include "hzpch.h"
#include "WindowsWindow.h"

namespace Hazel {

	static bool s_GLFWInitialized = false;

	// 在WindowsWindow子类定义在Window父类声明的函数
	Window* Window::Create(const WindowProps& props) 
	{
		return new WindowsWindow(props);
	}

	WindowsWindow::WindowsWindow(const WindowProps& props) 
	{
		Init(props);
	}

	// Windows窗口销毁代码
	WindowsWindow::~WindowsWindow() 
	{
		Shutdown();
	}

	// Windows窗口初始化代码
	void WindowsWindow::Init(const WindowProps& props) 
	{
		m_Data.Title = props.Title;
		m_Data.Width = props.Width;
		m_Data.Height = props.Height;

		HZ_CORE_INFO("Create window {0} ({1}, {2})", props.Title, props.Width, props.Height);

		if (!s_GLFWInitialized)
		{

			int success = glfwInit();
			HZ_CORE_ASSERT(success, "Could not initialize GLFW!");

			s_GLFWInitialized = true;
		}

		// 创建窗口//////////////////////////////////////////////
		m_Window = glfwCreateWindow((int)props.Width, (int)props.Height, m_Data.Title.c_str(), nullptr, nullptr);
		// 设置glfw当前的上下文
		glfwMakeContextCurrent(m_Window);
		///////////////////////////////////////////////////////
		// 设置窗口用户指针为窗口数据结构体的地址
		// 这样可以在回调函数中访问窗口数据
		glfwSetWindowUserPointer(m_Window, &m_Data);// 设置窗口回调函数代码
		SetVSync(true);// 默认启用垂直同步
	}

	void WindowsWindow::OnUpdate() 
	{
		// Windows窗口更新代码
		glfwPollEvents();// 处理窗口事件,轮询事件
		glfwSwapBuffers(m_Window);// 交换前后缓冲区
	}

	void WindowsWindow::SetVSync(bool enabled) {
		// 设置垂直同步的代码
		if(enabled)
			glfwSwapInterval(1);
		else
			glfwSwapInterval(0);

		m_Data.VSync = enabled;
	}
	bool WindowsWindow::IsVSync() const {
		// 返回垂直同步状态的代码
		return m_Data.VSync;
	}

	void WindowsWindow::Shutdown() {
		glfwDestroyWindow(m_Window);
	}
}	
```

**HZ_CORE_ASSERT**

```cpp
$Core.h

#ifdef HZ_ENABLE_ASSERTS
#define HZ_ASSERT(x, ...) { if(!(x)) { HZ_ERROR("Assertion Failed: {0}", __VA_ARGS__); __debugbreak(); } }
#define HZ_CORE_ASSERT(x, ...) { if(!(x)) { HZ_CORE_ERROR("Assertion Failed: {0}", __VA_ARGS__); __debugbreak(); } }
#else
#define HZ_ASSERT(x, ...)
#define HZ_CORE_ASSERT(x, ...)
#endif
```

----------------------------------------

# 构建一个GLFW窗口

```cpp
$Application.h

class HAZEL_API Application
{
private:
	std::unique_ptr<Window> m_Window;
	bool m_Running = true;
};
```

```
$Application.cpp

void Application::Run()
{
	while (m_Running)
	{
		// 背景颜色
		glClearColor(1, 0, 1, 1);
		glClear(GL_COLOR_BUFFER_BIT);
		m_Window->OnUpdate();
	}
}
```