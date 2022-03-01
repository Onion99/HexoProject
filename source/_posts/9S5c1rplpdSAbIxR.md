---
title: 源码学习-ViewModel
tags:
  - 源码解析
categories: Android
updated: 1638199831000
date: 2021-11-29 23:30:53
---

首先看看ViewModel是怎么样实例化
```kotlin
    protected open fun <T : ViewModel?> getFragmentScopeViewModel(modelClass: Class<T>): T {
        if (!::fragmentProvider.isInitialized) {
            fragmentProvider = ViewModelProvider(this)
        }
        return fragmentProvider.get(modelClass)
    }

    protected open fun <T : ViewModel?> getActivityScopeViewModel(modelClass: Class<T>): T {
        if (!::activityProvider.isInitialized) {
            activityProvider = ViewModelProvider(requireActivity())
        }
        return activityProvider.get(modelClass)
    }
```
我超,原来是通过一个ViewModelProvider去get到的,他妈的这是干什么的呢,为什么要通过ViewModelProvider获取呢
![](http://img.doutula.com/production/uploads/image/2019/06/06/20190606755268_TSvBec.jpg)
我怎么知道,瞧瞧官方正儿八经的回答吧
`ViewModel`:旨在以注重生命周期的方式存储和管理界面相关的数据。ViewModel 类让数据可在发生屏幕旋转等配置更改后继续留存
`ViewModelProvider`: ViewModel的辅助程序类，该类负责为界面准备数据。在配置更改期间会自动保留 ViewModel 对象，以便它们存储的数据立即可供下一个 activity 或 fragment 实例使用
<!-- more -->
好家伙,原来是这对组合,以生命周期的方式耶,看看有什么不同
### Activty与ViewModel的生命周期

> 牛逼,战斗直至Finish

![I8L0jU.png](https://z3.ax1x.com/2021/11/08/I8L0jU.png)

官方说:ViewModel对象存在的时间范围是获取 ViewModel 时传递给 ViewModelProvider 的Lifecycle。ViewModel将一直留在内存中，直到限定其存在时间范围的Lifecycle永久消失：
对于activity，是在activity完成时；
对于fragment，是在 fragment分离时

好了,来看看ViewModelProvider是怎样实现构造的

### ViewModelProvider()

```java
public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {
        // 尼玛的,偷偷调用另一构造函数
        // 1.创建用来存储ViewModel的ViewModelStore, 2. 创建用于实例化新ViewModel的Factory
        this(owner.getViewModelStore(), owner instanceof HasDefaultViewModelProviderFactory
                ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
                : NewInstanceFactory.getInstance());
}
```

#### ViewModelStore

这个是接口何许玩意,原来到后面Activity,Fragment等都实现了这个接口,以来获取跟当前生命周期相关的ViewModelStore,看到没有,有图有真相
![I8xIk4.png](https://z3.ax1x.com/2021/11/08/I8xIk4.png)
看看`ComponentActivity.getViewModelStore()`:
```java
    public ViewModelStore getViewModelStore() {
        if (getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");
        }
        // 叉,这里照我猜想肯定是实现ViewModelStore的单例
        ensureViewModelStore();
        // 返回与此Activity关联的ViewModelStore
        return mViewModelStore;
    }
    //...
    void ensureViewModelStore() {
        if (mViewModelStore == null) {
            // 检索先前由onRetainNonConfigurationInstance()返回的配置变更后的缓存配置
            NonConfigurationInstances nc = (NonConfigurationInstances) getLastNonConfigurationInstance();
            // 如果缓存配置不为空,则取缓存配置的viewModelStore
            if (nc != null) {
                mViewModelStore = nc.viewModelStore;
            }
            // 否则自己实例一个
            if (mViewModelStore == null) {
                mViewModelStore = new ViewModelStore();
            }
        }
    }    
```

嘿嘿,存储ViewModel的ViewModelStore,牛逼啊,用一个HashMap来缓存看到有木有:

```java
public class ViewModelStore {
    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) oldViewModel.onCleared();
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }

    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
}
```
![IG8UgA.png](https://z3.ax1x.com/2021/11/08/IG8UgA.png)
#### ViewModelProviderFactory

同样的后面Activity,Fragment等也实现了HasDefaultViewModelProviderFactory接口,实现自己创建ViewModel的ViewModelProviderFactory, 来看图
![IGEOT1.png](https://z3.ax1x.com/2021/11/08/IGEOT1.png)
再来看看`ComponentActivity.getViewModelStore()`:
```java
    public ViewModelProvider.Factory getDefaultViewModelProviderFactory() {
        if (getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");
        }
        // 妈的,就是这么简单,直接实例化
        if (mDefaultFactory == null) {
            mDefaultFactory = new SavedStateViewModelFactory(
                    getApplication(),
                    this,
                    getIntent() != null ? getIntent().getExtras() : null);
        }
        return mDefaultFactory;
    }
```

有点东西,这样就实例`SavedStateViewModelFactory`了,看看具体怎么`get()`霍
```java
    // 第一步:小get
    public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
        String canonicalName = modelClass.getCanonicalName();
        // 好家伙,看到没有,如果这里判断是局部类或者匿名类,就直接给crash了,谷歌就是牛逼
        if (canonicalName == null) {
            throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
        }
        // 嘿,继续get下去
        return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
    }
    // 第二步:大get
    public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
        ViewModel viewModel = mViewModelStore.get(key);
        // 首先是判断是否有缓存
        if (modelClass.isInstance(viewModel)) {
            if (mFactory instanceof OnRequeryFactory) {
                ((OnRequeryFactory) mFactory).onRequery(viewModel);
            }
            return (T) viewModel;
        }
        // 然后如果为null,则通过具体工厂类去实例化ViewModel
        if (mFactory instanceof KeyedFactory) {
            viewModel = ((KeyedFactory) mFactory).create(key, modelClass);
        } else {
            viewModel = mFactory.create(modelClass);
        }
        // 嘿嘿,放进缓存
        mViewModelStore.put(key, viewModel);
        return (T) viewModel;
    } 
    // ViewModel的实例化
    public <T extends ViewModel> T create(@NonNull String key, @NonNull Class<T> modelClass) {
        boolean isAndroidViewModel = AndroidViewModel.class.isAssignableFrom(modelClass);
        Constructor<T> constructor;
        // 通过反射,去找到当前ViewModel的构造函数
        if (isAndroidViewModel && mApplication != null) {
            constructor = findMatchingConstructor(modelClass, ANDROID_VIEWMODEL_SIGNATURE);
        } else {
            constructor = findMatchingConstructor(modelClass, VIEWMODEL_SIGNATURE);
        }
        if (constructor == null) {
            return mFactory.create(modelClass);
        }
        SavedStateHandleController controller = SavedStateHandleController.create(
                mSavedStateRegistry, mLifecycle, key, mDefaultArgs);
        // 嘿嘿,调用构造函数,实例化        
        try {
            T viewmodel;
            if (isAndroidViewModel && mApplication != null) {
                viewmodel = constructor.newInstance(mApplication, controller.getHandle());
            } else {
                viewmodel = constructor.newInstance(controller.getHandle());
            }
            viewmodel.setTagIfAbsent(TAG_SAVED_STATE_HANDLE_CONTROLLER, controller);
            return viewmodel;
        } catch (IllegalAccessException e) {
            throw new RuntimeException("Failed to access " + modelClass, e);
        } catch (InstantiationException e) {
            throw new RuntimeException("A " + modelClass + " cannot be instantiated.", e);
        } catch (InvocationTargetException e) {
            throw new RuntimeException("An exception happened in constructor of "
                    + modelClass, e.getCause());
        }
    }   
```


### ViewModel的架构作用

向官方敬礼,懂不懂这张图的含金量,懂不懂MMVM
![IGd3ff.png](https://z3.ax1x.com/2021/11/08/IGd3ff.png)
![IGwfKg.png](https://z3.ax1x.com/2021/11/08/IGwfKg.png)
