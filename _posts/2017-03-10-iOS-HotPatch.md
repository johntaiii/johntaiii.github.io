---

title: iOS App的几种HotPatch方案

description: iOS App的几种HotPatch方案

header: iOS App的几种HotPatch方案

---

目前针对iOS平台上App的动态修复机制（HotFix）大致有四种方案：

1. 使用率最高的JSPatch/wax/rollout是先把OC手动翻译成脚本语言，通过服务端下发后，在运行时阶段利用OC的动态特性去调用和替换OC方法，实现实时修复bug。

2. 游戏客户端开发中常用的通过服务端下发lua脚本，动态执行并调用游戏引擎提供的函数。这种与ReactNative、Weex以及微信小程序类似，只能执行引擎（框架）已经封装的函数，并不能动态调用到iOS系统的任意API。

3. 滴滴的DynamicCocoa是从编译阶段入手，通过Clang把OC代码编译成自己定制的JS格式，再动态下发去执行，做到直接用原生OC编写补丁代码，动态运行，主打动态添加功能。

4. 手机QQ的OCS是定义了一套描述OC语义的字节码指令集(OCScript)，开发了一套基于Clang的全自动编译器（OCSCompiler），实现了一个高性能的虚拟机（OCSVM）以及一个可以跟底层对接的桥接器(OCSBridge)。首先使用OCS编译器把OC源码转化成OCS字节码，然后通过OCS桥接器实现OCS虚拟机与Native运行时的互联，最后使用OCS虚拟机对OCS字节码进行解释运算，并驱动Native运行时完成逻辑的执行，以此达到Native代码动态化的效果。

这次苹果警告去掉动态下发功能，综合网上各方信息来看，像ReactNative/Weex/小程序这样用JS做功能的并没有受到影响，主要禁的还是JSPatch/wax/rollout这样的热修复框架，这些框架的特点是可以通过JS脚本调用和替换任意OC方法。但是也有用户反馈他们在此之前对JSPatch中的关键字做了混淆，这次并没有收到邮件。所以Apple这次可能没有做详细的检测，只是在反编代码中检测到了关键字就发送warning邮件。

