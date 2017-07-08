---
title: 【译】RecyclerView 的使用
date: 2016-11-02
tags:
  - Android
---

原文地址: http://guides.codepath.com/android/Using-the-RecyclerView
### Overview

RecyclerView 是一个新的 ViewGroup，用来渲染任何基于适配器的视图。它被认为是 ListView 和 GridView 的继任者，它被包含在最新的 [support-v7][1] 中。RecyclerView 拥有一个扩展性很强的框架，特别是它既能实现水平布局，也能实现垂直布局。当你的数据集合中的元素由于用户的操作或者是网络事件，在运行时是会改变的，那么你就应该使用 RecyclerView。

如果你要使用 RecyclerView，你需要做下列的工作：

 - RecyclerView.Adapter - 将数据与视图进行绑定
 - LayoutManager - 决定每个元素是如何布局的
 - ItemAnimator - 常用操作的动画，比如说增加或者移除一个元素

此外，它还提供了支持 ListView 添加/删除元素的动画，但是实现起来很困难。RecyclerView 也开始强制使用 ViewHolder 模式，这是一种备受推荐的做法，现在已经集成到 RecyclerView 的框架中了。

想要了解更多的话，点击[这里][2]。
 
<!--more-->
 
#### Compared to ListView

RecyclerView 和它的先驱 ListView 主要有以下几点不同：

 - Required ViewHolder in Adapters - ListView 并不要求它的适配器使用 ViewHolder 模式来提高性能。相反的，RecyclerView 是强制使用 ViewHolder。
 - Customizable Item Layouts - ListView 的布局元素只能是垂直分布的，并且不能被定制化。而 RecyclerView 中的 RecyclerView.LayoutManager 可以让布局元素水平、垂直或者是瀑布式排列。
 - Easy Item Animations - ListView 没有包含增加 / 删除元素的动画。RecyclerView.ItemAnimator 能够处理动画效果。
 - Manual Data Source - ListView 使用不同的适配器来适配不同的数据源，比如说 ArrayAdapter 适配数组，CursorAdapter 适配数据库。RecyclerView.Adapter 实现不同的数据源需要自己来实现。
 - Manual Item Decoration - ListView 使用 android:divider 属性进行分割线的设置。RecyclerView 使用 RecyclerView.ItemDecoration 来手动设置更加具体的分割线。
 - Manual Click Dection - ListView 使用 AdapterView.OnItemClickListener 接口来绑定每个元素的点击事件。RecyclerView 仅支持 RecyclerView.OnItemTouchListener 来管理触摸事件，但是没有内置的点击处理接口。

### Components of a RecyclerView

#### LayoutManagers

RecyclerView 需要 LayoutManager 和 Adapter，才能初始化。布局管理器会对 RecyclerView 中的元素进行组织，并且决定什么时候重用对于用户不可见的视图元素。

RecyclerView 提供了如下布局管理器：

 - LinearLayoutManager 视图元素水平或者垂直显示
 - GridLayoutManager 视图元素以网格的形式显示
 - StaggerGridLayoutManager 视图元素以瀑布流的形式显示

通过继承 [RecyclerView.LayoutManager][3] 来创建一个自定义的布局管理器。

关于自定义布局管理器的视频：[Dave Smith's talk][4]。

#### RecyclerView.Adapter

RecyclerView 包含了一个新的适配器。它与你之前使用的适配器很类似，但它也有一些自己的特性，比如说需要 ViewHolder。你要重写两个主要的方法：一个用来渲染视图和 view holder，另一个用来绑定数据。有一个好处，就是第一个函数仅仅是当我们需要创建新的视图元素才调用。我们不用检查视图元素是否已经被回收了。

#### ItemAnimator

RecyclerView.ItemAnimator 用来组织 ViewGroup 的动画效果，比如说 增加/删除/选择 等操作，并通知了适配器。[DefaultItemAnimator][5] 是默认的动画组织者且效果良好。

### Using the RecyclerView

使用 RecyclerView 需要下面几步：
 1. 添加 RecyclerView 的支持库到 gradle 文件中
 2. 定义一个业务实体类来当作数据源
 3. 在你的 activity 中添加 RecyclerView
 4. 定义视图元素布局文件
 5. 创建 RecyclerView.Adapter 和 ViewHolder 来渲染视图元素
 6. 使用 adapter 绑定数据源来填充 RecyclerView

#### Installation

确保 RecyclerView 的支持库已经添加到了 app/build.gradle：

```java
dependencies {
    ...
    compile 'com.android.support:support-v4:24.2.1'
    compile 'com.android.support:recyclerview-v7:24.2.1'
}
```

点击 "Sync Project with Gradle files" 让你的 IDE 下载相关的资源。


#### Defining a Model

每一个 RecyclerView 都要有数据源的支持。在这篇文章中，我们定义一个 Contact 类表示 RecyclerView 中要展示的数据：

```java
public class Contact {
    private String mName;
    private boolean mOnline;

    public Contact(String name, boolean online) {
        mName = name;
        mOnline = online;
    }

    public String getName() {
        return mName;
    }

    public boolean isOnline() {
        return mOnline;
    }

    private static int lastContactId = 0;

    public static ArrayList<Contact> createContactsList(int numContacts) {
        ArrayList<Contact> contacts = new ArrayList<Contact>();

        for (int i = 1; i <= numContacts; i++) {
            contacts.add(new Contact("Person " + ++lastContactId, i <= numContacts / 2));
        }

        return contacts;
    }
}
```

#### Create the RecyclerView within Layout

在相关的布局文件中，添加 RecyclerView：

```java
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >
    
    <android.support.v7.widget.RecyclerView
      android:id="@+id/rvContacts"
      android:layout_width="match_parent"
      android:layout_height="match_parent" />

</RelativeLayout>
```

我们看到该布局的预览如下：

![ | center][6]

RecyclerView 现在已经嵌入我们的布局文件中了。下一步，我们可以为视图元素定义布局。

#### Creating the Custom Row Layout

在我们创建 adapter 之前，让我们为视图元素创建布局文件。在布局文件中，将会有一个水平布局的 LinearLayout，在 LinearLayout 中，使用 TextView 用来显示姓名，Button 来向该联系人发送消息：

![ | center][7]

该布局文件在 res/layout/item_contact.xml 中，被用作渲染每一个视图元素。**注意**，layout_height 的属性值应该使用 wrap_content，因为在 23.2.1 之前 RecyclerView 的版本中会忽略掉 layout 参数。点击[这里][8]获得详细信息。

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:orientation="horizontal"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:paddingTop="10dp"
        android:paddingBottom="10dp"
        >

    <TextView
        android:id="@+id/contact_name"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        />

    <Button
        android:id="@+id/message_button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:paddingLeft="16dp"
        android:paddingRight="16dp"
        android:textSize="10sp"
        />
</LinearLayout>
```

自定义视图元素布局完成以后，让我们来创建适配器向 RecyclerView 中填充数据。

#### Creating the RecyclerView.Adapter

adapter 所要做的事情是 **convert an object at a position into a list row item to be inserted**。

RecyclerView 的 adapter 需要 ViewHolder 对象的存在，来提供对视图元素内部的访问。我们可以在 ContactsAdapter.java 中创建空的 adapter 和 holder：

```java
// Create the basic adapter extending from RecyclerView.Adapter
// Note that we specify the custom ViewHolder which gives us access to our views
public class ContactsAdapter extends 
    RecyclerView.Adapter<ContactsAdapter.ViewHolder> {

    // Provide a direct reference to each of the views within a data item
    // Used to cache the views within the item layout for fast access
    public static class ViewHolder extends RecyclerView.ViewHolder {
        // Your holder should contain a member variable
        // for any view that will be set as you render a row
        public TextView nameTextView;
        public Button messageButton;

        // We also create a constructor that accepts the entire item row
        // and does the view lookups to find each subview
        public ViewHolder(View itemView) {
            // Stores the itemView in a public final member variable that can be used
            // to access the context from any ViewHolder instance.
            super(itemView);

            nameTextView = (TextView) itemView.findViewById(R.id.contact_name);
            messageButton = (Button) itemView.findViewById(R.id.message_button);
        }
    }
}
```

现在我们已经定义了基本的 adapter 和 Viewholder，我们需要开始填充 adapter。首先，我们为
联系人存储成员变量，并通过构造函数传入联系人数据：

```java
public class ContactsAdapter extends 
    RecyclerView.Adapter<ContactsAdapter.ViewHolder> {

    // ... view holder defined above...

    // Store a member variable for the contacts
    private List<Contact> mContacts;
    // Store the context for easy access
    private Context mContext;

    // Pass in the contact array into the constructor
    public ContactsAdapter(Context context, List<Contact> contacts) {
        mContacts = contacts;
        mContext = context;
    }

    // Easy access to the context object in the recyclerview
    private Context getContext() {
       return mContext;
    }
}
```

任何的 adapter 都有三个基本的方法：onCreateViewHolder 渲染视图布局并创建 holder，onBindViewHolder 根据数据设置视图元素的属性，getItemCount 返回视图元素的个数。我们需要实现这三个方法：

```java
public class ContactsAdapter extends 
    RecyclerView.Adapter<ContactsAdapter.ViewHolder> {

    // ... constructor and member variables

    // Usually involves inflating a layout from XML and returning the holder
    @Override
    public ContactsAdapter.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        Context context = parent.getContext();
        LayoutInflater inflater = LayoutInflater.from(context);

        // Inflate the custom layout
        View contactView = inflater.inflate(R.layout.item_contact, parent, false);

        // Return a new holder instance
        ViewHolder viewHolder = new ViewHolder(contactView);
        return viewHolder;
    }
    
    // Involves populating data into the item through holder
    @Override
    public void onBindViewHolder(ContactsAdapter.ViewHolder viewHolder, int position) {
        // Get the data model based on position
        Contact contact = mContacts.get(position);

        // Set item views based on your views and data model
        TextView textView = viewHolder.nameTextView;
        textView.setText(contact.getName());
        Button button = viewHolder.messageButton;
        button.setText("Message");
    }

    // Returns the total count of items in the list
    @Override
    public int getItemCount() {
        return mContacts.size();
    }
}
```

我们已经完成了 adapter 的创建，剩下的是将 RecyclerView 与 adapter 绑定。

#### Binding the Adapter to the RecyclerView

在 Activity 中，我们会创建一些简单的示例数据来填充 RecyclerView。

```java
public class UserListActivity extends AppCompatActivity {

     ArrayList<Contact> contacts;

     @Override
     protected void onCreate(Bundle savedInstanceState) {
         // ...
         // Lookup the recyclerview in activity layout
         RecyclerView rvContacts = (RecyclerView) findViewById(R.id.rvContacts);

         // Initialize contacts
         contacts = Contact.createContactsList(20);
         // Create adapter passing in the sample user data
         ContactsAdapter adapter = new ContactsAdapter(this, contacts);
         // Attach the adapter to the recyclerview to populate items
         rvContacts.setAdapter(adapter);
         // Set layout manager to position the items
         rvContacts.setLayoutManager(new LinearLayoutManager(this));
         // That's all!
     }
}
```

最后，编译并且运行该 app，你会看到如下的截图。如果你创建了足够多的数据，并且滑动该列表，列表中的视图会回收再利用，并且比 ListView 平滑很多：

![ | center][9]

#### Notifying the Adapter

不像 ListView，RecylerView 不能直接添加或者删除元素。你需要改变数据源，并通知相应的 adapter。无论何时在数据源上做出的改变，都会影响到**存在的**列表。比如，对联系人列表重新初始化并不会影响到 adapter，因为 adapter 引用的是内存中旧的列表数据：

```java
// do not reinitialize an existing reference used by an adapter
contacts = Contact.createContactsList(5);
```

所以，你需要直接修改已经存在的引用：

```java
// add to the existing list
contacts.addAll(Contact.createContactsList(5));
```
**(译者注：就在今天，我就犯了这个错误，引以为戒)**

当需要通知 adapter 数据源发生改变时，有很多方法可供调用：

|方法|描述|
|:--|:--|
|nitifyItemChanged(int pos)|Notify that item at position has changed.|
|notifyItemInserted(int pos)|Notify that item reflected at position has been newly inserted.|
|notifyItemRemoved(int pos)|Notify that items previously located at position has been removed from the data set.|
|notifyDataSetChanged()|Notify that the dataset has changed. Use only as last resort.|

我们可以在 activity 或者 fragment 中使用这些方法：

```java
// Add a new contact
contacts.add(0, new Contact("Barney", true));
// Notify the adapter that an item was inserted at position 0
adapter.notifyItemInserted(0);
```

每次我们想要 RecyclerView 添加/删除 视图元素，我们需要显式通知 adatper 这些事件的发生。和 ListView 的 adapter 不一样，RecyclerView adapter 不应该依赖 notifyDataSetChanged()，要调用颗粒更小的方法，[了解更多][10]。

如果你准备更新已存在的列表，在做任何改变之前，确保要获得当前列表的个数。比如，我们调用 getItemCount() 的返回值，作为列表变化的一个索引值：

```java
// record this value before making any changes to the existing list
int curSize = adapter.getItemCount();

// replace this line with wherever you get new records
ArrayList<Contact> newItems = Contact.createContactsList(20); 

// update the existing list
contacts.addAll(newItems);
// curSize should represent the first element that got added
// newItems.size() represents the itemCount
adapter.notifyItemRangeInserted(curSize, newItems.size());
```

#### Diffing Larger Changes

很多时候，列表发生的改变往往会很复杂（比如，对列表进行排序），这种情况下，并不能很容易的判断出每一行是否发生改变。面对如此情形，不得不调用 notifyDataSetChanged() 来更新整个列表，这个方法会清除展示数据变化的动画序列。

在 v24.2.0 的支持库中，增加了 DiffUtil 类来帮助计算新旧列表间的差异。该类使用的算法与计算源代码差异算法一致(the [diff utility][11] program)，所以它运行起来相当的快。如果你的列表很庞大，那么推荐你用后台线程来进行计算。

为了使用 DiffUtil 类，你需要创建一个类，并且继承 DiffUtil.Callback 接口，该接口接收新旧列表。

```java
public class ContactDiffCallback extends DiffUtil.Callback {

    private List<Contact> mOldList;
    private List<Contact> mNewList;

    public ContactDiffCallback(List<Contact> oldList, List<Contact> newList) {
        this.mOldList = oldList;
        this.mNewList = newList;
    }
    @Override
    public int getOldListSize() {
        return mOldList.size();
    }

    @Override
    public int getNewListSize() {
        return mNewList.size();
    }

    @Override
    public boolean areItemsTheSame(int oldItemPosition, int newItemPosition) {
        // add a unique ID property on Contact and expose a getId() method
        return mOldList.get(oldItemPosition).getId() == mNewList.get(newItemPosition).getId();
    }

    @Override
    public boolean areContentsTheSame(int oldItemPosition, int newItemPosition) {
        Contact oldContact = mOldList.get(oldItemPosition);
        Contact newContact = mNewList.get(newItemPosition);

        if (oldContact.getName() == newContact.getName() && oldContact.isOnline() == newContact.isOnline()) {
            return true;
        }
        return false;
    }
}
```

接下来，你需要在 adapter 中实现 swapItems() 方法，进行差异计算，并调用 dispatchUpdates() 来通知 adapter 该元素执行的操作是 插入/移除/移动/修改。

```java
public class ContactsAdapter extends
        RecyclerView.Adapter<ContactsAdapter.ViewHolder> {

  public void swapItems(List<Contact> contacts) {
        // compute diffs
        final ContactDiffCallback diffCallback = new ContactDiffCallback(this.mContacts, contacts);
        final DiffUtil.DiffResult diffResult = DiffUtil.calculateDiff(diffCallback);

        // clear contacts and add 
        this.mContacts.clear();
        this.mContacts.addAll(contacts);

        diffResult.dispatchUpdatesTo(this); // calls adapter's notify methods after diff is computed
    }
}
```

[这里][12]有示例代码。

#### Scrolling to New Items

如果我们在列表的表头插入数据，且希望 RecyclerView 保持显示第一个元素，我们可以设置滚动定位为第一个元素：

```java
adapter.notifyItemInserted(0);  
rvContacts.scrollToPosition(0);   // index 0 position
```

如果我们在列表的末尾插入数据，且希望视图滚动到我们插入数据的底部，我们可以通知 adapter 有新的数据插入，并在 RecyclerView 上调用 scrollToPosition()：

```java
adapter.notifyItemInserted(contacts.size() - 1);  // contacts.size() - 1 is the last element position
rvContacts.scrollToPosition(mAdapter.getItemCount() - 1); // update based on adapter 
```

#### Implementing Endless Scrolling

如果要实现这种效果，当用户滑动到视图的底部，就获取并展示更多的数据。那么要给 RecyclerView 添加 addOnScrollListener() 监听器，并增加 onLoadMore 方法，[详细EndlessScrollViewScrollListener][13]。

### Configuring the RecyclerView

RecyclerView 相当的灵活和高度定制化。下面是一些相关的选项。

#### Performance

如果视图元素是静态的且当剧烈滑动的时候也不会变化，那么我们可以开启优化选项：

```java
recyclerView.setHasFixedSize(true);
```

#### Layouts

视图元素的布局由 layout manager 决定。默认可以选择 LinearLayoutManager、GridLayoutManager、StaggeredGridLayoutManager 中的一个。Linear 以横向或者垂直对视图元素进行布局：

```java
// Setup layout manager for items with orientation
// Also supports `LinearLayoutManager.HORIZONTAL`
LinearLayoutManager layoutManager = new LinearLayoutManager(this, LinearLayoutManager.VERTICAL, false);
// Optionally customize the position you want to default scroll to
layoutManager.scrollToPosition(0);
// Attach layout manager to the RecyclerView
recyclerView.setLayoutManager(layoutManager);
```

以 grid 和 staggered 对元素进行布局的写法是类似的：

```java
// First param is number of columns and second param is orientation i.e Vertical or Horizontal
StaggeredGridLayoutManager gridLayoutManager = 
    new StaggeredGridLayoutManager(2, StaggeredGridLayoutManager.VERTICAL);
// Attach the layout manager to the recycler view
recyclerView.setLayoutManager(gridLayoutManager);
```

比如，staggered grid 的布局如下图所示：

![ | center][14]

我们可以定义[自己的布局][15]。

#### Decorations

我们可以使用不同的装饰器对视图元素进行装饰，比如说 [DividerItemDecoration][16]：

```java
RecyclerView.ItemDecoration itemDecoration = new 
    DividerItemDecoration(this, DividerItemDecoration.VERTICAL_LIST);
recyclerView.addItemDecoration(itemDecoration);
```

该装饰器展示视图元素之间的分割线：

![ | center][17]

#### Grid Spacing Decorations

装饰器也能被用作网格布局或者瀑布流布局中视图元素的间距。拷贝 [SpacesItemDecoration.java decorator][18] 到你的项目中，并且在 RecyclerView 中使用 addIemDecoration 方法。点击 [this staggered grid tutorial][19] 查看更多。

#### Animators

RecyclerView 通过使用 ItemAnimator 来支持视图元素的 添加/移动/删除 自定义动画。默认的动画效果通过 [DefaultItemAnimator][20] 实现，复杂的实现展示了逻辑的必要性来确保特定动画序列的效果。

目前，实现动画最快的方式就是使用第三库。[third-party recyclerview-animators library][21] 包含了丰富的动画效果，不需要你自己来构建他们。只要在你的 app/build.gradle 编辑如下：

```java
repositories {
    jcenter()
}

//If you are using a RecyclerView 23.1.0 or higher.
dependencies {
    compile 'jp.wasabeef:recyclerview-animators:2.2.3'
}

//If you are using a RecyclerView 23.0.1 or below.
dependencies {
    compile 'jp.wasabeef:recyclerview-animators:1.3.0'
}
```

接下来，我们可以定义任何的动画来改变 RecyclerView 的行为：

```java
recyclerView.setItemAnimator(new SlideInUpAnimator());
```

例如，下图是自定义动画的效果：

![ | center][22]

#### New ItemAnimator interface

从 [support v23.1.0][23] 开始，为 [ItemAnimator][24] 提供了一个新的接口。旧的接口因为 SimpleItemAnimator 已经弃用了。新的接口增加了 [ItemHolderInfo][25] 类，它与 DefaultItemAnimator 中的 [MoveInfo][26] 类似，但是它能在动画的过渡状态传递更多的状态信息。就好像是 DefaultItemAnimator 的下一个版本有了更好用的类，和修订了一些接口。

#### Heterogeneous Views

如果你想在一个 RecyclerView 中使用多种视图布局那么请看[这里][27]：

![ | center][28]

这时很有用的，如果一个列表内包含了不同类型的视图元素。

#### Handling Touch Events

RecyclerView 通过如下的方式来处理触摸事件：

```java
recyclerView.addOnItemTouchListener(new RecyclerView.OnItemTouchListener() {

    @Override
    public void onTouchEvent(RecyclerView recycler, MotionEvent event) {
        // Handle on touch events here
    }

    @Override
    public boolean onInterceptTouchEvent(RecyclerView recycler, MotionEvent event) {
        return false;
    }
    
});
```

#### Snap to Center Effect

在某些情况下，我们可能有一个水平的 RecyclerView，用户在上面滑动视图元素。当用户滑动的时候，我们想要视图元素始终显示在屏幕的中心，如下图所示：

![ | center][29]

#### LinearSnapHelper

为了在用户滑动的时候，能够让视图元素中心对齐，从 support library 24.2.0 开始，我们可以使用内置的 [LinearSnapHelper][30]：

```java
SnapHelper snapHelper = new LinearSnapHelper();
snapHelper.attachToRecyclerView(recyclerView);
```

了解更多相关信息， [read more about customizing these helpers][31] 和 reviewreview related sample code here][24]。

#### SnappyRecyclerView

如果需要自定义一些方法，我们可以创建一个继承自 RecyclerView 名叫 SnappyRecyclerView 的类，该类能够使视图元素在用户滑动d的时候居中对齐：

1. 拷贝 [SnappyRecyclerView.java][32] 到你的项目中。
2. 使用一个水平的 LinearLayoutManager 来配置你的 SnappyRecyclerView。

```java
LinearLayoutManager layoutManager = new LinearLayoutManager(this, LinearLayoutManager.HORIZONTAL, false);
snappyRecyclerView.setLayoutManager(layoutManager);
```

1. 在你的 RecyclerView 中添加 adapter 来填充水平列表的数据。
2. 你可以通过 snappyRecyclerView.getFirstVisibleItemPosition() 获取当前 "snapped" 元素的位置。

综上，你应该水平列表设置中心对齐！

#### Attaching Click Handlers to Items
##### Attaching Click Listeners with Decorators

RecyclerView 处理点击事件最简单的方法是使用一个装饰类。有了 [this clever ItemClickSupport decorator][33]，通过下面的代码来添加点击处理：

```java
ItemClickSupport.addTo(mRecyclerView).setOnItemClickListener(
  new ItemClickSupport.OnItemClickListener() {
      @Override
      public void onItemClicked(RecyclerView recyclerView, int position, View v) {
          // do it
      }
  }
);
```

Under the covers, this is wrapping the interface pattern described in detail below. If you apply this code above, you don't need to any of the manual item click handling below. This technique was originally [outlined in this article][34].（这段话不会翻译 :(）

##### Simple Click Handler within ViewHolder

RecyclerView 不像 ListView 提供了 setOnItemClickListener。为了获得类似的效果，我们可以在 ViewHolder 内部增加点击事件：

```java
public class ContactsAdapter extends RecyclerView.Adapter<ContactsAdapter.ViewHolder> {
    // ...

    // Used to cache the views within the item layout for fast access
    public class ViewHolder extends RecyclerView.ViewHolder implements View.OnClickListener {
        public TextView tvName;
        public TextView tvHometown;
        private Context context;

        public ViewHolder(Context context, View itemView) {
            super(itemView);
            this.tvName = (TextView) itemView.findViewById(R.id.tvName);
            this.tvHometown = (TextView) itemView.findViewById(R.id.tvHometown);
            // Store the context
            this.context = context;
            // Attach a click listener to the entire row view
            itemView.setOnClickListener(this);
        }

        // Handles the row being being clicked
        @Override
        public void onClick(View view) {
            int position = getAdapterPosition(); // gets item position
            if (position != RecyclerView.NO_POSITION) { // Check if an item was deleted, but the user clicked it before the UI removed it
                User user = users.get(position);
                // We can access the data within the views
                Toast.makeText(context, tvName.getText(), Toast.LENGTH_SHORT).show();
            }
        }
    }
    
    // ...
}
```

如果我们需要视图按下的时候有一种被选择的效果，我们可以为视图元素的根布局 android:background 设置为 ?android:attr/selectableItemBackground：

```java
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="horizontal" android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="?android:attr/selectableItemBackground"> 
  <!-- ... -->
</LinearLayout>
```

这将产生下面的效果：

![ | center][35]

##### Attaching Click Handlers using Listeners

在某些情况下，你需要在 RecyclerView 的内部设置点击处理，但是具体的逻辑实现在 Activity 或者 Fragment 中。为了做到这点，在 adapter 中创建一个[自定义监听器][36]，然后在 Activity 或者 Fragment 中实现这个接口：

```java
public class ContactsAdapter extends RecyclerView.Adapter<ContactsAdapter.ViewHolder> {
    // ...

    /***** Creating OnItemClickListener *****/
    
    // Define listener member variable
    private OnItemClickListener listener;
    // Define the listener interface
    public interface OnItemClickListener {
        void onItemClick(View itemView, int position);
    }
    // Define the method that allows the parent activity or fragment to define the listener
    public void setOnItemClickListener(OnItemClickListener listener) {
        this.listener = listener;
    }

    public static class ViewHolder extends RecyclerView.ViewHolder {
        public TextView tvName;
        public TextView tvHometown;

        public ViewHolder(final View itemView) {
            super(itemView);
            this.tvName = (TextView) itemView.findViewById(R.id.tvName);
            this.tvHometown = (TextView) itemView.findViewById(R.id.tvHometown);
            // Setup the click listener
            itemView.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    // Triggers click upwards to the adapter on click
                    if (listener != null) {
                        int position = getAdapterPosition();
                        if (position != RecyclerView.NO_POSITION) {
                            listener.onItemClick(itemView, position);
                        }
                    }
                }
            });
        }
    }
    
    // ...
}
```

接下来，我们可以为 adapter 增加点击事件了：

```java
// In the activity or fragment
ContactsAdapter adapter = ...;
adapter.setOnItemClickListener(new ContactsAdapter.OnItemClickListener() {
    @Override
    public void onItemClick(View view, int position) {
        String name = users.get(position).name;
        Toast.makeText(UserListActivity.this, name + " was clicked!", Toast.LENGTH_SHORT).show();
    }
});
```

点击 [this detailed stackoverflow post][37] 查看如何设置 item-level click handlers。

#### Implementing Pull to Refresh

RecyclerView 使用 SwipeRefreshLayout 通过 垂直滑动手势来对内容进行刷新。[这里][38]有相关的教程。

#### Swipe Detection

RecyclerView 中的 onFlingListener 方法被用来自定义滑动手势。下载 [RecyclerViewSwipeListener][38] 文件加入到你的项目中：

```java
RecyclerView rvMyList;

rvMyList.setOnFlingListener(new RecyclerViewSwipeListener(true) {
   @Override
   public void onSwipeDown() {
      Toast.makeText(MainActivity.this, "swipe down", Toast.LENGTH_SHORT).show();
   }

   @Override
   public void onSwipeUp() {
      Toast.makeText(MainActivity.this, "swipe up", Toast.LENGTH_SHORT).show();
   }
});
```



  [1]: https://developer.android.com/reference/android/support/v7/app/package-summary.html
  [2]: http://www.grokkingandroid.com/first-glance-androids-recyclerview/
  [3]: https://developer.android.com/reference/android/support/v7/widget/RecyclerView.LayoutManager.html
  [4]: https://www.youtube.com/watch?v=gs_C1E8HwvE&index=22&list=WL
  [5]: http://guides.codepath.com/android/Using-the-RecyclerView#animators
  [6]: http://odc50o546.bkt.clouddn.com/Using-the-RecyclerView-1.png
  [7]: http://odc50o546.bkt.clouddn.com/Using-the-RecyclerView-2.png
  [8]: http://android-developers.blogspot.tw/2016/02/android-support-library-232.html
  [9]: http://odc50o546.bkt.clouddn.com/Using-the-RecyclerView-3.gif
  [10]: https://developer.android.com/reference/android/support/v7/widget/RecyclerView.Adapter.html
  [11]: https://en.wikipedia.org/wiki/Diff_utility
  [12]: https://github.com/mrmike/DiffUtil-sample
  [13]: http://guides.codepath.com/android/Endless-Scrolling-with-AdapterViews-and-RecyclerView#implementing-with-recyclerview
  [14]: http://odc50o546.bkt.clouddn.com/Using-the-RecyclerView-4.png
  [15]: http://wiresareobsolete.com/2014/09/building-a-recyclerview-layoutmanager-part-1/
  [16]: https://gist.githubusercontent.com/alexfu/0f464fc3742f134ccd1e/raw/abe729359e5b3691f2fe56445644baf0e40b35ba/DividerItemDecoration.java
  [17]: http://odc50o546.bkt.clouddn.com/Using-the-RecyclerView-5.png
  [18]: https://gist.github.com/nesquena/db922669798eba3e3661
  [19]: http://blog.grafixartist.com/pinterest-masonry-layout-staggered-grid/
  [20]: https://developer.android.com/intl/ko/reference/android/support/v7/widget/DefaultItemAnimator.html
  [21]: https://github.com/wasabeef/recyclerview-animators
  [22]: http://odc50o546.bkt.clouddn.com/Using-the-RecyclerView-6.gif
  [23]: https://developer.android.com/intl/ko/tools/support-library/index.html#revisions
  [24]: https://developer.android.com/intl/ko/reference/android/support/v7/widget/RecyclerView.ItemAnimator.html#pubmethods
  [25]: https://developer.android.com/intl/ko/reference/android/support/v7/widget/RecyclerView.ItemAnimator.ItemHolderInfo.html
  [26]: https://github.com/android/platform_frameworks_support/blob/master/v7/recyclerview/src/android/support/v7/widget/DefaultItemAnimator.java#L53-L63
  [27]: http://guides.codepath.com/android/Heterogenous-Layouts-inside-RecyclerView
  [28]: http://odc50o546.bkt.clouddn.com/Using-the-RecyclerView-7.png
  [29]: http://odc50o546.bkt.clouddn.com/Using-the-RecyclerView-8.gif
  [30]: https://developer.android.com/reference/android/support/v7/widget/LinearSnapHelper.html
  [31]: https://rubensousa.github.io/2016/08/recyclerviewsnap
  [32]: https://gist.github.com/nesquena/b26f9a253bbbb6a2e4890891e8a57eb9
  [33]: https://gist.github.com/nesquena/231e356f372f214c4fe6
  [34]: http://www.littlerobots.nl/blog/Handle-Android-RecyclerView-Clicks/
  [35]: http://odc50o546.bkt.clouddn.com/Using-the-RecyclerView-9.gif
  [36]: http://guides.codepath.com/android/Creating-Custom-Listeners
  [37]: http://stackoverflow.com/a/24933117/313399
  [38]: https://gist.github.com/rogerhu/9e769149f9550c0a6ddb4987b94caee8