# NotePad







## 简述
    这个应用主要可以用于记录，类似于备忘录，可以对此标题下的内容进行修改等等，可以切换不同的颜色背景，默认是根据修改笔记的时间来进行排序，当然排序方式可以选择。


首先会有一个菜单，可以选择背景颜色或者排序方式等（我自己的手机上是长按菜单键）

![menu](menu.jpg)




## 应用功能————基础功能

### 添加时间戳
找到NoteList的ItemStyle -->lauout/notelist_item.xml，这个xml是定义ListView每个Item的样式，修改它就行了。

<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:padding="5dp">

    <TextView xmlns:android="http://schemas.android.com/apk/res/android"
        android:id="@android:id/text1"
        android:layout_width="match_parent"
        android:textColor="@android:color/black"
        android:layout_height="?android:attr/listPreferredItemHeight"
        android:textAppearance="?android:attr/textAppearanceLarge"
        android:gravity="center_vertical"
        android:paddingLeft="5dip"
        android:singleLine="true"
    />
    <TextView
        android:layout_width="match_parent"
        android:layout_height="?android:attr/listPreferredItemHeight"
        android:id="@+id/cdate_text"
        android:textColor="@android:color/black"
        android:text="2017-04-26 20:20:20"
        android:layout_alignRight="@android:id/text1"
        android:paddingLeft="200dip"
        android:gravity="bottom"/>
</RelativeLayout>

进入notelist.java，点击应用之后它就加载数据
final Cursor cursor = managedQuery(
            getIntent().getData(),            // Use the default content URI for the provider.
            PROJECTION,                       // Return the note ID and title for each note.
            null,                             // No where clause, return all records.
            null,                             // No where clause, therefore no where column values.
          //  NotePad.Notes.DEFAULT_SORT_ORDER  // Use the default sort order.
           null
        );

下面这个适配器就是用来显示数据的
private SimpleCursorAdapter adapter;
...

// SimpleCursorAdapter adapter
adapter
    = new SimpleCursorAdapter(
                      this,                             // The Context for the ListView
                      R.layout.noteslist_item,          // Points to the XML for a list item
                      cursor,                           // The cursor to get items from
                      dataColumns,
                      viewIDs
    );

NodePadProvider类
public void onCreate(SQLiteDatabase db) {
           db.execSQL("CREATE TABLE " + NotePad.Notes.TABLE_NAME + " ("
                   + NotePad.Notes._ID + " INTEGER PRIMARY KEY,"
                   + NotePad.Notes.COLUMN_NAME_TITLE + " TEXT,"
                   + NotePad.Notes.COLUMN_NAME_NOTE + " TEXT,"
                   + NotePad.Notes.COLUMN_NAME_CREATE_DATE + " TEXT,"
                   //+ NotePad.Notes.COLUMN_NAME_CREATE_DATE + " INTEGER,"
                  // + NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE + " INTEGER"
                   + NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE + " TEXT"
                   + ");");
       }


![start](start.png)


### 模糊查询
借鉴别人的方法，来到NoteList，创建一个SearchView成员变量：//成员变量：SearchView组件
    private SearchView searchView;
看他博客是在onCreateOptionsMenu 里面初始化样式的。
@Override
    public boolean onCreateOptionsMenu(Menu menu) {
        // Inflate menu from XML resource
        MenuInflater inflater = getMenuInflater();
        inflater.inflate(R.menu.list_options_menu, menu);
        setSearchView(menu);//添加搜索框
        ...
 /**
     * 设置查询菜单项
     * @param menu
     */
    private void setSearchView(Menu menu) {
        //1.找到menuItem并动态设置SearchView
        MenuItem item = menu.getItem(0);
        searchView = new SearchView(this);
        item.setActionView(searchView);

        //2.设置搜索的背景为白色
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
            item.collapseActionView();
        }
        searchView.setQuery("", false);
        //searchView.setBackgroundResource(R.drawable.ic_menu_edit);
        //3.设置为默认展开状态，图标在外面
        searchView.setIconifiedByDefault(true);
        searchView.setQueryHint("Search");
        

//还原显示所有的数据
        case R.id.menu_all:
          Cursor cursor = getContentResolver().query
                  (getIntent().getData(),PROJECTION,null,null,null);
          adapter.swapCursor(null);
          adapter.swapCursor(cursor);
            return true;
这里面whereClause和Args填空值就没有where了,就显示全部信息了。


![search](search.png)

## 应用功能————附加功能

### 修改背景颜色
这一块自己不是很会，借鉴了别人的

因为是自定义的颜色按钮（textView点击事件），用户偏好设置要自己保存到SharedPreferences，新建一个类，获取和保存SharedPreferences的键值对：
public class PreferencesService {
    private Context context;

    public PreferencesService(Context context) {
        this.context = context;
    }

    /**
     * 保存参数
     * @param color 颜色ID
     * @param sortType 排序类型
     */
    public void save(int color,int sortType) {
        //获得SharedPreferences对象
        SharedPreferences preferences = context.getSharedPreferences("pdata", Context.MODE_PRIVATE);
        SharedPreferences.Editor editor = preferences.edit();
        editor.putInt("color", color);
        editor.putInt("sort", sortType);
        editor.commit();
    }

    /**
     * 获取各项参数
     * @return
     */
    public Map<String, String> getPerferences() {
        Map<String, String> params = new HashMap<String, String>();
        SharedPreferences preferences = context.getSharedPreferences("pdata", Context.MODE_PRIVATE);
        params.put("color", String.valueOf(preferences.getInt("color", 0)));
        params.put("sort", String.valueOf(preferences.getInt("sort", 0)));
        return params;
    }

}

list_options_menu添加颜色菜单，其实就是在右上角列表里的一个按钮:
<!-- 背景颜色 -->
    <item android:id="@+id/menu_color"
        android:title="@string/menu_color"
        android:alphabeticShortcut='c' />NoteList里添加一个操作配置文件和一个属性保存背景颜色值的属性：
        //偏好设置
private PreferencesService service;
//背景颜色
private int color;
NoteList-->OnCreate加载时需要加载配置文件，并获取配置文件中的颜色值
//设置背景色
        service = new PreferencesService(this);
        //获取配置文件的键值对
        Map<String, String> params = service.getPerferences();
        //获取颜色值
        String value = params.get("color");
        if(!"0".equals(value))
            color= Integer.parseInt(value);
        else
            color = R.drawable.bkwhite;
        //修改背景颜色
        Resources res = getResources();
        Drawable bdrawable = res.getDrawable(color);
        this.getListView().setBackgroundDrawable(bdrawable);
择框
            case R.id.menu_color:
                //设置自定义布局
                LinearLayout form=(LinearLayout)getLayoutInflater()
                        .inflate(R.layout.color_style,null);
                final AlertDialog dialog=new AlertDialog.Builder(this)
                        .setView(form)//加载布局
                        .setTitle(R.string.title_color_list)//设置标题
                        .setPositiveButton("Exit", new DialogInterface.OnClickListener() {//退出按钮
                    @Override
                    public void onClick(DialogInterface dialog, int which) {
                    }

                })
                .create();
                dialog.show();
                //更新背景颜色
                setTextViewOnclick(dialog);
                return true;


//设置颜色点击事件监听
        color_a.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                setColor(R.drawable.bkshibanhui);
            }
        });
        color_b.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                setColor(R.drawable.bkwhite);
            }
        });

        color_c.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                setColor(R.drawable.bkdougello);
            }
        });
        color_d.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                setColor(R.drawable.bkbanana);
            }
        });
        color_e.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                setColor(R.drawable.bkfenhong);
            }
        });
        color_f.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                setColor(R.drawable.bkmeiguihong);
            }
        });
        color_g.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                setColor(R.drawable.bkqianhuilan);
            }
        });
        color_h.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                setColor(R.drawable.bkbohe);
            }
        });

    }


![background](background.png)



### 排序

前面那个SharedPreferences个里已经定义了sort类型了，就可以直接set进去。
!-- 排序菜单列表-->
    <item android:title="@string/menu_sort">
        <menu>
            <!-- 默认按修改时间降序，见NotePad契约类-->
            <item android:title="@string/menu_sort_default"
                android:id="@+id/sort_default" />
            <!-- 标题升序-->
            <item android:title="@string/menu_sort_title_asc"
                android:id="@+id/sort_title_asc"/>
            <!-- 标题降序-->
            <item android:title="@string/menu_sort_title_desc"
                android:id="@+id/sort_title_desc" />
            <!-- 创建时间升序-->
            <item android:title="@string/menu_sort_date_asc"
                android:id="@+id/sort_date_asc" />
            <!-- 创建时间降序-->
            <item android:id="@+id/sort_date_desc"
                android:title="@string/menu_sort_date_desc" />
        </menu>
    </item>添加属性并在OnCreate加载配置：//排序类型
    private int sortType;
    ...
    
    //获取排序值
    value = params.get("sort");
    sortType = Integer.parseInt(value);
排序类型先在NotePad中写个契约类吧：public static final class SortType implements BaseColumns{
        /**
         * 排序类型契约：
         * SORT_DEFALUT 默认排序，即修改时间降序
         * SORT_TITLE_ASC 标题升序
         * SORT_TITLE_DESC 标题降序
         * SORT_CREATEDATE_ASC 创建时间升序
         * SORT_CREATEDATE_DESC 创建时间降序
         */
        public static final  int  SORT_DEFALUT=0;
        public static final  int  SORT_TITLE_ASC=1;
        public static final  int  SORT_TITLE_DESC=2;
        public static final  int  SORT_CREATEDATE_ASC=3;
        public static final  int  SORT_CREATEDATE_DESC=4;
    }
写个函数返回排序字符串：/**
     * 获取排序类型
     * @param type 排序类型
     * @return 返回数据库排序字符串
     */
    private String getSortType(int type){
        if(type == NotePad.SortType.SORT_TITLE_ASC)//标题升序
            return NotePad.Notes.COLUMN_NAME_TITLE + " ASC";
        if(type == NotePad.SortType.SORT_TITLE_DESC)//标题降序
            return NotePad.Notes.COLUMN_NAME_TITLE + " DESC";
        if(type == NotePad.SortType.SORT_CREATEDATE_ASC)//创建时间升序
            return NotePad.Notes.COLUMN_NAME_CREATE_DATE + " ASC";
        if(type == NotePad.SortType.SORT_CREATEDATE_DESC)//创建时间降序
            return NotePad.Notes.COLUMN_NAME_CREATE_DATE + " DESC";
        else//默认
            return NotePad.Notes.DEFAULT_SORT_ORDER;
    }

![paixu](paixu.png)
