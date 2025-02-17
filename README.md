[![](https://jitpack.io/v/thefuntasty/mvvm-android.svg)](https://jitpack.io/#thefuntasty/mvvm-android)
[![Build Status](https://app.bitrise.io/app/7bcc28f1329d6773/status.svg?token=RvtAnWzfOq-0n7TIW5By8g&branch=master)](https://app.bitrise.io/app/7bcc28f1329d6773)

# MVVM Android


MVVM Android is the framework based on Android Architecture components, which gives you set of
base classes to implement concise, testable and solid application. It combines built-in
support for Dagger 2 dependency injection, View DataBinding, ViewModel and RxJava
interactors (use cases). Architecture described here is used among wide variety of
projects and it's production ready.

# Download
`build.gradle.kts`
```groovy
allprojects {
    repositories {
        maven { url 'https://jitpack.io' }
    }
}
```

`app/build.gradle.kts`
```groovy
dependencies {
    implementation("com.github.thefuntasty:mvvm-android:LatestVersion")
}    
```

# Table of contents

1. [Getting started - Minimal project file hierarchy](#getting-started---minimal-project-file-hierarchy)
2. [Interactors (use-cases)](#interactors-use-cases)
3. [UI changes flow](#ui-changes-flow)
4. [Stores (Repositories)](#stores-repositories)
5. [About](#about)

# Getting started - Minimal project file hierarchy
Minimal working project must contain files as presented in `example-minimal`
module. File hierarchy might looks like this:
```
example-minimal
`-- src/main
    |-- java/com/example
    |   |-- injection  
    |   |   |-- ActivityBuilderModule.kt
    |   |   |-- ApplicationComponent.kt
    |   |   `-- ApplicationModule.kt
    |   |-- ui 
    |   |   |-- base/BaseActivity.kt
    |   |   `-- main
    |   |       |-- MainActivity.kt
    |   |       |-- MainActivityModule.kt
    |   |       |-- MainView.kt
    |   |       |-- MainViewModel.kt
    |   |       |-- MainViewModelFactory.kt
    |   |       `-- MainViewState.kt
    |   `-- App.kt 
    `-- res/layout/activity_main.xml  
```

Keep in mind this description focuses on architecture `.kt` files. Android related files like an 
`AndroidManifest.xml` are omitted. Let's describe individual files one by one:

#### `ActivityBuilderModule.kt` 
File contains Dagger module class that takes responsibility of proper injection
into Activities. This is the place where every Activity and its `ActivityModule` 
in project must be specified to make correct ViewModel injection work.
 
```kotlin
@Module
abstract class ActivityBuilderModule {

    @ContributesAndroidInjector(modules = [MainActivityModule::class])
    abstract fun mainActivity(): MainActivity
}
``` 

#### `ApplicationComponent.kt`

ApplicationComponent interface combines your singleton Dagger modules and defines
how `DaggerApplicationComponent` should be generated.
```kotlin
@Singleton
@Component(
    modules = [
        AndroidInjectionModule::class,
        AndroidSupportInjectionModule::class,
        ActivityBuilderModule::class,
        ApplicationModule::class
    ]
)
interface ApplicationComponent : AndroidInjector<App> {

    @Component.Builder
    interface Builder {

        @BindsInstance
        fun application(app: App): Builder

        fun build(): ApplicationComponent
    }
}
```

#### `ApplicationModule.kt`

Application module definition. Your singleton scoped objects might
be specified here and injected wherever needed. Example implementation:
```kotlin
@Module
class ApplicationModule {

    @Singleton
    @Provides
    fun moshi(): Moshi = Moshi.Builder().build()
}
```

#### `BaseActivity.kt`

All of Activities in the project should inherit from this class to make DataBinding work properly.
Be aware of fact BR class used in this class is generated when there is at least one layout file 
with correctly defined data variables. Read more [here](#activity_mainxml).
```kotlin
abstract class BaseActivity<VM : BaseViewModel<VS>, VS : ViewState, B : ViewDataBinding> :
    BaseDaggerBindingActivity<VM, VS, B>() {

    override val brViewVariableId = BR.view
    override val brViewModelVariableId = BR.viewModel
    override val brViewStateVariableId = BR.viewState
}
```

#### `MainActivity.kt`

Example Activity implementation. `viewModelFactory` and `layoutResId` must be overridden in every
Activity in order to make ViewModel injection and DataBinding work. `ActivityMainBinding` used
in `BaseActivity` constructor is generated from related `activity_main.xml` layout file. Make sure this file
exists and have root tag `<layout>` before you try to build your code. `ViewModel` can be
accessed through derived `viewModel` field.
```kotlin
class MainActivity : BaseActivity<MainViewModel, MainViewState, ActivityMainBinding>(), MainView {

    @Inject override lateinit var viewModelFactory: MainViewModelFactory

    override val layoutResId = R.layout.activity_main
}
```

#### `MainActivityModule.kt`

`MainActivity` scoped module. It becomes useful when you want to provide specific
activity related configuration e.g.:
  
```kotlin
@Module
abstract class MainActivityModule {

    @Provides
    fun provideUser(activity: MainActivity): User = 
            activity.intent.getParcelableExtra("user")
}
```

#### `MainView.kt`

Interface representing actions executable on your Activity/Fragment. These actions
might be invoked directly from xml layout thanks to `view` data variable.  
```kotlin
interface MainView : BaseView
```

#### `MainViewModel.kt`

Activity/Fragment specific ViewModel implementation. You can choose between extending
`BaseViewModel` or `BaseRxViewModel` with build-in support for RxJava based use cases.
```kotlin
class MainViewModel @Inject constructor() : BaseViewModel<MainViewState>() {

    override val viewState = MainViewState
}
```

#### `MainViewModelFactory.kt`

Factory responsible for `ViewModel` creation. It is injected in Activity/Fragment. 
```kotlin
class MainViewModelFactory @Inject constructor(
    override val viewModelProvider: Provider<MainViewModel>
) : BaseViewModelFactory<MainViewModel>() {
    override val viewModelClass = MainViewModel::class
}
```

#### `MainViewState.kt`

State representation of an screen. Should contain set of `LiveData` fields observed
by Activity/Fragment. State is stored in `ViewModel` thus survives screen rotation. 
```kotlin
object MainViewState : ViewState {
    val user = DefaultValueLiveData<User>(User.EMPTY)
}
```

#### `activity_main.xml`

Layout file containing proper DataBinding variables initialization. Make sure correct
types are defined.
```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android">

    <data>
        <variable name="view" type="com.thefuntasty.mvvmsample.ui.main.MainView"/>
        <variable name="viewModel" type="com.thefuntasty.mvvmsample.ui.main.MainViewModel"/>
        <variable name="viewState" type="com.thefuntasty.mvvmsample.ui.main.MainViewState"/>
    </data>

    <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:orientation="vertical"
            android:gravity="center">

    </LinearLayout>
</layout>
```

# Interactors (use-cases)

Module `interactors` contains set of base classes useful to easy execution of
background tasks through RxJava streams. There are five basic types of 
interactors: `BaseObservabler`, `BaseSingler`, `BaseFlowabler`, `BaseMayber`
and finally `BaseCompletabler`. 

Following example describes how to make an API call and how to deal with 
result of this call. 

#### LoginSingler.kt
```kotlin
class LoginSingler @Inject constructor(
    private val apiManager: ApiManager // Retrofit Service
) : BaseSingler<User>() {

    private lateinit var email: String
    private lateinit var pass: String

    fun init(email: String, pass: String) = apply {
        this.email = email
        this.pass = pass
    }

    override fun prepare(): Single<User> {
        return apiManager.login(email, pass)
    }
}
```
#### LoginViewState.kt
```kotlin
class LoginViewState : ViewState {
    // IN - values provided by UI
    val email = DefaultValueLiveData("")
    val password = DefaultValueLiveData("")

    // OUT - Values observed by UI
    val fullName = MutableLiveData<String>()
}
```

#### LoginViewModel.kt
```kotlin
class LoginViewModel @Inject constructor(
    private val LoginSingler: LoginSingler // Inject interactor
) : BaseRxViewModel<LoginViewState>() {
    override val viewState = LoginViewState()

    fun logIn() = with(viewState) {
        loginInteractor.init(email.value, pass.value).execute ({ user ->
            fullName.value = user.fullName // handle success & manipulate state
        }, {
            // handle error
        })
    }
}
```

# UI changes flow
There are two main ways how to reflect data changes in UI. Through `ViewState` observation
or one-shot `Events`. 

## ViewState observation

You can observe state changes and reflect these changes in UI via DataBinding 
observation directly in xml layout:

 ```xml
 <layout xmlns:android="http://schemas.android.com/apk/res/android">
 
     <data>
         <variable name="view" type="com.thefuntasty.mvvmsample.ui.detail.DetailView"/>
         <variable name="viewModel" type="com.thefuntasty.mvvmsample.ui.detail.DetailViewModel"/>
         <variable name="viewState" type="com.thefuntasty.mvvmsample.ui.detail.DetailViewState"/>
     </data>
     
     <TextView
             android:layout_width="wrap_content"
             android:layout_height="wrap_content"
             android:text="@{viewState.myTextLiveData}"/>
 </layout>
```

## Events
Events are one-shot messages sent from `ViewModel` to an Activity/Fragment. They
are based on `LiveData` bus. Events are guaranteed to be delivered only once even when
there is screen rotation in progress. Basic event communication might look like this:

#### `MainEvents.kt`
```kotlin
sealed class MainEvent : Event<MainViewState>()

object ShowDetailEvent : MainEvent()
```

#### `MainViewModel.kt`
```kotlin
class MainViewModel @Inject constructor() : BaseViewModel<MainViewState>() {

    override val viewState = MainViewState

    fun onDetail() {
        sendEvent(ShowDetailEvent)
    }
}
```

#### `MainActivity.kt`
```kotlin
class MainActivity : BaseActivity<MainViewModel, MainViewState, ActivityMainBinding>(), MainView {

    // ...

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        observeEvent(ShowDetailEvent::class) { 
            startActivity(DetailActivity.getStartIntent(this)) 
        }
    }
}
```

# Stores (Repositories)
All our applications respect broadly known repository pattern. The main message this
pattern tells: Define `Store` (Repository) classes with single entity related business logic 
eg. `UserStore`, `OrderStore`, `DeviceStore` etc. Let's see this principle on `UserStore` class
from sample app:

```kotlin
@Singleton
class UserStore @Inject constructor() {
    private val userRelay = BehaviorRelay.createDefault(User.EMPTY)

    fun setUser(user: User) {
        userRelay.accept(user)
        // ... optionally persist user
    }

    fun getUser(): Observable<User> {
        return userRelay.hide()
    }
}
```

With this approach only one class is responsible for `User` related data access. Besides 
custom classes, Room library `Dao`s or for example Retrofit API interfaces might be 
perceived on the same domain level as stores. Thanks to interactors we can easily access, 
manipulate and combine this kind of data on background threads. 

```kotlin
class GetUserFullNameObservabler @Inject constructor(
    private val userStore: UserStore
) : BaseObservabler<String>() {

    override fun prepare(): Observable<String> {
        return userStore.getUser()
            .map { "${it.firstName} ${it.lastName}" }
    }
}
```

We strictly respect this injection hierarchy:

| Application Component | Injects |
| --------- | --------------------- |
| Activity/Fragment | `ViewModel` |
| ViewModel | `ViewState`, Interactor |
| Interactor | `Store` |
| Store | `Dao`, `Persistence`, `ApiService` |



# About
Created with &#x2764; at The Funtasty. Inspired by [Alfonz library](https://github.com/petrnohejl/Alfonz). Licence MIT.
