---
layout: post
title: 在andorid中使用kotlin的协程
category: kotlin
tags: 协程
description: java 的协程太迟了，估计要到java 13了，但是目前大多数的项目还停留在java 8, 估计要等很长时间。而kotlin 1.3 已经提供了稳定的协程, 至少个人用起来还没有什么坑遇到，并且使用体验还是不错的。
---

## 从进程到线程再到协程

首先我们先从进程的概念开始，什么进程？进程是具有独立功能的程序在某个数据集合上的一次运行活动，也是操作系统进行资源分配和保护的基本单位。换句话说，进程首先包括一个可执行文件也就是程序，还包括操作系统为运行这个程序分配的用与运行程序的一系列资源。

接下来再谈线程，为什么会需要线程呢，其实进程已经可以解决一些并发场景下的需求了，其实主要是为了效率考虑，引入线程是为了减少程序并发执行时所付出的时空开销，使得并发颗粒度更细，并发性更好。解决问题的基本思路是：把进程的两项功能“独立分配资源”和“被调度分派执行”分开，前一项任务仍由进程完成，作为系统资源调度和分派的基本单位，后一项交给线程完成，线程作为系统调度和分派的基本单位（同一个进程内的线程切换不会更改地址空间），会被频繁的调度和切换。线程相比进程，消耗资源更少，切换消耗更低，共享地址空间，易于通信。

好了既然线程已经这么好，为什么还需要协程呢，首先，虽然线程相比进程消耗资源少，单还是消耗资源的，系统支持的最大线程数是有限的，其次，cpu的核心是有数的，太对的线程会导致cpu消耗大量的时间在线程调度上。那什么是协程呢？协程相比线程，是没有“被调度分派执行”的功能的，操作系统对于协程是无感知的，一个线程中可以具备多个协程，协程之间是竞争的，一个协程执行时，其他协程是等待的，程序可以在适当的地方从一个协程跳转到另一个协程或回到之前的协程。协程相比线程，不需要操作系统分配资源和调度，就减少了资源消耗和损耗，协程只需要保存在跳转时保存执行上下文，一边跳回时继续执行。一般有两种实现一种是有栈协程（利用栈来保存上下文），比如说go, 还有java种计划的协程, 一种是无栈协程（在跳转的地方将上下文信息保存到堆上）c#, python, kotlin。

## kotlin的协程语法  

如果你使用c#、python或者是es 7的协程，跟定会觉得kotlin的协程语法有些怪异，这是因为kotlin的定位决定的，kotlin并不是做一个新的独立的编程语言，它可以编译到jvm, android, js, native。并且具有很强的互操作性，这就使得，kotlin只加入一些语法，并没有做一些很基础的改变，这是也是为了互操作性，以及移植性性考虑。比如kotlin种将协程调度器这种东西暴露给用户，而不是像c#中隐藏起来。

好了，不多说了，让我们来看一下语法把，相比无栈协程，不能进行随意的调换，像c#, 使用await等待一个Task对象时，就是跳出当前上下文，等到Task任务完成后，再跳回到这里执行，await只能在async函数中使用， async函数可以被看做一个协程。kotlin的语法有些相似，不过没有await关键字， 只要在调用一个suspend函数时，就会跳出，当syspend函数完成后再跳回，suspend函数可以被看做一个协程。

c# 协程语法

```c#
async void FunAsync()
{
    //do something
    await Task.Sleep(1);// 在此处会跳出
    //do something
    var client = new HttpClient();
    var html = await client.GetStringAsync("https://www.baidu.com")
}
```



kolint 协程语法

```kotlin

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L)// 在此出会跳出
    return 13
}

```

分别了解了c#和kotlin中的协程后，再看一下他们如何使用 C#中的async函数可以随意调用，调用后会返回一个Task包裹的结构。而kotlin中则必须使用launch或者其他CoroutineScope中的函数来启动一个协程，Coroutine中包含协程上下文和调度器，协程上下文包含了协程的Job， 对比c#中的Task， 协程调度器用来调度不同协程执行。你可能会想，为什么c#不需要，CoroutineScope这样的东西呢，其实是C#默认帮我们决定了，协程的调度器默认使用的是当前线程中的ThreadContext， 如果当前线程没有，就在调用协程的线程中执行协程的后半部分。而kotlin 运行于jvm， android上，没有能力说于更改java, 或者说andorid 的Thread结构，以及更改一些库来支持协程，就只好将协程上下文以及调度器暴露出来，这样相比c#麻烦性，不过灵活性更高，而且无侵入式，更容易和别的库框架搭配使用。

```kotlin
import kotlinx.coroutines.*

fun main() {
    GlobalScope.launch { // 在后台启动一个新的协程并继续
        delay(1000L) // 非阻塞的等待 1 秒钟（默认时间单位是毫秒）
        println("World!") // 在延迟后打印输出
    }
    println("Hello,") // 协程已在等待时主线程还在继续
    Thread.sleep(2000L) // 阻塞主线程 2 秒钟来保证 JVM 存活
}
```



## 在andorid中使用kotlin协程

android中网络通信库大家一般使用的都是retrofit 吧，retrofit2 目前已经支持kotlin的协程，可以极大的方便我们代码的编写。比如说，我的一个首页，请求轮播图设置轮播组件数据的代码如下：

```kotlin
    private fun pullBanners() = launch {
            try {
                val banners = api.banner.all().await().data
                Log.i("banners", banners.toString())
                bannerView.setPages(banners, ::BannerViewHolder)
            } catch (e: Exception) {
                Log.e("errror", e.toString())
            }
    }
```

当然，为了支持协程，在HomeFragment类委托MainScrop()返回的对象实现了CoroutineScope接口。其中使用的调度器是Dispathers.Main, 这个在不同平台下不同，在android中是Main thread dispathers。

```kotlin
class HomeFragment : Fragment(), CoroutineScope by MainScope() {
...
}
```

### kotlin 协程在andoroid中的集成

首先，先确定你在项目中启用了kotlin ,  查看你app的build.gradle中有没有启用kotlin-android插件就可以了。

首先，我们主要是使用kotlin的协程请求数据，其中我们使用retrofit来发送请求，使用gson来解析内容。

在app的build.gradle加入这两个库。

```groovy
    implementation 'com.squareup.retrofit2:retrofit:2.3.0'
    implementation 'com.squareup.retrofit2:converter-gson:2.3.0'
```

如果要让retrofit支持kotlin的协程接口，添加如下依赖。

```groovy
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-core:1.1.0'

    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.1.0'

    implementation 'com.jakewharton.retrofit:retrofit2-kotlin-coroutines-adapter:0.9.2'
```

其中kotlinx-coroutines-core和kotlinx-coroutines-android是kotlin的协程库，而ertrofit2-kotlin-coroutines-adapter是让retrofit支持kotlin协程的库。

### 使用retrofit 2的协程接口来请求数据

配置完成后，接下来就来实验一下吧。retrofit的使用大家大概应该都知道吧。首先，我们定义一个接口，使用注解来表明请求方式和路径。如下：

```kotlin
interface BannerApi {

    @GET("/api/banner/all")
    fun all(): Deferred<Response<List<Banner>>>
}
```

注意函数的返回类型，Deferred就是Kotlin协程的返回结果，可以调用await()函数来挂起等待（对比c#中的await), Deferred中包裹就是返回内容，如果是自定义类型，默认会使用gson解析，注意这里的Response是我自定义的一个数据结构。如果想要获取全部内容，可以使用okhttp中的Response结构。

```kotlin
data class Response<T>(var errCode: Int, var msg: String?, var data: T?)
```

之后，我们利用Retrofit创建一个BannerApi对象。这里索性就用我项目中的代码吧，我用了一个Api把所有的Api都集中起来了，并且使用dagger来进行依赖注入，向大家推荐这个框架。

```kotlin
@Module
class ApiModule {

    @Singleton
    @Provides
    fun provideApi(interceptor: AuthInterceptor): Api {
        val client = OkHttpClient.Builder()
                .addInterceptor(interceptor)
                .build()
        val gson = GsonBuilder().setDateFormat("yyyy-MM-dd HH:mm:ss").create()
        val builder = Retrofit.Builder()
                .baseUrl(BASE_URL)
                .client(client)
                .addCallAdapterFactory(CoroutineCallAdapterFactory())
                .addConverterFactory(GsonConverterFactory.create(gson))
                .build()
        return Api(
                builder.create(BannerApi::class.java),
                builder.create(ProjectApi::class.java),
                builder.create(CommentApi::class.java),
                builder.create(NewsApi::class.java),
                builder.create(AuthApi::class.java),
                builder.create(UserApi::class.java),
                builder.create(TransactionApi::class.java)
        )
    }

}
```

好了，看一下我们HomeFragment中的代码。

```kotlin
class HomeFragment : Fragment(), CoroutineScope by MainScope() {

    lateinit var api: Api
        @Inject set

    lateinit var bannerView: MZBannerView<Banner>

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?,
                              savedInstanceState: Bundle?): View? {
        return inflater.inflate(R.layout.fragment_home, container, false)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        activity?.let{it.application.castTo<App>().appComponent.inject(this)}
        bannerView = activity?.findViewById(R.id.banner)!!
        pullBanners()

      
    }

    private fun pullBanners() = launch {
            try {
                val banners = api.banner.all().await().data
                Log.i("banners", banners.toString())
                bannerView.setPages(banners, ::BannerViewHolder)
            } catch (e: Exception) {
                Log.e("errror", e.toString())
            }
    }


}
```

