# 阅读器解析

阅读器具有的功能

1. 目录的显示 (当前页，已加载，未加载)
2. 夜间模式
3. 亮度调试(自动、自己设置)
4. 字体大小
5. 阅读的背景颜色
6. 范围点击弹出菜单
7. 翻页的动画效果
8. 文本的显示效果(根据字的大小，行间距显示文本)
9. 文本的来源(网络加载，还是存储，还有网络加载问题，是保留数据还是不保留数据，网络阅读的架构问题)
10. StatusBar的变化
11. 底部battery和时间、百分比的显示

作者是如何实现的

主要类:

ReadActivity:主要实现了显示切换还有一系列的点击事件。设置功能(夜间模式，亮度、目录、
字体大小的设置，文本的来源，StatusBar的变化)

BaseReadView:主要实现了加载文本上次的位置、判断点击事件(左边上一页，右边下一页，中间
弹出菜单并且判断是否相应翻页事件)、绘制文本页面的接口。总结而言实现的功能是对于点击事件
的判断并回调、开放翻页效果动画的接口、阅读位置的跳转与存储。之后所有的动画效果都是继承了
这个类。

PageFactory:其实BaserReadView还有的作用就是作为PageFactory的代理。PageFactory是
绘制的具体实现，绘制的内容有背景、字体大小、每页显示的文字、电池、时间、获取小说的段落等功能。

SettingManager:保存与恢复用户的阅读状态(性别、夜间模式、上一次阅读的节点、屏幕亮度、是否使用音量键翻阅等)

ThemeManager:设置阅读的背景的。

基本的了解完成了，我们现在来找难点在哪里，如何一步步的实现。

个人想法:

1. 参考掌阅还有UC阅读


## 实现阶段

1. 首先需要有一个可以显示含有菜单的界面，并实现展示目录的功能+设置版面的外形。

在设置之前，我们遇到的一个问题在于数据存储的设计(目录是否是永久保存的，会在退出的时候删除)。现在需要思考架构上的问题。
就是文章目录的问题，到底如何存储。

   1. 由于本身架构和能力的限制，当阅读的时候加载目录，并且默认缓存5章。当退出时候提示加入书架，并且提示不加入不保存进度

   2. 当书籍加入收藏的时候需要保存阅读的进度，还有章节的目录。这个应该由哪一方存储。

首先制作整体布局的问题，我需要的功能。 (大致布局框架完成)

### 阶段一

具体实现:

1. 实现菜单栏的显示和消失。 (完成)
2. 实现Toolbar顶部栏的显示和消失
3. 实现侧滑栏显示目录。(思考开始阅读和收藏下的不同状态 收藏保存目录保存状态。未收藏加载目录和状态)

沉浸式导航栏:在制作之前需要弄明白的几个问题

1. 如何将头部设置成半透明
2. 在半透明状态下，如何将布局不伸到StatusBar上
3. 如何设置隐藏StatusBar和NavigationBar，并且点击不会恢复，只有显示菜单的时候才恢复
4. 重新设置界面的StatusBar的问题。(1. 不设置StatusBar 2. 设置StatusBar 3. )

### 阶段二

具体实现:
1. 查看别人是如何构建阅读器，需要哪些类，每个类的作用是什么？
2. 自己如何组织如何实现。

根据treader作者代码的总结:

主要类分析:

PageWidget:原理是接收Bitmap作为展示，内部处理点击事件，最后通过AnimatorProvider处理翻页动画的类。对Bitmap进行处理
显示并绘制。在PageWidget中通过Scroller不断调用AnimatorProvider绘制翻页动画

AnimatorProvider:处理翻页动画接口类。真正的实现类是SimuationAnimator(仿真)、SlideAnimator(滑动)等。其真正的原理
是根据PageWidget传过来的具体点，然后对Bitmap进行转换而成。

PageFactory:制作Page的Factory。即创建显示在PageWidget中的Bitmap。

下面分析主要类的开发接口，以及做的具体的功能:

PageWidget:

开放的方法:

setCurrentPage(): 设置当前页面显示的Bitmap

setNextPage():设置下一个页面显示的bitmap

setBgColor():设置背景颜色 (不是被Bitmap覆盖了么，有什么用- -)

setPageMode():设置页面的切换模式

onTouchListener: 比如说是否存在下一页和上一页的判断接口。是否点击到中间的接口

整个的核心原理是TouchEvent事件的逻辑处理(详见PageWidget的onTouchEvent()事件)

这里的PageWidget还有AnimatorProvider都是照搬原作者的，我只开发PageFactory逻辑。

## 进度执行

1. 可以将head变成Toolbar，减小视图  (完成)
2. 可以将Setting移出来，定义为一个Dialog，减少ReadActivity的视图逻辑。(完成)
3. 重新修改SystemBar的弹出方式。(完成)
4. 实现屏幕的变化 (基本完成)
5. 实现PageFactory

PageFactory需要实现的东西:

1. 实现章节的显示(如何确保获取章节，然后根据视图获取每个page显示的行数量)
2. 实现电池的现实
3. 实现百分比的显示

## 如何实现PageFactory

1. 首先定义下载。首先从CollBook中查看当前进度，然后获取具体章节，再判断当前章节及其后五章是否已经下载，如果存在没有加载
就从网络中加载。之后将章节内容传递给PageFactory,PageFactory获取具体进度，并将数据进行分割，显示。每进行翻页的时候，
判断是否存在next如果不存在，则向ReadActivity告急，ReadActivity再从文本中导入数据给PageFactory,PageFactory再
进行切割。

顺序就是 ReadActivity传值给PageFactory->PageFactory切割并显示，当next章显示完了，再向ReadActivity进行请求。
ReadActivity在从数据库中加载文本(文本过程中下一章显示正在加载状态)。如果文本存在


数据下载的流程:

1. 初始化的时候从指定位置获取5章，然后放到下载队列中去，如果存在缓存则进行下载，然后村到数据库中。当下载完成一项的时候
就从文件中获取数据生成对象，然后执行PageFactory方法。(这里将文件转成对象是否妥当，如果文件特别大会造成内存溢出的问题)

2. PageFactory进行段落的分割，还有当前的进度设置问题。

ReadActivity获取章节，传递给PageFactory,PageFactory分割章节，并且跳转到上一次阅读的位置，最后显示。

这里文章是如何分割的。(是当达到下一章的时候，进行分割呢，还是全部分割完然后显示呢)

这里使用全部分割完再显示，那么如何制定有效的分割算法呢？

首先大致判断每行需要的数量，然后根据长度比较，如果超过则-1，如果没超过则+1。直到平衡。之后的段落分解就以初始的大小
为基准，进行分割判断。(其中我们还可以根据字体的大小，再判断可能需要多少个字符才能填满一行)当全部分解完毕

然后就是绘制的问题，(Reading通过控制PageFactory的nextPage和prePage获取对应页面，并显示)。如果页面不存在了，
马上装载下一章节进行显示。

重新构造思路。。。。分割线

1. 对章节的加载问题 流程介绍:首先进入的时候直接下载5章，对章节进行缓存。当最前面的一章缓存完成的时候，就提示PageFactory
显示。

2. PageFactory根据章节名，从缓存中获取文章，同时对内容使用Weak进行缓存。(问题，如何知道我阅读到了哪个章节呢，
CollBook会存储一个记录点，这个记录点存储了阅读到了哪个章节和章节的哪一段)

3. 如何进行段落的分割，当一个章节完成之后，如何载入下一个章节。 PageWidget通过getPre和getNext判断是否存在上一页。

这个时候首先对PageFactory进行getPre操作创建Page对象和Bitmap，在创建Page对象的过程中，进行段落分割。在段落分割的
时候判断是否已经到达最前面或者最后的章节，如果到达了最后的章节，那么就进行二次加载，即载入下一章节到PageFactory中。

那么如何分割段落，并且如何判断是否到达最后的章节，就是一个问题了。


剩下的就是如何进行绘制的问题，根据开放的样式进行调整的问题。

那么这些步骤该如何完成:

1. 首先完成背景的绘制流程。(通过setPageWidget的方式，然后进行初始化绘制，配置当前状态)

当进入ReadActivity，通过setPageWidget，初始化正在加载状态。如果Book已收藏，则获取chapter的位置，切换List，
导入后面5章到缓存中。当第一章加载完成后，改变PageFactory状态，进行数据的显示。PageFactory首先从缓存中获取数据，
然后再从文件中获取数据，进行显示。每次PageWidget调用getNext()的时候，PageFactory就会对是否加载到最后进行判断
如果是，则回到ReadActivity中，ReadActivity重新导入数据，并且将下一章的状态变成Running,直到回调完成位置。

上面的想法全部推翻了。。。。。之后会深入的写一篇原理。

1. 显示内容文本。(当前页的显示，下一页的显示，上一页的显示) (提供章节切换)
2. 绘制头部
3. 绘制底部的按钮


文本内容显示的问题:

1. 跳转到上一章的时候，如何保证空白部分。 (难道是直接将章节分割为一页？，反正分割方式肯定成了问题，需要改进)

2. 章节跳转之间存在显示问题。

3. 对于章节加载的优化

重置阅读分段算法，重新分析步骤:

1. 首先当进入阅读页面的时候自动加载5章，当文章加载完成时候，开启PageFactory。
2. PageFactory根据缓存位置，初始化BookManager，加载具体章节文件的缓存。
3. 将章节全部分割成TPage，设计页面分割算法。  (字体大小切换时候会很耗时，并且分割很耗时)

BookManager设计段落获取算法，然后在分割章节的时候，首先获取每个段落，通过段落填充lines，如果lines不满足则再次
获取段落。直到满足为止，最后如果存在多出来的段落，则重置position。

4. 上下章直接获取List<TPage>就可以了。

明天需要完成的任务:

1. 解决PrevLast代码绘制优化的问题 (完成)

2. 解决当每次点击下一章，加载后五章的代码架构问题。还有弱网环境下，等待加载，显示的问题

如果是向前翻页，则加载前面3章。如果是向后翻页就是加载后面三章。然后PageFactory直接从硬盘中获取。
如果没有从硬盘中获取到，需要等待网络加载的情况怎们办。
首先如果没有获取到数据，那么就进入LOADING状态，然后等待onChange回复，再次加载。
如果能够在加载中自动切换这个过程太复杂了，为了简化过程，采用如下方法:
如果使用章节跳转之后如果是正在加载，那么就显示正在加载，不允许翻页。
如果下一章或者上一章正在加载那么就提示正在加载，不允许翻页

显示加载上一章，后三章。

能不能对章节切换进行优化(每次切换都需要进行多次查询遍历，并且下一章已经在加载的话，会造成重复加载的问题)

侧滑栏的修改与优化(完成)

3. 缓存位置的问题  (对收藏的文件进行进度缓存，对没有收藏的书籍进行文件的清除)  (突然崩溃情况下的数据保存问题,ReadActivity的数据保存和回调，PageFactory的数据保存和回调)

(1. 退出的时候还在进行文章加载的问题 2. 退出时候删除数据造成的文件读写问题 3.当退出马上在进入造成的文件读写问题)
(重新判断是否缓存)
(明天的任务) (BookDetailActivity进入ReadActivity收藏的问题)
4. 菜单栏的配置
5. 页面的显示绘制
6. 目录页的切换   (完成)
7. 页面缩进的问题
8. 外拉StatusBar显示，然后就消失不掉了

之后就是:

1. 书籍删除。
2. 支持全部缓存和全部暂停。