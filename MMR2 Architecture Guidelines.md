
# Maple MMR2 Architecture Guidelines

Maple (MMR2) uses MMR1 output as a reference.

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

![](./maple_mvvm_aproach_neethu.PNG)

### How to define the responsibilities of MVVM

 It is based on "Separation of concerns , Better testable, Follow SOLID principles' .

| MODEL Responsibilities |
| ------------------| 
| Retrofit         |
| Shared Preferences   | 


| VIEW(UI) Responsibilities |
| ------------------| 
| Working with android.view, android.widget, etc. |
| Showing Dialogs, Toasts, Snackbars   | 
| Event listeners*   | 
| Starting Activities*   | 
| Handling Menu   | 
| Handling permisstions   | 
| Other Android specific stuff & methods which require reference to the Activity Context | 
* Event Listeners is defined in the view.


| VIEWMODEL Responsibilities |
| ------------------| 
| Exposing state(progress, offline, empty, error,etc.)        |
| Exposing data*   | 
| Handling visibility   | 
| Handling Extras & Arguments(Bundle)   | 
| Input validation   | 
| Executing data calls in the Model   | 
| Executing UI commands in the View*   | 
* ViewModel layer should be separated from Android specific classes.
* ViewModel should use only the Application Context (start service , send broadcast, load resource values).

'*' it is changeable in this project.

p.s) reference : https://speakerdeck.com/petrnohejl/mvvm-architecture-on-android


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
automatically `ActivityXyzBiding`.

```xml
//screen file: activity_xyz.xml

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

```java

public class XyzActivity extends BaseActivity<ActivityXyzBiding, XyzViewModel> {

    //view models
    private XyzViewModel mXyzViewModel;

    //onCreate
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        ....

        //LiveData observe 
        mXyzViewModel.observeXyz().observe(this, xyzDataModel ->{
           // do something..

        });
    }

    @Override
    public int getBindingVariable() {
        return BR.xyzViewModel;
    }

    @Override
    public int getLayoutId() {
        return R.layout.activity_xyz;
    }

    @Override
    public XyzViewModel getViewModel() {
        mXyzViewModel = ViewModelProviders.of(this).get(XyzViewModel.class);
        return mXyzViewModel;
    }



```
6. Create an new Java Class called `NutritionRepository`. You should add this to Dagger.

```java

public class NutritionRepository extends BaseRepository {

    private final MediatorLiveData<Resource<XyzDataModel>> mXyzResponse = new MediatorLiveData<>();
    private Observer<ApiResponse<XyzDataModel>> mXyzResponseObserver;

    private XyzDataModel mXyzDataModel = new XyzDataModel();

    public XyzRepository getXyzDataModel() {
        return mXyzDataModel;
    }

    public LiveData<Resource<XyzDataModel>> observeXyzData() {
        mXyzResponseObserver = xyzDataGenericModel -> {
            if (xyzDataGenericModel == null) return;
            mXyzDataModel = xyzDataGenericModel.body;
        };

        return mXyzResponse;
    }

    //if there are parameters..
    //public void getLoadData(..param1,..param2,...) {
    //esle no paramters
    public void getLoadData() {

        //.. set parameter
        XyzPayload payload = new XyzPayload();
        payload.param1 = param1;
        payload.param2 = param2;
        
        genericCall(mXyzResponse, mApi.getNutritionService().getData(payload),mXyzResponseObserver);
    }
```

7. Add an new Interface Class called `NutritionService`
```java

public interface NutritionService {

    ...

    @GET("xxx/yyyy")
    LiveData<ApiResponse<XyzDataModel>> getData();

    ...
}
    
```


8. Update method called `getNutritionService()` at ApiController(../data/sources/remote/rest).

```java

public class ApiController {
    ...
    //add
    private NutritionService mNutritionService;
    ...
    //update mBaseUrlNutrition
    private String mBaseUrlMaple, mBaseUrlMedeo, mBaseUrlOkta, mBaseUrlNutrition;
    ...
    
    //update String nutritionBaseUrl
    public void init(final Context context, String mapleBaseUrl, String medeoBaseUrl, String oktaBaseUrl, 
String nutritionBaseUrl, String environment) {
    
    ...
    //add
    mBaseUrlNutrition = nutritionBaseUrl;
    ...
    
    //add
    mNutritionService = buildService(mBaseUrlNutrition).create(NutritionService.class);
    
    }
    
    //add
    public NutritionService getNutritionService() {
        return mNutritionService;
    }
    
```

9. Add method called `provideNutritionRepository()` at RepositoriesModule(../data/di/module).

```java

   @Singleton
   @Provides
   public NutritionRepository provideNutritionRepository(){
      return new NutritionRepository();
   }
   
```

10. Call & Event Flow.
![](./CallandEventFlow.PNG)


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
 



