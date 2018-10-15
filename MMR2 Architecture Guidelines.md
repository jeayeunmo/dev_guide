
# Maple MMR2 Architecture Guidelines

Maple Android app (MMR2) that we use MMR1 output as a reference.

Libraries and tools included:

- Support libraries
- [Architecture Components](https://developer.android.com/topic/libraries/architecture/)
- [Retrofit 2](http://square.github.io/retrofit/)
- [Dagger 2](http://google.github.io/dagger/)
- [Picasso](http://square.github.io/picasso/)
- [Hockey](https://hockeyapp.net/)
- [Glide](https://github.com/bumptech/glide)

- Functional tests with [Espresso](https://google.github.io/android-testing-support-library/docs/espresso/index.html)
- [Robolectric](http://robolectric.org/)
- [Mockito](http://mockito.org/)
- [Checkstyle](http://checkstyle.sourceforge.net/), [PMD](https://pmd.github.io/) and [Findbugs](http://findbugs.sourceforge.net/) for code analysis

## Requirements

- JDK 1.8
- [Android SDK](http://developer.android.com/sdk/index.html).
- Min:Android 5.0(LOLLIPOP) [(API 21) ](http://developer.android.com/tools/revisions/platforms.html).
- Max:Android 8.0(Oreo) [(API 27) ](http://developer.android.com/tools/revisions/platforms.html).
- Latest Android SDK Tools and build tools.


## Architecture

This project follows MRR1's Android architecture guidelines that are based on [MVVM (Model-View-View Model)](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel). Read more about them [Guide to app architecture](https://developer.android.com/jetpack/docs/guide). 

![](https://github.com/ribot/android-guidelines/raw/master/architecture_guidelines/architecture_diagram.png)

### How to implement a new screen following MVVM

Imagine you have to implement a nutirition's some screen. 

1. Create a new package under `maple/ui/modules/` called `nutirition`
2. Create a new sub packages under `nutirition` 

| package           | description        | Example                      |
| ------------------| ----------------   | ---------------------------- |
| nutirition        | pillar             |                              |
|  - views          | activity           | `QuickRefillActivity`        |
|  -- fragments     | fragment           | `HistoryFragment`            |
|  - viewmodels     | viewmodel          | `PrescriptionsViewModel`     |
|  - models         | model              | `PrescriptionStatusCodes`    |
|  - interfaces     | interface          | `TransferStepValidation`     |
|  - bindingmodels  | bindingmodel        | `AddPharmacyBindingModel`   |
|  - adapter        | RecyclerViewAdapter | `HistoryAdapter`            |

3. Create an new screen for Activity or Fragment called `activity_xyz`,`fragment_xyz`. as a result binding file is generated
automatically `activityXyzBiding`.

```xml
//define variable
<layout xmlns:android="http://schemas.android.com/apk/res/android">

    <data>
        <variable
            name="xyzViewModel"
            type="ca.shoppersdrugmart.maple.ui.modules.nutirition.viewmodels.XyzViewModel"/>
    </data>
 
 ...
 
 <include
    android:id="@+id/xyz_cta"
    layout="@layout/view_generic_button"
    android:layout_width="match_parent"
    android:layout_height="@dimen/default_button_height"
    android:layout_marginTop="@dimen/margin_xlarge"
    app:click='@{(v) -> xyzViewModel.loadData()}'
  ....   
          
```

4. Create an new ViewModel for Activity or Fragment called `XyzViewModel`. 
 
```java
//define variable
private final LiveData<Resource<XyzDataModel>> mXyz;

//define observemethod for activity
public LiveData<Resource<XyzDataModel>> observeXyz() {
        return mXyz;
}

//add Transformations.map to  in constructor
public XyzViewModel() {
        mXyz = Transformations.map(mXyzRepo.observeXyzData(), xyz -> {
            mLoading.set(xyz.status == Status.LOADING);
            return xyz;
        });
}

//define method to call Repo
public void loadData() {
        mXyzRepo.getLoadData();
}

```

5. Create an new Activity called `XyzActivity`. You could also use a Fragment.

-- 원래는 이부분을 다양하게 종류별로 만들면 좋을텐데.. 일단 액티비에만 치우치게 작성하고
나중에 나머지는 추가한다.


3. Create an new Activity called `XyzActivity`. You could also use a Fragment.
4. Define the view interface that your Activity is going to implement. Create a new interface called `SignInMvpView` that extends `MvpView`. Add the methods that you think will be necessary, e.g. `showSignInSuccessful()`

Imagine you have to implement a sign in screen. 

1. Create a new package under `ui` called `signin`
2. Create an new Activity called `ActivitySignIn`. You could also use a Fragment.
3. Define the view interface that your Activity is going to implement. Create a new interface called `SignInMvpView` that extends `MvpView`. Add the methods that you think will be necessary, e.g. `showSignInSuccessful()`
4. Create a `SignInPresenter` class that extends `BasePresenter<SignInMvpView>`
5. Implement the methods in `SignInPresenter` that your Activity requires to perform the necessary actions, e.g. `signIn(String email)`. Once the sign in action finishes you should call `getMvpView().showSignInSuccessful()`.
6. Create a `SignInPresenterTest`and write unit tests for `signIn(email)`. Remember to mock the  `SignInMvpView` and also the `DataManager`.
7. Make your  `ActivitySignIn` implement `SignInMvpView` and implement the required methods like `showSignInSuccessful()`
8. In your activity, inject a new instance of `SignInPresenter` and call `presenter.attachView(this)` from `onCreate` and `presenter.detachView()` from `onDestroy()`. Also, set up a click listener in your button that calls `presenter.signIn(email)`.

## Code Quality

This project integrates a combination of unit tests, functional test and code analysis tools. 

### Tests

To run **unit** tests on your machine:

``` 
./gradlew test
``` 

To run **functional** tests on connected devices:

``` 
./gradlew connectedAndroidTest
``` 

Note: For Android Studio to use syntax highlighting for Automated tests and Unit tests you **must** switch the Build Variant to the desired mode.

### Code Analysis tools 

The following code analysis tools are set up on this project:

* [PMD](https://pmd.github.io/): It finds common programming flaws like unused variables, empty catch blocks, unnecessary object creation, and so forth. See [this project's PMD ruleset](config/quality/pmd/pmd-ruleset.xml).

``` 
./gradlew pmd
```

* [Findbugs](http://findbugs.sourceforge.net/): This tool uses static analysis to find bugs in Java code. Unlike PMD, it uses compiled Java bytecode instead of source code.

```
./gradlew findbugs
```

* [Checkstyle](http://checkstyle.sourceforge.net/): It ensures that the code style follows [our Android code guidelines](https://github.com/ribot/android-guidelines/blob/master/project_and_code_guidelines.md#2-code-guidelines). See our [checkstyle config file](config/quality/checkstyle/checkstyle-config.xml).

```
./gradlew checkstyle
```

### The check task

To ensure that your code is valid and stable use check: 

```
./gradlew check
```

This will run all the code analysis tools and unit tests in the following order:

![Check Diagram](https://github.com/mojeayeun/dev_guide/blob/master/check-task-diagram.png)
 
## Distribution

The project can be distributed based MMR2 policy.


