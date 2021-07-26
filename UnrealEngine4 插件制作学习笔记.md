## UnrealEngine4 插件制作学习笔记

#### **一、模块、插件之间的相互引用**

首先从ue4的官方网站关于插件的介绍我们能看到这样一张图：

<img src="https://docs.unrealengine.com/4.26/Images/ProductionPipelines/Plugins/PluginAndModuleDependency.jpg" alt="PluginAndModuleDependency.png" style="zoom:Infinity%;" />

从图中我们可以看到基本的关系：1、游戏模块直接可以相互引用，2、游戏模块可以引用游戏插件，3、游戏插件之间可以相互引用，4、项目中的模块（无论是游戏模块还是插件模块）都可以引用引擎插件，5、引擎插件只能引用引擎模块。从这些基本关系中，我们就可以明确哪些功能是我们可以做的了。

接下来，以对第三方库插件的引用为例，介绍一下模块之间相互引用的基本框架：

![1625662690391](https://user-images.githubusercontent.com/19146127/126928171-ead9a758-4b5c-4862-ae2e-f1f81db5c84a.png)

[^图1]: 插件模块的基本架构

如图，这是实现模块之间相互引用的基本架构。基本概念即：每个模块单独实现一个Invoker类（继承自UObject）来实现与其他模块类的相耦合的业务逻辑。模块中的主要模块类通过调用本模块中Invoker类来实现与其他模块无关的自己本身的主要业务逻辑。（个人觉得与IModuleInterface本身提供的属性和方法无关的逻辑均可以放到Invoker类中，比如将原本MyThirdLibPlugin中用来调用第三方库的业务逻辑放到ThirdLibInvoker当中。这样更加方便扩展和复用）。

创建类后，仍有一些需要操作的点：

1、在模块内部，由于要使用UObject类，所以在module_name.build.cs配置文件中需要添加“CoreUObject"依赖![1625731671507](https://user-images.githubusercontent.com/19146127/126928245-5d548326-56d6-4e22-ad4b-09448beab584.png)

2、跨模块之间，还需要添加所依赖的其他模块，如图![1625735277928](https://user-images.githubusercontent.com/19146127/126928277-b3ab686c-aef7-4350-880f-dbafb4505518.png)

3、插件之间互相引用还需要在对应的.uplugin文件中添加对应的插件说明：![1625736802082](https://user-images.githubusercontent.com/19146127/126928296-d9d38495-6fda-4508-82e6-d7ea96eff382.png)

遇到的问题与解决方案：

1、在模块中制作Invoker类时，在#include xxx.generated.h 和GENERATE_BODY() 处VS会报红，![1625741758004](https://user-images.githubusercontent.com/19146127/126928393-3e20d650-28c8-4397-b187-e0b5991a5f32.png)

其原因是，此时UnrealHeadTool并没有生成对应的头文件。此时只要编译一遍，在对应插件路径**Intermediate\Build\Win64\UE4Editor\Inc\plugin_name\**中找到其头文件。然后vside即可。

2、在将第三方库编译成dll和lib时，采用新建一个项目来进行编译。在最初的编译过程中，无论属性如何选择build时都会报如下错误：![1625750433360](https://user-images.githubusercontent.com/19146127/126928432-6c900843-02db-4261-b737-614d4d3f8c46.png)
后来发现在代码中亮着的宏是WIN_32![1625750504438](https://user-images.githubusercontent.com/19146127/126928449-13499c9e-21fa-400c-8529-d9bf47e89183.png)

奇怪的是明明属性里配置的编译平台是x64为什么亮着的确实32位的宏呢？于是我打开了配置管理器，终于发现，虽然属性页配置的编译时release+x64但是在解决方案中，真实引用的项目文件仍然是32位。才会导致如此冲突。

![1625750701720](https://user-images.githubusercontent.com/19146127/126928509-171b3c3b-cc23-4b87-a6c0-64d33a63a4e0.png)

将其也改成release+x64后再build，就会生成指定的lib和dll了。

3、在最初跨插件引用类时，为了图省事，希望能直接#include ThirdLibInvoker.h 使用它的方法，但是在NewObject时，发现报了如下的错误![1625759439854](https://user-images.githubusercontent.com/19146127/126928524-3e2e6c08-c521-4285-babe-2a02cbd9cfd1.png)

后面按照基本架构写完后编译了一遍，再直接引用ThirdLibInvoker.h后又能正常使用NewObject了。其原因是，Definitions.OneButtonPlugin.h在这个头文件里并没有生成对NewObject的重载，导致模版类无法使用。通过这点，也能看出，使用图1的文件夹结构的用意。

#### **二、再各个编辑器内加按钮**

之前辛辛苦苦写的第二条，不知道为何没有保存上。简单来说：按照模板中提供的独立窗口插件和按钮插件RegisterMenu的方式，用例很少，不宜查找。所以采用传统的方式，完成按钮的注册。

![1625829048179](https://user-images.githubusercontent.com/19146127/126928540-7219e03f-10cb-40a4-99a7-01be05ae5430.png)
![1625830544578](https://user-images.githubusercontent.com/19146127/126928546-9bf4ab8c-fe90-4260-8dea-a1383c9fc759.png)

做法如一图所示，先获得要插入按钮的编辑器模块，通过MenuExtender来扩展编辑器。扩展位置可以通过编辑-》偏好设置-》通用\其他-》显示UI扩展，之后重启UE就可以看到绿色字体的UI扩展点了。至于能扩展哪些编辑器以及有哪些位置可以扩展，可以在引擎的源代码位置输入 ` grep -r "^class [I|F]\S*EditorModule"`即可看到该编辑器有哪些扩展能力了。
![1625833991220](https://user-images.githubusercontent.com/19146127/126928714-b6073d58-bcd3-4d84-a82f-005e847369f2.png)
![1625834022107](https://user-images.githubusercontent.com/19146127/126928732-aafec9a5-b449-487e-a6f9-d1f1d8e6a125.png)
![1625835850699](https://user-images.githubusercontent.com/19146127/126928650-e290abe7-e8b6-4324-8409-e3b746919388.png)

#### **三、利用Slate制作窗口**

有源码是这个世界上最幸福的事！直接贴几个官网：

https://docs.unrealengine.com/4.26/zh-CN/ProgrammingAndScripting/Slate/Widgets/ 控件示例展示了几种常用布局的示例

https://docs.unrealengine.com/4.26/zh-CN/ProgrammingAndScripting/Slate/Overview/ Slate概述，概述了Slate的基本特性和使用方法

https://docs.unrealengine.com/4.26/zh-CN/ProgrammingAndScripting/Slate/DetailsCustomization/ 详情面板的自定义示例。

通过这几个网站，基本上一个窗口的布局所需要知道的知识框架就全在其中了。了解了原理和基本实现之后，就是抄作业的环节了。首先我们如何知道我们需要哪些容器呢？我们可以新建一个UMG的蓝图编辑器，在里面构造一下我们想要的编辑器的基本框架。找到我们想要的元件。![1625927794417](https://user-images.githubusercontent.com/19146127/126928772-b2a26093-183d-4fd5-b852-d097db916aa8.png)
在细节的详情页中有打开该控件头文件的按钮。在头文件中我们可以找到该控件对应到slate上的共享指针类型。

![1625928025610](https://user-images.githubusercontent.com/19146127/126928806-588c3c9b-eba2-4011-a2c2-525c6737d5c5.png)

我们甚至可以通过在对应private的cpp文件中看到该slate类的一些常见用法，对之后在写控件属性有很大的帮助。

还有一种方法就是抄现有编辑器的代码。UE4为我们提供了一个强大的工具叫“控件反射器”，其中有一个功能就是显示控件层级。点击选择可测试命中控件，然后选择你在编辑器中想要看的区域或者控件，接着按下esc，就能看到你选中的控件的层级关系。以及该控件所属的源文件。点击源文件，Ctrl+G到对应行数，就能看到这个控件的完整用法![1625928957450](https://user-images.githubusercontent.com/19146127/126928834-cbe807c2-6f22-439c-88be-ed6a0b279e2d.png)

![1625928969276](https://user-images.githubusercontent.com/19146127/126928850-febe8ac0-8bf8-4ddd-8ee8-7829970ffa0b.png)

这个工具使我们借鉴源码的道路越发的平顺。另外我们还可以在runtime\appframework找到很多测试框架。都可以摘取自己想要的段落。

![1626941647622](https://user-images.githubusercontent.com/19146127/126928949-f74de13c-d947-4b18-8d20-48aff99b4a46.png)

#### **四、在standalone内添加菜单栏**

原本认为只要继承了Extensibility的界面就可以拓展和添加菜单和工具。后来发现，其实standaloneWindow即使继承了 Extensibility依然没法扩展menu。后来熟读了AssetEditor 的源码后发现了其中的奥妙。实际上ExtensbilityManager只是一个用于管理众多需要扩展到窗口的extender的中转站。我们所创建的窗口要在构造时会从manager中获取合并好的extender（ExtensbilityManager::getAllExtender()会将该管理器下的所有Extender合并起来。）传入MenuBuilder用于创建菜单栏。通过MenuBuilder.MakeWidget() 来显示菜单。![1626941647622](https://user-images.githubusercontent.com/19146127/126928978-0e96b56c-89e0-4716-9b85-a74460f393da.png)
还有第二种做法，使用UToolMenus在插件模块startModule时注册菜单。这里要注意的是每个菜单项都是一个subMenu，对菜单项进行扩展才是添加下拉菜单项。这套方法对从头写菜单栏提供了极大的便利。但对于扩展已知编辑器的菜单和工具栏是有一定困难的（具体方法详见第二大点）。![1626958924093](https://user-images.githubusercontent.com/19146127/126928969-5082556d-d732-4e0f-9825-ea4dd808f7f8.png)

![1626958977666](https://user-images.githubusercontent.com/19146127/126928988-431f6e64-ffd3-4d75-b283-c72c18b80066.png)

![1626959508195](https://user-images.githubusercontent.com/19146127/126928995-22206375-13ee-4ab4-a08f-8cb748a72865.png)

