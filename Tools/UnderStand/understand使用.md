
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 设置](#1-设置)
* [2 导入项目](#2-导入项目)
* [3 主要功能](#3-主要功能)
	* [3.1 代码知识](#31-代码知识)
	* [3.2 指标和报告](#32-指标和报告)
	* [3.3 制图](#33-制图)
	* [3.4 标准测试](#34-标准测试)
	* [3.5 依赖性分析](#35-依赖性分析)
	* [3.6 编辑](#36-编辑)
	* [3.7 搜索](#37-搜索)
	* [3.8 语言](#38-语言)
* [4 搜索功能](#4-搜索功能)
* [5 项目视图](#5-项目视图)
	* [4.1 层级关系视图分类](#41-层级关系视图分类)
	* [4.2 结构关系视图分类](#42-结构关系视图分类)
* [术语 Terminology](#术语-terminology)
	* [Architecture 层级:](#architecture-层级)
	* [Database 数据库](#database-数据库)

<!-- /code_chunk_output -->

# 1 设置

1. 设置字体和颜色风格

修改默认字体: Preference → Editor → Default style 

"SourceCodePro Nerd Font Mono" 13

修改颜色: Preference → Editor → Styles, Predefined选为"silver"

2. 

# 2 导入项目

1. new → project, 输入项目名称(linux\-latest), understand的项目数据库(之前的目录)

![](./images/2019-05-31-09-50-12.png)

2. 语言选择

Assembly、C/C\+\+、Python、Web

注: 其中Strict模式包含Object-C和Object—C\+\+，还有Web的注释

![](./images/2019-05-31-09-53-57.png)

3. 导入文件和设置, 这里选第一个

![](./images/2019-05-31-09-54-38.png)

4. 添加目录和文件, 这里添加目录

![](./images/2019-05-31-09-55-35.png)

![](./images/2019-05-31-09-56-09.png)

5. 代码分析

有两个选项，一个是立即分析代码，一个选择配置，对于我们来说只需要默认即可，然后点击OK按钮，此时软件开始分析代码

![](./images/2019-05-31-09-56-44.png)

# 3 主要功能

## 3.1 代码知识

理解为您提供有关您的代码的相关信息。快速查看关于函数，类，变量等的所有信息，如何使用，调用，修改和交互。轻松查看您想要了解代码的呼叫树，指标，参考信息和任何其他信息。

![](./images/2019-05-31-11-10-24.png)

## 3.2 指标和报告

理解非常有效地收集有关代码的度量标准并为您提供不同的查看方式。当我们没有完全满足您的需求时，可以快速获得大量标准指标以及编写您自己的自定义指标的选项。

![](./images/2019-05-31-11-10-44.png)

## 3.3 制图

了解提供图表，使您可以查看代码连接（依赖关系），流程如何（控制流程图），使用哪些函数调用其他函数（调用图表）等等。有许多自定义选项可轻松让您仅显示您感兴趣的内容，因此该图最适合您的需求。

![](./images/2019-05-31-11-11-04.png)

## 3.4 标准测试

理解提供了一种使用已发布的编码标准或您自己的自定义标准来检查代码的方法。这些检查可用于验证命名准则，度量标准要求，已发布的最佳做法或对您的团队而言重要的任何其他规则或约定。

![](./images/2019-05-31-11-11-25.png)

## 3.5 依赖性分析

查看代码中的所有依赖关系以及它们如何连接。使用Understanding的交互式图形或使用文本依赖浏览器查看这些依赖关系。两者都可以让您快速轻松地查看所有依赖关系，或者深入了解详细信息。

![](./images/2019-05-31-11-11-46.png)

## 3.6 编辑

![](./images/2019-05-31-11-12-02.png)

## 3.7 搜索

在“理解”中搜索有多个选项。要获得即时结果，请使用我们的“即时搜索”功能，该功能可在打字完成之前提供结果。了解还提供更多自定义和复杂搜索的搜索选项，例如正则表达式和通配符搜索。

![](./images/2019-05-31-11-12-20.png)

## 3.8 语言

了解支持十几种语言，并且可以处理以多种语言编写的代码库。这允许您查看语言之间的调用和依赖关系，以便您可以获取有关完整系统的信息。

![](./images/2019-05-31-11-12-40.png)

# 4 搜索功能

1. 搜索文件: 在这个搜索中你可以快速搜索你要查看的文件，

快捷键，鼠标点击左侧上面项目结构窗口，然后按command + F键会出现如下图所示的搜索框，在框中输入你想要的类回车即可

![](./images/2019-05-31-10-15-15.png)

2. 在打开的文件中搜索: 将鼠标定位到右侧代码中，点击command + F，会弹出搜索框，输入方法回车即可：

![](./images/2019-05-31-10-18-33.png)

3. 全局搜索: 快捷键F5或者去上面菜单栏中的search栏中查找

![](./images/2019-05-31-10-22-08.png)

4. 实体搜索: 菜单栏的find entity, 根据结构体、方法等搜索

![](./images/2019-05-31-10-27-56.png)

# 5 项目视图

项目视图包含很多的功能，能够自动生成各种流程图结构图，帮助你快速理清代码逻辑、结构等，以便快速理解项目流程，快速开发.

视图查看方式有两种，

一种是鼠标点击你要查看的类或者方法等上面，然后右键弹出菜单，鼠标移动到Graphical Views，然后弹出二级菜单，如下图所示：

![](./images/2019-05-31-10-33-33.png)

另一种方式是点击要查看的类或者方法，然后找到代码上面菜单栏中的如下图标：

![](./images/2019-05-31-10-36-18.png)

然后点击图标右下角的下拉箭头，弹出如下菜单，即可选择查看相关视图：

## 4.1 层级关系视图分类

1. Butterfly：如果两个实体间存在关系，就显示这两个实体间的调用和被调用关系；

如下图为Activity中的一个方法的关系图：

![](./images/2019-05-31-11-21-18.png)

2. Calls：展示从你选择的这个方法开始的整个调用链条；

3. Called By：展示了这个实体被哪些代码调用，这个结构图是从底部向上看或者从右到左看；

![](./images/2019-05-31-11-24-57.png)

4. Calls Relationship/Calledby Relationship:展示了两个实体之间的调用和被调用关系，操作方法：首先右键你要选择的第一个实体，然后点击另一个你要选择的实体，如果选择错误，可以再次点击其他正确即可，然后点击ok；

![](./images/2019-05-31-11-25-12.png)

![](./images/2019-05-31-11-25-16.png)

5. Contains:展示一个实体中的层级图，也可以是一个文件，一条连接线读作”x includes y“；

![](./images/2019-05-31-11-25-31.png)

6. Extended By:展示这个类被哪些类所继承，

![](./images/2019-05-31-11-25-45.png)

7. Extends:展示这个类继承自那个类：

![](./images/2019-05-31-11-25-59.png)

## 4.2 结构关系视图分类

1.Graph Architecture：展示一个框架节点的结构关系；

2.Declaration:展示一个实体的结构关系，例如：展示参数，则返回类型和被调用函数，对于类，则展示私有成员变量（谁继承这个类，谁基于这个类）

3.Parent Declaration:展示这个实体在哪里被声明了的结构关系；

4.Declaration File:展示所选的文件中所有被定义的实体（例如函数，类型，变量，常量等）；

5.Declaration Type:展示组成类型；

6.Class Declaration:展示定义类和父类的成员变量；

7.Data Members:展示类或者方法的组成，或者包含的类型；

8.Control Flow:展示一个实体的控制流程图或者类似实体类型；

![](./images/2019-05-31-11-26-28.png)

9.Cluster Control Flow:展示一个实体的流程图或者类似实体类型，这个比上一个更具有交互性；

10.UML Class Diagram:展示这个项目中或者一个文件中定义的类以及与这个类关联的类

![](./images/2019-05-31-11-27-46.png)

11.UML Sequence Diagram:展示两个实体之间的时序关系图；

![](./images/2019-05-31-11-28-09.png)

12.Package:展示给定包名中声明的所有实体

13.Task:展示一个任务中的参数，调用，实体

14.Rename Declaration:展示实体中被重命名的所有实体

由于视图比较多，所以就一一贴上代码，主要还是需要自己去调试，查看各个功能视图的展示结构以及作用，孰能生巧，多操作几下就会了，所以不再做过多的解释。最终希望这款软件能够帮助你快速开发，快速阅读别人的或者自己的代码。

# 术语 Terminology

## Architecture 层级:

An architecture is a hierarchical aggregation of source code units (entities). An architecture can be user created or automatically generated. Architectures need not be complete (that is, an architecture’s flattened expansion need not reference every source entity in the database), nor unique (that is, an architecture’s flattened expansion need not maintain the set property).

层级表示代码单元（或者实体）组成的层次结构，可以由用户手动创建，也可由本软件自动生成。一个层级可以不完整（例如一个层级的扁平化扩展有可能不会关联数据库中的所有代码实体），也可能不唯一（扁平化扩展的层级可能不会处理其预设属性）。

## Database 数据库

The database is where the results of the source code analysis, as well as project settings, are stored. By default, this is a project’s “.udb” file.

代码经分析后产生的中间结果，以及工程设置保存在数据库，其缺省扩展名为“.udb”。

## Entity 实体

An Understand “entity” is anything it has information about. In practice this means anything declared or used in your source code and the files that contain the project. Subroutines, variables, and source files are all examples of entities.

Understand 描述的“实体”表示任何包含信息的事物，具体来说，代码中声明或 
者使用的标识、包含工程的文件、子程序、变量、源文件都可以被称为实体。

Project 工程

The set of source code you have analyzed and the settings and parameters chosen. A “project file” contains the list of source files and the project settings.

表示源代码的集合以及相关的配置和参数，工程文件包含源文件清单和工程设置。

Relationship 关联

A particular way that entities relate to one another. The names of relationships come from the syntax and semantics of a programming language. For instance, subroutine entities can have “Call” relationships and “CalledBy” relationships.

互作用的实体之间的关系，关联的名称来源于编程语言的语法和语义，例如过程式实体具有“调用”和“被调用”的关联对象。

Script 脚本

Generally a Perl script. These can be run from within Understand’s GUI, or externally via the “uperl” command. The Understand Perl API provides easy and direct access to all information stored in an Understand database.
通常指perl脚本，脚本可以通过Understand 2.5的图形用户界面或者外部的脚本命令执行。Understand Perl API提供了快捷的访问Understand数据库所有信息的接口。

parts 部件
下面的图形展示了一些Understand 图形用户界面(GUI) 中常用的部件: