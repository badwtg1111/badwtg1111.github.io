---
layout: post
title: "Android平台Gallery2应用分析(七)---PhotoPage图片解码"
date: 2015-01-16 18:38:14 +0800
comments: true
categories: 
---
Android平台Gallery2应用分析(七)---PhotoPage图片解码
分类： Android多媒体 2013-12-23 16:15 1347人阅读 评论(1) 收藏 举报
AndroidGallery2应用相册多媒体
PhotoPage图片解码
从前文可知，PhotoPage的图片解码始于PhotoPage的onResume()调用updateImageRequests()。先看下代码：
[java] view plaincopy在CODE上查看代码片派生到我的代码片
private void updateImageRequests() {  
    ……  
    int currentIndex = mCurrentIndex;  
    MediaItem item = mData[currentIndex % DATA_CACHE_SIZE];  
    ……  
    // 1. 遍历sImageFetchSeq，查看当前图片符合哪种类型，调用startTaskIfNeeded  
    Future<?> task = null;  
    for (int i = 0; i < sImageFetchSeq.length; i++) {  
        int offset = sImageFetchSeq[i].indexOffset;  
        int bit = sImageFetchSeq[i].imageBit;  
        if (bit == BIT_FULL_IMAGE && !mNeedFullImage) continue;  
        task = startTaskIfNeeded(currentIndex + offset, bit);  
        if (task != null) break;  
    }  
  
    // 2. 释放任务和内存  
    for (ImageEntry entry : mImageCache.values()) {  
        if (entry.screenNailTask != null && entry.screenNailTask != task) {  
            entry.screenNailTask.cancel();  
            entry.screenNailTask = null;  
            entry.requestedScreenNail = MediaObject.INVALID_DATA_VERSION;  
        }  
        if (entry.fullImageTask != null && entry.fullImageTask != task) {  
            entry.fullImageTask.cancel();  
            entry.fullImageTask = null;  
            entry.requestedFullImage = MediaObject.INVALID_DATA_VERSION;  
        }  
    }  
接下来，重点分析startTaskIfNeeded()，看它是如何对一张图片做解析的。还是先看下面代码：
[java] view plaincopy在CODE上查看代码片派生到我的代码片
private Future<?> startTaskIfNeeded(int index, int which) {  
    if (index < mActiveStart || index >= mActiveEnd) return null;  
  
    ImageEntry entry = mImageCache.get(getPath(index));  
    if (entry == null) return null;  
    // 先得到当前图片，类型为LocalImage  
    MediaItem item = mData[index % DATA_CACHE_SIZE];  
    Utils.assertTrue(item != null);  
    long version = item.getDataVersion();  
      
    // 第一次代码执行暂时screenNailTask和fullImageTask为null  
    if (which == BIT_SCREEN_NAIL && entry.screenNailTask != null  
            && entry.requestedScreenNail == version) {  
        return entry.screenNailTask;  
    } else if (which == BIT_FULL_IMAGE && entry.fullImageTask != null  
            && entry.requestedFullImage == version) {  
        return entry.fullImageTask;  
    }  
    // 匹配到格式后，先创建ScreenNailJob、ScreenNailListener并返回screenNailTask  
    if (which == BIT_SCREEN_NAIL && entry.requestedScreenNail != version) {  
        entry.requestedScreenNail = version;  
        entry.screenNailTask = mThreadPool.submit(  
                new ScreenNailJob(item),  
                new ScreenNailListener(item));  
        // request screen nail  
        return entry.screenNailTask;  
    }  
配到格式后，先创建FullImageJob、FullImageListener并返回fullImageTask  
    if (which == BIT_FULL_IMAGE && entry.requestedFullImage != version  
            && (item.getSupportedOperations()  
            & MediaItem.SUPPORT_FULL_IMAGE) != 0) {  
        entry.requestedFullImage = version;  
        entry.fullImageTask = mThreadPool.submit(  
                new FullImageJob(item),  
                new FullImageListener(item));  
        // request full image  
        return entry.fullImageTask;  
    }  
    return null;  
参数which就是静态数组sImageFetchSeq中的图片类型，android原生代码默认两种BIT_SCREEN_NAIL和BIT_FULL_IMAGE。当然我们也可以自己加入一种解析图片的格式，例如BIT_GIF_IMAGE。在详细分析下面的代码后，详细你自己加入一种图片格式应该问题不大。流程大致如下：
1） 从传入参数index获取当前图片MediaItem，图片为LocalImage类型。第一次执行时screenNailTask和fullImageTask为null，匹配到格式后，创建ScreenNailJob、ScreenNailListener（或者screenNailTask和fullImageTask为null），并最终返回screenNailTask或者fullImageTask。
2） 以FullImage为例。ThreadPool的机制这里再大致讲述一下。看下mThreadPool.submit的代码：
[java] view plaincopy在CODE上查看代码片派生到我的代码片
public <T> Future<T> submit(Job<T> job, FutureListener<T> listener) {  
    Worker<T> w = new Worker<T>(job, listener);  
    mExecutor.execute(w);  
    return w;  
}  
由前文分析，execute会启动Worker的线程进入run()函数：
[java] view plaincopy在CODE上查看代码片派生到我的代码片
@Override  
 public void run() {  
     T result = null;  
  
     // A job is in CPU mode by default. setMode returns false  
     // if the job is cancelled.  
     if (setMode(MODE_CPU)) {  
         try {  
             // 1、执行job的run()  
             result = mJob.run(this);  
         } catch (Throwable ex) {  
             Log.w(TAG, "Exception in running a job", ex);  
         }  
     }  
  
     synchronized(this) {  
         setMode(MODE_NONE);  
         mResult = result;  
         mIsDone = true;  
         notifyAll();  
     }  
     // 2、执行完毕，调用listener的onFutureDone  
     if (mListener != null) mListener.onFutureDone(this);  
 }  
这里面分两步：
        2.1）执行job的run()。这里会调用传入参数new FullImageJob (item).run()。
[java] view plaincopy在CODE上查看代码片派生到我的代码片
private class FullImageJob implements Job<BitmapRegionDecoder> {  
    private MediaItem mItem;  
  
    public FullImageJob(MediaItem item) {  
        mItem = item;  
    }  
  
    @Override  
    public BitmapRegionDecoder run(JobContext jc) {  
        if (isTemporaryItem(mItem)) {  
            return null;  
        }  
        return mItem.requestLargeImage().run(jc);  
    }  
}  
这段代码实际就是在线程池的某个线程中执行LocalImage的requestLargeImage().run(jc)。而该函数会创建一个LocalLargeImageRequest对象，而run(jc)实际就是DecodeUtils.createBitmapRegionDecoder(jc, mLocalFilePath, false)，最终创建一个BitmapRegionDecoder实例。该实例会调用JNI层的BitmapRegionDecoder中的nativeNewInstanceFromStream接口，在doBuildTileIndex(JNIEnv* env, SkStream* stream)里可以看到，图片会根据stream的header判断解码器是SkJPEGImageDecoder，SkPNGImageDecoder，SkBMPImageDecoder还是SkWEBPImageDecoder等，最后得到解码数据。
        2.2） 调用FullImageListener的onFutureDone。此时会将2.1)中创建好的BitmapRegionDecoder实例返回传给mFuture。而FullImageListener则再发送一个MSG_RUN_OBJECT消息给MainThread，MainThread再执行FullImageListener的run()，即再执行updateFullImage(mPath, mFuture)。
[java] view plaincopy在CODE上查看代码片派生到我的代码片
    private void updateFullImage(Path path, Future<BitmapRegionDecoder> future) {  
        ImageEntry entry = mImageCache.get(path);  
        ……  
        entry.fullImageTask = null;  
        entry.fullImage = future.get();  
        if (entry.fullImage != null) {  
            if (path == getPath(mCurrentIndex)) {  
                updateTileProvider(entry);  
                mPhotoView.notifyImageChange(0);  
            }  
        }  
        updateImageRequests();  
}  
其中mPhotoView.notifyImageChange(0)取到当前图片后并reload，其中mPictures当前为FullPicture。
[java] view plaincopy在CODE上查看代码片派生到我的代码片
public void notifyImageChange(int index) {  
    ……  
    mPictures.get(index).reload();  
    ……  
    invalidate();  
那么再看看FullPicture的reload(), 该函数会对mTileView的screenNail做更新。
[java] view plaincopy在CODE上查看代码片派生到我的代码片
@Override  
public void reload() {  
    // mImageWidth and mImageHeight will get updated  
    mTileView.notifyModelInvalidated();  
    ……  
    setScreenNail(mModel.getScreenNail(0));  
    ……  
}  
代码段中的mModel就是PhotoDataAdapter。mModel.getScreenNail(0)得到当前图片的ScreenNail以做更新。由此可知，我们看到的全屏的图片，就是TileImageView类型的。
欢迎转载和技术交流，转载请帮忙注明出处，http://blog.csdn.net/discovery_by_joseph，谢谢！

