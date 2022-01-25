

# Recyclerview使用

对于其不仅可以实现和ListView同样的效果

本文主要是一个RecyclerView的开篇，主要针对RecyclerView空间的使用。至于RecyclerView的观察者模式和缓冲机制，后续会在源码分析持续更新。

# RecyclerView的使用说明

笔者对于RecyclerView的记忆主要是一个口诀，一类继承，四个方法。

对于口诀使用的类就是Adapter

## 1）一类继承

里面MyViewHolder继承 RecyclerView.ViewHolder，并实现RecyclerView.ViewHolder的构造，一般用于findviewbyid绑定控件

而MyAdapter继承RecyclerView.Adpter<MyAdapter.MyViewHolder>

## 2）四个方法

### 一个构造

**一般为有参构造，用于传递参数进来，初始化控件信息**

举例MyAdapter(List< SettingItem > list)

### 三个继承

1)**通过viewType来实现加载不同的布局**

public MyViewHolder onCreateViewHolder(ViewGroup parent, int viewType)

2)**将数据和控件绑定，*会在每个子项被滚动到屏幕内时执行*，position从0开始**

public void onBindViewHolder(ViewHolder holder, int position)

3)**多少个item数**

public int getItemCount()

## 3）外部使用

设置完成之后，在Activity的使用也很简单，如下所示外部设置

**1)RecyclerView获取他的对象**
RecyclerView recyclerView = (RecyclerView) findViewById(R.id.recycler_view);
**2)LayoutManager用于指定RecyclerView的布局方式，设置layoutManager 的展示方式设置为竖直方向**
LinearLayoutManager layoutManager = new LinearLayoutManager(this);
layoutManager .setOrientation(LinearLayoutManager.VERTICAL);
recyclerView.setLayoutManager(layoutManager);

**3)设置适配器**

MyAdapter adapter = new MyAdapter(fruitList);
recyclerView.setAdapter(adapter);

# RecyclerView的demo

## 1）settings

效果图如下



### 拆解demo

首先是Adapter

```kotlin
//构造内部类，并使用kotlin的构造函数，继承内部Holder类
private inner class SimpleAdapter constructor(list :List<SettingItem>) : RecyclerView.Adapter<SimpleAdapter.Holder>() {
        private var mSettingItem: List<SettingItem>? = null
		//*一个继承* Holder继承RecyclerView.ViewHolder，并且初始化构造
        inner class Holder(itemView: View) : RecyclerView.ViewHolder(itemView){
            var textView : TextView? = null
            var imageView : ImageView? = null

            //Hoder构造器初始化
            init {
                textView = itemView.findViewById(R.id.settingname)
                imageView = itemView.findViewById(R.id.settinginfo)
                imageView?.setOnClickListener(View.OnClickListener {
                    Log.d(TAG, "->onclicked imageview")
                    //可以直接通过getposition获取到postion的值
                    Toast.makeText(context!!, "onclicked imageview pos = $position", Toast.LENGTH_LONG).show()
                })
                itemView?.setOnClickListener(View.OnClickListener {
                    Log.d(TAG, "->onclicked item")
                    Toast.makeText(context!!, "onclicked item pos = $position", Toast.LENGTH_LONG).show()
                })
            }

        }
        //*一个构造*SimpleAdapter构造器初始化，将list传递进来
        init {
            mSettingItem = list
        }
		//*三个方法*加载唯一布局
        override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): Holder {
            var view: View = LayoutInflater.from(parent.context).inflate(R.layout.item_settings, parent, false)
            var viewHolder: Holder = Holder(view)
            return viewHolder
        }
		//*三个方法*将数据和控件绑定，数据来源于SimpleAdapter构造器
        override fun onBindViewHolder(holder: Holder, position: Int) {
            var settingItem = mSettingItem?.get(position)
            holder.textView?.text = settingItem?.getName()
            settingItem?.getInfo()?.let { holder.imageView?.setImageResource(it) }

        }
		//*三个方法*设置多少个item数为list长度
        override fun getItemCount(): Int {
            return mSettingItem?.let { it.size } ?: run { 0 }
        }
    }
```

接着拆解外部使用

```kotlin
//1)RecyclerView获取他的对象
recyclerView = findViewById(R.id.moresettings)
//2)LayoutManager用于指定RecyclerView的布局方式，展示方式设置为竖直方向
var ll: LinearLayoutManager = LinearLayoutManager(this)
ll.orientation = LinearLayoutManager.VERTICAL
recyclerView?.layoutManager = ll
//3)设置适配器
recyclerView?.adapter = SimpleAdapter(mList)
```

拆解之后就感觉使用会很简单

## 2）wifiSettings

效果图如下



### 拆解demo

首先是Adapter

```kotlin
//构造内部类，并使用kotlin的构造函数，具体的逻辑代码省略部分，因为此demo有多个viewholder，所以不继承使用父类即可
private inner class WifiAdapter internal constructor(context: Context) :
        RecyclerView.Adapter<RecyclerView.ViewHolder>() {
        private val inflater: LayoutInflater = LayoutInflater.from(context)
		//*三个方法*增加一个方法来判断viewholder类型，总共有itemCount个item
        //范围从0---itemCount-1，其中下标为0，表示为viewType为2
        //下标为1---itemCount-2表示viewType为1
        //下标为其他itemCount-1表示viewType为0
        override fun getItemViewType(position: Int): Int {
            return when (position) {
                0 -> 2
                itemCount - 1 -> 1
                else -> 0
            }
        }
		//*三个方法*加载布局,这里默认有三个布局，根据上面的getItemViewType的返回值viewtype来判断
        //viewType为0 ,布局就是数据布局，这里代表wifi
        //viewType为1 ,布局就是数据布局的下面布局，这里代表手动添加wifi
        //viewType为2 ,布局就是数据布局的上面布局，这里代表关闭/打开wifi
        override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
            return when (viewType) {
                0 ->
                    ItemViewHolder(inflater.inflate(R.layout.item_access_point_2, parent, false))
                1 ->
                    ManualViewHolder(
                        inflater.inflate(
                            R.layout.item_access_point_manual,
                            parent,
                            false
                        )
                    )
                2 -> {
                    val holder = WifiSwitchViewHolder(
                        inflater.inflate(
                            R.layout.item_wifi_switch,
                            parent,
                            false
                        )
                    )
                    ...
                }
                else ->
                    ItemViewHolder(inflater.inflate(R.layout.item_access_point_2, parent, false))
            }
        }
		//*三个方法*将数据和控件绑定，数据来源于scans
        override fun onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int) {
            if (holder is ItemViewHolder) {
                scans?.get(position - 1)?.let { scan ->
                    holder.bind(scan)
                }
            }
        }
		//*三个方法*设置多少个item数，wifi打开为scans.size + 2，wifi关闭为1
        override fun getItemCount(): Int {
            val isWifiEnabled = wifiSwitch?.isChecked == true
            return if (isWifiEnabled) (scans?.size ?: 0) + 2 else 1
        }
		//*一个继承* ManualViewHolder继承RecyclerView.ViewHolder,并且初始化构造
        internal inner class ManualViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
            init {
                itemView.setOnClickListener {
                    handleAddManual()
                }
            }
        }
		//*一个继承* WifiSwitchViewHolder继承RecyclerView.ViewHolder
        internal inner class WifiSwitchViewHolder(itemView: View) :
            RecyclerView.ViewHolder(itemView)
		//*一个继承* ItemViewHolder继承RecyclerView.ViewHolder,并且初始化构造
        internal inner class ItemViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
            private val icon: ImageView = itemView.findViewById(R.id.icon)
            private val ssid: TextView = itemView.findViewById(R.id.ssid)
            private val status: TextView = itemView.findViewById(R.id.status)
            private val info: ImageView = itemView.findViewById(R.id.info)

            init {
                itemView.setOnClickListener { handleOnItemClick(scans!![this@ItemViewHolder.layoutPosition - 1]) }
                info.setOnClickListener { handleOpenWifiInfo(scans!![this@ItemViewHolder.layoutPosition - 1]) }
            }
        }
    }
```

接着拆解外部使用

```kotlin
private var adapter: WifiAdapter? = nul 
adapter = context?.let { WifiAdapter(it) }
//1)RecyclerView获取他的对象
val list =findViewById<RecyclerView>(R.id.wifi_list)
//2)设置展示方式，这里将item之间添加横线，默认方向为竖向
list.addItemDecoration(
    DividerItemDecoration.Builder(list.context)
    .setPadding(resources.getDimensionPixelSize(R.dimen.dp_40))
    .setDividerColor(ContextCompat.getColor(list.context, R.color.dividerLight))
    .setDividerWidth(resources.getDimensionPixelSize(R.dimen.dp_1))
    .build()
)
//4)设置适配器
list.adapter = adapter
```



## 整理wifisettings.kt

主要流程如下

1.布局加载

2.数据加载

3.notify更新



# 总结

RecyclerView的使用已经完结，主要针对具体demo分析如何去快速的上手使用，由于RecyclerView的功能非常强大，其他功能各位宝宝可以自行探索。



# 猜你喜欢

recyclerview观察者模式

recyclerview缓存机制



本文所有实例代码下载

# 参考

