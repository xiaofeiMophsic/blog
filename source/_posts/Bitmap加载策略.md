---
title: Bitmap加载策略
author: xiaofei
date: 2016-08-24 10:04:05
tags: [android, bitmap]
categories:
- android
---
<!-- toc -->
本篇文章内容观点均来自于[android官网](https://developer.android.com/training/displaying-bitmaps/load-bitmap.html#load-bitmap)，欢迎转载，转载请注明出处。文中有不对的地方欢迎交（tu）流（cao）。

## 前言    
在app开发过程中，尤其是涉及到图片比较多的页面中，如果图片很大，不对图片做优化处理的话，很容易导致应用崩溃，并抛出如下异常：     
`java.lang.OutofMemoryError: bitmap size exceeds VM budget.`   
<!-- more -->
虽然有很多优秀的开源图片加载框架可供选择，比如Facebook的`fresco`，它对图片的的内存管理有更好的方式，比如真正实现了三级缓存（两级内存缓存和一级的磁盘缓存），并且使用起来也很简单。但是我们还是有必要对bitmap的加载方式有更深的了解，知其然更要知其所以然，从根本上杜绝OOM的发生。     

一般来讲，android 应用处理`bitmap`很多问题是由以下几个原因引起的：
* android设备本身内存限制。设备分配给应用的内存容量是有限的（16M），所以，你的应用应该做好优化使之占用内存空间在最小限制之下；
* bitmap占用大量内存空间，尤其是手机摄像图片。比如一张2592x1936像素的图片，如果图片采用ARGB_8888（ARGB32位）彩色格式，那么加载这张完整图片大概需要19M的内存空间，会立即耗尽设备分配给应用的内存，导致应用崩溃；
* android应用组件如`ListView`, `GridView`会一次性加载多张图片。

## 高效加载大Bitmap
### 获取Bitmap的大小和类型
如果只是获取bitmap的大小和类型的话，可以通过设置`BitmapFactory.Options#inJustDecodeBounds`字段为`true`，然后再去解析`bitmap`。设置了这个属性之后，`BitmapFactory.decodeResource()`会返回一个值为`null`的`bitmap`，也就是说，这个方法并不会为`bitmap`分配内存空间，但是却可以获取到图片的宽高和类型。如以下代码：

```java
BitmapFactory.Options options = new BitmapFactory.Options();
options.inJustDecodeBounds = true;// 这个属性需要在decode之前设置
BitmapFactory.decodeResource(getResources(), R.id.myimage, options);
int imageHeight = options.outHeight;
int imageWidth = options.outWidth;
String imageType = options.outMimeType;
```

建议加载图片之前先计算下图片大小是否会引发OOM，除非你能确定图片大小不会占用很多内存。
### 加载按比例缩放的bitmap
经过上一个步骤我们已经获取到了图片的宽高和类型信息，这些信息可以用来决定是否加载完整的图片，否则的话，我们就要进行二级采样来代替原图，下面是一些需要考虑的因素：
* 预估加载完整图片所占用的内存大小；
* 加载这张图片到内存中时需要考虑应用的其他的内存需求；
* 加载图片目标组件（ImageView）或者其他组件的大小；
* 当前设备屏幕密度。

例如，加载一张大分辨率的图片到一个显示缩略图的`ImageView`中，可以通过`inSampleSize`设置采样大小。比如设置`inSampleSize=4`表示加载bitmap宽高均为原图的1/4。

```java
public static int calculateInSampleSize(
    BitmapFactory.Options options, int reqWidth, int reqHeight) {
    // Raw height and width of image
    final int height = options.outHeight;
    final int width = options.outWidth;
    int inSampleSize = 1;

    if (height > reqHeight || width > reqWidth) {

        final int halfHeight = height / 2;
        final int halfWidth = width / 2;

        // Calculate the largest inSampleSize value that is a power of 2 and keeps both
        // height and width larger than the requested height and width.
        while ((halfHeight / inSampleSize) >= reqHeight
                && (halfWidth / inSampleSize) >= reqWidth) {
            inSampleSize *= 2;
        }
    }

    return inSampleSize;
}
```
上面代码表示以2的指数为基准，获取最接近指定宽高的采样率。

在使用上面方法之前，需要设置`inJustDecodeBounds`属性为`true`，在获取到宽高之后呢，再将其设置为`false`，然后再解码bitmap。代码如下：
```java
public static Bitmap decodeSampledBitmapFromResource(Resources res, int resId,
        int reqWidth, int reqHeight) {

    // First decode with inJustDecodeBounds=true to check dimensions
    final BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    BitmapFactory.decodeResource(res, resId, options);

    // Calculate inSampleSize
    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);

    // Decode bitmap with inSampleSize set
    options.inJustDecodeBounds = false;
    return BitmapFactory.decodeResource(res, resId, options);
}
```
使用上面的方法就可以很方便的将bitmap加载到`ImageView`中并且以缩略图的方式显示出来，具体使用方式如下代码所示：
```java
mImageView.setImageBitmap(decodeSampledBitmapFromResource(getResources(), R.id.myimage, 100, 100));
```
你可以使用类似的方式去处理来自其他源的bitmap，无非就是替换掉对应的`BitmapFactory.decode*`方法。

## 不要在UI线程中处理Bitmap
在使用`BitmapFactory.decode*`加载网络或者磁盘上的图片的时候，由于多种因素，加载图片所用时间无法预知，如果在UI线程中去加载的话，往往会阻塞UI线程，极易造成应用程序无响应。因此本章节内容主要介绍使用后台异步任务加载bitmap。

### 使用AsyncTask

`AsyncTask`类为我们提供了一种很简单的方式去执行一些任务，并且可以把任务的执行结果发布到主线程中。要想使用这个类，我们先创建它的一个子类，并重写相关方法。示例代码如下：
```java
class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
    private final WeakReference<ImageView> imageViewReference;
    private int data = 0;

    public BitmapWorkerTask(ImageView imageView) {
        // Use a WeakReference to ensure the ImageView can be garbage collected
        imageViewReference = new WeakReference<ImageView>(imageView);
    }

    // Decode image in background.
    @Override
    protected Bitmap doInBackground(Integer... params) {
        data = params[0];
        // 查看上一章节提供的方法
        return decodeSampledBitmapFromResource(getResources(), data, 100, 100));
    }

    // Once complete, see if ImageView is still around and set bitmap.
    @Override
    protected void onPostExecute(Bitmap bitmap) {
        if (imageViewReference != null && bitmap != null) {
            final ImageView imageView = imageViewReference.get();
            if (imageView != null) {
                imageView.setImageBitmap(bitmap);
            }
        }
    }
}
```
简单解释下上面的实例代码。使用`WeakReference`确保`ImageView`使用结束之后能被垃圾回收，在`onPostExecute`方法中对`imageView`作空值校验是因为我们并不能保证在执行`onPostExecute`的时候`ImageView`实例一直存在，如果后台任务完成之前配置发生变化比如切换屏幕方向，那么就有可能导致`ImageView`不在了。      
使用下面代码开始这个异步任务：
```java
public void loadBitmap(int resId, ImageView imageView) {
    BitmapWorkerTask task = new BitmapWorkerTask(imageView);
    task.execute(resId);
}
```
### 处理并发
android UI中一些公共组件如`ListView`和`GridView`等在结合上个小节演示的`AsyncTask`一起工作的时候会引入另外一个问题。为了节省内存空间，这些组件会在用户滑动控件的时候回收不可见的子视图。问题是不能保证在每一次子view执行`AsyncTask`完之后，它所关联的view能够被回收以便于显示下一个view，此外，也不能保证异步任务开始和结束顺序的一致性。      
之前有一篇文章[Multithreading For Performance](http://android-developers.blogspot.com/2010/07/multithreading-for-performance.html)（这篇文章很早以前了啊）提供了一种解决方式，就是在`ImageView`保存了最近的`AsyncTask`引用，然后异步任务完成的时候再拿出来。我们可以使用类似的方式，将前面定义的`AsyncTask`改造成下面的模式。    
首先创建一个`Drawable`的子类用于存储专门返回工作任务的引用，在这种情况下，可以使用`BitmapDrawable`作为任务完成后的显示在`ImageView`里的图像占位符。示例代码：
```java
static class AsyncDrawable extends BitmapDrawable {
    private final WeakReference<BitmapWorkerTask> bitmapWorkerTaskReference;

    public AsyncDrawable(Resources res, Bitmap bitmap, BitmapWorkerTask bitmapWorkerTask) {
        super(res, bitmap);
        bitmapWorkerTaskReference = new WeakReference<BitmapWorkerTask>(bitmapWorkerTask);
    }

    public BitmapWorkerTask getBitmapWorkerTask() {
        return bitmapWorkerTaskReference.get();
    }
}
```
在执行`BitmapWorkerTask`之前，把创建好的`AsyncDrawable`绑定到目标`ImageView`上。
```java
// 这个算是上一节loadBitmap()方法的改进版
public void loadBitmap(int resId, ImageView imageView) {
    if (cancelPotentialWork(resId, imageView)) {
        final BitmapWorkerTask task = new BitmapWorkerTask(imageView);
        final AsyncDrawable asyncDrawable = new AsyncDrawable(getResources(), mPlaceHolderBitmap, task);
        imageView.setImageDrawable(asyncDrawable);
        task.execute(resId);
    }
}
```
上面代码中`cancelPotentialWork()`这个方法的作用主要是判断目标`ImageView`是否已经关联了一个正在运行着不同的任务，如果是的话，就取消正在运行的任务。
```java
public static boolean cancelPotentialWork(int data, ImageView imageView) {
    final BitmapWorkerTask bitmapWorkerTask = getBitmapWorkerTask(imageView);

    if (bitmapWorkerTask != null) {
        final int bitmapData = bitmapWorkerTask.data;
        // If bitmapData is not yet set or it differs from the new data
        if (bitmapData == 0 || bitmapData != data) {
            // Cancel previous task
            bitmapWorkerTask.cancel(true);
        } else {
            // The same work is already in progress
            return false;
        }
    }
    // No task associated with the ImageView, or an existing task was cancelled
    return true;
}
```
`getBitmapWorkerTask(imageView)`是个辅助方法，用于查询关联在`ImageView`上的任务。
```java
private static BitmapWorkerTask getBitmapWorkerTask(ImageView imageView) {
   if (imageView != null) {
       // 获取通过imageView.setImageDrawable(asyncDrawable);设置的drawable
       final Drawable drawable = imageView.getDrawable();
       if (drawable instanceof AsyncDrawable) {
           final AsyncDrawable asyncDrawable = (AsyncDrawable) drawable;
           // 获取关联的Task
           return asyncDrawable.getBitmapWorkerTask();
       }
    }
    return null;
}
```
最后一步就是修改下`BitmapWorkerTask`的`onPostExecute()`方法，判断当前执行的任务是否取消了或者是否匹配先前关联在`ImageView`上的任务。
```java
class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
    //...

    @Override
    protected void onPostExecute(Bitmap bitmap) {
        if (isCancelled()) {
            bitmap = null;
        }

        if (imageViewReference != null && bitmap != null) {
            final ImageView imageView = imageViewReference.get();
            final BitmapWorkerTask bitmapWorkerTask = getBitmapWorkerTask(imageView);
            if (this == bitmapWorkerTask && imageView != null) {
                imageView.setImageBitmap(bitmap);
            }
        }
    }
}
```
ok, `ListView`和`GridView`的问题解决了，现在你就可以直接使用`loadBitmap`方法了。例如，在`GridView`的`Adapter#getView()`方法中直接调用它就可以了。

## 缓存Bitmap
直接使用缓存过的Bitmap要比重新加载Bitmap在UI上流畅的多，因此缓存就很有必要了啊。这里缓存主要分内存缓存和磁盘缓存来讲。
### 使用内存缓存
针对内存缓存算法，这里主要介绍LRU Cache（最近最少使用缓存淘汰算法，关于LRU cache，可以看[这篇](http://flychao88.iteye.com/blog/1977653)比较接地气的文章）。LRU特别适合用来缓存bitmap，保持最近使用对象在一个强引用的`LinkedHashMap`中，在缓存达到指定的容量的时候删除（evicting）最少使用的对象。  
> 在过去，缓存Bitmap常用的实现是使用软/弱引用的方式，这种方式强烈**不推荐**。因为从2.3开始，GC对于软/弱引用对象的回收更加具有侵略性，导致这种方式失败。另外，在android 3.0以前，bitmap的数据是存放在native memory中的，而native memory的垃圾回收是不可控的，因为它不受GC的管理。有可能导致应用由于暂时超过内存限制引发崩溃。       

为了给`LruCache`选择合适的大小，请考虑以下几个因素：
* 其余Activity和Application的内存使用情况；
* 一次性显示在屏幕上图片的数量和准备好随时显示在屏幕上图片的数量；
* 设备屏幕大小和像素密度。在显示同样多图片的基础上高分屏需要更大的缓存空间；
* 图片的尺寸信息和每张图片各自占用的内存情况
* 图片被访问的频率，以及根据图片访问频率的不同可能需要对图片进行分组缓存
* 图片的质量和数量的权衡。先存一堆低分辨率图片，然后再后台任务中再去加载高分辨率图片，这种方式有时更有用。

关于`LruCache`的大小没有一个指定的准则，由有你去决定缓存大小。下面是一个代码示例：
```java
private LruCache<String, Bitmap> mMemoryCache;

@Override
protected void onCreate(Bundle savedInstanceState) {
    ...
    // Get max available VM memory, exceeding this amount will throw an
    // OutOfMemory exception. Stored in kilobytes as LruCache takes an
    // int in its constructor.
    // 虚拟机最大可用内存
    final int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);

    // Use 1/8th of the available memory for this memory cache.
    final int cacheSize = maxMemory / 8;

    mMemoryCache = new LruCache<String, Bitmap>(cacheSize) {
        @Override
        protected int sizeOf(String key, Bitmap bitmap) {
            // The cache size will be measured in kilobytes rather than
            // number of items.
            return bitmap.getByteCount() / 1024;
        }
    };
    ...
}

public void addBitmapToMemoryCache(String key, Bitmap bitmap) {
    if (getBitmapFromMemCache(key) == null) {
        mMemoryCache.put(key, bitmap);
    }
}

public Bitmap getBitmapFromMemCache(String key) {
    return mMemoryCache.get(key);
}
```
>说明：在上面程序中，应用可用内存的1/8分配给缓存，在hdpi的设备上，这个值大概4MB(8/32)。在充满图片的全屏的GridView控件中，在800*480分辨率下大概使用1.5MB(800*480*4)内存，所以，这个内存大概可以缓存2.5页的图片。

下面是一个使用LRU缓存的示例程序：
```java
public void loadBitmap(int resId, ImageView imageView) {
    final String imageKey = String.valueOf(resId);

    // 先从缓存中取图片，如果没有的话，则使用后台task去加载图片
    final Bitmap bitmap = getBitmapFromMemCache(imageKey);
    if (bitmap != null) {
        mImageView.setImageBitmap(bitmap);
    } else {
        mImageView.setImageResource(R.drawable.image_placeholder);
        BitmapWorkerTask task = new BitmapWorkerTask(mImageView);
        task.execute(resId);
    }
}
```
这里用到的`BitmapWorkerTask`需要做一点变化:
```java
class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
    ...
    // Decode image in background.
    @Override
    protected Bitmap doInBackground(Integer... params) {
        final Bitmap bitmap = decodeSampledBitmapFromResource(
                getResources(), params[0], 100, 100));
        //加载完图片之后，将其添加到缓存里
        addBitmapToMemoryCache(String.valueOf(params[0]), bitmap);
        return bitmap;
    }
    ...
}
```
### 使用磁盘缓存
虽然使用内存缓存提高图片访问速度，但是由于内存容量有限，并且当应用处于后台的时候，内存缓存很容易被清理掉，当应用恢复的时候还得重复去处理图片，这个时候磁盘缓存就应运而生。使用磁盘可以持久化图片并且可以在内存缓存不可用的时候降低图片的装载次数，当然，从磁盘上获取图片肯定要比直接在内存中获取要慢，并且加载的过程也需要放到后台任务中去做，同样，从磁盘中读取图片的时间也是不可预知的。
> 提示，如果访问图片的操作很频繁的话，`ContentProvider`更加适合用于存放缓存的图片。

下面是一个磁盘缓存的示例程序：
```java
private DiskLruCache mDiskLruCache;
private final Object mDiskCacheLock = new Object();
private boolean mDiskCacheStarting = true;
private static final int DISK_CACHE_SIZE = 1024 * 1024 * 10; // 10MB
private static final String DISK_CACHE_SUBDIR = "thumbnails";

@Override
protected void onCreate(Bundle savedInstanceState) {
    ...
    // Initialize memory cache
    ...
    // Initialize disk cache on background thread
    File cacheDir = getDiskCacheDir(this, DISK_CACHE_SUBDIR);
    new InitDiskCacheTask().execute(cacheDir);
    ...
}

class InitDiskCacheTask extends AsyncTask<File, Void, Void> {
    @Override
    protected Void doInBackground(File... params) {
        synchronized (mDiskCacheLock) {
            File cacheDir = params[0];
            mDiskLruCache = DiskLruCache.open(cacheDir, DISK_CACHE_SIZE);
            mDiskCacheStarting = false; // Finished initialization
            mDiskCacheLock.notifyAll(); // Wake any waiting threads
        }
        return null;
    }
}

class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
    ...
    // Decode image in background.
    @Override
    protected Bitmap doInBackground(Integer... params) {
        final String imageKey = String.valueOf(params[0]);

        // Check disk cache in background thread
        Bitmap bitmap = getBitmapFromDiskCache(imageKey);

        if (bitmap == null) { // Not found in disk cache
            // Process as normal
            final Bitmap bitmap = decodeSampledBitmapFromResource(
                    getResources(), params[0], 100, 100));
        }

        // Add final bitmap to caches
        addBitmapToCache(imageKey, bitmap);

        return bitmap;
    }
    ...
}

public void addBitmapToCache(String key, Bitmap bitmap) {
    // Add to memory cache as before
    if (getBitmapFromMemCache(key) == null) {
        mMemoryCache.put(key, bitmap);
    }

    // Also add to disk cache
    synchronized (mDiskCacheLock) {
        if (mDiskLruCache != null && mDiskLruCache.get(key) == null) {
            mDiskLruCache.put(key, bitmap);
        }
    }
}

public Bitmap getBitmapFromDiskCache(String key) {
    synchronized (mDiskCacheLock) {
        // Wait while disk cache is started from background thread
        // 等待直到DiskCache初始化结束
        while (mDiskCacheStarting) {
            try {
                mDiskCacheLock.wait();
            } catch (InterruptedException e) {}
        }
        if (mDiskLruCache != null) {
            return mDiskLruCache.get(key);
        }
    }
    return null;
}

// Creates a unique subdirectory of the designated app cache directory. Tries to use external
// but if not mounted, falls back on internal storage.
public static File getDiskCacheDir(Context context, String uniqueName) {
    // Check if media is mounted or storage is built-in, if so, try and use external cache dir
    // otherwise use internal cache dir
    // 如果有SD卡的话使用SD卡作为缓存路径
    final String cachePath =
            Environment.MEDIA_MOUNTED.equals(Environment.getExternalStorageState()) ||
                    !isExternalStorageRemovable() ? getExternalCacheDir(context).getPath() :
                            context.getCacheDir().getPath();

    return new File(cachePath + File.separator + uniqueName);
}
```
使用内存缓存获取图片的操作是在主线程中，而使用磁盘缓存读取图片的操作是在后台任务中，对磁盘的操作绝对不要放到主线程中去做，对图片缓存的时候需要同时将图片缓存到内存和磁盘中。
### 处理配置更改
主要针对运行时用户配置发生变化时的处理情况，如屏幕旋转操作。下面一个程序示例使用了`Fragment`特性`setRetainInstance()`获取缓存对象的操作，这个用法很巧妙啊。
```java
private LruCache<String, Bitmap> mMemoryCache;

@Override
protected void onCreate(Bundle savedInstanceState) {
    ...
    RetainFragment retainFragment =
            RetainFragment.findOrCreateRetainFragment(getFragmentManager());
    mMemoryCache = retainFragment.mRetainedCache; //对缓存做恢复操作
    if (mMemoryCache == null) {
        mMemoryCache = new LruCache<String, Bitmap>(cacheSize) {
            ... // Initialize cache here as usual
        }
        retainFragment.mRetainedCache = mMemoryCache;
    }
    ...
}

class RetainFragment extends Fragment {
    private static final String TAG = "RetainFragment";
    public LruCache<String, Bitmap> mRetainedCache;

    public RetainFragment() {}

    public static RetainFragment findOrCreateRetainFragment(FragmentManager fm) {
        RetainFragment fragment = (RetainFragment) fm.findFragmentByTag(TAG);
        if (fragment == null) {
            fragment = new RetainFragment();
            fm.beginTransaction().add(fragment, TAG).commit();
        }
        return fragment;
    }

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setRetainInstance(true); //设置这个属性可以在activity重新重建的时候获取先前的fragment
    }
}
```
## Bitmap内存管理
紧接着上面的章节，android给出了一些有利于垃圾回收和bitmap重用的建议，这些建议和你的app的target API版本有关。
>* 在2.2及其之前的版本中，当发生gc的时候，会挂起app所有的线程（感觉整个世界都停了），引发卡顿并且降低性能。而在2.3的时候，增加了并发gc的机制，也就意味着在bitmap不在被引用的时候，它所占用的内存可以及时回收；
* 在2.3.3及其之前的版本中，bitmap的像素数据是存放到native memory中的，这和存放到dalvik heap中的bitmap对象本身是分开的。由于存放到native memory中的数据释放时机是不可控的（不受GC的管理），有可能触发低内存限制导致崩溃。而在3.0的时候，bitmap的像素数据也存放到dalvik heap中，并且和bitmap对象关联起来。

针对以上情况，下面也要对不同的android版本进行分开描述。
### 2.3.3 及其之前版本的内存管理
在2.3.3及之前版本建议在不使用bitmap的时候调用`recyle()`方法进行回收，如果加载大量bitmap在你的应用中，极易引发OOM，`recyle()`方法能够使app尽量回收不用内存。
> 需要注意的是：一定要在你确定bitmap不在被使用的时候调用`recyle()`方法，否则的话会导致以下错误：`"Canvas: trying to use a recycled bitmap"`。

下面的代码片段使用了引用计数的方式来确定显示和缓存的bitmap，当满足下面两个条件的时候，bitmap会被回收：
* 引用计数变量`mDisplayRefCount`和`mCacheRefCount`的值同时为0；
* bitmap不为`null`并且未被回收。

```java
private int mCacheRefCount = 0;
private int mDisplayRefCount = 0;
...
// Notify the drawable that the displayed state has changed.
// Keep a count to determine when the drawable is no longer displayed.
// 图像的显示状态发生变化的时候被调用。
public void setIsDisplayed(boolean isDisplayed) {
    synchronized (this) {
        if (isDisplayed) {
            mDisplayRefCount++;
            mHasBeenDisplayed = true;
        } else {
            mDisplayRefCount--;
        }
    }
    // Check to see if recycle() can be called.
    checkState();
}

// Notify the drawable that the cache state has changed.
// Keep a count to determine when the drawable is no longer being cached.
// 图像缓存状态变化的时候被调用
public void setIsCached(boolean isCached) {
    synchronized (this) {
        if (isCached) {
            mCacheRefCount++;
        } else {
            mCacheRefCount--;
        }
    }
    // Check to see if recycle() can be called.
    checkState();
}

// 每一次图片状态发生变化的时候被调用
private synchronized void checkState() {
    // If the drawable cache and display ref counts = 0, and this drawable
    // has been displayed, then recycle.
    if (mCacheRefCount <= 0 && mDisplayRefCount <= 0 && mHasBeenDisplayed
            && hasValidBitmap()) {
        getBitmap().recycle();
    }
}

// 检验bitmap的有效性
private synchronized boolean hasValidBitmap() {
    Bitmap bitmap = getBitmap();
    return bitmap != null && !bitmap.isRecycled();
}
```
### Android 3.0 及之后版本内存管理
在Android 3.0版本引入了`BitmapFactory.Options.inBitmap`字段，设置了这个字段之后，在对bitmap解码的时候会对已有bitmap数据内容进行重用。这儿有个限制，在4.4版本以前，只支持jpeg或者png格式的图片，并且bitmap的图片大小必须相同。而在4.4之后，只要保证原`inBitmap`不小于将要加载的bitmap大小即可。具体细节可以参考[inBitmap](https://developer.android.com/reference/android/graphics/BitmapFactory.Options.html#inBitmap)。
#### 保存bitmap为以后使用
下面的代码片段示范了app运行在Android 3.0及以上版本的时候，当bitmap从`LruCache`缓存移除的时候，将会在`HashSet`保存这个bitmap的软引用，以便后面的重用。
```java
Set<SoftReference<Bitmap>> mReusableBitmaps;
private LruCache<String, BitmapDrawable> mMemoryCache;

// If you're running on Honeycomb or newer, create a
// synchronized HashSet of references to reusable bitmaps.
if (Utils.hasHoneycomb()) {
    mReusableBitmaps =
            Collections.synchronizedSet(new HashSet<SoftReference<Bitmap>>());
}

mMemoryCache = new LruCache<String, BitmapDrawable>(mCacheParams.memCacheSize) {

    // Notify the removed entry that is no longer being cached.
    @Override
    protected void entryRemoved(boolean evicted, String key,
            BitmapDrawable oldValue, BitmapDrawable newValue) {
        if (RecyclingBitmapDrawable.class.isInstance(oldValue)) {
            // The removed entry is a recycling drawable, so notify it
            // that it has been removed from the memory cache.
            ((RecyclingBitmapDrawable) oldValue).setIsCached(false);
        } else {
            // The removed entry is a standard BitmapDrawable.
            if (Utils.hasHoneycomb()) {
                // We're running on Honeycomb or later, so add the bitmap
                // to a SoftReference set for possible use with inBitmap later.
                mReusableBitmaps.add
                        (new SoftReference<Bitmap>(oldValue.getBitmap()));
            }
        }
    }
....
}
```
#### 使用已有的bitmap
正在运行的app的时候会查看是否有可用bitmap可供decode*方法使用。例如以下代码:
```java
public static Bitmap decodeSampledBitmapFromFile(String filename,
        int reqWidth, int reqHeight, ImageCache cache) {

    final BitmapFactory.Options options = new BitmapFactory.Options();
    ...
    BitmapFactory.decodeFile(filename, options);
    ...

    // If we're running on Honeycomb or newer, try to use inBitmap.
    if (Utils.hasHoneycomb()) {
        addInBitmapOptions(options, cache);
    }
    ...
    return BitmapFactory.decodeFile(filename, options);
}
```
`addInBitmapOptions()`方法的实现如下:
```java
private static void addInBitmapOptions(BitmapFactory.Options options,
        ImageCache cache) {
    // inBitmap only works with mutable bitmaps, so force the decoder to
    // return mutable bitmaps.
    options.inMutable = true;

    if (cache != null) {
        // Try to find a bitmap to use for inBitmap.
        Bitmap inBitmap = cache.getBitmapFromReusableSet(options);

        if (inBitmap != null) {
            // If a suitable bitmap has been found, set it as the value of
            // inBitmap.
            options.inBitmap = inBitmap;
        }
    }
}

// This method iterates through the reusable bitmaps, looking for one
// to use for inBitmap:
protected Bitmap getBitmapFromReusableSet(BitmapFactory.Options options) {
        Bitmap bitmap = null;

    if (mReusableBitmaps != null && !mReusableBitmaps.isEmpty()) {
        synchronized (mReusableBitmaps) {
            final Iterator<SoftReference<Bitmap>> iterator
                    = mReusableBitmaps.iterator();
            Bitmap item;

            while (iterator.hasNext()) {
                item = iterator.next().get();

                if (null != item && item.isMutable()) {
                    // Check to see it the item can be used for inBitmap.
                    if (canUseForInBitmap(item, options)) {
                        bitmap = item;

                        // Remove from reusable set so it can't be used again.
                        iterator.remove();
                        break;
                    }
                } else {
                    // Remove from the set if the reference has been cleared.
                    iterator.remove();
                }
            }
        }
    }
    return bitmap;
}

static boolean canUseForInBitmap(
        Bitmap candidate, BitmapFactory.Options targetOptions) {

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
        // From Android 4.4 (KitKat) onward we can re-use if the byte size of
        // the new bitmap is smaller than the reusable bitmap candidate
        // allocation byte count.
        // 在4.4及其以后的版本中只要保证原inBitmap的大小大于新的bitmap大小就可以了
        int width = targetOptions.outWidth / targetOptions.inSampleSize;
        int height = targetOptions.outHeight / targetOptions.inSampleSize;
        int byteCount = width * height * getBytesPerPixel(candidate.getConfig());
        return byteCount <= candidate.getAllocationByteCount();
    }

    // On earlier versions, the dimensions must match exactly and the inSampleSize must be 1
    // 而在早期版本中必须要求原bitmap大小和新bitmap的大小一致并且inSampleSize的值必须为1
    return candidate.getWidth() == targetOptions.outWidth
            && candidate.getHeight() == targetOptions.outHeight
            && targetOptions.inSampleSize == 1;
}

/**
 * A helper function to return the byte usage per pixel of a bitmap based on its configuration.
 * 辅助函数，表示在不同的颜色位数情况下单个像素所占空间大小
 */
static int getBytesPerPixel(Config config) {
    if (config == Config.ARGB_8888) {
        return 4;
    } else if (config == Config.RGB_565) {
        return 2;
    } else if (config == Config.ARGB_4444) {
        return 2;
    } else if (config == Config.ALPHA_8) {
        return 1;
    }
    return 1;
}
```
## 在界面上显示Bitmap
这个小节结合之前的内容介绍下在并发或者配置发生变化的情况下，使用`ViewPager`或者`GridView`组件的时候，利用后台线程加载bitmap和对bitmap进行缓存。
### 在ViewPager中加载Bitmap的实现
`FragmentStatePagerAdapter`是一种比`PagerAdapter`更好的适配器。它能够在离开屏幕的时候自动销毁和保存`Fragment`的状态，从而降低内存使用率。
下面是一个持有`ImageView`的`ViewPager`的实现方式，`MainActivity`持有`ViewPager`及其`adapter`。
```java
public class ImageDetailActivity extends FragmentActivity {
    public static final String EXTRA_IMAGE = "extra_image";

    private ImagePagerAdapter mAdapter;
    private ViewPager mPager;

    // A static dataset to back the ViewPager adapter
    public final static Integer[] imageResIds = new Integer[] {
            R.drawable.sample_image_1, R.drawable.sample_image_2, R.drawable.sample_image_3,
            R.drawable.sample_image_4, R.drawable.sample_image_5, R.drawable.sample_image_6,
            R.drawable.sample_image_7, R.drawable.sample_image_8, R.drawable.sample_image_9};

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.image_detail_pager); // Contains just a ViewPager

        mAdapter = new ImagePagerAdapter(getSupportFragmentManager(), imageResIds.length);
        mPager = (ViewPager) findViewById(R.id.pager);
        mPager.setAdapter(mAdapter);
    }

    public static class ImagePagerAdapter extends FragmentStatePagerAdapter {
        private final int mSize;

        public ImagePagerAdapter(FragmentManager fm, int size) {
            super(fm);
            mSize = size;
        }

        @Override
        public int getCount() {
            return mSize;
        }

        @Override
        public Fragment getItem(int position) {
            return ImageDetailFragment.newInstance(position);
        }
    }
}
```
下面是一个初级的`ImageDetailFragment`：
```java
public class ImageDetailFragment extends Fragment {
    private static final String IMAGE_DATA_EXTRA = "resId";
    private int mImageNum;
    private ImageView mImageView;

    static ImageDetailFragment newInstance(int imageNum) {
        final ImageDetailFragment f = new ImageDetailFragment();
        final Bundle args = new Bundle();
        args.putInt(IMAGE_DATA_EXTRA, imageNum);
        f.setArguments(args);
        return f;
    }

    // Empty constructor, required as per Fragment docs
    public ImageDetailFragment() {}

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mImageNum = getArguments() != null ? getArguments().getInt(IMAGE_DATA_EXTRA) : -1;
    }

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
            Bundle savedInstanceState) {
        // image_detail_fragment.xml contains just an ImageView
        final View v = inflater.inflate(R.layout.image_detail_fragment, container, false);
        mImageView = (ImageView) v.findViewById(R.id.imageView);
        return v;
    }

    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        final int resId = ImageDetailActivity.imageResIds[mImageNum];
        mImageView.setImageResource(resId); // Load image into ImageView
    }
}
```
估计你也注意到了，在上面的代码中加载image的时候是在UI线程中去加载的。根据前面讲到的，在UI线程中处理图片并不可取，所以我们要将其放到后台线程中去处理。
```java
public class ImageDetailActivity extends FragmentActivity {
    ...

    public void loadBitmap(int resId, ImageView imageView) {
        mImageView.setImageResource(R.drawable.image_placeholder);
        BitmapWorkerTask task = new BitmapWorkerTask(mImageView);
        task.execute(resId);
    }

    ... // 这儿可以参考前面章节中的BitmapWorkerTask
}

public class ImageDetailFragment extends Fragment {
    ...

    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        if (ImageDetailActivity.class.isInstance(getActivity())) {
            final int resId = ImageDetailActivity.imageResIds[mImageNum];
            // Call out to ImageDetailActivity to load the bitmap in a background thread
            ((ImageDetailActivity) getActivity()).loadBitmap(resId, mImageView);
        }
    }
}
```
对图片的处理比如设置大小或者从网络中加载图片等都可以放到后台线程中而不会影响UI，这儿也可以在后台线程中对图片进行缓存处理。
```java
public class ImageDetailActivity extends FragmentActivity {
    ...
    private LruCache<String, Bitmap> mMemoryCache;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        ...
        // 这儿可以参考之前讲到的内存缓存
    }

    public void loadBitmap(int resId, ImageView imageView) {
        final String imageKey = String.valueOf(resId);

        final Bitmap bitmap = mMemoryCache.get(imageKey);
        if (bitmap != null) {
            mImageView.setImageBitmap(bitmap);
        } else {
            // 先显示一张正在占位图片，然后再后台去加载真正的图片
            mImageView.setImageResource(R.drawable.image_placeholder);
            BitmapWorkerTask task = new BitmapWorkerTask(mImageView);
            task.execute(resId);
        }
    }

    ... // 其他代码参考缓存那节
}
```
### 在GridView中加载Bitmap的实现
和在`VIewPager`中一样，一个比较常见的实现方式如下：
```java
public class ImageGridFragment extends Fragment implements AdapterView.OnItemClickListener {
    private ImageAdapter mAdapter;

    // A static dataset to back the GridView adapter
    public final static Integer[] imageResIds = new Integer[] {
            R.drawable.sample_image_1, R.drawable.sample_image_2, R.drawable.sample_image_3,
            R.drawable.sample_image_4, R.drawable.sample_image_5, R.drawable.sample_image_6,
            R.drawable.sample_image_7, R.drawable.sample_image_8, R.drawable.sample_image_9};

    // Empty constructor as per Fragment docs
    public ImageGridFragment() {}

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mAdapter = new ImageAdapter(getActivity());
    }

    @Override
    public View onCreateView(
            LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
        final View v = inflater.inflate(R.layout.image_grid_fragment, container, false);
        final GridView mGridView = (GridView) v.findViewById(R.id.gridView);
        mGridView.setAdapter(mAdapter);
        mGridView.setOnItemClickListener(this);
        return v;
    }

    @Override
    public void onItemClick(AdapterView<?> parent, View v, int position, long id) {
        final Intent i = new Intent(getActivity(), ImageDetailActivity.class);
        i.putExtra(ImageDetailActivity.EXTRA_IMAGE, position);
        startActivity(i);
    }

    private class ImageAdapter extends BaseAdapter {
        private final Context mContext;

        public ImageAdapter(Context context) {
            super();
            mContext = context;
        }

        @Override
        public int getCount() {
            return imageResIds.length;
        }

        @Override
        public Object getItem(int position) {
            return imageResIds[position];
        }

        @Override
        public long getItemId(int position) {
            return position;
        }

        @Override
        public View getView(int position, View convertView, ViewGroup container) {
            ImageView imageView;
            if (convertView == null) { // if it's not recycled, initialize some attributes
                imageView = new ImageView(mContext);
                imageView.setScaleType(ImageView.ScaleType.CENTER_CROP);
                imageView.setLayoutParams(new GridView.LayoutParams(
                        LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT));
            } else {
                imageView = (ImageView) convertView;
            }
            imageView.setImageResource(imageResIds[position]); // Load image into ImageView
            return imageView;
        }
    }
}
```
和之前问题类似，同样是在UI线程中加载图片，这儿也可以使用异步任务的方式去处理，但是要小心`GridView`在回收子View的时候引起的并发问题。下面是更新之后的代码:
```java
public class ImageGridFragment extends Fragment implements AdapterView.OnItemClickListener {
    ...

    private class ImageAdapter extends BaseAdapter {
        ...

        @Override
        public View getView(int position, View convertView, ViewGroup container) {
            ...
            loadBitmap(imageResIds[position], imageView)
            return imageView;
        }
    }

    public void loadBitmap(int resId, ImageView imageView) {
        if (cancelPotentialWork(resId, imageView)) {
            final BitmapWorkerTask task = new BitmapWorkerTask(imageView);
            final AsyncDrawable asyncDrawable =
                    new AsyncDrawable(getResources(), mPlaceHolderBitmap, task);
            imageView.setImageDrawable(asyncDrawable);
            task.execute(resId);
        }
    }

    static class AsyncDrawable extends BitmapDrawable {
        private final WeakReference<BitmapWorkerTask> bitmapWorkerTaskReference;

        public AsyncDrawable(Resources res, Bitmap bitmap,
                BitmapWorkerTask bitmapWorkerTask) {
            super(res, bitmap);
            bitmapWorkerTaskReference =
                new WeakReference<BitmapWorkerTask>(bitmapWorkerTask);
        }

        public BitmapWorkerTask getBitmapWorkerTask() {
            return bitmapWorkerTaskReference.get();
        }
    }

    public static boolean cancelPotentialWork(int data, ImageView imageView) {
        final BitmapWorkerTask bitmapWorkerTask = getBitmapWorkerTask(imageView);

        if (bitmapWorkerTask != null) {
            final int bitmapData = bitmapWorkerTask.data;
            if (bitmapData != data) {
                // Cancel previous task
                bitmapWorkerTask.cancel(true);
            } else {
                // The same work is already in progress
                return false;
            }
        }
        // No task associated with the ImageView, or an existing task was cancelled
        return true;
    }

    private static BitmapWorkerTask getBitmapWorkerTask(ImageView imageView) {
       if (imageView != null) {
           final Drawable drawable = imageView.getDrawable();
           if (drawable instanceof AsyncDrawable) {
               final AsyncDrawable asyncDrawable = (AsyncDrawable) drawable;
               return asyncDrawable.getBitmapWorkerTask();
           }
        }
        return null;
    }

    ... // include updated BitmapWorkerTask class
```
> 提示：这一套代码也可以用于`ListView`
相信在看完这篇教程之后，你能够更好的处理bitmap了吧。
