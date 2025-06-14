---
title: OpenGL环境搭建与第一个窗口
tags:
  - OpenGL
---

## GLFW
所用库为`GLFW`：GLFW是一个专门针对OpenGL的C语言库，它提供了一些渲染物体所需的最低限度的接口。

### 构建GLFW
下载地址[这里](https://www.glfw.org/download.html)

下载后的源码包，只需要**编译后的lib库**和**include文件夹**

### 搭建构建环境，cmake配置

构建系统选择CMake，虽然可以通过cmake-gui生成ide的工程文件，但个人目前主用`emacs`，且平台为linux，因此暂时采用手写`CMakeLists.txt` 来编译.

整体目录结构如下：
```shell
.
├── build
├── CMakeLists.txt
├── compile_commands.json -> ./build/compile_commands.json
├── include
├── libs
├── pic
└── src
```

`build` : 编译产物目录
`CMakeLists.txt` : 顶层cmake配置文件
`compile_commands.json` ：用来补全，使lsp 工作
`include` 头文件和源码形式的库
`libs` ：外置库，已经编译好的库，比如glfw库就会放在这个目录里
`pic` ：放置图片
`src` ：源码目录

顶层的 `CMakeLists.txt` 内容如下
```cmake
cmake_minimum_required(VERSION 3.4...3.28 FATAL_ERROR)

project(LearnOpenGL VERSION 1.0
  LANGUAGES CXX
)

set(CMAKE_CXX_STANDARD 11)

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

# 将include目录包括进整个项目
include_directories("${PROJECT_SOURCE_DIR}/include")

# 各个include里的库，自己写的类，默认编译成静态库
set(GLAD_SRC "${PROJECT_SOURCE_DIR}/include/glad/glad.cpp")
set(SHADER_SRC "${PROJECT_SOURCE_DIR}/include/shaders/shader.cpp")
set(STB_IMAGE_SRC "${PROJECT_SOURCE_DIR}/include/stb_image/stb_image.h")
set(CAMERA_SRC "${PROJECT_SOURCE_DIR}/include/camera/camera.cpp")

# 各个静态库
add_library(glad STATIC ${GLAD_SRC})
add_library(shader STATIC ${SHADER_SRC})
# add_library(stb_image STATIC ${STB_IMAGE_SRC})
# set_target_properties(stb_image PROPERTIES LINKER_LANGUAGE CXX)

# stb_image 为单头文件库（header only library），作如下配置
add_library(stb_image INTERFACE)
target_sources(stb_image INTERFACE
  FILE_SET HEADERS
  FILES "${PROJECT_SOURCE_DIR}/include/stb_image/stb_image.h"
)

add_library(camera_static STATIC ${CAMERA_SRC})

# For glm
set(BUILD_STATIC_LIBS "ON") # set before add_subdirectory
add_subdirectory(${PROJECT_SOURCE_DIR}/include/glm)

# For imgui
add_subdirectory(${PROJECT_SOURCE_DIR}/include/imgui)

# For libglfw3.a
set(GLFW3_STATIC_LIB "${PROJECT_SOURCE_DIR}/libs/libglfw3.a")
add_library(glfw3_static STATIC IMPORTED)
set_target_properties(glfw3_static PROPERTIES IMPORTED_LOCATION ${GLFW3_STATIC_LIB})

# 系统的共享库，一般只需要rt，m， dl这几个
set(SHARED_LIB rt;m;dl)
set(STATIC_LIB glad;shader;stb_image;glfw3_static;glm_static;camera_static;imgui_opengl_static)

add_subdirectory(src)
```

而 `src` 目录下的 `CMakeLists.txt` 就能简单很多

```cmake
cmake_minimum_required(VERSION 3.10)

# 设置好需要编译的目标，先编译后链接即可，也可以针对不同的项目配置不同的需要链接的库
# set(GUI_TARGETS create_window texture triangle transformations coordinate camera light)
set(GUI_TARGETS light)
foreach(target ${GUI_TARGETS})
  add_executable(${target} ${target}.cpp)
  target_link_libraries(${target} ${SHARED_LIB} ${STATIC_LIB})
endforeach()
```

### 编译

先生成 `Makefile` ： `cd build/ && cmake -S .. -B .`

后执行编译即可： `cmake --build .`

### 第一个窗口

需要准备两个库，一个是 `glfw`，另一个是 `glad`，下载可以直接网上搜，下载后，先编译 `glfw` 静态库，复制到项目的 `lib` 文件夹里； `glad` 下载后解压，将文件同目录结构复制到 `include` 文件夹里

第一个窗口的代码 `create_window.c` ：

```c
#define GLFW_INCLUDE_NONE
#include <GLFW/glfw3.h>
#include <glad/glad.h>

#include <stdio.h>

void framebuffer_size_callback(GLFWwindow *window, int width, int height);
void processInput(GLFWwindow *window);

int main(void)
{
    // 初始化
	glfwInit();
	# 设定glfw版本
	glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
	glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
	# 核心模式渲染
	glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);

    # 创建一个抽口上下文
	GLFWwindow *window = glfwCreateWindow(800, 600, "LearnOpenGL", NULL, NULL);
	if (window == NULL) {
		printf("Failed to create GLFW window\n");
		glfwTerminate();
		return 1;
	}

    # 生成窗口上下文
	glfwMakeContextCurrent(window);
	glfwSetFramebufferSizeCallback(window, framebuffer_size_callback);

	if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress)) {
		printf("Failed to initialize GLAD\n");
		return 1;
	}

	while (!glfwWindowShouldClose(window)) {
		processInput(window);

		glClearColor(0.2f, 0.2f, 0.2f, 1.0f);
		glClear(GL_COLOR_BUFFER_BIT);
		glfwSwapBuffers(window);
		glfwPollEvents();
	}
	glfwTerminate();
	return 0;
}

void processInput(GLFWwindow *window)
{
	if (glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS) {
		glfwSetWindowShouldClose(window, 1);
	}
}

void framebuffer_size_callback(GLFWwindow *window, int width, int height)
{
	glViewport(0, 0, width, height);
}
```
