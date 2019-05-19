1、NoteList中显示条目增加时间戳显示

  创建的数据库
  
        public void onCreate(SQLiteDatabase db) {
           db.execSQL("CREATE TABLE " + NotePad.Notes.TABLE_NAME + " ("
                   + NotePad.Notes._ID + " INTEGER PRIMARY KEY,"
                   + NotePad.Notes.COLUMN_NAME_TITLE + " TEXT,"
                   + NotePad.Notes.COLUMN_NAME_NOTE + " TEXT,"
                   + NotePad.Notes.COLUMN_NAME_CREATE_DATE + " INTEGER,"
                   + NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE + " INTEGER"
                   + NotePad.Notes.COLUMN_NAME_TEXT_COLOR + "INTEGER"
                   + NotePad.Notes.COLUMN_NAME_TEXT_SIZE + "INTEGER" 
                   + ");");
      }
      
  创建时显示的时间是在NotePadProvider的insert函数实现的
      
      public Uri insert(Uri uri, ContentValues initialValues) {
        if (sUriMatcher.match(uri) != NOTES) {
            throw new IllegalArgumentException("Unknown URI " + uri);
        }
        ContentValues values;
        if (initialValues != null) {
            values = new ContentValues(initialValues);
        } else {
            values = new ContentValues();
        }
        Long now = Long.valueOf(System.currentTimeMillis());
        Date date = new Date(now);
        SimpleDateFormat format = new SimpleDateFormat("yyyy.MM.dd HH:mm:ss");
        format.setTimeZone(TimeZone.getTimeZone("GMT+8:10"));
        String dateTime = format.format(date);
        if (values.containsKey(NotePad.Notes.COLUMN_NAME_CREATE_DATE) == false) {
            values.put(NotePad.Notes.COLUMN_NAME_CREATE_DATE, dateTime);
        }
        if (values.containsKey(NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE) == false) {
            values.put(NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE, dateTime);
        }
        if (values.containsKey(NotePad.Notes.COLUMN_NAME_TITLE) == false) {
            Resources r = Resources.getSystem();
            values.put(NotePad.Notes.COLUMN_NAME_TITLE, r.getString(android.R.string.untitled));
        }

        if (values.containsKey(NotePad.Notes.COLUMN_NAME_NOTE) == false) {
            values.put(NotePad.Notes.COLUMN_NAME_NOTE, "");
        }
        SQLiteDatabase db = mOpenHelper.getWritableDatabase();
        long rowId = db.insert(
            NotePad.Notes.TABLE_NAME,        
            NotePad.Notes.COLUMN_NAME_NOTE,                                  
            values                                                           
        );
        if (rowId > 0) {
            Uri noteUri = ContentUris.withAppendedId(NotePad.Notes.CONTENT_ID_URI_BASE, rowId);
            getContext().getContentResolver().notifyChange(noteUri, null);
            return noteUri;
        }
        throw new SQLException("Failed to insert row into " + uri);
      }
    
  实现时间的显示，先在noteslist_item.xml中增加一个用来显示时间的TextView
  
      <TextView
      android:id="@+id/text2"
      android:layout_width="match_parent"
      android:layout_height="wrap_content"
      android:textAppearance="?android:attr/textAppearanceLarge"
      android:textColor="@color/yellow"
      android:textSize="20sp"
      />
    
  然后在NotesList的数据定义中增加修改时间
  
      private static final String[] PROJECTION = new String[] {
        NotePad.Notes._ID, // 0
        NotePad.Notes.COLUMN_NAME_TITLE, // 1
        NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE,//在这里加入了修改时间的显示
      };
      
  然后通过SimpleCursorAdapter来进行装配
  
      SimpleCursorAdapter adapter
        = new SimpleCursorAdapter(
              this,                             
              R.layout.noteslist_item,         
              cursor,                           
              dataColumns,
              viewIDs
        );
        
  结果截图：![](https://github.com/linyu0119/NotePad/blob/master/image/1.jpg)
  
2.添加笔记查询功能
  
  在list_options_menu.xml中新建一个查询的按钮
  
    <item
    android:id="@+id/menu_search"
    android:icon="@drawable/search"
    android:title="@string/menu_search"
    android:showAsAction="always"
    />
  
  onOptionsItemSelected的switch (item.getItemId())中添加对应menu_search的case：
    
    case R.id.menu_search:
    startActivity(new Intent(Intent.ACTION_SEARCH,getIntent().getData()));
    return true;
    
  在AndroidManifest.xml中加入相应的声名
    
    <activity
    android:name=".NoteSearch"
    android:label="NoteSearch"
    >
    <intent-filter>
        <action android:name="android.intent.action.NoteSearch" />
        <action android:name="android.intent.action.SEARCH" />
        <action android:name="android.intent.action.SEARCH_LONG_PRESS" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="vnd.android.cursor.dir/vnd.google.note" />
        <!--1.vnd.android.cursor.dir代表返回结果为多列数据-->
        <!--2.vnd.android.cursor.item 代表返回结果为单列数据-->
    </intent-filter>
    </activity>
  
  搜索采用SearchView实现，并通过实现implements SearchView.OnQueryTextListener接口来完成查询NoteSearch.java
    
    public class NoteSearch extends ListActivity implements SearchView.OnQueryTextListener{
    private static final String[] PROJECTION = new String[]{
            NotePad.Notes._ID, 
            NotePad.Notes.COLUMN_NAME_TITLE, 
            NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE,
    };
    SearchView mSearchView;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.note_search);
        Intent intent = getIntent();
        if (intent.getData()==null) {
            intent.setData(NotePad.Notes.CONTENT_URI);
        }
        ListView list = (ListView)findViewById(android.R.id.list);
        list.setOnCreateContextMenuListener(this);
        mSearchView = (SearchView)findViewById(R.id.search_view);
        mSearchView.setIconifiedByDefault(false); 
        mSearchView.setSubmitButtonEnabled(true); 
        mSearchView.setOnQueryTextListener(this);
    }
    @Override
    public boolean onQueryTextSubmit(String s) {
        return false;
    }
    @Override
    public boolean onQueryTextChange(String s) { 
        String selection = NotePad.Notes.COLUMN_NAME_TITLE + " Like ? ";
        String[] selectionArgs = { "%"+s+"%" };
        Cursor cursor = managedQuery(
                getIntent().getData(),            
                PROJECTION,                      
                selection,                        
                selectionArgs,                    
                NotePad.Notes.DEFAULT_SORT_ORDER  
        );
        String[] dataColumns = { NotePad.Notes.COLUMN_NAME_TITLE ,  NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE };
        int[] viewIDs = { android.R.id.text1 , R.id.text2 };
        SimpleCursorAdapter adapter = new SimpleCursorAdapter(
                this,                   
                R.layout.noteslist_item,         
                cursor,                        
                dataColumns,                   
                viewIDs                         
        );
        setListAdapter(adapter);
        return true;
    }
    @Override
    protected void onListItemClick(ListView l, View v, int position, long id) {
        Uri uri = ContentUris.withAppendedId(getIntent().getData(), id);
        String action = getIntent().getAction();
        if (Intent.ACTION_PICK.equals(action) || Intent.ACTION_GET_CONTENT.equals(action)) {
            setResult(RESULT_OK, new Intent().setData(uri));
        } else {
            startActivity(new Intent(Intent.ACTION_EDIT, uri));
        }
    }
    }
    
  结果截图：![](https://github.com/linyu0119/NotePad/blob/master/image/2.jpg)

3.UI美化

  修改一下条目显示的颜色，以及鼠标点击之后的颜色list_selector.xml，以及更改编辑和搜索的图标
    
    <?xml version="1.0" encoding="utf-8"?>
    <selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:state_focused="true" android:drawable="@color/colorAccent"/>
    <item android:state_pressed="true" android:drawable="@color/colorPrimary" />
    <item android:drawable="@color/blue"/>
    </selector>
    
   结果截图：![](https://github.com/linyu0119/NotePad/blob/master/image/3.jpg)

4.在文本编辑界面中添加改变字体大小和字体颜色的功能
  
  editor_options_menu.xml
    
    <?xml version="1.0" encoding="utf-8"?>
    <menu xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:id="@+id/menu_save"
          android:icon="@drawable/ic_menu_save"
          android:alphabeticShortcut='s'
          android:title="@string/menu_save"
          android:showAsAction="ifRoom|withText" />
    <item android:id="@+id/menu_revert"
          android:icon="@drawable/ic_menu_revert"
          android:title="@string/menu_revert" />
    <item android:id="@+id/menu_delete"
          android:icon="@drawable/ic_menu_delete"
          android:title="@string/menu_delete"
          android:showAsAction="ifRoom|withText" />
    <item android:id="@+id/menu_output"
        android:title="@string/menu_output" />

    <item
        android:id="@+id/font_size"
        android:title="@string/font_size">
        <!--子菜单-->
        <menu>
            <!--定义一组单选菜单项-->
            <group>
                <!--定义多个菜单项-->
                <item
                    android:id="@+id/font_10"
                    android:title="@string/font10"
                    />

                <item
                    android:id="@+id/font_16"
                    android:title="@string/font16" />
                <item
                    android:id="@+id/font_20"
                    android:title="@string/font20" />
            </group>
        </menu>
    </item>
    <item
        android:title="@string/font_color"
        android:id="@+id/font_color"
        >
        <menu>
            <!--定义一组普通菜单项-->
            <group>
                <!--定义两个菜单项-->
                <item
                    android:id="@+id/red_font"
                    android:title="@string/red_title" />
                <item
                    android:title="@string/black_title"
                    android:id="@+id/black_font"/>
            </group>
        </menu>
    </item>
    </menu>
    
  在NoteEditor的onOptionsItemSelected(MenuItem item)中添加相应的case来相应事件
  
    case R.id.font_10:
    mText.setTextSize(20);
    Toast toast =Toast.makeText(NoteEditor.this,"修改成功",Toast.LENGTH_SHORT);
    toast.show();
    finish();
    break;
    case R.id.font_16:
    mText.setTextSize(32);
    Toast toast2 =Toast.makeText(NoteEditor.this,"修改成功",Toast.LENGTH_SHORT);
    toast2.show();
    finish();
    break;
    case R.id.font_20:
    mText.setTextSize(40);
    Toast toast3 =Toast.makeText(NoteEditor.this,"修改成功",Toast.LENGTH_SHORT);
    toast3.show();
    finish();
    break;
    case R.id.red_font:
    mText.setTextColor(Color.RED);
    Toast toast4 =Toast.makeText(NoteEditor.this,"修改成功",Toast.LENGTH_SHORT);
    toast4.show();
    finish();
    break;
    case R.id.black_font:
    mText.setTextColor(Color.BLACK);
    Toast toast5 =Toast.makeText(NoteEditor.this,"修改成功",Toast.LENGTH_SHORT);
    toast5.show();
    finish();
    break;
    
  结果截图：![](https://github.com/linyu0119/NotePad/blob/master/image/4.jpg)
  ![](https://github.com/linyu0119/NotePad/blob/master/image/5.jpg)
  ![](https://github.com/linyu0119/NotePad/blob/master/image/6.jpg)
  ![](https://github.com/linyu0119/NotePad/blob/master/image/7.jpg)
  

五：笔记导出
  
  在editor_options_menu.xml中添加一个item：
  
    <item android:id="@+id/menu_output"
    android:title="@string/menu_output" />
    
  在NoteEditor中onOptionsItemSelected()方法里的switch加一个case：
  
     case R.id.menu_output:
        outputNote();
        break;
  
  OutputText.java
  
    public class OutputText extends Activity {
    private static final String[] PROJECTION = new String[] {
            NotePad.Notes._ID, // 0
            NotePad.Notes.COLUMN_NAME_TITLE, // 1
            NotePad.Notes.COLUMN_NAME_NOTE, // 2
            NotePad.Notes.COLUMN_NAME_CREATE_DATE, // 3
            NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE, // 4
    };
    private String TITLE;
    private String NOTE;
    private String CREATE_DATE;
    private String MODIFICATION_DATE;
    private Cursor mCursor;
    private EditText mName;
    private Uri mUri;
    private boolean flag = false;
    private static final int COLUMN_INDEX_TITLE = 1;
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.output_text);
        mUri = getIntent().getData();
        mCursor = managedQuery(
                mUri,       
                PROJECTION, 
                null,        
                null,       
                null         
        );
        mName = (EditText) findViewById(R.id.output_name);
    }
    @Override
    protected void onResume(){
        super.onResume();
        if (mCursor != null) {
            // The Cursor was just retrieved, so its index is set to one record *before* the first
            // record retrieved. This moves it to the first record.
            mCursor.moveToFirst();
            mName.setText(mCursor.getString(COLUMN_INDEX_TITLE));
        }
    }
    @Override
    protected void onPause() {
        super.onPause();
        if (mCursor != null) {
            TITLE = mCursor.getString(mCursor.getColumnIndex(NotePad.Notes.COLUMN_NAME_TITLE));
            NOTE = mCursor.getString(mCursor.getColumnIndex(NotePad.Notes.COLUMN_NAME_NOTE));
            CREATE_DATE = mCursor.getString(mCursor.getColumnIndex(NotePad.Notes.COLUMN_NAME_CREATE_DATE));
            MODIFICATION_DATE = mCursor.getString(mCursor.getColumnIndex(NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE));
            if (flag == true) {
                write();
            }
            flag = false;
        }
    }
    public void OutputOk(View v){
        flag = true;
        finish();
    }
    private void write()
    {
        try
        {
            if (Environment.getExternalStorageState().equals(
                    Environment.MEDIA_MOUNTED)) {
                File sdCardDir = Environment.getExternalStorageDirectory();
                File targetFile = new File(sdCardDir.getCanonicalPath() + "/" + mName.getText() + ".txt");
                PrintWriter ps = new PrintWriter(new OutputStreamWriter(new FileOutputStream(targetFile), "UTF-8"));
                ps.println(TITLE);
                ps.println(NOTE);
                ps.println("创建时间：" + CREATE_DATE);
                ps.println("最后一次修改时间：" + MODIFICATION_DATE);
                ps.close();
                Toast.makeText(this, "保存成功,保存位置：" + sdCardDir.getCanonicalPath() + "/" + mName.getText() + ".txt", Toast.LENGTH_LONG).show();
            }
        }
        catch (Exception e)
        {
            e.printStackTrace();
        }
    }
    }
    
  output_text.xml
    
    <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    android:paddingLeft="6dip"
    android:paddingRight="6dip"
    android:paddingBottom="3dip">
    <EditText android:id="@+id/output_name"
        android:maxLines="1"
        android:layout_marginTop="2dp"
        android:layout_marginBottom="15dp"
        android:layout_width="wrap_content"
        android:ems="25"
        android:layout_height="wrap_content"
        android:autoText="true"
        android:capitalize="sentences"
        android:scrollHorizontally="true" />
    <Button android:id="@+id/output_ok"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="right"
        android:text="@string/output_ok"
        android:onClick="OutputOk" />
    </LinearLayout>
    
  结果截图：![](https://github.com/linyu0119/NotePad/blob/master/image/8.jpg)
  ![](https://github.com/linyu0119/NotePad/blob/master/image/9.jpg)
  ![](https://github.com/linyu0119/NotePad/blob/master/image/10.jpg)









    
   



    



