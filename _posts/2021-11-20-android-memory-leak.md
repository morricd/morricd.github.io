---
title: Android 性能优化实战总结
author: morric
date: 2021-11-20 00:10:00 +0800
categories: [开发记录, Android]
tags: [android]
---

Android 性能优化是一个贯穿整个开发周期的话题。本文总结了我在实际项目中遇到的性能问题以及对应的解决方案，涵盖内存泄漏、布局优化、图片加载、APK 瘦身等方向。

---

## 一、内存优化

内存问题是 Android 开发中最高频的问题，轻则 GC 频繁导致卡顿，重则 OOM 直接崩溃。项目中遇到的内存问题主要集中在以下几类。

### 1.1 Handler 内存泄漏

这是我遇到最多的泄漏场景。通过匿名内部类创建的 `Handler` 会隐式持有外部 `Activity` 的引用，如果此时消息队列中还有未处理的消息，`Activity` 销毁后无法被 GC 回收。

**错误写法：**

```java
// 匿名内部类 Handler，隐式持有 Activity 引用
private Handler mHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
        // 更新 UI
    }
};
```

**正确做法：** 改为静态内部类 + 弱引用，并在 `onDestroy` 中清空消息队列：

```java
private static class SafeHandler extends Handler {
    private final WeakReference<MainActivity> mRef;

    SafeHandler(MainActivity activity) {
        mRef = new WeakReference<>(activity);
    }

    @Override
    public void handleMessage(Message msg) {
        MainActivity activity = mRef.get();
        if (activity != null) {
            // 安全地更新 UI
        }
    }
}

@Override
protected void onDestroy() {
    // 清空消息队列，避免延迟消息继续持有引用
    mHandler.removeCallbacksAndMessages(null);
    super.onDestroy();
}
```

> **实际经验**：项目中有一处倒计时功能，Handler 每秒发一条延迟消息更新 UI。用户快速进出页面时，Activity 已销毁但消息仍在队列中，导致该页面无法被回收，用 LeakCanary 检测到后按上述方案修复。

### 1.2 AsyncTask 没取消导致崩溃

`AsyncTask` 在后台执行耗时任务时，如果 `Activity` 提前销毁，`onPostExecute` 回调回来时尝试更新已销毁的 UI 就会崩溃。

**解决方案：** 在 `onDestroy` 中取消任务：

```java
@Override
protected void onDestroy() {
    if (mAsyncTask != null && mAsyncTask.getStatus() == AsyncTask.Status.RUNNING) {
        mAsyncTask.cancel(true);
    }
    super.onDestroy();
}
```

同时在 `doInBackground` 中定期检查是否已被取消：

```java
@Override
protected Void doInBackground(Void... params) {
    for (int i = 0; i < total; i++) {
        if (isCancelled()) break; // 及时退出
        // 执行任务
    }
    return null;
}
```

> **实际经验**：后来项目中逐步将 `AsyncTask` 替换为 `ExecutorService` + `Handler` 或 `Kotlin Coroutines`，彻底规避了这类问题。`AsyncTask` 在 API 30 已被官方废弃，新项目不建议再使用。

### 1.3 单例持有 Context 泄漏

项目中有一个工具类使用单例模式，构造时传入了 `Activity` 的 `Context`，导致单例的生命周期等于 App 生命周期，而 `Activity` 无法被回收。

```java
// 错误：持有 Activity Context
public static Manager getInstance(Context context) {
    if (instance == null) {
        instance = new Manager(context); // context 是 Activity
    }
    return instance;
}
```

**解决方案：** 统一改为传入 `Application Context`：

```java
public static Manager getInstance(Context context) {
    if (instance == null) {
        instance = new Manager(context.getApplicationContext()); // 使用 Application Context
    }
    return instance;
}
```

### 1.4 Bitmap OOM

项目早期直接用 `BitmapFactory.decodeFile` 加载原图，在低端机上频繁 OOM。

**解决方案：** 加载前先获取图片尺寸，计算合适的 `inSampleSize`：

```java
public static Bitmap decodeSampledBitmap(String path, int reqWidth, int reqHeight) {
    BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true; // 只读尺寸，不加载像素
    BitmapFactory.decodeFile(path, options);

    // 计算采样率
    int inSampleSize = 1;
    if (options.outHeight > reqHeight || options.outWidth > reqWidth) {
        int halfHeight = options.outHeight / 2;
        int halfWidth = options.outWidth / 2;
        while ((halfHeight / inSampleSize) >= reqHeight
                && (halfWidth / inSampleSize) >= reqWidth) {
            inSampleSize *= 2;
        }
    }

    options.inSampleSize = inSampleSize;
    options.inJustDecodeBounds = false;
    return BitmapFactory.decodeFile(path, options);
}
```

对于不再使用的 Bitmap，及时回收：

```java
if (bitmap != null && !bitmap.isRecycled()) {
    bitmap.recycle();
    bitmap = null;
}
```

> **实际经验**：自从项目全面切换到 Glide 后，Bitmap 的 OOM 问题基本消失，Glide 内部已经做了完善的采样率计算和缓存管理。

### 1.5 动画没取消导致泄漏

属性动画（`ValueAnimator`、`ObjectAnimator`）如果设置了无限循环，Activity 销毁后动画仍在运行，会一直持有 View 的引用，进而持有 Activity 引用。

```java
@Override
protected void onDestroy() {
    if (mAnimator != null && mAnimator.isRunning()) {
        mAnimator.cancel();
    }
    super.onDestroy();
}
```

---

## 二、布局优化

### 2.1 减少布局层级

过深的布局层级会增加 measure 和 draw 的时间，直接影响页面的渲染速度。

**优化原则：**

- 能用 `FrameLayout` 或 `LinearLayout` 实现的，不用 `RelativeLayout`（`RelativeLayout` 会对子 View 进行两次 measure）；
- 使用 `<merge>` 标签减少 `include` 引入布局时产生的冗余根节点；
- 复杂布局优先使用 `ConstraintLayout`，一层布局实现复杂排版，避免多层嵌套。

**布局性能对比：**

| 布局                        | onMeasure 次数 | 适用场景       |
| --------------------------- | -------------- | -------------- |
| `FrameLayout`               | 1 次           | 简单叠层布局   |
| `LinearLayout`（无 weight） | 1 次           | 线性排列       |
| `LinearLayout`（有 weight） | 2 次           | 有权重分配时   |
| `RelativeLayout`            | 2 次           | 相对位置复杂时 |
| `ConstraintLayout`          | 1 次           | 复杂布局首选   |

### 2.2 使用 ViewStub 延迟加载

对于初始状态不可见、只在特定条件下才显示的布局（如错误页、空状态页），使用 `ViewStub` 代替 `GONE`：

```xml
<ViewStub
    android:id="@+id/stub_empty"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:layout="@layout/layout_empty_state"/>
```

需要显示时再 inflate：

```java
if (data.isEmpty()) {
    ViewStub stub = findViewById(R.id.stub_empty);
    if (stub != null) {
        stub.inflate(); // 只执行一次
    }
}
```

> **实际经验**：用 `GONE` 隐藏的布局在页面初始化时仍会被实例化，`ViewStub` 真正做到了按需加载，对于含有复杂空状态页的列表页面，启动速度有明显提升。

### 2.3 过度绘制优化

开启开发者选项中的"显示过渡绘制区域"后，发现项目中几个列表页面大面积红色，原因是每个 item 和父容器都设置了相同的白色背景。

**解决方案：**

```xml
<!-- 移除主题中的默认窗口背景 -->
<style name="AppTheme" parent="Theme.AppCompat.Light">
    <item name="android:windowBackground">@null</item>
</style>
```

同时排查 item 布局中的冗余背景，子 View 与父 View 背景相同时，子 View 不需要重复设置。

---

## 三、图片优化

### 3.1 Glide vs Picasso 的选择

项目初期用过 Picasso，后来全面切换到 Glide，主要原因：

| 对比项       | Glide                    | Picasso           |
| ------------ | ------------------------ | ----------------- |
| 图片质量     | RGB_565（稍低）          | ARGB_8888（更高） |
| 内存占用     | 较低                     | 较高              |
| 缓存策略     | 按 ImageView 尺寸缓存    | 缓存原始尺寸      |
| GIF 支持     | ✅ 支持                   | ❌ 不支持          |
| 生命周期感知 | ✅ 支持 Activity/Fragment | 只支持 Context    |
| 加载速度     | 更快（缓存尺寸匹配）     | 每次需要调整尺寸  |

**Glide 的实际使用配置：**

```java
RequestOptions options = new RequestOptions()
        .placeholder(R.drawable.ic_placeholder)   // 加载中占位图
        .error(R.drawable.ic_error)               // 加载失败占位图
        .diskCacheStrategy(DiskCacheStrategy.ALL) // 缓存所有尺寸
        .override(400, 400)                       // 指定加载尺寸，避免加载超大图
        .centerCrop();

Glide.with(context)
        .load(imageUrl)
        .apply(options)
        .into(imageView);
```

> **实际经验**：切换到 Glide 后，图片列表页的 OOM 崩溃率下降了 90% 以上。Glide 最关键的优化是缓存的图片尺寸与 ImageView 一致，避免了 Picasso 每次显示前还要缩放的开销。

### 3.2 图片格式选择

- **PNG**：无损，支持透明通道，适合 icon 和切图素材；
- **JPEG**：有损，不支持透明，适合色彩丰富的照片类图片；
- **WebP**：有损/无损均支持，体积比 PNG 小约 26%，比 JPEG 小约 25%~35%，推荐在 API 18+ 的项目中优先使用；
- **.9 图**：适配不同屏幕尺寸的拉伸图，体积小，强烈推荐用于背景和按钮。

---

## 四、APK 瘦身

项目上线前 APK 体积接近 20MB，经过以下几步优化压缩到 11MB 左右。

### 4.1 开启代码混淆和资源压缩

```gradle
buildTypes {
    release {
        minifyEnabled true      // 开启代码混淆，移除未使用的类和方法
        shrinkResources true    // 移除未使用的资源文件（需配合 minifyEnabled）
        proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
    }
}
```

### 4.2 指定语言包

项目只需支持中文，但依赖的第三方库（如 AppCompat）内置了几十种语言的字符串资源：

```gradle
defaultConfig {
    resConfigs "zh" // 只保留中文资源，其余语言包全部剔除
}
```

### 4.3 使用 lint 清理无用资源

```
Analyze → Run Inspection By Name → unused resources
```

扫描后删除未被引用的图片、布局、字符串等资源文件。

### 4.4 图片压缩

使用 [TinyPNG](https://tinypng.com) 对项目中所有 PNG 图片进行无损压缩，平均压缩率 60%~80%，图片质量肉眼无差异。

> **实际经验**：单纯开启 `shrinkResources` 并不会真正删除资源文件，而是将未使用的图片替换为 1×1 像素的占位图。配合 lint 手动清理，效果才最彻底。

---

## 五、列表优化

列表卡顿是用户感知最直接的性能问题，主要优化方向：

### 5.1 ViewHolder 复用

使用 `RecyclerView` 替代 `ListView`，`RecyclerView` 内置了 ViewHolder 机制，复用效率更高。`Adapter` 中避免在 `onBindViewHolder` 里做耗时操作。

### 5.2 分页加载

数据量大时不能一次性全部加载，使用 `RecyclerView` + `Paging` 库实现分页，或手动监听滚动到底部时加载下一页。

### 5.3 图片异步加载 + 缓存

列表中的图片全部交给 Glide 处理，利用其内存缓存 + 磁盘缓存机制，避免每次滑动都重新请求网络。

### 5.4 减少 item 布局复杂度

item 布局层级越深，每次 inflate 和 measure 的耗时越长。将复杂 item 布局用 `ConstraintLayout` 扁平化，同时避免在 item 中使用 `wrap_content` 高度（会触发多次 measure）。

---

## 六、多线程优化

项目早期直接 `new Thread()` 处理网络请求，大量并发请求时线程数失控，GC 频繁，页面明显卡顿。

**解决方案：** 统一使用线程池：

```java
// 全局线程池，避免频繁创建销毁线程
public class ThreadPoolManager {
    private static final int CORE_SIZE = Runtime.getRuntime().availableProcessors();
    private static ExecutorService sExecutor;

    public static ExecutorService get() {
        if (sExecutor == null) {
            sExecutor = new ThreadPoolExecutor(
                CORE_SIZE,
                CORE_SIZE * 2,
                60L,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(128),
                new ThreadPoolExecutor.CallerRunsPolicy() // 队列满时在调用线程执行，不丢任务
            );
        }
        return sExecutor;
    }
}
```

> **实际经验**：核心线程数设置为 CPU 核数，最大线程数为核数的两倍，这是 CPU 密集型任务的经验值。如果是 IO 密集型任务（如大量网络请求），可以适当调大最大线程数。

---

## 总结

| 方向     | 核心原则                                                  |
| -------- | --------------------------------------------------------- |
| 内存泄漏 | 内部类用静态 + 弱引用；onDestroy 清理 Handler、动画、监听 |
| 布局优化 | 减少层级；用 ConstraintLayout；ViewStub 延迟加载          |
| 图片优化 | 全面使用 Glide；WebP 替换 PNG；指定加载尺寸               |
| APK 瘦身 | 混淆 + 资源压缩 + 指定语言包 + TinyPNG                    |
| 列表优化 | RecyclerView + 分页 + 图片缓存 + 扁平化 item 布局         |
| 多线程   | 统一线程池，禁止随意 new Thread()                         |

性能优化没有银弹，最重要的是在开发阶段就养成好习惯，而不是等到上线后再回头修。LeakCanary 和 Android Profiler 是排查问题最有效的两个工具，建议在开发环境中常态化使用。