---
layout: post
title: "Android平台Gallery2应用分析(一)---背景知识"
date: 2015-01-16 18:09:49 +0800
comments: true
categories: 
---

 Android平台Gallery2应用分析(一)---背景知识
分类： Android多媒体 2013-12-23 09:51 1631人阅读 评论(4) 收藏 举报
Android多媒体系统应用线程池Gallery2
Android系统概括来讲可分为GUI、多媒体以及网络相关三个部分，在学习了GUI部分如何去编写应用外，多媒体系统是接下来重点分析掌握的重点。本文着重介绍Android中的Gallery2应用以及该应用的框架设计。
概要：本文先对Gallery2中涉及的线程池ThreadPool，OpenGL ES的背景知识略作讲解。再以Gallery2应用从Launcher点击到查看图片的操作过程为线索， 将依次分析AlbumSetPage、AlbumPage、PhotoPage、PhotoView以及PhotoPage上的触屏操作代码流程。
背景知识
1.1 EGL介绍
        EGL是OpenGL ES和底层Native平台视窗系统之间的接口。EGL是为OpenGL ES提供平台独立性而设计的。OpenGL ES本质上是一个图形渲染管线的状态机，而EGL则是用于监控这些状态以及维护FrameBuffer和其他渲染Surface的外部层。
        EGL的数据类型：
EGL Boolean ：EGL_TRUE = 1， EGL_FALSE = 0
EGL int       : int 数据类型
EGLDisplay   : 系统显示ID或句柄
EGLConfig    : Surface的EGL配置
EGLSurface   : 系统窗口或framebuffer句柄
EGLContext   : OpenGL ES图形上下文
NativeDisplayType   : Native系统显示类型
NativeWindowType  : Native系统窗口缓存类型
NativePixmapType   : Native系统framebuffer


OpenGL ES的初始化过程：

         Surface实际上是一个FrameBuffer， 通过EGLSurface eglCreateWindowSurface(…)创建一个可显示的Surface。系统通常支持另两种Surface：PixmapSurface和PBufferSurface。这两种都不是可显示的Surface。
PixmapSurface：系统内存中的位图
PBufferSurface：保存在显存中的帧
应用程序通过OpenGL API进行绘制，一帧完成后，调用eglSwapBuffers来显示。


1.2 GLSurfaceView
GLSurfaceView是一个视图，继承至SurfaceView，它内嵌的surface专门负责OpenGL渲染。GLSurfaceView提供了下列特性：
1> 管理一个surface，这个surface就是一块特殊的内存，能直接排版到android的视图view上。
2> 管理一个EGL display，它能让opengl把内容渲染到上述的surface上。
3> 用户自定义渲染器(render)。
4> 让渲染器在独立的线程里运作，和UI线程分离。
5> 支持按需渲染(on-demand)和连续渲染(continuous)。
6> 一些可选工具，如调试。
使用GLSurfaceView 
通常会继承GLSurfaceView，并重载一些和用户输入事件有关的方法。如果你不需要重载事件方法，GLSurfaceView也可以直接使用，你可以使用set方法来为该类提供自定义的行为。例如，GLSurfaceView的渲染被委托给渲染器在独立的渲染线程里进行，这一点和普通视图不一 样，setRenderer(Renderer)设置渲染器。
初始化GLSurfaceView 
初始化过程其实仅需要你使用setRenderer(Renderer)设置一个渲染器(render)。当然，你也可以修改GLSurfaceView一些默认配置。
[java] view plaincopy在CODE上查看代码片派生到我的代码片
* setDebugFlags(int)  
* setEGLConfigChooser(boolean)  
* setEGLConfigChooser(EGLConfigChooser)  
* setEGLConfigChooser(int, int, int, int, int, int)  
* setGLWrapper(GLWrapper)  

getHolder().setFormat()的参数需要与setEGLConfigChooser的参数相匹配，否则就会失败。如果想设置surface背景为透明，代码参考如下：
surfaceView.setZOrderOnTop(true);
surfaceView.setEGLConfigChooser(8, 8, 8, 8, 16, 0);
surfaceView.getHolder().setFormat(PixelFormat.TRANSLUCENT);


定制android.view.Surface 
GLSurfaceView默认会创建像素格式为PixelFormat.RGB_565的surface。如果需要透明效果，调用 getHolder().setFormat(PixelFormat.TRANSLUCENT)。透明(TRANSLUCENT)的surface的像 素格式都是32位，每个色彩单元都是8位深度，像素格式是设备相关的，这意味着它可能是ARGB、RGBA或其它。
选择EGL配置 
Android设备往往支持多种EGL配置，可以使用不同数目的通道(channel)，也可以指定每个通道具有不同数目的位(bits)深度。因此，在 渲染器工作之前就应该指定EGL的配置。GLSurfaceView默认EGL配置的像素格式为RGB_656，16位的深度缓存(depth buffer)，默认不开启遮罩缓存(stencil buffer)。
如果你要选择不同的EGL配置，请使用setEGLConfigChooser方法中的一种。
调试行为 
你可以调用调试方法setDebugFlags(int)或setGLWrapper(GLSurfaceView.GLWrapper)来自定义 GLSurfaceView一些行为。在setRenderer方法之前或之后都可以调用调试方法，不过最好是在之前调用，这样它们能立即生效。
设置渲染器 
总之，你必须调用setRenderer(GLSurfaceView.Renderer)来注册一个GLSurfaceView.Renderer渲染器。渲染器负责真正的GL渲染工作。
渲染模式 
渲染器设定之后，你可以使用setRenderMode(int)指定渲染模式是按需(on demand)还是连续(continuous)。默认是连续渲染。
Activity生命周期 
Activity窗口暂停(pause)或恢复(resume)时，GLSurfaceView都会收到通知，此时它的onPause方法和 onResume方法应该被调用。这样做是为了让GLSurfaceView暂停或恢复它的渲染线程，以便它及时释放或重建OpenGL的资源。
事件处理 
为了处理事件，一般都是继承GLSurfaceView类并重载它的事件方法。但是由于GLSurfaceView是多线程操作，所以需要一些特殊的处 理。由于渲染器在独立的渲染线程里，你应该使用Java的跨线程机制跟渲染器通讯。queueEvent(Runnable)方法就是一种相对简单的操 作，例如下面的例子。
[java] view plaincopy在CODE上查看代码片派生到我的代码片
class MyGLSurfaceView extends GLSurfaceView {  
  
private MyRenderer mMyRenderer;  
  
public void start() {  
    mMyRenderer = ...;  
    setRenderer(mMyRenderer);  
}  
  
public boolean onKeyDown(int keyCode, KeyEvent event) {  
    if (keyCode == KeyEvent.KEYCODE_DPAD_CENTER) {  
        queueEvent(new Runnable() {  
            // 这个方法会在渲染线程里被调用  
            public void run() {  
                mMyRenderer.handleDpadCenter();  
            }});  
            return true;  
        }  
        return super.onKeyDown(keyCode, event);  
    }  
}  
(注：如果在UI线程里调用渲染器的方法，很容易收到“call to OpenGL ES API with no current context”的警告，典型的误区就是在键盘或鼠标事件方法里直接调用opengl es的API，因为UI事件和渲染绘制在不同的线程里。更甚者，这种情况下调用glDeleteBuffers这种释放资源的方法，可能引起程序的崩溃， 因为UI线程想释放它，渲染线程却要使用它。)

欢迎转载和技术交流，转载请帮忙注明出处，http://blog.csdn.net/discovery_by_joseph，谢谢！
