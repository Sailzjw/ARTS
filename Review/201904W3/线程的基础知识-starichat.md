
# 多线程的入门指导
> 原文：https://www.internalpointers.com/post/gentle-introduction-multithreading
> 翻译：starichat

现代计算机能够同时执行多个操作。在硬件改进和更智能的操作系统的支持下，这个特性使你的程序在执行速度和响应速度方面运行得更快。（Supported by hardware advancements and smarter operating systems, this feature makes your programs run faster, both in terms of speed of execution and responsiveness.）

编写采用这种性能的软件是既迷人又棘手的：它要求你去理解计算机底层发生了什么。在第一季中我们将尝试去触及线程的表面，这是操作系统提供的一种工具，用来执行这钟神奇的功能。现在就开始吧

## 进程和线程:以正确的方式命名
现代操作系统可以同时运行多个程序。这就是为什么您可以在浏览器（程序）中阅读本文，同时在您的媒体播放器（另一个程序）上听音乐。每个程序都被称为正在执行的进程。操作系统知道许多软件技巧，以使进程与其他进程一起运行，并利用底层硬件。无论哪种方式，最终结果是您感觉所有程序同时运行。

在操作系统中运行进程不是同时执行多个操作的唯一方法。每个进程都能够在其自身内部同时执行子任务，称为线程。您可以将线程视为流程本身的一部分。每个进程在启动时至少触发一个线程，称为主线程。然后，根据程序/程序员的需要，可以启动或终止其他线程。多线程是在单个进程中运行多个线程。

例如，您的媒体播放器可能会运行多个线程：一个用于呈现界面 - 这通常是主线程，另一个用于播放音乐，等等。

您可以将操作系统视为包含多个进程的容器，其中每个进程都是一个容纳多个线程的容器。在本文中，我将仅关注线程，但整个主题非常吸引人，并且值得在未来进行更深入的分析。

![image.png](https://s3.imgsha.com/2019/04/12/image.png)

## 进程与线程之间的差异

每个进程都有自己的操作系统分配的内存块。默认情况下，内存无法与其他进程共享：您的浏览器无法访问分配给您的媒体播放器的内存，反之亦然。如果您运行同一进程的两个实例，即两次启动浏览器，则会发生同样的情况。操作系统将每个实例视为一个新进程，并分配了自己独立的内存部分。因此，默认情况下，两个或多个进程无法共享数据，除非它们执行高级技巧 - 即所谓的进程间通信（IPC）。

与进程不同，线程共享由操作系统分配给其父进程的相同内存块：媒体播放器主界面中的数据可以由音频引擎轻松访问，反之亦然。因此，两个线程更容易相互通信。最重要的是，线程通常比进程更轻：它们占用的资源更少，创建速度更快，这就是为什么它们也被称为轻量级进程。

线程是使程序同时执行多个操作的便捷方式。如果没有线程，则必须为每个任务编写一个程序，将它们作为进程运行并通过操作系统进行同步。这将更加困难（IPC很棘手）而且速度较慢（流程比线程更重）。

绿色线，纤维
到目前为止提到的线程是操作系统的事情：想要触发新线程的进程必须与操作系统通信。但并非每个平台本身都支持线程。绿色线程（也称为光纤）是一种仿真，它使多线程程序在不提供该功能的环境中工作。例如，如果底层操作系统没有本机线程支持，则虚拟机可能会实现绿色线程。

绿色线程的创建和管理速度更快，因为它们完全绕过操作系统，但也有缺点。我会在下一集中写下这个话题。

“绿色线程”这个名称是指Sun Microsystem的Green Team，它在90年代设计了原始的Java线程库。今天Java不再使用绿色线程：它们在2000年转向本地线程。其他一些编程语言 - Go，Haskell或Rust等等 - 实现等效的绿色线程而不是本机线程。

## 线程被用作什么？
为什么进程应该使用多个线程？正如我之前提到的，（in parallel）并行处理可以大大加快速度。假设您要在电影编辑器中渲染电影。编辑器可以足够聪明，可以跨多个线程传播渲染操作，每个线程处理最终影片的一大块。因此，如果使用一个线程，任务将花费一个小时，两个线程需要30分钟; 用四个线程15分钟，依此类推。

它真的那么简单吗？有三个要点需要考虑：

- 1.并非每个程序都需要多线程。如果您的应用程序执行顺序操作或经常等待用户执行某些操作，多线程可能不是那么有用;
- 2.你只是不向应用程序抛出更多线程以使其运行更快：每个子任务都必须仔细考虑和设计以执行并行操作;
- 3.并非100％保证线程将真正并行执行其操作，同时：它实际上取决于底层硬件。

最后一个是至关重要的：如果您的计算机不同时支持多个操作，操作系统必须伪造它们。我们将在一分钟内看到。现在让我们将并发视为同时运行任务的感知，而将真正的并行视为同时运行的任务。


## 什么使并发和并行成为可能
在中央处理单元（CPU）在您的电脑上运行的程序的辛勤工作。它由几个部分组成，主要部分是所谓的核心：即实际执行计算的地方。核心一次只能运行一个操作。

这当然是一个主要缺点。出于这个原因，操作系统开发了先进的技术，使用户能够同时运行多个进程（或线程），尤其是在图形环境中，甚至在单核机器上。最重要的一种叫做抢占式多任务处理，抢占是指中断任务，切换到另一个任务然后在以后恢复第一个任务的能力。

因此，如果您的CPU只有一个核心，那么操作系统的一部分工作就是将该单核计算能力分散到多个进程或线程中，这些进程或线程在一个循环中一个接一个地执行。这个操作让你觉得有多个程序并行运行，或者一个程序同时执行多个程序（如果是多线程的）。并发性得到满足，但真正的并行性 - 同时运行进程的能力- 仍然缺失。

如今，现代CPU在引擎盖下有多个核心，每个核心一次执行独立操作。这意味着使用两个或更多内核可以实现真正的并行性。例如，我的英特尔酷睿i7有四个内核：它可以同时运行四个不同的进程或线程。

操作系统能够检测CPU核心的数量，并为每个核心分配进程或线程。线程可以分配给操作系统喜欢的任何核心，并且这种调度对于正在运行的程序是完全透明的。此外，如果所有内核都处于繁忙状态，可以启动抢占式多任务处理。这使您能够运行比计算机中可用的实际数量或核心数更多的进程和线程。

## 单核上的多线程应用程序：它有意义吗？
单核机器上的真正并行性是不可能实现的。然而，如果您的应用程序可以从中受益，那么编写多线程程序仍然是有意义的。当进程使用多个线程时，即使其中一个线程执行缓慢或阻塞任务，抢占式多任务也可以使应用程序保持运行。

比如说你正在开发一个从非常慢的磁盘读取一些数据的桌面应用程序。如果只用一个线程编写程序，整个应用程序将冻结，直到磁盘操作完成：分配给唯一线程的CPU功率在等待磁盘唤醒时被浪费。当然，操作系统除此之外还运行许多其他进程，但您的特定应用程序将不会取得任何进展。

让我们以多线程的方式重新思考您的应用。线程A负责磁盘访问，而线程B负责主接口。如果线程A由于设备运行缓慢而等待，则线程B仍然可以运行主界面，从而使程序保持响应。这是可能的，因为有两个线程，操作系统可以在它们之间切换CPU资源而不会卡在较慢的线程上。

 
## 更多线程，更多问题
众所周知，线程共享其父进程的相同内存块。这使得它们中的两个或更多个在同一应用程序内交换数据非常容易。例如：电影编辑器可能包含大部分包含视频时间轴的共享内存。这些共享内存正被指定用于将电影渲染到文件的几个工作线程读取。它们都只需要一个指向该存储区的句柄（例如指针），以便从中读取并将渲染帧输出到磁盘。

只要两个或多个线程从同一个内存位置读取，事情就会顺利进行。当至少其中一个人写入共享内存时，其他人正在从中读取问题。此时可能会出现两个问题：

数据争用 - 当编写器线程修改内存时，读者线程可能正在读取它。如果作者尚未完成其工作，读者将获得损坏的数据;

竞争条件 - 读者线程只有在作者写完后才能读取。如果相反的情况怎么办？比数据竞争更微妙，竞争条件是关于两个或更多线程以不可预测的顺序执行其工作，而实际上操作应该以正确的顺序执行以正确完成。您的程序即使受到数据竞争保护也可以触发竞争条件。

## 线程安全的概念
如果一段代码正常工作，即没有数据竞争或竞争条件，即使许多线程同时执行它，也会说它是线程安全的。您可能已经注意到某些编程库声明自己是线程安全的：如果您正在编写多线程程序，则需要确保可以跨不同线程使用任何其他第三方函数，而不会触发并发问题。

## 数据竞争的根本原因
我们知道CPU核心一次只能执行一条机器指令。这样的指令被认为是原子的，因为它是不可分割的：它不能分解成更小的操作。希腊语“atom”（ἄτομος; atomos）意味着不可切割。

不可分割的属性使原子操作本质上是线程安全的。当线程对共享数据执行原子写入时，没有其他线程可以读取修改半完成。相反，当线程对共享数据执行原子读取时，它会读取单个时刻出现的整个值。线程无法通过原子操作，因此不会发生数据争用。

坏消息是绝大多数的操作都是非原子的。即使像x = 1某些硬件上那样的微不足道的任务也可能由多个原子机器指令组成，这使得赋值本身就是非原子的。因此，如果线程读取x而另一个线程执行分配，则会触发数据争用。

种族条件的根本原因
抢占式多任务处理使操作系统可以完全控制线程管理：它可以根据高级调度算法启动，停止和暂停线程。您作为程序员无法控制执行的时间或顺序。实际上，无法保证像这样的简单代码：

writer_thread.start()
reader_thread.start()
将按特定顺序启动两个线程。多次运行此程序，您将注意到每次运行时它的行为方式如何：有时候编写器线程首先启动，有时候读者会这样做。如果您的程序需要编写器始终在读者之前运行，您肯定会遇到竞争状态。

此行为称为非确定性：结果每次都会更改，您无法预测。受竞争条件影响的调试程序非常烦人，因为您无法始终以受控方式重现问题。

教导线程相处：并发控制
数据竞赛和竞争条件都是现实世界的问题：有些人甚至因为他们而死亡。容纳两个或多个并发线程的技术称为并发控制：操作系统和编程语言提供了几种解决方案来处理它。最重要的是：

同步 - 一种确保资源一次只能由一个线程使用的方法。同步是将代码的特定部分标记为“受保护”，以便两个或多个并发线程不会同时执行它，从而搞砸了共享数据;

原子操作 - 由于操作系统提供的特殊指令，一堆非原子操作（如之前提到的赋值）可以转换为原子操作。这样，无论其他线程如何访问共享数据，共享数据始终保持有效状态;

不可变数据 - 共享数据被标记为不可变，没有任何东西可以改变它：只允许线程从中读取，消除了根本原因。我们知道线程可以安全地从相同的内存位置读取，只要它们不修改它。这是函数式编程背后的主要哲学。

我将在这个关于并发的迷你系列的下一集中介绍所有这些引人入胜的主题。敬请关注！

> sub-task:子任务
  并行
  安全