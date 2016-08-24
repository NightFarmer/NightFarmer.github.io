---
title: 浅析Google官方AndroidMVP架构示例
date: 2016-08-24 10:51:06
tags: [Android, Android架构]
category: Android
---

传统的MVC架构是个非常经典的设计，它将系统的任务进行分层，将代码分割到模型(model)－视图(view)－控制器(controller)三个层面来实现解耦，从而简化开发流程，实现开发任务的分离。
而在android平台中，xml布局文件作为视图的承载能力并不强，通常会将一部分的view操作放在Activity/Fragment中来处理，而Activity/Fragment通常又担任了controller的角色，这就造成了V和C在Android中通常融合在一起，以至MVC的设计架构在Android中并不能进行很好的分层，形成了一种貌合神离的现象。
在这样的背景下，MVP设计架构应运而生。在MVP中单独抽离了一个P(Presenter)层来负责数据流向控制的角色，而xml布局文件和Activity则统一作为V(View)来单一的负责视图的生成和变更。
对于Android中MVP设计架构Google也提供了一些sample，本篇摘取一个待办事项的示例来初步解析Google官方对MVP的实现方式。

<!-- more -->
github：
https://github.com/googlesamples/android-architecture/tree/todo-mvp/
分支：
todo-mvp
todo-mvp-rxjava
todo-mvp-dagger
todo-mvp-loaders
todo-mvp-clean
todo-mvp-tablet
这个库有很多个分支，这里只采用todo—mvp来进行解析

这个项目分支的官方介绍是这样的
### 概述
这个示例是很多其他变体版本的基础，它展示了一个简单的没有其使用其他框架的MVP模式的实现。它使用手动依赖注入，以提供一个本地和远程数据源的存储。异步任务通过callback进行回调处理
![项目结构](https://github.com/googlesamples/android-architecture/wiki/images/mvp.png)

在MVP中，"view"的含义是被重新定义的：
android.view.View类被称为Android View
在MVP的presenter中接受收指令的view则被简单的称作view

### Fragments
使用Fragment有两个原因：
Activity和Fragment的分离对MVP的实现是非常合适的：Activity可以作为views和presenters的创建和关联的控制器
多个view的碎片化的视图充分利用了Fragment的优势

### 概念
在应用程序中有四个功能：
Tasks
TaskDetail
AddEditTask
Statistics
每个功能都有：
- 一个**Contract**作为view和presenter的约定
- 一个**Activity**负责Fragment和presenter的创建
- 一个**Fragment**实现view接口
- 一个**presenter**实现presenter接口

一般情况下业务逻辑存在于presenter中，并通过view操作UI视图

view几乎不含有任何逻辑：它将presenter的指令转换为UI操作，并监听用户操作传递给presenter

Contract是用来定义view和presenter之间通讯的一些接口


### 解析
这个sample有四个功能模块，分别是事项列表/事项详情/新增修改事项/数据统计。项目结构如下：
![项目结构](http://nightfarmer.github.io/public/static/image/mvpPro.png)

根据项目结构可以看到每个功能模块分别定义了一个包，而包下则会为这个功能把Activity/Contract/Fragment/Presenter各定义一份。
这里针对事项详情(taskdetail)来进行剖析。
#### Contract
首先我们查看TaskDetailContract文件，可以看到这是一个接口，而里面定义了另外两个接口分别是View和Presenter
```java
/**
 * This specifies the contract between the view and the presenter.
 */
public interface TaskDetailContract {

    interface View extends BaseView<Presenter> {
        void setLoadingIndicator(boolean active);
        void showMissingTask();
        void hideTitle();
        void showTitle(String title);
        void hideDescription();
        void showDescription(String description);
        void showCompletionStatus(boolean complete);
        void showEditTask(String taskId);
        void showTaskDeleted();
        void showTaskMarkedComplete();
        void showTaskMarkedActive();
        boolean isActive();
    }

    interface Presenter extends BasePresenter {
        void editTask();
        void deleteTask();
        void completeTask();
        void activateTask();
    }
}

```
View和Presenter两个接口分别提供了代办事件的View相关操作和数据相关任务处理的接口，而这两个接口分别继承了另外两个父接口，为所有的View和Presenter提供统一的行为约束。
```java
public interface BaseView<T> {
    void setPresenter(T presenter);//让所有的view可以持有一个presenter
}
public interface BasePresenter {
    void start();//所有的presenter在开始的时候会被回调此方法
}
```
#### Fragment
在google为我们提供的这个示例项目中，作为view的并不是Activity而是Fragment。也就是说在这种模式下所有的Activity中都要存在一个Fragment来承载这个界面的UI元素。而Activity则只负责Fragment和Presenter的初始化和关联操作。
既然Fragment是作为view存在的，那么定义的Fragment理应实现上面定义的view接口：
```java
/**
 * Main UI for the task detail screen.
 */
public class TaskDetailFragment extends Fragment implements TaskDetailContract.View {
    public static final String ARGUMENT_TASK_ID = "TASK_ID";
    public static final int REQUEST_EDIT_TASK = 1;
    ...
    public static TaskDetailFragment newInstance(String taskId) {
        Bundle arguments = new Bundle();
        arguments.putString(ARGUMENT_TASK_ID, taskId);
        TaskDetailFragment fragment = new TaskDetailFragment();
        fragment.setArguments(arguments);
        return fragment;
    }

    @Override
    public void onResume() {
        super.onResume();
        mPresenter.start();//回调presenter的start方法，初始化请求数据并对view进行数据回显
    }

    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
        View root = inflater.inflate(R.layout.taskdetail_frag, container, false);
        setHasOptionsMenu(true);
        mDetailTitle = (TextView) root.findViewById(R.id.task_detail_title);
        mDetailDescription = (TextView) root.findViewById(R.id.task_detail_description);

        // Set up floating action button
        FloatingActionButton fab = (FloatingActionButton) getActivity().findViewById(R.id.fab_edit_task);
        fab.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                mPresenter.editTask();//修改操作，业务逻辑由presenter来处理，实则由presenter来调用本view的showEditTask来进行跳转
            }
        });
        return root;
    }

    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        switch (item.getItemId()) {
            case R.id.menu_delete:
                mPresenter.deleteTask();//删除操作，业务逻辑由presenter来处理，presenter处理完删除任务之后会回调本view的showTaskDeleted方法
        ...
    }
```
view的回调方法
```java

    @Override
    public void showEditTask(String taskId) {//presenter的editTask方法会回调此方法
        Intent intent = new Intent(getContext(), AddEditTaskActivity.class);
        intent.putExtra(AddEditTaskFragment.ARGUMENT_EDIT_TASK_ID, taskId);
        startActivityForResult(intent, REQUEST_EDIT_TASK);
    }

    @Override
    public void showTaskDeleted() {//删除回调
        getActivity().finish();
    }

    @Override
    public void showTitle(String title) {
        mDetailTitle.setVisibility(View.VISIBLE);
        mDetailTitle.setText(title);
    }

    @Override
    public void showDescription(String description) {
        mDetailDescription.setVisibility(View.VISIBLE);
        mDetailDescription.setText(description);
    }
    ...
    @Override
    public boolean isActive() {
        return isAdded();
    }
}

```
所有的view都会实现setPresenter方法来支持presenter的注入
```java
    @Override//由Activity调用此方法为Fragment(view)注入presenter
    public void setPresenter(@NonNull TaskDetailContract.Presenter presenter) {
        mPresenter = checkNotNull(presenter);
    }
```
在Fragment中除了视图的控制/事件的监听/presenter对事件的发送外没有进行任何的业务逻辑处理，而是将任务交给presenter，处理结果则由presenter发起对view的ui控制接口的回调进行通知。
#### Presenter

```java
/**
 * Listens to user actions from the UI ({@link TaskDetailFragment}), retrieves the data and updates
 * the UI as required.
 */
public class TaskDetailPresenter implements TaskDetailContract.Presenter {
    private final TasksRepository mTasksRepository;
    private final TaskDetailContract.View mTaskDetailView;

    @Nullable
    private String mTaskId;

    //presenter由Activity创建，同时presenter持有对view的引用，用于回调view的控制接口，并讲自身交给view引用，也就是说view和presenter相互持有对方的引用。
    //同时presenter持有着当前界面需要的一些临时数据。
    public TaskDetailPresenter(@Nullable String taskId,
                               @NonNull TasksRepository tasksRepository,
                               @NonNull TaskDetailContract.View taskDetailView) {
        this.mTaskId = taskId;
        mTasksRepository = checkNotNull(tasksRepository, "tasksRepository cannot be null!");
        mTaskDetailView = checkNotNull(taskDetailView, "taskDetailView cannot be null!");
        mTaskDetailView.setPresenter(this);
    }

    @Override
    public void start() {
        openTask();
    }

    private void openTask() {
        if (null == mTaskId || mTaskId.isEmpty()) {
            mTaskDetailView.showMissingTask();
            return;
        }

        mTaskDetailView.setLoadingIndicator(true);//ui显示加载
	//根据taskid异步获取详情，并根据查询结果的不同进行相应的ui回调。
        mTasksRepository.getTask(mTaskId, new TasksDataSource.GetTaskCallback() {
            @Override
            public void onTaskLoaded(Task task) {
                // The view may not be able to handle UI updates anymore
                if (!mTaskDetailView.isActive()) {//Fragment不再活动，忽略结果
                    return;
                }
                mTaskDetailView.setLoadingIndicator(false);//ui停止加载
                if (null == task) {
                    mTaskDetailView.showMissingTask();//查询结果不存在，调用ui提示
                } else {
                    showTask(task);//查询结果正常，调用回显接口
                }
            }

            @Override
            public void onDataNotAvailable() {
                // The view may not be able to handle UI updates anymore
                if (!mTaskDetailView.isActive()) {
                    return;
                }
                mTaskDetailView.showMissingTask();
            }
        });
    }

    @Override
    public void editTask() {
        if (null == mTaskId || mTaskId.isEmpty()) {
            mTaskDetailView.showMissingTask();
            return;
        }
        mTaskDetailView.showEditTask(mTaskId);
    }

    @Override
    public void deleteTask() {
        mTasksRepository.deleteTask(mTaskId);//此方法不采用异步，同步执行
        mTaskDetailView.showTaskDeleted();//执行完成后回调ui操作
    }
    ...

    private void showTask(Task task) {
        String title = task.getTitle();
        String description = task.getDescription();

        if (title != null && title.isEmpty()) {
            mTaskDetailView.hideTitle();
        } else {
            mTaskDetailView.showTitle(title);
        }

        if (description != null && description.isEmpty()) {
            mTaskDetailView.hideDescription();
        } else {
            mTaskDetailView.showDescription(description);
        }
        mTaskDetailView.showCompletionStatus(task.isCompleted());
    }
}
```

#### Activity
Activity在这里不担任MVP中的任何一个角色，只作为View和Presenter的创建和关联的控制器
```java
/**
 * Displays task details screen.
 */
public class TaskDetailActivity extends AppCompatActivity {
    public static final String EXTRA_TASK_ID = "TASK_ID";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        setContentView(R.layout.taskdetail_act);

        // Set up the toolbar.
        Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);

        // Get the requested task id
        String taskId = getIntent().getStringExtra(EXTRA_TASK_ID);

        TaskDetailFragment taskDetailFragment = (TaskDetailFragment) getSupportFragmentManager()
                .findFragmentById(R.id.contentFrame);

        if (taskDetailFragment == null) {
            taskDetailFragment = TaskDetailFragment.newInstance(taskId);
            ActivityUtils.addFragmentToActivity(getSupportFragmentManager(),
                    taskDetailFragment, R.id.contentFrame);
        }

        // Create the presenter
        new TaskDetailPresenter(
                taskId,
                Injection.provideTasksRepository(getApplicationContext()),
                taskDetailFragment);
    }
}
```
这里我们看到Activity只是将上个界面传入的数据放入Presenter中，进而由presenter在onstar方法被调用的时候对view(fragment)中的数据进行初始化。


到此为止google为我们提供的MVP的实现方式也就串了一遍，这和现在网上一些文章中的实现可能并不是很一致，大部分开发人员会选择使用Activity作为View的载体，而不是在每个Activity中定义一个Fragment再来展现View。
google的做法可能更符合MVP的设计，Activity只作为路由来使用，而V和P则是游离的存在。
将Activity作为View来使用的话代码则会显得更加简洁，定义的类会少一些，对于view的生命周期控制也会更加简便。
这两种作法孰优孰劣博主也暂时无法定夺，至于想在项目中使用mvp设计架构则建议视情况而定。








