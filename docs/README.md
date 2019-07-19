# GacUI

* https://github.com/vczh-libraries/GacUI

GacUI由轮子哥开发，目前已发布22个release，最新为0.10.0.0版本。功能在持续开发和支持中，由其单人维护windows版本。

## 优点：开源免费，功能强大

* 充分利用相对布局方式，无需繁琐的代码中计算位置和处理
* 脚本引擎支持将xml编译为C++代码和字节码，速度和效率更快
* 支持GDI和D2D
* 原生支持MVVM和数据绑定，支持动画效果
* 支持主流控件，熟悉后可缩短开发周期


## 缺点：

* API完善，但是文档缺乏；社区无，技术文档匮乏
* 需要知识迁移学习，来掌握GacUI的控件、布局、脚本逻辑
* 参与者太少，bug和稳定性未知，更新依赖单人频次较低

# 学习曲线

目前关于GacUI的文档较少，代码仓库内包含各个大大小小的demo可供参考，其本身的特性和优点值得大家学习迁移。

## 一、代码结构

windows版本可以从[Release](https://github.com/vczh-libraries/Release)获取，包含如下几个部分：
* Vlpp：跨平台C++基础库
* Workflow：脚本引擎，将xml文件编译成字节码，嵌入到二进制资源文件中
* GacUI：界面库代码
* GacUIWindows：windows平台下的Window Provider和Renderer

## 二、体系架构

我们先了解下GacUI的界面体系及其概念，在了解其基本概况的基础下，结合代码实践将有助于我们快速深入的掌握界面库的精髓。

GacUI的架构是一个分层体系，从底层到顶层分别为：
* Window Provider
* Renderers
* Elements+Compositions
* Controls+Templates
* XML Resource Compiler

### 1. Window Provider

操作系统提供的原生窗口、图片资源、鼠标资源、异步原语和其他属性和功能。跨平台支持只需要新增此部分支持即可，然后适配对应的Renderers。

### 2. Renderers

Renderers跟Window Provider分离，从而区分不同操作系统的不同或者相同的图像技术绘图。OpenGL在很多平台上均为二等公民，所以GacUI未使用OpenGL来开发Renderers。在程序开始(WinMain中)，选择一个Renderer后就不可以变更。

* Windows：GDI，Direct2D 1.0，Direct2D 1.1
* Linux：Cairo+Pango
* OSX：CoreGraphics+CoreText

Direct2D的不同版本仅初始化代码不同，但是决定了是否可以使用Direct3D 11.0。

### 3. Elements + Compositions

在GacUI中代表了基础的图元和排版功能，每一个element运行在具体平台时，需要具体的Renderer对象。一个COmposition对象代表了窗口上的一个长方形区域，提供核心的排版功能。每一个区域可以嵌入element，当一个区域确定其位置后，element就会被填充到长方形区域中，并渲染处理。

几乎所有element都是简单的几何体，除了渲染文字的GuiColorizedTextElement和GuiDocumentElement。在制作控件皮肤(template的部分功能)时，文本框由于功能复杂，皮肤需要提供一个区域让控件放置这两个Elemment，而不是跟普通控件一样，全权处理所有的渲染对象。目前composition支持定位、stack、table、flow和其他一些功能。使用GuiDocumentElement的GuiDocumentLabel和GuiDocumentViewer可以在富文本文档中嵌入Composition。

### 4. Controls + Templates

Control结构复杂，包含代表控件本身的操作和数据逻辑的对象，包含如何渲染这个控件的IStyleController或者IStyleProvider对象。IStyleController拥有整个Composition和Element的控制权。如果当一个Control只决定让皮肤控制一部分的Composition和Element的时候，那么他会提供IStyleProvider对象。每一个Control类都有自己相应的Template类。

IStyleController / IStyleProvider 和 Template的区别，就在于一个是Pull模型的，一个是Push模型的。Push模型做data binding特别容易，因此在XML里面都是通过创建Template对象来修改一个Control是如何渲染的。

对于列表控件、譬如 GuiTextList、GuiListView、GuiTreeView 和 GuiVirtualDataGrid 等，除了Template以外，还有ItemTemplate。Template和ItemTemplate是可以分开指定的。Template确立了整个控件的外观，而ItemTemplate确定了每一个列表项的外观。

如果需要对容器的内容做数据绑定的话，那么需要使用上述4个控件的Bindable版本，分别是它们的子类：GuiBindableTextList、GuiBindableListView、GuiBindableTreeView 和 GuiBindableDataGrid 。在使用这些控件的时候，可以通过在XML里面的Workflow脚本把一个C++的容器对象绑定到ListView上：
```
<GuiBindableListView ItemSource-eval="ViewModel.Something"/>
```

每个ListViewItem拿到的容器的每一个对象，可能最终类型是不一样的。你可以通过给ListView的ItemTemplate指定一系列的Template对象，通过在XML里面写好的这些Template的构造函数的参数的类型，来让ListView决定到底要使用哪个Template。于是一个异构的列表就这么轻松的造出来了。


### 5. XML Resource Compiler

目前GacUI的XML资源文件，支持构造window、userControl、Template、类CSS的instanceStyle(通过XPath来批量设置XML属性，比选择器好用，精确控制)和一些共享的workflow脚本。共享脚本可以定义一些窗口的逻辑代码，还有MVVM模式需要的ViewModel的接口和数据结构。

GacGen.exe可以将xml编译成一个二进制的资源文件，还有一系列C++代码。C++代码模拟了C#的partial class能力，可以像Windows Forms一样，准备控件的事件处理和窗口初始化时候的一系列任务。xml更新时，GacGen.exe重新生成的C++代码会跟你修改后的代码自动合并。窗口初始化时会去资源文件找到对应的脚本运行，并按照要求创建控件和data binding。

## 二、排版

### 1. GuiGraphicsComposition

GacUI的图形都是由Composition和Element组成的，Composition的基类为GuiGraphicsComposition。

* MinSizeLimitation：控制当前composition如何被children影响的
* Margin：子容器的外部padding
* InternalMargin：子容器相对父容器的padding
* PreferredMinSize：最小大小

### 2. GuiBoundsComposition

* AlignmentToParent：设置Composition的四个方向，margin的外边与parent的internalMargin的距离
* Bounds：很鸡肋，设置float pos，但是与自动布局冲突时就失效了

### 3. GuiSharedSize(Root|Item)Composition

### 4. GuiStackComposition

GuiStackComposition的属性如下：

* Direction：生长的方向，从左到右、从右到左、从上到下、从下到上的生长方向
* Padding：StackItem之间的间距，可以使用InternalMargin替换，或者设置Margin或AlignmentToParent来模拟
* ExtraMargin：仅对StackItem起作用，类似于Stack的InternalMargin
* IsStackItemClipped：是否有item因大小而不可见
* EnsureVisible：所有item根据direction的要求进行滑动，使某一个item显示出来，例如很多的tab显示不全的时候，控制某一个tab页签展示

GuiStackItemComposition的属性如下：
* ExtraMargin：Stack告诉item位置，item据此属性变大一点，例如tab在MoveChild时可以变大，并且可以挡住旁边的两个页签展示在最上边

### 5. GuiFlowComposition 

Flow相对与Stack是二维对一维，生长方向有8个。

* Axis：AxisDirection，生长方向
* RowPadding：虚拟行的行距，根据Axis属性的Y轴方向的间距
* ColumnPadding：虚拟行的列距
* ExtraMargin：与Stack一致
* Alignment：当一个虚拟行放不下item，但是还有空间时如何处理。左对齐、居中、扩展三种方法

GuiFlowItemComposition与GuiFlowComposition配合使用，包含以下属性：
* ExtraMargin：
* FlowOption：计算基线的方法

### 6. GuiTableComposition


### 7. Document

布局的更多细节和使用详情请参考[GacUI_Layout](https://github.com/vczh-libraries/Release/tree/master/Tutorial/GacUI_Layout).




## 三、界面库扩展

### 1. 扩展Element

### 2. 扩展Composition

### 3. 扩展Control
* Control表示与逻辑分离
* 皮肤接口定义
* 创建皮肤接口的Template
* 控件反射
* XML资源增加新功能

### 4. 绘图API的移植

### 5. 操作系统移植

## 四、实战

对象生命周期管理：


### 1. 工程创建

### 2. 资源文件生成二进制字节码和C++代码

### 3. Workflow脚本引擎定义接口、数据结构和表达式

### 4. 创建窗口

### 5. 创建控件

### 6. 皮肤

* 创建皮肤(Template)
* 列表控件的列表项皮肤(Item Template)


### 7. 控件的数据绑定

* 普通控件的数据绑定
* 列表控件的数据绑定

### 8. MVVM与异步操作

### 9. 动画和状态机


# 代码剖析

本章节我们将深入到GacUI界面库的代码中，分析其框架和内部实现。







