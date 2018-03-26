title: Material Design 标注图
date: 2016-01-01 18:40:33
tags: 
	- Material Design
categories:
	- 设计
---

&emsp;最近发现自己在实现产品需求时,在界面元素的细节部分把握不够准确,经常被leader告知需要微调一下界面元素的各类尺寸.虽然导致这个因素的最重要的原因是我场的设计太懒,不给标注,但是作为一个终端工程师,了解并掌握足够的设计相关的知识还是十分必要的,于是最近就开始研究其google的Material Design设计风格.
&emsp;本篇文章主要是收集一些MD风格的设计标注图和注意事项,主要内容都来自于[重磅教程！帮你全面彻底搞定MATERIAL DESIGN的学习笔记](http://www.uisdc.com/comprehensive-material-design-note#).
> MD风格的精髓在于
> 把物理世界的体验带进屏幕,去掉现实的杂质和随机性,保留保留其最原始纯净的形态、空间关系、变化与过渡，配合虚拟世界的灵活特性，还原最贴近真实的体验，达到简洁与直观的效果

&emsp;其实以我现在的理解就是:
-  界面元素的层次性
-  交互时强调带有意义并且具有现实特性的动画
-  两种纯色作为应用的基本颜色

&emsp;这篇只是收藏MD风格中关于一些特定界面元素的标注图以备以后使用

#### 文字排版
常用字号：

- 12sp 小字提示
- 14sp（桌面端13sp） 正文/按钮文字
- 16sp（桌面端15sp） 小标题
- 20sp Appbar文字
- 24sp 大标题
- 34sp/45sp/56sp/112sp 超大号文字

&emsp;长篇幅正文，每行建议60字符（英文）左右。短文本，建议每行30字符（英文）左右。

#### 常用尺寸
>所有可操作元素最小点击区域尺寸：48dp X 48dp。
  栅格系统的最小单位是8dp，一切距离、尺寸都应该是8dp的整数倍。以下是一些常见的尺寸与距离：

- 顶部状态栏高度：24dp
- Appbar最小高度：56dp
- 底部导航栏高度：48dp
- 悬浮按钮尺寸：56x56dp/40x40dp
- 用户头像尺寸：64x64dp/40x40dp
- 小图标点击区域：48x48dp
- 侧边抽屉到屏幕右边的距离：56dp
- 卡片间距：8dp
- 分隔线上下留白：8dp
- 大多元素的留白距离：16dp
- 屏幕左右对齐基线：16dp
- 文字左侧对齐基线：72dp
![enter image description here](http://7xjsjy.com1.z0.glb.clouddn.com/MD_a-33.png)
#### Button sheets
![enter image description here](http://7xjsjy.com1.z0.glb.clouddn.com/MD_button_sheet_1.png)
![enter image description here](http://7xjsjy.com1.z0.glb.clouddn.com/MD_button_sheet_2.png)
#### Buttons
&emsp;按钮分为悬浮按钮,凸起按钮和扁平按钮

- 悬浮按钮的操作都是正向操作
- 建议只用一个悬浮按钮
- 悬浮按钮可以贴在纸片边缘或者接缝处，但不要贴在对话框、侧边抽屉和菜单的边缘
- 悬浮按钮不能被其他元素盖住，也不能挡住其他按钮
- 列表滚动至底部时，悬浮按钮应该隐藏，防止它挡住列表项
悬浮按钮的位置不能随意摆放，可以贴着左右两边的对齐基线

#### Cards

- 卡片统一带有2dp的圆角
- 同时展现多种不同内容
- 卡片内容之间不需要进行比较
- 包含了长度不确定的内容，比如评论
- 包含丰富的内容与操作项，比如赞、滚动条、评论
- 本该是列表，但文字超过3行
- 本该是网格，但需要展现更多文字
![enter image description here](http://7xjsjy.com1.z0.glb.clouddn.com/MD_cards_1.png)
#### Dialogs

- 注意有些对话框点击周围会消失,但是有些是不会消失的
- 对话框的四周留白比较大，通常是24dp。
![enter image description here](http://7xjsjy.com1.z0.glb.clouddn.com/MD_dialog_1.png)
#### Dividers

- 列表中有头像、图片等元素时，使用内嵌分隔线，左端与文字对齐。
- 没有头像、图标等元素时，需要用通栏分隔线。
- 谨慎使用分隔线，留白和小标题也能起到分隔作用。能用留白的地方，优先使用留白。分隔线的层级高于留白。

#### Lists
- 列表由行构成，行内包含瓦片。如果列表项内容文字超过3行，请改用卡片。如果列表项的主要区别在于图片，请改用网格。 
![enter image description here](http://7xjsjy.com1.z0.glb.clouddn.com/MD_list_1.png)
![enter image description here](http://7xjsjy.com1.z0.glb.clouddn.com/MD_list_2.png)
#### Pickers
![日期选择器](http://7xjsjy.com1.z0.glb.clouddn.com/MD_pickers_1.png)
![时间选择器](http://7xjsjy.com1.z0.glb.clouddn.com/MD_pickers_1.png)
#### Progress & activity

- 进度条分为确定时间和不确定时间,不同的类型,动画效果不同
- 进度条可以使用在一些本来就有的元素上,比如悬浮按钮

#### Snackbars
![snackbars尺寸](http://7xjsjy.com1.z0.glb.clouddn.com/MD_snackbars_1.png)
#### Text fields
![Text1](http://7xjsjy.com1.z0.glb.clouddn.com/MD_text_fields_1.png)
![Text2](http://7xjsjy.com1.z0.glb.clouddn.com/MD_text_fields_2.png)
![Text3](http://7xjsjy.com1.z0.glb.clouddn.com/MD_text_fields_3.png)
#### NavigatorBar
![navigator尺寸](http://7xjsjy.com1.z0.glb.clouddn.com/MD_navigator_1.png)


