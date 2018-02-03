---
layout: post
title: Getting started with android architecture components and MVVM
header-img: "img/post-bg-06.jpg"
comments: true
---

There were some interesting stuff at this year's Google I/O 17. One such session was [Architecture Components - Introduction](https://events.google.com/io/schedule/?section=may-17&sid=77a07bfa-52e2-4488-8166-53f5c5a15ebc)
by Yigit Boyar and Luke Bergstorm. The announcement included new [architecture components library](https://developer.android.com/topic/libraries/architecture/index.html).

### What's Android architecture components


>A new collection of libraries that help you design robust, testable, and maintainable apps. Start with classes for managing your >UI component lifecycle and handling data persistence.


The architecture components come with lot of new components such as  `LiveData, ViewModel, LifecycleObserver and LifecycleOwner` and `Room` Persistence library. Together these components would help developers follow the right architecture pattern with proper separation of responsibility for each layer (more on that later) and help survive configuration changes, avoid memory leaks and updating the UI in case of changes in our data.

1. **LifecycleOwner** - Interface for components having Lifecycle (such as Activities and Fragments).

2. **LifecycleObserver** - LifecycleObserver subscribes to LifecycleOwner's lifecycle state changes and are self sufficient. It does not have to rely on Activity or fragment's `onStart()` or `onStop()` lifecycle callback to initialise and stop it.

3. **LiveData** - It is an observable data holder, it notifies the observers in case of data changes. It is also lifecycle aware component i.e. it respects the lifecycle state of the `LifecycleOwner` (Activities or Fragments). This helps preventing memory leaks and other issues.

4. **ViewModel**- Objects that provide data for the UI components. They does not have the references to the view and are unaffected by the lifecycle of LifecycleOwner.

5. **Room** - SQLite based object mapping library. Room abstracts the underlying layer of implementation details like creating and managing database and tables. Room uses annotations to generate code at compile time. Furthermore Room also supports LiveData and Rx Java 2 Flowables

### MVVM Pattern using Architecture components

To demonstrate the MVVM architecture pattern, let's create a simple android app to fetch the issues of any Github repository. In this post We will be mostly be covering ViewModel and LiveData components.

![MVVM Architecture]({{ site.baseurl }}/img/mvvm-architecture.png)
Picture from: [Android developers site](https://developer.android.com/topic/libraries/architecture/guide.html)

We will be having the following components layers:

1. **View** - This layer contains UI components and is responsible for view related code such as initialising child views, displaying progress bar, receiving input from user and handling animations, etc. Example: Activities and Fragments.

2. **ViewModel** - ViewModels provide data to the UI components. In our case views will be using LiveData to observe the data changes in ViewModel.

3. **Repository** - Repositories abstract the underlying implementation of Data sources that an app can use cache and fetch the data. This abstraction is helpful in two ways: 1) the code is not dependent on the concrete implementation of data store, and 2) Because of previous point, we can swap the implementation of datastore at any time such as for testing.

4. **Data Service** - This layer contains the actual code that caches (by using SQLite for example) or fetches data (like by fetching data from backend api's using retrofit).


### Enough of talk. Let's Code

It is assumed that you have some understanding of [retrofit](https://github.com/square/retrofit).
The code described below can be found in the Github [here](https://github.com/shahbazahmed1269/androidgithubissues).

Open project level `build.gradle` file and add the following:
{% highlight groovy %}
allprojects {
    repositories {
        jcenter()
        maven { url 'https://maven.google.com' }
    }
}
{% endhighlight %}
Add the retrofit 2 and architecture components dependencies to app level `build.gradle` file.

{% highlight groovy %}
compile 'com.squareup.retrofit2:retrofit:2.1.0'
compile 'com.squareup.retrofit2:converter-gson:2.1.0'

compile 'android.arch.lifecycle:runtime:1.0.0'
compile 'android.arch.lifecycle:extensions:1.0.0-alpha9-1'
annotationProcessor 'android.arch.lifecycle:compiler:1.0.0-alpha9-1'
{% endhighlight %}

Next we will create the entities.

Enter the following URL in your browser [https://api.github.com/repos/square/retrofit/issues](https://api.github.com/repos/square/retrofit/issues) to get a list of JSON objects. Grab one one JSON object and head to the following site: [jsonschema2pojo](http://www.jsonschema2pojo.org/) and paste the copies JSON object in the text area. Select source type as `JSON` and Annotation Style as `GSON` since we will be using GSON converter to convert JSON to entity.

Now click preview and grab the generated code. Create 2 entities `Issue` and `User`.

![jsonschema2pojo]({{ site.baseurl }}/img/jsonschema2pojo.png)

Next, lets create another entity `ApiResponse`.

{% highlight java %}
public class ApiResponse {
    private List<Issue> issues;
    private Throwable error;

    public ApiResponse(List<Issue> issues) {
        this.issues = issues;
        this.error = null;
    }

    public ApiResponse(Throwable error) {
        this.error = error;
        this.issues = null;
    }

    // Getters...
}
{% endhighlight %}

We will use `ApiResponse` to communicate data from Repository to ViewModel and ultimately to Activity.
So if we get any error while fetching data from the remote api, we will set Error in the `ApiResponse`, else we will set the list of `Issue` objects into it.

Next let's create Retrofit Service interface:

{% highlight java %}
public interface GithubApiService {
    @GET("/repos/{owner}/{repo}/issues")
    Call<List<Issue>> getIssues(@Path("owner") String owner, @Path("repo") String repo);
}
{% endhighlight %}

Next, we'll create Repository. First lets create `IssueRepository` interface

{% highlight java %}
public interface IssueRepository {
    LiveData<ApiResponse> getIssues(String owner, String repo);
}
{% endhighlight %}

Now we'll create `IssueRepositoryImpl` which implements `IssueRepository` interface. As discussed above Repositories are used to abstract the communication of rest of the code to the Data sources (such as Database or API calls). In our case `IssueRepositoryImpl` will use `GithubApiService` to fetch data from Github API and return the value as a `LiveData`.
{% highlight java %}
public class IssueRepositoryImpl implements IssueRepository {

    public static final String BASE_URL = "https://api.github.com/";
    private GithubApiService mApiService;

    public IssueRepositoryImpl() {
        Retrofit retrofit = new Retrofit.Builder()
                .addConverterFactory(GsonConverterFactory.create())
                .baseUrl(BASE_URL)
                .build();
        mApiService = retrofit.create(GithubApiService.class);
    }

    public LiveData<ApiResponse> getIssues(String owner, String repo) {
        final MutableLiveData<ApiResponse> liveData = new MutableLiveData<>();
        Call<List<Issue>> call = mApiService.getIssues(owner, repo);
        call.enqueue(new Callback<List<Issue>>() {
            @Override
            public void onResponse(Call<List<Issue>> call, Response<List<Issue>> response) {
                liveData.setValue(new ApiResponse(response.body()));
            }

            @Override
            public void onFailure(Call<List<Issue>> call, Throwable t) {
                liveData.setValue(new ApiResponse(t));
            }
        });
        return liveData;
    }

}
{% endhighlight %}

In the `getIssues()` method we create a `MutableLiveData` from the data obtained from retrofit. `MutableLiveData` is the subclass of LiveData that has `setValue(T)` method that can be used to modify the value it holds.

Next let's create our ViewModel class named `ListIssuesViewModel` which extends `ViewModel` abstract class:

{% highlight java %}
public class ListIssuesViewModel extends ViewModel {

    private MediatorLiveData<ApiResponse> mApiResponse;
    private IssueRepository mIssueRepository;

    // No argument constructor
    public ListIssuesViewModel() {
        mApiResponse = new MediatorLiveData<>();
        mIssueRepository = new IssueRepositoryImpl();
    }

    @NonNull
    public LiveData<ApiResponse> getApiResponse() {
        return mApiResponse;
    }

    public LiveData<ApiResponse> loadIssues(@NonNull String user, String repo) {
        mApiResponse.addSource(
                mIssueRepository.getIssues(user, repo),
                apiResponse -> mApiResponse.setValue(apiResponse)
        );
        return mApiResponse;
    }

}
{% endhighlight %}

`ListIssuesViewModel` will fetch the data requested by the UI from the IssueRepository. It has `MediatorLiveData` `mApiResponse` which is observed by the UI. `MediatorLiveData` is a subclass of `MutableLiveData` which allows us to observe one or more LiveData (`LiveData` from Repository's `getIssues()` method in our case) and propagate the changes to it own observers (Activity in our case).

**Note**:
1. In case you want to have a ViewModel class with non-empty constructor, you have to create a Factory class which would create instance of you ViewModel and that Factory class has to implement `ViewModelProvider.Factory` interface.

2. If you want reference to Application context in your View Model class, you can use [AndroidViewModel](https://developer.android.com/reference/android/arch/lifecycle/AndroidViewModel.html) class instead of `ViewModel` class.

Finally, create an activity which extends `AppCompatActivity` class (Since `LifecycleActivity` has been deprecated as of version architecture components `1.0.0-alpha9–1` and both `AppCompatActivity` and Support Fragment now implement the `LifecycleOwner` interface) with EditText and Recycler View.
In `onCreate()` we will initialise the ViewModel, observe the `MediatorLiveData` property `mApiResponse` and take appropriate action to display the view.
If user initiates a new search query, we will call `viewModel. loadIssues(@NonNull String user, String repo)` method with appropriate parameters.

{% highlight java %}
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mViewModel = ViewModelProviders.of(this).get(ListIssuesViewModel.class);
        setupView();
        // Handle changes emitted by LiveData
        mViewModel.getRes().observe(this, apiResponse -> {
            if (apiResponse.getError() != null) {
                handleError(apiResponse.getError());
            } else {
                handleResponse(apiResponse.getIssues());
            }
        });
    }

  // Rest of the code

}
{% endhighlight %}

So we now have an app which uses android recommended architecture pattern by using MVVM, LiveData and Repository.
Our app also persists data across configuration changes such as screen rotations.

<img src="{{ site.baseurl }}/img/github-issues-shot-1.png" alt="github-issues-shot-1" style="width: 480px;"/>

### What's next

Android developer's [Guide to app architecture](https://developer.android.com/topic/libraries/architecture/guide.html) suggests using Dagger 2 library for dependency injection and Room ORM for data persistence.
