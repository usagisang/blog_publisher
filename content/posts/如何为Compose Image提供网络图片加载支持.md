---
tags:
  - android/compose
date: 2022-02-12
title: 如何为Compose Image提供网络图片加载支持
---

> 本文是源码分析类文章

如何为Compose Image提供网络图片加载支持？目前（Compose 1.0.5）最好的选择似乎是使用图片框架Coil，Coil对Jetpack Compose相关的支持文档[在这](https://coil-kt.github.io/coil/compose/)。

Compose内的Image组件类似于ImageView，仅支持从本地加载图片资源，要想从网络中获取图片并加载，我们首先就得要使用能够处理网络请求的框架，将远程图片资源载入到本地才行。目前主流的图片加载框架Picasso、Glide、Coil等，它们更多面对的仍是传统的View系统下，将图片加载到ImageView中并显示这样的应用场景，而不是为Compose量身打造的，基于此，Accompanist库曾提供了一些图片加载框架的扩展库，为Compose的Image显示网络图片进行简便支持。时过境迁，后来Coil为Image加载图片提供了相关支持，故Accompanist以前关于图片加载框架扩展的依赖都被废弃并不推荐使用了。（Picasso可能在长期内都不会支持Compose Image，[详情](https://github.com/square/picasso/issues/2203)）

接下来我们将分析Accompanist曾经是如何对图片框架做扩展适配，使之能够与Compose配合工作的。

## Picasso(in version 0.6.2)

Accompanist在0.3.0版本就提供了Picasso的支持，不过，在版本0.7.0该集成被移除（相关的pull参见[https://github.com/google/accompanist/pull/253](https://github.com/google/accompanist/pull/253)）

在0.6.2版本中，想要加载网络图片，你可能会使用如下代码：

```kotlin
PicassoImage(
    data = "http://..."
    modifier = Modifier.size(50.dp),
) { imageLoadState ->
    when(imageLoadState) {
        ...
    }
}
CoilImage(
    data = "https://i.imgur.com/StXm8nf.jpg",
    contentDescription = null,
    onRequestCompleted = {
        println("LoadingCoilImage onRequestCompleted $it")
    },
    contentScale = ContentScale.Crop,
    modifier = Modifier.fillMaxWidth(),
) {
    ...
}
```

在version 0.6.2中，加载远程图片的方法是使用专用的Image组件，使用Picasso框架的调用PicassoImage，使用Coil的则调用CoilImage，等等。它们都依赖于一个imageloader-core的核心库来进行图片加载，我们不难想象这个加载图片的方法，为了糅合各类框架，肯定要用不少泛型，事实上它长下面这样：

```kotlin
@Composable
fun <R : Any, TR : Any> ImageLoad(
    request: R,
    executeRequest: suspend (TR) -> ImageLoadState,
    modifier: Modifier = Modifier,
    requestKey: Any = request,
    transformRequestForSize: (R, IntSize) -> TR?,
    shouldRefetchOnSizeChange: (currentResult: ImageLoadState, size: IntSize) -> Boolean = DefaultRefetchOnSizeChangeLambda,
    onRequestCompleted: (ImageLoadState) -> Unit = EmptyRequestCompleteLambda,
    content: @Composable BoxScope.(imageLoadState: ImageLoadState) -> Unit
) {
    ...
}
```

泛型R代表请求的值，这个值之所以是泛型，是因为实际上各种框架都支持多类型的图片加载请求，这个请求可能是基于一个URL的String，也可能单纯是一个resource的id，或者就是一个Bitmap，等等。泛型TR代表了不同图片框架内收集本次图片请求信息的实体类（或者是Builder），在Picasso中这个类叫RequestCreator，在Glide中这个类叫RequestBuilder。

我们继续观察它的实现：

```kotlin
@Composable
fun <R : Any, TR : Any> ImageLoad(
    request: R,
    executeRequest: suspend (TR) -> ImageLoadState,
    modifier: Modifier = Modifier,
    requestKey: Any = request,
    transformRequestForSize: (R, IntSize) -> TR?,
    shouldRefetchOnSizeChange: (currentResult: ImageLoadState, size: IntSize) -> Boolean = DefaultRefetchOnSizeChangeLambda,
    onRequestCompleted: (ImageLoadState) -> Unit = EmptyRequestCompleteLambda,
    content: @Composable BoxScope.(imageLoadState: ImageLoadState) -> Unit
) {
    // 三个rememberUpdatedState，目的是为了避免更改后重组
    val updatedOnRequestCompleted by rememberUpdatedState(onRequestCompleted)
    val updatedTransformRequestForSize by rememberUpdatedState(transformRequestForSize)
    val updatedExecuteRequest by rememberUpdatedState(executeRequest)

    // 这个state拿来缓存控件大小，因为控件大小要等到Compose内容传入constraints才能确定
    var requestSize by remember(requestKey) { mutableStateOf<IntSize?>(null) }

    // 重点，这里使用produceState将executeRequest返回的非Compose状态转换为一个State
    // 之所以连加载图片的过程都抽象成一个叫executeRequest的lambda，还是因为要糅合多个框架
    val loadState by produceState<ImageLoadState>(
        initialValue = ImageLoadState.Loading,
        key1 = requestKey,
        key2 = requestSize,
    ) {
        // value一开始肯定被赋值为ImageLoadState.Loading，因为requestSize为空。
        // 当requestSize被赋值后，首先将开始执行transformRequestForSize这个lambda
        // 传入原来的request和新获得的size，要求返回一个类似RequestBuilder的结果
        value = requestSize?.let { updatedTransformRequestForSize(request, it) }
            ?.let { transformedRequest ->
                   // 这里传入刚才的RequestBuilder
                try {
                    // 发起图片加载请求，这里可能会挂起
                    updatedExecuteRequest(transformedRequest)
                } catch (e: CancellationException) {
                    // We specifically don't do anything for the request coroutine being
                    // cancelled: https://github.com/google/accompanist/issues/217
                    // 如果我们响应了协程的CancellationException，让ImageLoadState变成了Error
                    // 有可能会出问题，因为如果取消的协程在新协程完成后执行，
                    // 会导致新的图片状态（Success）被上次取消的结果（Error）覆盖
                    throw e
                } catch (e: Error) {
                    // Re-throw all Errors
                    throw e
                } catch (e: IllegalStateException) {
                    // Re-throw all IllegalStateExceptions
                    throw e
                } catch (t: Throwable) {
                    // Anything else, we wrap in a Error state instance
                    // 除了CancellationException、Error、IllegalStateException之外，
                    // 其余的错误将会令状态转变为Error
                    ImageLoadState.Error(painter = null, throwable = t)
                    // also内，加载完成，回调onRequestCompleted
                }.also(updatedOnRequestCompleted)
            } ?: ImageLoadState.Loading
    }

    BoxWithConstraints(
        modifier = modifier,
        propagateMinConstraints = true,
    ) {
        val size = IntSize(
            width = if (constraints.hasBoundedWidth) constraints.maxWidth else -1,
            height = if (constraints.hasBoundedHeight) constraints.maxHeight else -1
        )
        if (requestSize == null ||
            (requestSize != size && shouldRefetchOnSizeChange(loadState, size))
        ) {
            requestSize = size
        }

        content(loadState)
    }
}
```

ImageLoad的思路清晰明了：调用方告诉它如何build一个请求，并在使用图片框架的过程中产生ImageLoadState状态，它会把ImageLoadState转换为可以观察的`State<ImageLoadState>`。

直接使用通用实现的缺点在于会产生很多模板代码，可以基于通用实现进行更简洁的封装，我们以特定的PicassoImage的实现为例进行分析：

```kotlin
// 这个API封装更彻底，不需要写when(state)，直接在函数中传入error、loading的内容即可
@Composable
fun PicassoImage(
    data: Any,
    contentDescription: String?,
    modifier: Modifier = Modifier,
    alignment: Alignment = Alignment.Center,
    contentScale: ContentScale = ContentScale.Fit,
    colorFilter: ColorFilter? = null,
    fadeIn: Boolean = false,
    picasso: Picasso = LocalPicasso.current,
    requestBuilder: (RequestCreator.(size: IntSize) -> RequestCreator)? = null,
    shouldRefetchOnSizeChange: (currentResult: ImageLoadState, size: IntSize) -> Boolean = DefaultRefetchOnSizeChangeLambda,
    onRequestCompleted: (ImageLoadState) -> Unit = EmptyRequestCompleteLambda,
    error: @Composable (BoxScope.(ImageLoadState.Error) -> Unit)? = null,
    loading: @Composable (BoxScope.() -> Unit)? = null,
) {
    PicassoImage(
        data = data,
        modifier = modifier,
        requestBuilder = requestBuilder,
        picasso = picasso,
        shouldRefetchOnSizeChange = shouldRefetchOnSizeChange,
        onRequestCompleted = onRequestCompleted,
    ) { imageState ->
        when (imageState) {
            is ImageLoadState.Success -> {
                // MaterialLoadingImage是0.6.2版本中存在的一个实现fadeIn效果的控件
                // 原理是使用Compose动画中的Transition托管三个动画
                // alpha(透明度),brightness(亮度),saturation(饱和度), 
                // 同时修改传入Image内的colorFliter的这三个值，从而实现渐入效果
                MaterialLoadingImage(
                    result = imageState,
                    contentDescription = contentDescription,
                    fadeInEnabled = fadeIn,
                    alignment = alignment,
                    contentScale = contentScale,
                    colorFilter = colorFilter
                )
            }
            is ImageLoadState.Error -> if (error != null) error(imageState)
            ImageLoadState.Loading -> if (loading != null) loading()
            ImageLoadState.Empty -> Unit
        }
    }
}


@Composable
fun PicassoImage(
    data: Any,
    modifier: Modifier = Modifier,
    picasso: Picasso = LocalPicasso.current,
    requestBuilder: (RequestCreator.(size: IntSize) -> RequestCreator)? = null,
    shouldRefetchOnSizeChange: (currentResult: ImageLoadState, size: IntSize) -> Boolean = DefaultRefetchOnSizeChangeLambda,
    onRequestCompleted: (ImageLoadState) -> Unit = EmptyRequestCompleteLambda,
    content: @Composable BoxScope.(imageLoadState: ImageLoadState) -> Unit
) {
    ImageLoad(
        request = data.toRequestCreator(picasso),
        requestKey = data, // Picasso RequestCreator doesn't support equality so we use the data
        executeRequest = { r ->
            @OptIn(ExperimentalCoroutinesApi::class)
            suspendCancellableCoroutine { cont ->
                // 初始化了一个Target，这个Target用来获取图片加载结果
                val target = object : com.squareup.picasso.Target {
                    override fun onBitmapLoaded(bitmap: Bitmap, from: Picasso.LoadedFrom) {
                        val state = ImageLoadState.Success(
                            painter = BitmapPainter(bitmap.asImageBitmap()),
                            source = from.toDataSource()
                        )
                        // 协程恢复
                        cont.resume(state) {
                            // Not much we can do here. Ignore this
                        }
                    }

                    override fun onBitmapFailed(exception: Exception, errorDrawable: Drawable?) {
                        val state = ImageLoadState.Error(
                            throwable = exception,
                            painter = errorDrawable?.toPainter(),
                        )
                        // 协程恢复
                        cont.resume(state) {
                            // Not much we can do here. Ignore this
                        }
                    }

                    override fun onPrepareLoad(placeholder: Drawable?) = Unit
                }

                cont.invokeOnCancellation {
                    // 取消图片加载
                    picasso.cancelRequest(target)
                }

                // Now kick off the image load into our target
                r.into(target)
            }
        },
        transformRequestForSize = { r, size ->
            val sizedRequest = when {
                // 如果尺寸包含未指定尺寸的尺寸，我们不会在Coil请求中指定尺寸
                size.width < 0 || size.height < 0 -> r
               
                size != IntSize.Zero -> {
                    r.resize(size.width, size.height)
                        .centerInside()
                        .onlyScaleDown()
                }
                // Otherwise we have a zero size, so no point executing a request
                // 未获得size，因此暂时无法生成请求
                else -> null
            }

            // 根据参数来build请求
            if (sizedRequest != null && requestBuilder != null) {
                // If we have a transformed request and builder, let it run
                requestBuilder(sizedRequest, size)
            } else {
                // Otherwise we just return the sizedRequest
                sizedRequest
            }
        },
        shouldRefetchOnSizeChange = shouldRefetchOnSizeChange,
        onRequestCompleted = onRequestCompleted,
        modifier = modifier,
        content = content
    )
}
```

现在让我们来总结一下，在0.6.2版本，实现网络图片加载的集成库思路如下：

1. 图片加载：使用Target回调获取加载的结果（各个框架都有类似的抽象的Target而不是限制目标必须是ImageView）。结果返回的过程是阻塞式，协程将在produceState内执行到` updatedExecuteRequest(transformedRequest)`后挂起，直到这个lambda返回结果，State的值将会在结果返回后产生变化。当然，如果协程被取消，Picasso也会取消加载到Target那个图片请求。
2. 图片大小约束：依赖于BoxWithConstraints获得的约束大小。
3. 渐入动画实现：使用动画API `Transition `对ColorFliter的alpha,brightness,saturation进行动态修改，从而实现渐入动画。
4. loading占位图、error显示等：依赖于用户传入的@Composable内容.根据produceState生成的状态，PicassoImage内显示的@Composable内容会动态变化。

## Glide(in version 0.13.0)

0.3.0版本诞生于2020年10月份，而当时间来到了2021年4月，Accompanist发布0.8.0版本，Coil 和 Glide 集成库进行了大规模的重构。上面提到的类似于`CoilImage()`和`GlideImage()`API都已经被弃用了。

以下对Glide集成库的分析基于版本0.13.0的代码。

如果在0.13.0版本想要加载远程图片，或许你会写出以下的代码：

```kotlin
Image(
    painter = rememberGlidePainter(request = "http://..."),
    contentDescription = null
)
```

新的API不再需要专门的Image组件，而是使用Painter这种概念来表现加载的结果。新的API对性能的提升似乎有所提升：Compose内容重组后，需要重绘的不再是不同的Loading组件或Success组件，现在核心组件一定是一个Image，随加载状态变化的只不过是Image内绘制的内容而已，重绘范围有所缩小。这很符合我们对ImageView的想象：在加载的时候显示一张placeholder占位图，成功显示最终结果，否则显示error图片，而placeholder和error都可以发起图片加载请求的时候设置。

Painter是一个什么样的概念？我们可以先看一下类注释是怎么介绍它的：

```kotlin
/**
* 对可以画出来的东西的抽象。除了能够绘制到指定的有界区域外，Painter还提供了一些高级机制，消费者可以使用
* 这些机制来配置内容的绘制方式。其中包括alpha、ColorFilter和RTL
* 实现应该提供一个有意义的equals方法来比较不同Painter子类的值，而不仅仅依赖于引用相等
*/
abstract class Painter {
    ...
    protected abstract fun DrawScope.onDraw()
}
```

描述看起来有点像Drawable，但实际上Drawable比Painter更加复杂一些，除了上述的alpha、ColorFilter、LayoutDirection之外，Drawable还具有动画Callback、Level、Hotspot等属性。`DrawScope.onDraw()`方法类似于Drawable的`draw(Canvas canvas)`。

继续观察rememberGlidePainter的具体实现：

```kotlin
@Composable
fun rememberGlidePainter(
    request: Any?,
    requestManager: RequestManager = GlidePainterDefaults.defaultRequestManager(),
    shouldRefetchOnSizeChange: ShouldRefetchOnSizeChange = ShouldRefetchOnSizeChange { _, _ -> false },
    // 注意这里的requestBuilder，加载的结果类型已经被固定为drawable
    requestBuilder: (RequestBuilder<Drawable>.(size: IntSize) -> RequestBuilder<Drawable>)? = null,
    // 新的API也能开启fadeIn效果
    fadeIn: Boolean = false,
    fadeInDurationMs: Int = LoadPainterDefaults.FadeInTransitionDuration,
    // 是不是很疑惑为什么这里有个占位图id的参数？Glide本身就支持占位图设置，
    // 在Build Request的时候设置不就行了吗？其实这个参数是给Compose预览模式用的
    @DrawableRes previewPlaceholder: Int = 0,
): LoadPainter<Any> {
    // GlideLoader是加载逻辑实现类，稍后展示
    val glideLoader = remember {
        GlideLoader(requestManager, requestBuilder)
    }.apply {
        // 这里的逻辑并不是多余的，要知道如果key没有变化，remember函数会直接返回上次计算的结果，
        // 这里想表达的是，对上次的结果调用apply，更新requestManager和requestBuilder
        this.requestManager = requestManager
        this.requestBuilder = requestBuilder
    }
    // rememberLoadPainter位于之前所说的imageloading-core的核心库
    // 在0.13.0版本Coil和Glide都用到这个库来获取LoadPainter
    return rememberLoadPainter(
        loader = glideLoader,
        request = checkData(request),
        shouldRefetchOnSizeChange = shouldRefetchOnSizeChange,
        fadeIn = fadeIn,
        fadeInDurationMs = fadeInDurationMs,
        previewPlaceholder = previewPlaceholder
    )
}
// checkData检查了request的类型
private fun checkData(data: Any?): Any? {
    when (data) {
        is Drawable -> {
            throw IllegalArgumentException(....)
        }
        is ImageBitmap -> {
            throw IllegalArgumentException(....)
        }
        is ImageVector -> {
            throw IllegalArgumentException(....)
        }
        is Painter -> {
            throw IllegalArgumentException(....)
        }
    }
    return data
}
```

imageloading-core这次如何抽象图片加载行为？我们先观察一下`rememberLoadPainter`的参数列表：

```kotlin
@Composable
fun <R> rememberLoadPainter(
    loader: Loader<R>,
    request: R?,
    shouldRefetchOnSizeChange: ShouldRefetchOnSizeChange,
    fadeIn: Boolean = false,
    fadeInDurationMs: Int = LoadPainterDefaults.FadeInTransitionDuration,
    @DrawableRes previewPlaceholder: Int = 0,
): LoadPainter<R> {...}

@Stable
fun interface Loader<R> {
    fun load(request: R, size: IntSize): Flow<ImageLoadState>
}
```

与0.6.2版本不同，加载逻辑实现类需要返回一个状态流`Flow<ImageLoadState>`，而不再是单一的`ImageLoadState`，虽然请求类型仍然是泛型的，但是已经不需要表达类似于RequestBuilder这样的泛型类型，如何构建、发起请求由Loader自己决定。

ImageLoadState的实现如下

```kotlin
sealed class ImageLoadState {
    object Empty : ImageLoadState()
    data class Loading(
        val placeholder: Painter?,
        val request: Any,
    ) : ImageLoadState()
    data class Success(
        val result: Painter,
        val source: DataSource,
        val request: Any,
    ) : ImageLoadState()
    data class Error(
        val request: Any,
        val result: Painter? = null,
        val throwable: Throwable? = null
    ) : ImageLoadState()
}
```

不难发现所有的图片加载结果都要求封装成Painter进行返回，但尴尬的是，Drawable与Painter并不是天生互通的类型（Compose 1.0.5只有三种Painter，BitmapPainter、VectorPainter、ColorPainter），好在Accompanist提供了一个DrawablePainter。不过话又说回来，为什么非得要求生产者Loader返回Painter不可呢？那是因为加载请求是多类型的，消费者LoadPainter其实无法确定生产者返回的结果的类型，自然也不确定如何绘制它，因此LoadPainter采用了类似于装饰者模式的设计，图片结果绘制交由State内的Painter完成。

`GlideLoader`的实现如下：

```kotlin
internal class GlideLoader(
    requestManager: RequestManager,
    requestBuilder: (RequestBuilder<Drawable>.(size: IntSize) -> RequestBuilder<Drawable>)?,
) : Loader<Any> {
    var requestManager by mutableStateOf(requestManager)
    var requestBuilder by mutableStateOf(requestBuilder)

    /**
     * 不要删除callbackFlow上的显式类型<ImageLoadState>。IR编译器不喜欢隐式类型。
     */
    @Suppress("RemoveExplicitTypeArguments")
    @OptIn(ExperimentalCoroutinesApi::class)
    override fun load(
        request: Any,
        size: IntSize
    ): Flow<ImageLoadState> = callbackFlow<ImageLoadState> {
        var failException: Throwable? = null
        // 这里同时使用Target与Listener两种机制来监听加载状态，并向flow发送对应状态
        // Target并不会去处理Success的状态，Listener已经抢先处理并拦截了Target的Success调用
        val target = object : EmptyCustomTarget(
            if (size.width > 0) size.width else Target.SIZE_ORIGINAL,
            if (size.height > 0) size.height else Target.SIZE_ORIGINAL
        ) {
            override fun onLoadStarted(placeholder: Drawable?) {
                trySendBlocking(
                    ImageLoadState.Loading(
                        placeholder = placeholder?.let(::DrawablePainter),
                        request = request
                    )
                )
            }

            override fun onLoadFailed(errorDrawable: Drawable?) {
                trySendBlocking(
                    ImageLoadState.Error(
                        result = errorDrawable?.let(::DrawablePainter),
                        request = request,
                        throwable = failException
                            ?: IllegalArgumentException("Error while loading $request")
                    )
                )
                // Close the channel[Flow]
                channel.close()
            }

            override fun onLoadCleared(resource: Drawable?) {
                // Glide想要释放资源，所以我们需要清除结果，否则我们可能会绘制已经被回收的视图
                trySendBlocking(ImageLoadState.Empty)
                // Close the channel[Flow]
                channel.close()
            }
        }

        val listener = object : RequestListener<Drawable> {
            override fun onResourceReady(
                drawable: Drawable,
                model: Any,
                target: Target<Drawable>,
                dataSource: com.bumptech.glide.load.DataSource,
                isFirstResource: Boolean
            ): Boolean {
                // 这里发送的Painter类型
                trySendBlocking(
                    ImageLoadState.Success(
                        result = DrawablePainter(drawable),
                        source = dataSource.toDataSource(),
                        request = request
                    )
                )
                // Close the channel[Flow]
                channel.close()
                // Return true so that the target doesn't receive the drawable
                // 这里返回true，Target就收不到结果了
                return true
            }

            override fun onLoadFailed(
                e: GlideException?,
                model: Any,
                target: Target<Drawable>,
                isFirstResource: Boolean
            ): Boolean {
                // Glide只为Listener派发错误的Exception，因此这里需要缓存一下
                failException = e
                // 返回false，允许Target被回调onLoadFailed
                return false
            }
        }

        // Start the image request into the target
        requestManager.load(request)
            .apply { requestBuilder?.invoke(this, size) }
            .addListener(listener)
            .into(target)

        // Await the channel being closed and request finishing...
        awaitClose {
            // 这里没有调用Glide.clear()，因为clear之后Painter进行绘制的位图可能会被回收，这会报错
            // See https://github.com/google/accompanist/issues/419
        }
    }
}
```

总体来说状态转换逻辑和以前类似，只不过使用callbackFlow生成数据流后，状态发送显得更加优雅了。

接下来关注rememberLoadPainter的具体实现：

```kotlin
/**
一个通用的 image loading painter，它为要实现的图像加载库提供Loader接口。应用程序通常不应该使用此功能，而更推荐使用在此基础上构建的扩展库，例如Coil和Glide库。
*/
@Composable
fun <R> rememberLoadPainter(
    loader: Loader<R>,
    request: R?,
    shouldRefetchOnSizeChange: ShouldRefetchOnSizeChange,
    fadeIn: Boolean = false,
    fadeInDurationMs: Int = LoadPainterDefaults.FadeInTransitionDuration,
    @DrawableRes previewPlaceholder: Int = 0,
): LoadPainter<R> {
    val coroutineScope = rememberCoroutineScope()

    // Our LoadPainter. This invokes the loader as appropriate to display the result.
    val painter = remember(loader, coroutineScope) {
        LoadPainter(loader, coroutineScope)
    }
    painter.request = request
    painter.shouldRefetchOnSizeChange = shouldRefetchOnSizeChange
    // 缓存父布局的大小，在计算图片请求的大小时会参考此值
    painter.rootViewSize = LocalView.current.let { IntSize(it.width, it.height) }

    // fadeIn动画的ColorFilter
    // 实现原理和0.6.2版本类似，也是修改了ColorFliter的alpha(透明度),
    // brightness(亮度),saturation(饱和度)，不过这次的ColorFliter由LoadPainter直接进行处理
    animateFadeInColorFilter(
        painter = painter,
        enabled = { result ->
            // 从 disk/network 才去展示fadeIn动画
            // 这使我们可以近似地只在“首次加载”时运行动画
            fadeIn && result is ImageLoadState.Success && result.source != DataSource.MEMORY
        },
        durationMs = fadeInDurationMs,
    )

    // Our result painter, created from the ImageState with some composition lifecycle
    // callbacks
    // 我们的result painter，通过一些composition生命周期的回调从ImageState创建
    updatePainter(painter, previewPlaceholder)

    return painter
}
```

LoaderPainter的实现如下。这里要特别注意RememberObserver这个接口，RememberObserver是一个能够实现对remember行为的观察的接口，如果composition记住或者遗忘的是一个RememberObserver对象，RememberObserver能够收到这个事件，这些事件对LoaderPainter很有用。因为LoaderPainter毕竟并不是一个Compose组件，但是它必须了解它所在的父组件在什么时候离开了屏幕被销毁了（例如高速滑动列表时），这样它能够及时取消对状态流`Flow<ImageLoadState>`的收集，这是避免发生图片闪烁、错位等问题的关键。

```kotlin
class LoadPainter<R> internal constructor(
    private val loader: Loader<R>,
    private val coroutineScope: CoroutineScope,
) : Painter(), RememberObserver {
    private val paint by lazy(LazyThreadSafetyMode.NONE) { Paint() }

    internal var painter by mutableStateOf<Painter>(EmptyPainter)
    // 这个ColorFilter和渐入动画有关
    internal var transitionColorFilter by mutableStateOf<ColorFilter?>(null)
    // CoroutineScope for the current request
    private var requestCoroutineScope: CoroutineScope? = null
    /**
     * The current request object.
     */
    var request by mutableStateOf<R?>(null)
    /**
     * The root view size.
     */
    internal var rootViewSize by mutableStateOf(IntSize(0, 0))
    /**
     * Lambda which will be invoked when the size changes, allowing
     * optional re-fetching of the image.
     */
    var shouldRefetchOnSizeChange by mutableStateOf(ShouldRefetchOnSizeChange { _, _ -> false })

    /**
     * The current [ImageLoadState].
     * 被观察的ImageLoadState
     */
    var loadState: ImageLoadState by mutableStateOf(ImageLoadState.Empty)
        private set

    private var alpha: Float by mutableStateOf(1f)
    private var colorFilter: ColorFilter? by mutableStateOf(null)

    /**
     * 执行图像加载请求时要使用的大小
     */
    private var requestSize by mutableStateOf<IntSize?>(null)

    // Painter内的属性，指定边界大小
    override val intrinsicSize: Size
        get() = painter.intrinsicSize

    override fun applyAlpha(alpha: Float): Boolean {
        this.alpha = alpha
        return true
    }

    override fun applyColorFilter(colorFilter: ColorFilter?): Boolean {
        this.colorFilter = colorFilter
        return true
    }

    override fun DrawScope.onDraw() {
        // 根据Canvas的大小确定requestSize，是不是注意到requestSize的确定其实是存在延时的？
        updateRequestSize(canvasSize = size)
        
        // 下面是一些绘制逻辑
        val transitionColorFilter = transitionColorFilter
        if (colorFilter != null && transitionColorFilter != null) {
            // If we have a transition color filter, 
            // and a specified color filter we need to
            // draw the content in a layer for both to apply.
            // See https://github.com/google/accompanist/issues/262
           drawIntoCanvas { canvas ->
                paint.colorFilter = transitionColorFilter
                canvas.saveLayer(bounds = size.toRect(), paint = paint)
                with(painter) {
                    draw(size, alpha, colorFilter)
                }
                canvas.restore()
        } else {
            // Otherwise we just draw the content directly, using the filter
            with(painter) {
                draw(size, alpha, colorFilter ?: transitionColorFilter)
            }
        }
    }
    // RememberObserver的方法
    // remember运行了计算的lambda但是composition没记住这个对象时回调
    override fun onAbandoned() {
        // We've been abandoned from composition, so cancel our request scope
        requestCoroutineScope?.cancel()
        requestCoroutineScope = null
    }
    // RememberObserver的方法
    // composition忘记了这个对象时回调
    override fun onForgotten() {
        // We've been forgotten from composition, so cancel our request scope
        // onAbandoned和onForgotten时都会cancel运行中的协程
        requestCoroutineScope?.cancel()
        requestCoroutineScope = null
    }
    // RememberObserver的方法
    // 当composition成功记住此对象时调用。
    override fun onRemembered() {
        // Cancel any on-going scope (this shouldn't really happen anyway)
        // 先取消以前正running的协程
        requestCoroutineScope?.cancel()

        // 为当前请求创建新的scope，这允许我们取消作用域，而不影响父作用域的作业。
        val scope = coroutineScope.coroutineContext.let { context ->
            CoroutineScope(context + Job(context[Job]))
        }.also { requestCoroutineScope = it }

        // 我们已经被记住了，所以可以启动一个协程来观察当前的请求对象和请求大小。
        // 每当这些值中的任何一个发生变化时，collectLatest块将运行并执行图像加载（任何正在进行的请求都将被取消）。
        scope.launch {
            // combine方法如其名，能把两个流合并成一个流
            // 不过为什么这里要使用snapshotFlow把State转化成流呢？
            // 因为使用流来监听State变化的最大好处就是collectLatest能够
            // 取消掉上一次的execute调用并启动新一轮的加载
            combine(
                snapshotFlow { request },
                snapshotFlow { requestSize },
                transform = { request, size -> request to size }
            ).collectLatest { (request, size) ->
                execute(request, size)
            }
        }

        // 自动保险。如果没有从onDraw()获得合适的大小，
        // 我们会将请求大小更新为-1，-1，这将加载原始大小的图像。
        scope.launch {
            if (requestSize == null) {
                // 32ms should be enough time for measure/layout/draw to happen.
                // 微妙的32毫秒
                delay(32) 

                if (requestSize == null) {
                   // If we still don't have a request size, resolve the size without
                   // the canvas size
                    // 没获取到Canvas大小，使用原始尺寸
                    updateRequestSize(canvasSize = Size.Zero)
                }
            }
        }
    }

    /**
     * 执行图片加载请求并根据结果更新loadState的方法
     下面描述的是一些状态转换逻辑，比如如果请求为null，状态就转变为Empty
     */
    private suspend fun execute(request: R?, size: IntSize?) {
        if (request == null || size == null) {
            // If we don't have a request, set our state to Empty and return
            loadState = ImageLoadState.Empty
            return
        }
        // ...

        loader.load(request, size)
            .catch { throwable ->
                when (throwable) {
                    is Error -> throw throwable
                    is IllegalStateException -> throw throwable
                    is IllegalArgumentException -> throw throwable
                    else -> {
                        emit(
                            ImageLoadState.Error(
                                result = null,
                                throwable = throwable,
                                request = request
                            )
                        )
                    }
                }
            }
            .collect { loadState = it }
        // 上面collect收集了加载的状态，注意，代表图片结果的Painter没被设置到LoadPainter的字段内
    }

    private fun updateRequestSize(canvasSize: Size) {
        requestSize = IntSize(
            width = when {
                // If we have a canvas width, use it...
                canvasSize.width >= 0.5f -> canvasSize.width.roundToInt()
                // 还记得这个rootViewSize吗？它在rememberLoadPainter函数内被设置
                rootViewSize.width > 0 -> rootViewSize.width
                else -> -1
            },
            height = when {
                // If we have a canvas height, use it...
                canvasSize.height >= 0.5f -> canvasSize.height.roundToInt()
                // Otherwise we fall-back to the root view size as an upper bound
                rootViewSize.height > 0 -> rootViewSize.height
                else -> -1
            },
        )
    }
}
```

虽然说LoadPainter确实是实现了RememberObserver，但是，这个回调是怎么被注册的呢？答案藏在习以为常的remember函数中，传入remember的key，或者是calculation得出的值，它们如果是个RememberObserver，则会被插入到RememberManager的队列中，每当“记忆”和“遗忘”事件发生时都会得到通知。

```kotlin
@Composable
inline fun <T> remember(
    key1: Any?,
    calculation: @DisallowComposableCalls () -> T
): T {
    return currentComposer.cache(currentComposer.changed(key1), calculation)
}
// 注意检查key是否有变化的changed函数
@ComposeCompilerApi
override fun changed(value: Any?): Boolean {
    return if (nextSlot() != value) {
        updateValue(value)
        true
    } else {
        false
    }
}

@PublishedApi
@OptIn(InternalComposeApi::class)
internal fun updateValue(value: Any?) {
    // 两个if分支我们都可以看到 rememberManager.remembering()
    // rememberManager.forgetting()这些调用
    if (inserting) {
        writer.update(value)
        if (value is RememberObserver) {
            // 注意，判断value是不是RememberObserver
            record { _, _, rememberManager -> rememberManager.remembering(value) }
        }
    } else {
        val groupSlotIndex = reader.groupSlotIndex - 1
        recordSlotTableOperation(forParent = true) { _, slots, rememberManager ->
            if (value is RememberObserver) {
                abandonSet.add(value)
                rememberManager.remembering(value)
            }
            when (val previous = slots.set(groupSlotIndex, value)) {
                is RememberObserver ->
                    rememberManager.forgetting(previous)
                is RecomposeScopeImpl -> {
                    val composition = previous.composition
                    if (composition != null) {
                        previous.composition = null
                        composition.pendingInvalidScopes = true
                    }
                }
            }
        }
    }
}
// RememberManager是个接口
internal interface RememberManager {
    /**
     * The [RememberObserver] is being remembered by a slot in the slot table.
     */
    fun remembering(instance: RememberObserver)

    /**
     * The [RememberObserver] is being forgotten by a slot in the slot table.
     */
    fun forgetting(instance: RememberObserver)
    
    ...
}
// RememberManager的实现类
private class RememberEventDispatcher(
    private val abandoning: MutableSet<RememberObserver>
) : RememberManager {
    private val remembering = mutableListOf<RememberObserver>()
    private val forgetting = mutableListOf<RememberObserver>()
    private val sideEffects = mutableListOf<() -> Unit>()

    override fun remembering(instance: RememberObserver) {
        forgetting.lastIndexOf(instance).let { index ->
            if (index >= 0) {
                forgetting.removeAt(index)
                abandoning.remove(instance)
            } else {
                remembering.add(instance)
            }
        }
    }

    override fun forgetting(instance: RememberObserver) {
        remembering.lastIndexOf(instance).let { index ->
            if (index >= 0) {
                remembering.removeAt(index)
                abandoning.remove(instance)
            } else {
                forgetting.add(instance)
            }
        }
    }
    fun dispatchRememberObservers() {
        // 派发forgetting和remembering事件的逻辑
        if (forgetting.isNotEmpty()) {
            for (i in forgetting.size - 1 downTo 0) {
                val instance = forgetting[i]
                if (instance !in abandoning) {
                    instance.onForgotten()
                }
            }
        }
        if (remembering.isNotEmpty()) {
            remembering.fastForEach { instance ->
                abandoning.remove(instance)
                instance.onRemembered()
            }
        }
    }
    // ....
}
```

我们已经明白LoadPainter到底是怎么管理Loader返回的流结果了，最后一个需要注意的地方在函数updatePainter里，这个调用位于rememberLoadPainter最后，函数实现会根据图片加载State的变化来为LoadPainter设置Painter。不过这不是兜了个圈子吗？似乎也可以在collect更新State的同时把Painter更新一下？

```kotlin
/**
* 允许我们以状态观察当前结果。这个函数允许我们最小化重组范围，这样当loadState改变时，只有这个函数需要重新
* 启动。
*/
@Composable
private fun <R> updatePainter(
    loadPainter: LoadPainter<R>,
    @DrawableRes previewPlaceholder: Int = 0,
) {
    loadPainter.painter = if (LocalInspectionMode.current && previewPlaceholder != 0) {
        // 如果我们处于检查模式（预览），并且有一个预览占位符，只需使用图像绘制它并返回
        // 还记得rememberGlidePainter的参数吗？这里就是传入的参数previewPlaceholder的用途
        // 这个函数令LoadPainter完全忽略了State的变化，只展示静态图片
        painterResource(previewPlaceholder)
    } else {
        // remember在这里看上去像是毫无必要的调用，
        // 但这允许任何Painter实例接收记忆事件（如果它实现了RememberObserver）。不要移除。
        remember(loadPainter.loadState) { loadPainter.loadState.painter } ?: EmptyPainter
    }
}
```

现在来总结一下0.13.0版本的Glide远程图片扩展的实现思路：

1. 图片加载：依然是用Target回调获取加载的结果。但是加载状态的返回现在使用流（Flow）来封装，不管是发起加载，异常处理，加载取消都更加优雅直观了。Loader是彻彻底底的生产者，LoadPainter则是消费者。

   LoadPainter并不具有@Composable上下文，作为替代，它实现了RememberObserver来监听控件是否已经离屏销毁。

2. 图片大小约束：依赖于LoadPainter获取的Canvas的大小。

3. 渐入动画实现：跟0.6.2版本的思路相似，不过消费ColorFilter的类变成了LoadPainter。

4. loading占位图、error图等：这些功能直接依赖于具体的图片加载框架的实现，有则有，无则无。0.13.0版本稍微舍去了一些灵活性，不能够像PicassoImage一样直接传入error、loading的Compose内容（控件），不过仍然留有监听图片加载状态的方式，注意，LoadPainter的`loadState`字段是公开的：

   ```kotlin
   /**
    * The current [ImageLoadState].
    */
   var loadState: ImageLoadState by mutableStateOf(ImageLoadState.Empty)
       private set
   ```

## Coil

Accompanist内的Coil集成库最终集成到了Coil内部，成为其扩展，Glide的集成支持则在2021年8月的0.16.0版本被删除。

现在我们简要分析Coil的图片加载逻辑（版本2.0.0-alpha06）。Coil扩展库提供了两种方式来加载网络图片，两种方式正巧就是上面提到的在0.6.2版本与在0.13.0版本的两种实现形式：

```kotlin
// 实现形式1
@Composable
fun AsyncImage(
    model: Any?,
    contentDescription: String?,
    imageLoader: ImageLoader,
    modifier: Modifier = Modifier,
    loading: @Composable (AsyncImageScope.(State.Loading) -> Unit)? = null,
    success: @Composable (AsyncImageScope.(State.Success) -> Unit)? = null,
    error: @Composable (AsyncImageScope.(State.Error) -> Unit)? = null,
    alignment: Alignment = Alignment.Center,
    contentScale: ContentScale = ContentScale.Fit,
    alpha: Float = DefaultAlpha,
    colorFilter: ColorFilter? = null,
    filterQuality: FilterQuality = DefaultFilterQuality,
) {...}
// 实现形式2
@Composable
fun rememberAsyncImagePainter(
    model: Any?,
    imageLoader: ImageLoader,
    filterQuality: FilterQuality = DefaultFilterQuality,
): AsyncImagePainter {...}
```

我们重点分析第二种形式，即rememberAsyncImagePainter函数，其实该函数的实现逻辑与Glide扩展库比较类似，只在某些细节有所区别：

```kotlin
// 这里不再详细分析源码，挑重要的讲
@Composable
fun rememberAsyncImagePainter(
    model: Any?,
    imageLoader: ImageLoader,
    filterQuality: FilterQuality = DefaultFilterQuality,
): AsyncImagePainter {
    val request = requestOf(model)
    requireSupportedData(request.data)
    // 注意这里，这里要求request的target为null
    require(request.target == null) { "request.target must be null." }

    // Dispatchers.Main.immediate是一个有趣的协程调度器，具体效果见类注释
    val scope = rememberCoroutineScope { Dispatchers.Main.immediate }
    // AsyncImagePainter
    val painter = remember(scope) { AsyncImagePainter(scope, request, imageLoader) }
    painter.request = request
    painter.imageLoader = imageLoader
    painter.filterQuality = filterQuality
    // 是否处于预览模式
    painter.isPreview = LocalInspectionMode.current
    // 这里手动调用了一次onRemembered，onRemembered里有向ImageLoader提交request的逻辑
    painter.onRemembered() // Invoke this manually so `painter.state` is up to date immediately.
    // 这里的updatePainter更加复杂，里面有处理fadeIn动画的逻辑
    updatePainter(painter, request, imageLoader)
    return painter
}
```

Dispatchers.Main.immediate比单纯的Dispatchers.Main更加智能，它会减少不必要的调度，当它已经在正确的上下文中，它会立刻执行相应逻辑而无需额外的重新调度。效果类似于下面这样：

```kotlin
suspend fun updateUiElement(val text: String) {
  /*
   * 假设updateUiElement既会被Main线程调用也会被其他线程调用。
   * 那么，当updateUiElement是在Main线程被调用的，更新uiElement.text 这段代码会直接运行，而换成Dispatchers.Main的话，它会再进行一次到Main的调度（明显这是赘余的调度）。
   */
  withContext(Dispatchers.Main.immediate) {
    uiElement.text = text
  }
  // Do context-independent logic such as logging
}
```

接下来我们关注AsyncImagePainter的具体实现：

```kotlin
/**
 * 异步执行ImageRequest并呈现结果的Painter。
 */
class AsyncImagePainter internal constructor(
    private val parentScope: CoroutineScope,
    request: ImageRequest,
    imageLoader: ImageLoader
) : Painter(), RememberObserver {

    private var rememberScope: CoroutineScope? = null
    // 图片请求的协程的Job
    private var requestJob: Job? = null
    private var drawSize = MutableStateFlow(Size.Zero)

    private var alpha: Float by mutableStateOf(1f)
    private var colorFilter: ColorFilter? by mutableStateOf(null)

    internal var painter: Painter? by mutableStateOf(null)
    internal var filterQuality = DefaultFilterQuality
    internal var isPreview = false

    /** The current [AsyncImagePainter.State]. */
    var state: State by mutableStateOf(State.Empty)
        private set

    var request: ImageRequest by mutableStateOf(request)
        internal set

    var imageLoader: ImageLoader by mutableStateOf(imageLoader)
        internal set

    override val intrinsicSize: Size
        get() = painter?.intrinsicSize ?: Size.Unspecified

    override fun DrawScope.onDraw() {
        // 绘制逻辑非常清爽
        drawSize.value = size
        // Draw the current painter.
        painter?.apply { draw(size, alpha, colorFilter) }
    }
    ...

    override fun onRemembered() {
        // 如果我们处于检查模式（预览），请跳过执行图像请求，并将状态设置为加载。
        // 对于预览模式的支持
        if (isPreview) {
            val request = request.newBuilder().defaults(imageLoader.defaults).build()
            state = State.Loading(request.placeholder?.toPainter())
            return
        }
        // 与Glide扩展类似，创建了一个子作用域
        if (rememberScope != null) return
        val scope = parentScope + SupervisorJob(parentScope.coroutineContext.job)
        rememberScope = scope

        // 观察当前请求+请求大小，并根据需要启动新请求。
        // Coil天然支持Kotlin协程，无需为生产者额外编写代码
        scope.launch {
            snapshotFlow { request }.collect { request ->
                requestJob?.cancel()
                requestJob = launch {
                    // execute是挂起函数，返回ImageResult
                    state = imageLoader.execute(updateRequest(request)).toState()
                }
            }
        }
    }

    override fun onForgotten() {
        rememberScope?.cancel()
        rememberScope = null
        requestJob?.cancel()
        requestJob = null
    }

    override fun onAbandoned() = onForgotten()

    /** Update the [request] to work with [AsyncImagePainter]. */
    private fun updateRequest(request: ImageRequest): ImageRequest {
        return request.newBuilder()
            .target(
                onStart = { placeholder ->
                     // 这里获取到placeholder的Painter并更新State为Loading
                    state = State.Loading(placeholder?.toPainter())
                }
            )
            .apply {
                if (request.defined.sizeResolver == null) {
                    // Coil内关于设置图片大小的代码
                    // size接受一个SizeResolver，一个含suspend函数的接口
                    // 获取尺寸的函数是挂起函数，非常合理，因为很多时候需要等待控件测量完毕才知道大小
                    size(DrawSizeResolver())
                }
                if (request.defined.precision != Precision.EXACT) {
                    precision(Precision.INEXACT)
                }
            }
            .build()
    }

    private fun ImageResult.toState() = when (this) {....}
    private fun Drawable.toPainter() = when (this) {...}

    /** Suspends until the draw size for this [AsyncImagePainter] is unspecified or positive. */
    private inner class DrawSizeResolver : SizeResolver {

        override suspend fun size() = drawSize
            .mapNotNull { size ->
                when {
                    // mapNotNull会将drawSize转化为Flow，同时过滤null值，然后挂起函数first()
                    // 将会返回Flow中传送的第一个值
                    size.isUnspecified -> CoilSize.ORIGINAL
                    size.isPositive -> CoilSize(size.width.roundToInt(), size.height.roundToInt())
                    else -> null
                }
            }
            .first()
    }

    /**
     * The current state of the [AsyncImagePainter].
     * 状态定义
     */
    sealed class State {
        abstract val painter: Painter?
        object Empty : State() {
            override val painter: Painter? get() = null
        }
        data class Loading(
            override val painter: Painter?,
        ) : State()
        data class Success(
            override val painter: Painter,
            val result: SuccessResult,
        ) : State()
        data class Error(
            override val painter: Painter?,
            val result: ErrorResult,
        ) : State()
    }
}
```

与Glide扩展库的思路类似，updatePainter函数会监听AsyncImagePainter的加载状态变化，同时更新AsyncImagePainter内的Painter字段。

```kotlin
@Composable
private fun updatePainter(
    imagePainter: AsyncImagePainter,
    request: ImageRequest,
    imageLoader: ImageLoader
) {
    // This may look like a useless remember, but this allows any painter instances
    // to receive remember events (if it implements RememberObserver). Do not remove.
    // 与Glide扩展库一样，允许结果Painter实例接收remember事件（如果它实现了RememberObserver）
    val state = imagePainter.state
    val painter = remember(state) { state.painter }

    // 如果没有CrossfadeTransition（实现渐入变换）的话，直接设置imagePainter.painter并返回
    val transition = request.defined.transitionFactory ?: imageLoader.defaults.transitionFactory
    if (transition !is CrossfadeTransition.Factory) {
        imagePainter.painter = painter
        return
    }

    // ValueHolder是一个包含static field的数据类，目的是储存state.painter的值，
    // 避免在state.painter值更新后函数rememberCrossfadePainter重组，
    // 与rememberUpdatedState有异曲同工之妙，估计是因为rememberUpdatedState没有
    // 传入key的API（这里要监听request变化），所以这里提供了简易的避免重组的实现
    val loading = remember(request) { ValueHolder<Painter?>(null) }
    if (state is State.Loading) loading.value = state.painter

    // 必须位于Success状态且图片是从网络或磁盘加载的，才允许启动Crossfade，否则返回即可
    if (state !is State.Success || state.result.dataSource == DataSource.MEMORY_CACHE) {
        imagePainter.painter = painter
        return
    }

    // Set the crossfade painter.
    // 千呼万唤始出来的CrossfadePainter
    imagePainter.painter = rememberCrossfadePainter(
        key = state,
        start = loading.value,
        end = painter,
        scale = request.scale,
        durationMillis = transition.durationMillis,
        fadeStart = !state.result.isPlaceholderCached,
        preferExactIntrinsicSize = transition.preferExactIntrinsicSize
    )
}
/** A simple mutable value holder that avoids recomposition. */
// 使用静态字段（static）避免重组
private class ValueHolder<T>(@JvmField var value: T)
```

CrossfadePainter的实现如下：

```kotlin
@Stable
private class CrossfadePainter(
    private var start: Painter?,
    private val end: Painter?,
    private val scale: Scale,
    private val durationMillis: Int,
    private val fadeStart: Boolean,
    private val preferExactIntrinsicSize: Boolean,
) : Painter() {

    private var invalidateTick by mutableStateOf(0)
    private var startTimeMillis = -1L
    private var isDone = false

    private var maxAlpha: Float by mutableStateOf(1f)
    private var colorFilter: ColorFilter? by mutableStateOf(null)

    override val intrinsicSize get() = computeIntrinsicSize()

    override fun DrawScope.onDraw() {
        // 如果Alpha变化完毕，直接使用end绘制
        if (isDone) {
            drawPainter(end, maxAlpha)
            return
        }

        // Initialize startTimeMillis the first time we're drawn.
        val uptimeMillis = SystemClock.uptimeMillis()
        if (startTimeMillis == -1L) {
            startTimeMillis = uptimeMillis
        }

        // Alpha的百分比 = (当前时间 - 开始时间) / 持续时间
        val percent = (uptimeMillis - startTimeMillis) / durationMillis.toFloat()
        val endAlpha = percent.coerceIn(0f, 1f) * maxAlpha
        val startAlpha = if (fadeStart) maxAlpha - endAlpha else maxAlpha
        isDone = percent >= 1.0

        // Loading占位图渐出，Success图片结果渐入
        drawPainter(start, startAlpha)
        drawPainter(end, endAlpha)

        if (isDone) {
            start = null
        } else {
            // Increment this value to force the painter to be redrawn.
            invalidateTick++
        }
    }
    ...
}
```

现在来总结一下Coil远程图片扩展的实现思路：

1. 图片加载：Coil对协程提供直接的支持，size函数、execute加载函数本身就是挂起函数，因此无需额外的转换逻辑。而AsyncImagePainter则使用Job来控制图片加载协程。

   AsyncImagePainter并不具有@Composable上下文，作为替代，它实现了RememberObserver来监听控件是否已经离屏销毁。

2. 图片大小约束：依赖于DrawContext的Size。

3. 渐入动画实现：依赖于DrawScope.onDraw()内的重绘行为，通过对透明度Alpha的百分比计算来实现，令Loading状态的占位图渐出，Success状态的最终结果渐入。

4. loading占位图、error图等：由Coil提供具体的实现。

根据上述分析我们可以发现，相比于Glide或是Picasso，基于Kotlin协程实现的图片加载库Coil，的确能够很轻松与Jetpack Compose配合工作。

至此对扩展库的分析已经完毕。横向对比来说，无论是对Picasso还是Glide进行扩展，我们都得额外做一些处理，才能够令本身不支持协程的它们在Compose下正常工作。要注意的是，单纯使用自定义的Target把结果返回到某个State，这种简单的做法在列表中可能会遇到严重的性能问题，因为Glide也好，Picasso也好，它们内部实现中取消图片加载以避免图片错位、闪烁的重要参照物就是ImageView，随着列表滑动不断创建的自定义的Target无法被它们识别并进行相应处理。相比之下基于协程的Coil的加载能够变得简单得多，我们只需要利用Job本身就可以控制加载的协程。

