[TOC]

## 介绍

StarUML是一个复杂的软件建模工具，旨在支持敏捷和简洁的建模。

### 主要面向的用户

- 敏捷和小型开发团队
- 专业人员
- 教育机构

### StarUML的主要特点

- 支持多平台
- 符合UML 2.x标准
- 时序图
- 类图
- 用例图
- ERD（实体关系图）
- 数据流程图
- 流程图
- 多窗口
- 模型驱动开发
- Open APIs 
- 各种第三方扩展
- 异步模型验证
- 导出到HTML文档



## 基本概念

### 项目

- 项目是存储为单个文件（.mdj）的顶级元素。
- 建模一个软件系统需要描述多个模型，因为用一个透视图来描述系统是不够的，所以我们通常会制作多个模型，例如用例模型、设计模型、组件模型、部署模型或项目中的其他模型。
- 通常，项目组织为一组umlModels、umlPackages或umlSubSystems。如果您想了解更多关于UML元素的信息，请参考OMG UML规范。

### 模型与视图

- 许多用户混淆了图表或绘图工具（如Microsoft Visio）与建模工具（如StarUML或Rational Software Architect）之间的区别。首先，你需要了解一个图表不是一个模型。
- 模型或软件模型是对软件系统任何方面的描述，如结构、行为、需求等。软件模型可以用文本、数学或视觉形式表示。模型元素是软件模型的构建块。
- 图表是软件模型的视觉几何符号表示。软件模型可以用一个或多个具有不同方面的图来表示。例如，一个图可以关注类层次结构，而另一个图可以关注对象之间的交互。图表由视图元素组成，视图元素是模型元素的视觉表示。
- 模型元素可以有多个对应的视图元素。模型元素有自己的数据，如名称、原型、类型等。视图元素只是在图中呈现相应的模型元素。视图元素可能在一个图或不同的图中存在多次。如果模型元素的名称发生了更改，则所有相应的视图元素都会反映其关系图中的更改。

### 片段（Fragment）

- 片段是项目的一部分，另存为扩展名为.mfj的单独文件。任何元素都可以作为片段导出，但通常，umlPackage、umlModel和umlsubsystem都是候选元素。将片段导出为文件后，可以通过在项目中导入来重用片段。
- 可以导出/导入片段。

### 概要文件（Profile）

UML（统一建模语言）是一种通用的建模语言，可以用来表示各种软件密集型系统。因此，对特定域或平台使用UML是不够的，因此您可能需要定义UML概要文件。StarUML提供了可用于扩展UML的UML配置文件。例如，UML概要文件可以用于以下目的。

- 特定的编程语言（C/C++，Java，C语言，Python等）的配置文件
- 特定开发方法的概要（RUP、Catalysis、UML组件等）
- 特定域的配置文件(EAI, CRM, SCM, ERP等) 

### 扩展

扩展是向StarUML添加新特性的包。例如，扩展可以扩展菜单、UI、对话框、建模符号、首选项等。扩展可以用JavaScript、CSS3和HTML5编写，并且可以使用集成在StarUML中的node.js。扩展可以通过主扩展注册表轻松安装、卸载和更新。



## 管理项目

### 新的项目

创建建模项目，按`ctrl+n`或选择 `File|New`

### 根据模板新建项目

可以通过选择模板来启动建模项目。要使用模板启动项目，请选择 `File | New From Template | [TemplateName]` 。StarUML支持4个默认模板：

- **UMLMinimal**  ：一个带有UML标准配置文件的单一模型
- **UMLConventional**  ：使用UML标准配置文件的用例模型、分析模型、设计模型、实现模型和部署模型
- **4+1 View Model** ：Pilippe Kruchten's [4+1 Architectural View Model](http://en.wikipedia.org/wiki/4%2B1_architectural_view_model). 
- **Rational**  ：Approach of Rational Rose Tool 
- **Data Model**  ：一个简单的数据建模项目

如果不想使用预先定义的模板，则需要创建自己的项目结构。

### 打开项目

如果您有模型文件（.mdj），您可以在StarUML中打开它。要打开模型文件，请按`ctrl+o`或选择 `File | Open...` 然后在`Open Dialog` 中选择一个文件。

### 保存项目

您可以通过按`ctrl+s`或选择文件保存将工作项目保存到文件中。如果要另存为其他文件，请按`ctrl+shift+s`或选择`File | Save As` 。

### 关闭项目

要关闭工作项目，请选择`File | Close` 。如果您没有保存项目，将要求您保存或不保存。

### 导出片段（Export Fragment ）

要将项目的一部分导出为片段，请选择`File | Export | Fragment` ，然后在`Element Picker Dialog` 中选择要导出的元素。

### 导入片段（Import Fragment ）

要将片段导入到项目中，请选择`File | Import | Fragment` 。片段将作为项目的子级包含在内。

### 应用配置文件（Applying Profiles）

要包括UML标准配置文件，请在菜单栏中选择 `Model | Apply Profile | UML Standard Profile (v2)` 。



## 编辑元素

### 编辑图标（Diagram）

#### 创建图表

- 可以选择菜单栏中的`Model | Add Diagram | [DiagramType]` ，可以在本项目中创建一个图表
- StarUML界面的右侧有个`Model Explorer`,下面的红色正方体是项目的根目录，右击根目录，也可以添加图表

#### 删除图表

- 在`Explorer`右击删除
- 在`Explorer`中选中并按`Ctrl+Delete` 
- 选择`Edit | Delete from Model`  这时内部文件被删除，再`Edit | Delete from Model` ，这时全部内容被删除

#### 打开图表

- 双击`Explorer`中的目标图表

#### 关闭图表

- 在StarUML界面的左侧`Working Diagrams`中点击 `x`按钮，关闭目标图表
- 选中目标图表，按`F4`
- 选中目标图表，选择`View|Colse Diagram`关闭此图表，还有`Colse All Diagram`、`Colse Other Diagram`选项

#### 切换图表

- 按`Ctrl+Shift+]` 、`Ctrl+Shift+[`

### 编辑图表元素

#### 创建图表元素

可以使用以下选项创建模型元素和视图元素。

- 在StarUML界面的左下角的Toolbox栏中，选择元素
- 拖到图表上作为元素的大小，如果元素是一种关系，则链接两个元素

#### 删除元素

- 删除图表中的元素：选中图表中的元素，按 `Delete|only`
- 删除模块中的元素：选中图表中的元素，按 `Delete|model`或在`Explorer`中删除

#### 选择元素

- 只需单击一个元素，就可以在图中选择一个元素。如果要在保持当前选择的同时选择其他元素，请按`SHIFT`键单击元素。拖动区域时，元素与区域重叠将被选中。按`SHIFT`也可以拖动。

#### 复制粘贴

- 复制或剪切要粘贴的元素时，必须在模型元素和视图元素之间进行明确区分。如果复制了模型元素，则必须将其粘贴到模型元素下。在这种情况下，所选元素中包含的所有子元素都将复制到一起。视图元素可以复制到同一个图表中，也可以复制到不同的图表中。复制的视图元素只能粘贴到图表中；不能粘贴到模型元素。复制和粘贴也可能受到限制，具体取决于视图元素类型和图表类型。

注：有些元素不能`copy, cut, and paste`. 

#### 给元素添加属性

- 给一个元素（如一个类）添加方法等属性。



## 设置图表格式

### 设置字体

### 设置线颜色

### 设置线的样式



## 类图表元素（Class Diagram）





































https://docs.staruml.io/

https://github.com/staruml/staruml-java

https://www.52pojie.cn/thread-899541-1-1.html

https://www.cnblogs.com/luofay/p/6087808.html

https://blog.csdn.net/luansha0/article/details/82260678

https://www.jianshu.com/p/0cda2771caf2

https://wenku.baidu.com/view/e943ce8cd0d233d4b14e69de.html?sxts=1555571196593

