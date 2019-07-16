# Introduction

This project testing the uilib of [GacUI](https://github.com/vczh-libraries/GacUI).

# GacUI

Windows版本可以从Release仓库获取，包含如下几部分：
* Vlpp：跨平台的C++基础库
* Workflow：一个脚本引擎，靠反射实现对C++类的访问，对象模型使用引用计数
* GacUI：GacUI的主要部分
* GacUIReflection：GacUI所有可以被脚本访问的类的元数据，用于反射。
* GacUIWindows：windows平台下的Window Provider和Renderer

GacUI会将xml文件编译成workflow脚本引擎字节码，嵌入到生成后的二进制资源文件中。

## 一、体系架构

GacUI的架构是分层体系，从底层到顶层分别为：
* Window Provider
* Renderers
* Elements+Compositions
* Controls+Templates
* XML Resource Compiler

1. Window Provider

    操作系统提供的原生窗口、图片资源、鼠标资源、异步原语和其他属性和功能。跨平台支持只需要新增此部分支持即可，然后适配对应的Renderers。

2. Renderers

    Renderers跟Window Provider分离，从而区分不同操作系统的不同或者相同的图像技术绘图。OpenGL在很多平台上均为二等公民，所以GacUI未使用OpenGL来开发Renderers。在程序开始(WinMain中)，选择一个Renderer后就不可以变更。

    * Windows：GDI，Direct2D 1.0，Direct2D 1.1
    * Linux：Cairo+Pango
    * OSX：CoreGraphics+CoreText

    Direct2D的不同版本仅初始化代码不同，但是决定了是否可以使用Direct3D 11.0。

3. Elements + Compositions

    在GacUI中代表了基础的图元和排版功能，每一个element运行在具体平台时，需要具体的Renderer对象。一个COmposition对象代表了窗口上的一个长方形区域，提供核心的排版功能。每一个区域可以嵌入element，当一个区域确定其位置后，element就会被填充到长方形区域中，并渲染处理。
    
    几乎所有element都是简单的几何体，除了渲染文字的GuiColorizedTextElement和GuiDocumentElement。在制作控件皮肤(template的部分功能)时，文本框由于功能复杂，皮肤需要提供一个区域让控件放置这两个Elemment，而不是跟普通控件一样，全权处理所有的渲染对象。目前composition支持定位、stack、table、flow和其他一些功能。使用GuiDocumentElement的GuiDocumentLabel和GuiDocumentViewer可以在富文本文档中嵌入Composition。

4. Controls + Templates

    Control结构复杂，包含代表控件本身的操作和数据逻辑的对象，包含如何渲染这个控件的IStyleController或者IStyleProvider对象。IStyleController拥有整个Composition和Element的控制权。如果当一个Control只决定让皮肤控制一部分的Composition和Element的时候，那么他会提供IStyleProvider对象。每一个Control类都有自己相应的Template类。

    IStyleController / IStyleProvider 和 Template的区别，就在于一个是Pull模型的，一个是Push模型的。Push模型做data binding特别容易，因此在XML里面都是通过创建Template对象来修改一个Control是如何渲染的。

    对于列表控件、譬如 GuiTextList、GuiListView、GuiTreeView 和 GuiVirtualDataGrid 等，除了Template以外，还有ItemTemplate。Template和ItemTemplate是可以分开指定的。Template确立了整个控件的外观，而ItemTemplate确定了每一个列表项的外观。

    如果需要对容器的内容做数据绑定的话，那么需要使用上述4个控件的Bindable版本，分别是它们的子类：GuiBindableTextList、GuiBindableListView、GuiBindableTreeView 和 GuiBindableDataGrid 。在使用这些控件的时候，可以通过在XML里面的Workflow脚本把一个C++的容器对象绑定到ListView上：
    ```
    <GuiBindableListView ItemSource-eval="ViewModel.Something"/>
    ```
    每个ListViewItem拿到的容器的每一个对象，可能最终类型是不一样的。你可以通过给ListView的ItemTemplate指定一系列的Template对象，通过在XML里面写好的这些Template的构造函数的参数的类型，来让ListView决定到底要使用哪个Template。于是一个异构的列表就这么轻松的造出来了。


5. XML Resource Compiler

    目前GacUI的XML资源文件，支持构造window、userControl、Template、类CSS的instanceStyle(通过XPath来批量设置XML属性，比选择器好用，精确控制)和一些共享的workflow脚本。共享脚本可以定义一些窗口的逻辑代码，还有MVVM模式需要的ViewModel的接口和数据结构。

    GacGen.exe可以将xml编译成一个二进制的资源文件，还有一系列C++代码。C++代码模拟了C#的partial class能力，可以像Windows Forms一样，准备控件的事件处理和窗口初始化时候的一系列任务。xml更新时，GacGen.exe重新生成的C++代码会跟你修改后的代码自动合并。窗口初始化时会去资源文件找到对应的脚本运行，并按照要求创建控件和data binding。



## 二、排版




## 三、综合概述

1. GacUI基本概念

   * 工程如何创建？
   * 什么是Element
   * 什么是Composition
   * 什么是Control
   * 什么是Template
   * 对象生命周期的管理

2. GacUI资源文件

    * 资源文件生成二进制和C++代码
    * Workflow脚本引擎定义接口、数据结构和表达式
    * 创建窗口
    * 创建控件
    * 创建皮肤（Template）
    * 普通控件的数据绑定
    * 列表控件的数据绑定
    * 列表控件的列表项皮肤（Item Template）

3. GacUI开发模式

    * MVVM
    * 数据绑定
    * 异步操作
    * 动画
    * 状态机

4. GacUI扩展

    * 扩展Element
    * 扩展Composition
    * 扩展Control
      * Control表示与逻辑分离
      * 皮肤接口定义
      * 创建皮肤接口的Template
      * 控件反射
      * XML资源增加新功能
    * 绘图API移植
    * 操作系统移植

5. GacUI框架和内部实现