---
title: "Android Property详解"
date: 2022-08-21
thumbnailImagePosition: left
thumbnailImage: property/property_thumb.jpg
coverImage: property/property_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- Property
- 2022
- August 
tags:
- Android
- 源码
- SElinux
- Trie
- socket
- mmap
- 共享内存
- 匿名私有内存
- proto
- rc文件
- persist属性
showSocial: false
---

Android系统中的属性，我们可以通过get和set去获取，但是有时候App获取修改属性的操作被拒绝了，这个就需要深入了解属性系统。

<!--more-->
# Android Property详解

 # 0属性介绍

## 0.1简介

在 android 系统中，为同一管理系统的属性，设计了一个统一的属性系统。每个属性都有一个名字和值，他们都是字符串格式。属性被大量使用在 Android 系统中，用来记录系统设置或进程之间的信息交换。属性是在整个系统中全局可见的。每个进程可以 `get/set` 属性。在编译的过程中会将各种系统参数汇总到 `build.proc` 以及 `default.proc` 这两个文件中，主要属性集中在 `build.proc` 中。系统在开机后将读取配置信息并构建共享缓冲区，加快查询速度。本文以`Android11`来介绍属性系统。

{{< image classes="fancybox center fig-100" src="/property/property_2.png" thumbnail="/property/property_2.png" title="">}}

## 0.2Properties Type

​    系统属性根据不同的应用类型，分为不可变型，持久型，网络型，启动和停止服务等。

**特别属性：**

- 属性名称以 `ro.` 开头，那么这个属性被视为只读属性。一旦设置，属性值不能改变。
- 属性名称以`persist.` 开头，当设置这个属性时，其值也将写入 `/data/property`。
- 属性名称以 `net.` 开头，当设置这个属性时，`net.change` 属性将会自动设置，以加入到最后修改的属性名。（这是很巧妙的。`netresolve` 模块的使用这个属性来追踪在 `net.*` 属性上的任何变化。）
- 属性 `ctrl.start` 和 `ctrl.stop` 是用来启动和停止服务。每一项服务必须在 `/init.rc` 中定义。系统启动时，与 `init` 守护进程将解析 `init.rc` 和启动属性服务。一旦收到设置 `ctrl.start` 属性的请求，属性服务将使用该属性值作为服务名找到该服务，启动该服务。这项服务的启动结果将会放入 `init.svc.<服务名>` 属性中。客户端应用程序可以轮训那个属性值，以确定结果。



## 0.3Android toolbox

​    Android toolbox 程序提供了两个工具：`setprop` 和 `getprop` 获取和设置属性。其使用方法：

> getprop <属性名>
>
> setprop <属性名> <属性值>

**Java**

​    在 Java 应用程序可以使用 `System.getProperty()` 和 `System.setProperty()` 函数获取和设置属性。

**Action**

​     默认情况下，设置属性只会使 `init` 守护程序写入共享内存，它不会执行任何脚本或二进制程序。但是，您可以将您的想要的实现的操作与 `init.rc` 中某个属性的变化相关联。例如，在默认的 `init.rc` 中有：

```vbnet
# adbd on at boot in emulator
on property:ro.kernel.qemu=1
start adbd
on property:persist.service.adb.enbale=1
start adbd
on property:persist.service.adb.enable=0
stop adbd
```

 

## 0.4Properties Source

属性的设置可以出现在 **make android 的任何环节**。

> `alps/build/target/board/generic_arm64/system.prop`
>
> `alps/build/target/product/core.mk`
>
> `alps/buid/tools/buildinfo.sh`

编译好后，被设置的系统属性主要存放在：

这样，如果你设置 `persist.service.adb.enable` 为1，`init` 守护进程就知道需要采取行动：开启 `adbd` 服务。

> `/default.prop`         手机厂商自己定制使用
>
> `/system/build.prop`     系统属性主要存放处
>
> `/system/default.prop`    default properties，存放与 security 相关的属性
>
> `/data/property/persistent_properties`       保存着持久属性







# 系统初始化属性

初始化属性一定是在系统初始化的时候就存在，且存在的时刻不能太晚，后续好多rc文件去启动进程，都会大量的需要属性的读取和设置。

```c++
//system/core/init/init.cpp
static int property_fd = -1;
int SecondStageMain(int argc, char** argv) {
    PropertyInit();
    ...
    StartPropertyService(&property_fd);
    ...
}
```

# 1PropertyInit

用于属性的初始化，包括创建有关`PropertyInfo`的序列化，创建对应的`SElinux`的`Context`文件，初始化属性的导入，这里也会加载一些属性脚本文件等

```c++
//system/core/init/property_service.cpp
void PropertyInit() {
    //SElinux初始化配置，这个不是很重要
    selinux_callback cb;
    cb.func_audit = PropertyAuditCallback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);
	//1.创建属性文件，用于序列化操作
    mkdir("/dev/__properties__", S_IRWXU | S_IXGRP | S_IXOTH);
    CreateSerializedPropertyInfo();
    //2.初始化属性安全上下文
    if (__system_property_area_init()) {
        LOG(FATAL) << "Failed to initialize property area";
    }
    //3.属性安全上下文的文件导入
    if (!property_info_area.LoadDefaultPath()) {
        LOG(FATAL) << "Failed to load serialized property info file";
    }
	//4.其他的属性初始化
    ProcessKernelDt();
    ProcessKernelCmdline();

    ExportKernelBootProps();
    PropertyLoadBootDefaults();
}

```

### 1.1创建属性文件，用于序列化操作

首先在设备中创建`/dev/__properties__`，这个目录后续会对其进行**序列化**。

```c++
//system/core/init/property_service.cpp
void CreateSerializedPropertyInfo() {
    auto property_infos = std::vector<PropertyInfoEntry>();
    if (access("/system/etc/selinux/plat_property_contexts", R_OK) != -1) {
        if (!LoadPropertyInfoFromFile("/system/etc/selinux/plat_property_contexts",
                                      &property_infos)) {
            return;
        }
        // Don't check for failure here, so we always have a sane list of properties.
        // E.g. In case of recovery, the vendor partition will not have mounted and we
        // still need the system / platform properties to function.
        if (access("/system_ext/etc/selinux/system_ext_property_contexts", R_OK) != -1) {
            LoadPropertyInfoFromFile("/system_ext/etc/selinux/system_ext_property_contexts",
                                     &property_infos);
        }
        if (!LoadPropertyInfoFromFile("/vendor/etc/selinux/vendor_property_contexts",
                                      &property_infos)) {
            // Fallback to nonplat_* if vendor_* doesn't exist.
            LoadPropertyInfoFromFile("/vendor/etc/selinux/nonplat_property_contexts",
                                     &property_infos);
        }
        if (access("/product/etc/selinux/product_property_contexts", R_OK) != -1) {
            LoadPropertyInfoFromFile("/product/etc/selinux/product_property_contexts",
                                     &property_infos);
        }
        if (access("/odm/etc/selinux/odm_property_contexts", R_OK) != -1) {
            LoadPropertyInfoFromFile("/odm/etc/selinux/odm_property_contexts", &property_infos);
        }
    } else {
        if (!LoadPropertyInfoFromFile("/plat_property_contexts", &property_infos)) {
            return;
        }
        LoadPropertyInfoFromFile("/system_ext_property_contexts", &property_infos);
        if (!LoadPropertyInfoFromFile("/vendor_property_contexts", &property_infos)) {
            // Fallback to nonplat_* if vendor_* doesn't exist.
            LoadPropertyInfoFromFile("/nonplat_property_contexts", &property_infos);
        }
        LoadPropertyInfoFromFile("/product_property_contexts", &property_infos);
        LoadPropertyInfoFromFile("/odm_property_contexts", &property_infos);
    }

    auto serialized_contexts = std::string();
    auto error = std::string();
    if (!BuildTrie(property_infos, "u:object_r:default_prop:s0", "string", &serialized_contexts,
                   &error)) {
        LOG(ERROR) << "Unable to serialize property contexts: " << error;
        return;
    }

    constexpr static const char kPropertyInfosPath[] = "/dev/__properties__/property_info";
    if (!WriteStringToFile(serialized_contexts, kPropertyInfosPath, 0444, 0, 0, false)) {
        PLOG(ERROR) << "Unable to write serialized property infos to file";
    }
    selinux_android_restorecon(kPropertyInfosPath, 0);
}
```

这里的逻辑分成两块，第一块是`LoadPropertyInfoFromFile`用于将对应文件中的词条一条条导入到`property_infos`的数组中。

第二块是将上述的数组通过字典树的方式以二进制序列化的形式存储在变量`serialized_contexts`中，最后变成文件的形式存到`/dev/__properties__/property_info`里面。

### 1.1.1LoadPropertyInfoFromFile

```c++
//system/core/init/property_service.cpp
bool LoadPropertyInfoFromFile(const std::string& filename,
                              std::vector<PropertyInfoEntry>* property_infos) {
    auto file_contents = std::string();
    //将文件内容转化成字符串形式，导入到file_contents中
    if (!ReadFileToString(filename, &file_contents)) {
        PLOG(ERROR) << "Could not read properties from '" << filename << "'";
        return false;
    }

    auto errors = std::vector<std::string>{};
    bool require_prefix_or_exact = SelinuxGetVendorAndroidVersion() >= __ANDROID_API_R__;
    //将对应的/system/etc/selinux/plat_property_contexts，
    ///system/etc/selinux/plat_property_contexts等文件
    //依次变成字符串的形式去对应解析成PropertyInfoEntry数组的形式
    ParsePropertyInfoFile(file_contents, require_prefix_or_exact, property_infos, &errors);
    ...
    return true;
}
```

这里的property_infos类型是`vector<PropertyInfoEntry>*`，这个类似于函数运行结束的返回值写法，并且传入到ParsePropertyInfoFile函数中，是在这个函数中获取这个数组的。

```c++
//system/core/property_service/libpropertyinfoserializer/property_info_file.cpp
void ParsePropertyInfoFile(const std::string& file_contents, bool require_prefix_or_exact,
                           std::vector<PropertyInfoEntry>* property_infos,
                           std::vector<std::string>* errors) {
  errors->clear();
  //将对应文件中的"\n"来分割文件内容，即按照行来分割文件内容，注释和空白行自动过滤掉
  for (const auto& line : Split(file_contents, "\n")) {
    auto trimmed_line = Trim(line);
    if (trimmed_line.empty() || StartsWith(trimmed_line, "#")) {
      continue;
    }

    auto property_info_entry = PropertyInfoEntry{};
    auto parse_error = std::string{};
    //最终通过行解析得到具体的数组值，一个PropertyInfoEntry类型的实例
    if (!ParsePropertyInfoLine(trimmed_line, require_prefix_or_exact, &property_info_entry,
                               &parse_error)) {
      errors->emplace_back(parse_error);
      continue;
    }

    property_infos->emplace_back(property_info_entry);
  }
}
```

`ParsePropertyInfoLine`

```c++
bool ParsePropertyInfoLine(const std::string& line, bool require_prefix_or_exact,
                           PropertyInfoEntry* out, std::string* error) {
  //去除一行首部和尾部中的空格  
  auto tokenizer = SpaceTokenizer(line);
  //获取第一列数据，为property的name（string类型）
  auto property = tokenizer.GetNext();
  ...
  //获取第二列数据，为SElinux的context（string类型）
  auto context = tokenizer.GetNext();
  ...
  //获取第三列数据，为exact/prefix/...参数
  auto match_operation = tokenizer.GetNext();
  auto type_strings = std::vector<std::string>{};
  //获取第四列的数据类型，string/bool/int/uint/double/size/enum(定义在property_type.cpp中)
  auto type = tokenizer.GetNext();
  //这里就是为了获取enum类型存在的
  while (!type.empty()) {
    type_strings.emplace_back(type);
    type = tokenizer.GetNext();
  }

  bool exact_match = false;
  if (match_operation == "exact") {
    exact_match = true;
  } else if (match_operation != "prefix" && match_operation != "" && require_prefix_or_exact) {
    return false;
  }
  ...
  *out = {property, context, Join(type_strings, " "), exact_match};
  return true;
}
```

比如设备中的`/system/etc/selinux/plat_property_contexts`文件

```txt
#截取部分内容
bluetooth.              u:object_r:bluetooth_prop:s0
config.                 u:object_r:config_prop:s0
ctl.bootanim            u:object_r:ctl_bootanim_prop:s0
ctl.bugreport           u:object_r:ctl_bugreport_prop:s0
...
ro.boot.dynamic_partitions    u:object_r:exported_default_prop:s0              exact string
aaudio.hw_burst_min_usec      u:object_r:exported_default_prop:s0              exact string
ro.boot.hardware.revision     u:object_r:exported_default_prop:s0              exact string
ro.surfaceflinger.primary_display_orientation u:object_r:exported_default_prop:s0 exact
 ORENTATION_0 ORENTATION_180 ORENTATION_270 ORENTATION_90
cache_key.bluetooth           u:object_r:binder_cacherbluetooth_server_prop:s0 prefix string
graphics_gpu.profiler.support u:object_r:graphics_config_prop:s0               exact bool
...
```

转化为类似

```c++
//PropertyInfoEntry类型数组
...
{ro.boot.dynamic_partitions,   u:object_r:exported_default_prop:s0,              string, true},
{aaudio.hw_burst_min_usec,     u:object_r:exported_default_prop:s0,              string, true},
{ro.boot.hardware.revision,    u:object_r:exported_default_prop:s0,              string, true},
{ro.surfaceflinger.primary_display_orientation, u:object_r:exported_default_prop:s0, 
 {ORENTATION_0, ORENTATION_180, ORENTATION_270, ORENTATION_90}, true},
{cache_key.bluetooth,          u:object_r:binder_cacherbluetooth_server_prop:s0, string, false},
{graphics_gpu.profiler.support,u:object_r:graphics_config_prop:s0,               bool,   true},
...
```

{{< image classes="fancybox center fig-100" src="/property/property_4.png" thumbnail="/property/property_4.png" title="">}}

### 1.1.2BuildTrie

```c++
//system/core/property_service/libpropertyinfoserializer/property_info_serializer.cpp
//构建字典树，并且序列化
bool BuildTrie(const std::vector<PropertyInfoEntry>& property_info,
               const std::string& default_context, const std::string& default_type,
               std::string* serialized_trie, std::string* error) {
  // Check that names are legal first
  auto trie_builder = TrieBuilder(default_context, default_type);

  for (const auto& [name, context, type, is_exact] : property_info) {
    //生成字典树
    if (!trie_builder.AddToTrie(name, context, type, is_exact, error)) {
      return false;
    }
  }
  //实例化字典树序列器
  auto trie_serializer = TrieSerializer();
  //字典树序列化
  *serialized_trie = trie_serializer.SerializeTrie(trie_builder);
  return true;
}
```

显然，这边主要是一个装饰着模式的应用，主要的实现还在其他地方，这个函数主要作用就是用于把之前导入的属性构建成字典树然后将其序列化成字符串的形式，最终会调到`CreateSerializedPropertyInfo`函数，变成一个序列化的文件`/dev/__properties__/property_info`

#### 1.1.2.1TrieBuilder

```c++
//system/core/property_service/libpropertyinfoserializer/trie_builder.cpp
TrieBuilder::TrieBuilder(const std::string& default_context, const std::string& default_type)
    : builder_root_("root") {
  auto* context_pointer = StringPointerFromContainer(default_context, &contexts_);
  builder_root_.set_context(context_pointer);
  auto* type_pointer = StringPointerFromContainer(default_type, &types_);
  builder_root_.set_type(type_pointer);
}
```

如下图所示，是字典树的数据结构，简化后如右图所示，只显示`TrieBuilder`构造完成后对应的关系的数据结构图

{{< image classes="fancybox center fig-100" src="/property/property_5.png" thumbnail="/property/property_5.png" title="">}}

#### 1.1.2.2AddToTrie

```c++
//system/core/property_service/libpropertyinfoserializer/trie_builder.cpp
bool TrieBuilder::AddToTrie(const std::string& name, const std::string& context,
                            const std::string& type, bool exact, std::string* error) {
  auto* context_pointer = StringPointerFromContainer(context, &contexts_);
  auto* type_pointer = StringPointerFromContainer(type, &types_);
  return AddToTrie(name, context_pointer, type_pointer, exact, error);
}

bool TrieBuilder::AddToTrie(const std::string& name, const std::string* context,
                            const std::string* type, bool exact, std::string* error) {
  //找到根结点，就是TrieBuilder构造完成的结点
  TrieBuilderNode* current_node = &builder_root_;
  //通过属性name的点来划分属性名字，分成几段	
  auto name_pieces = Split(name, ".");

  bool ends_with_dot = false;
  if (name_pieces.back().empty()) {
    ends_with_dot = true;
    name_pieces.pop_back();
  }

  // Move us to the final node that we care about, adding incremental nodes if necessary.
  while (name_pieces.size() > 1) {
    //按照排列寻找对应的结点的每一段在字典树中查找（不排序）  
    auto child = current_node->FindChild(name_pieces.front());
    if (child == nullptr) {
      //找不到字典树对应的结点，就新增一个结点（数组中添加）  
      child = current_node->AddChild(name_pieces.front());
    }
    if (child == nullptr) {
      *error = "Unable to allocate Trie node";
      return false;
    }
    current_node = child;
    name_pieces.erase(name_pieces.begin());
  }

  //传入到这里代表着属性name点的最后一段，根据传入的exact参数值判断是否是exact类型或者是prefix类型来添加对应的结点
  if (exact) {
    if (!current_node->AddExactMatchContext(name_pieces.front(), context, type)) {
      *error = "Duplicate exact match detected for '" + name + "'";
      return false;
    }
  } else if (!ends_with_dot) {
    if (!current_node->AddPrefixContext(name_pieces.front(), context, type)) {
      *error = "Duplicate prefix match detected for '" + name + "'";
      return false;
    }
  } else {
    auto child = current_node->FindChild(name_pieces.front());
    if (child == nullptr) {
      child = current_node->AddChild(name_pieces.front());
    }
    if (child == nullptr) {
      *error = "Unable to allocate Trie node";
      return false;
    }
    if (child->context() != nullptr || child->type() != nullptr) {
      *error = "Duplicate prefix match detected for '" + name + "'";
      return false;
    }
    child->set_context(context);
    child->set_type(type);
  }
  return true;
}
```

真正开始添加成为一棵树，并开始添加子节点，以1.1.1中的`PropertyInfoEntry`类型数组为例说明

{{< image classes="fancybox center fig-100" src="/property/property_6.png" thumbnail="/property/property_6.png" title="">}}

#### 1.1.2.3SerializeTrie

```C++
//system/core/property_service/libpropertyinfoserializer/trie_serializer.cpp
std::string TrieSerializer::SerializeTrie(const TrieBuilder& trie_builder) {
  arena_.reset(new TrieNodeArena());
  //arena_指针是TrieNodeArena类型的，也就是这里的arena_所指的data_的头部为PropertyInfoAreaHeader类型
  //PropertyInfoAreaHeader类型是为了记录TrieBuilder中各个元素距离data_指针的偏移
  auto header = arena_->AllocateObject<PropertyInfoAreaHeader>(nullptr);
  header->current_version = 1;
  header->minimum_supported_version = 1;
  //开始存储TrieBuilder数据结构中的contexts_,并记录当前的位置，这个位置是与data_指针相差的距离
  header->contexts_offset = arena_->size();
  SerializeStrings(trie_builder.contexts());
  //开始存储TrieBuilder数据结构中的types_,并记录当前的位置，这个位置是与data_指针相差的距离
  header->types_offset = arena_->size();
  SerializeStrings(trie_builder.types());
  //记录types_之后的位置，主要是用于TrieBuilderNode序列化的寻址
  header->size = arena_->size();
  //开始对TrieBuilderNode这个数据结构图进行序列化
  uint32_t root_trie_offset = WriteTrieNode(trie_builder.builder_root());
  //root_offset记录的是TrieNodeInternal类型和data_指针的偏移地址
  header->root_offset = root_trie_offset;
  //最终记录TrieBuilderNode的与data_指针的偏移地址
  header->size = arena_->size();
  //截断多余的空间，使得data_使用空间大小和current_data_point_相等
  return arena_->truncated_data();
}
```

关于`arena_`，`TrieNodeArena`等定义都在下面的头文件中，这个头文件比较重要，包含了**如何序列化的一些函数**

{{< image classes="fancybox center fig-100" src="/property/property_7.png" thumbnail="/property/property_7.png" title="">}}

```c++
//system/core/property_service/libpropertyinfoserializer/trie_node_arena.h
template <typename T>
class ArenaObjectPointer {
    public:
    ArenaObjectPointer(std::string& arena_data, uint32_t offset)
        : arena_data_(arena_data), offset_(offset) {}

    T* operator->() { return reinterpret_cast<T*>(arena_data_.data() + offset_); }

    private:
    std::string& arena_data_;
    uint32_t offset_;
};

class TrieNodeArena {
    public:
    TrieNodeArena() : current_data_pointer_(0) {}

    // We can't return pointers to objects since data_ may move when reallocated, thus invalidating
    // any pointers.  Therefore we return an ArenaObjectPointer, which always accesses elements via
    // data_ + offset.
    template <typename T>
    ArenaObjectPointer<T> AllocateObject(uint32_t* return_offset) {
        uint32_t offset;
        AllocateData(sizeof(T), &offset);
        if (return_offset) *return_offset = offset;
        return ArenaObjectPointer<T>(data_, offset);
    }
    uint32_t AllocateUint32Array(int length) {
        uint32_t offset;
        AllocateData(sizeof(uint32_t) * length, &offset);
        return offset;
    }

    uint32_t* uint32_array(uint32_t offset) {
        return reinterpret_cast<uint32_t*>(data_.data() + offset);
    }

    uint32_t AllocateAndWriteString(const std::string& string) {
        uint32_t offset;
        char* data = static_cast<char*>(AllocateData(string.size() + 1, &offset));
        strcpy(data, string.c_str());
        return offset;
    }

    void AllocateAndWriteUint32(uint32_t value) {
        auto location = static_cast<uint32_t*>(AllocateData(sizeof(uint32_t), nullptr));
        *location = value;
    }

    void* AllocateData(size_t size, uint32_t* offset) {
        //这里的意思，实际上是数据结构的内存对齐，保证分配的大小是4的整数倍
        size_t aligned_size = size + (sizeof(uint32_t) - 1) & ~(sizeof(uint32_t) - 1);

        if (current_data_pointer_ + aligned_size > data_.size()) {
            auto new_size = (current_data_pointer_ + aligned_size + data_.size()) * 2;
            data_.resize(new_size, '\0');
        }
        if (offset) *offset = current_data_pointer_;

        uint32_t return_offset = current_data_pointer_;
        current_data_pointer_ += aligned_size;
        return &data_[0] + return_offset;
    }

    uint32_t size() const { return current_data_pointer_; }

    const std::string& data() const { return data_; }
    std::string truncated_data() const {
        auto result = data_;
        result.resize(current_data_pointer_);
        return result;
    }

    private:
    std::string data_;
    uint32_t current_data_pointer_;
};
```

序列化拆分的数据结构

{{< image classes="fancybox center fig-100" src="/property/property_8.png" thumbnail="/property/property_8.png" title="">}}

文件`/dev/__properties__/property_info`中的数据结构如下所示，后续可根据对应的`PropertyInfoAreaHeader`、`TrieNodeInternal`、`PropertyEntry`三个结构体存储的内容来具体找到对应的属性中的偏移。

{{< image classes="fancybox center fig-100" src="/property/property_9.png" thumbnail="/property/property_9.png" title="">}}

### 1.2初始化属性安全上下文

初始化属性安全上下文`__system_property_area_init`

```c++
//bionic/libc/bionic/system_property_api.cpp
//__BIONIC_WEAK_FOR_NATIVE_BRIDGE代表__attribute__((__weak__, __noinline__))
//声明weak function,链接器不会从库中加载对象来解析弱引用。仅当由于其他原因在镜像中包含了定义时，它才能解析弱引用。
//noinline设置函数为非内联函数
//#define PROP_FILENAME "/dev/__properties__"
 __BIONIC_WEAK_FOR_NATIVE_BRIDGE
int __system_property_area_init() {
  bool fsetxattr_failed = false;
  return system_properties.AreaInit(PROP_FILENAME, &fsetxattr_failed) && !fsetxattr_failed ? 0 : -1;
}
```

`system_properties.AreaInit`

```c++
//bionic/libc/system_properties/system_properties.cpp
bool SystemProperties::AreaInit(const char* filename, bool* fsetxattr_failed) {
  if (strlen(filename) >= PROP_FILENAME_MAX) {
    return false;
  }
  strcpy(property_filename_, filename);

  contexts_ = new (contexts_data_) ContextsSerialized();
  if (!contexts_->Initialize(true, property_filename_, fsetxattr_failed)) {
    return false;
  }
  //初始化结束之后，会把initialized_设置为true
  initialized_ = true;
  return true;
}
```

这里主要`property_filename`_赋值为`/dev/__properties__`，另外对`contexts_`进行序列化初值，并且初始化，这里的初始化传参是true，跟app传参false区别开来，说明只有系统root用户才有权限写入操作。

### 1.2.1ContextsSerialized

`contexts_`的基类是`Contexts`，`ContextsSerialized`是`Contexts`的子类。

记录`ContextsSerialized`类的组成

```c++
class ContextsSerialized : public Contexts {
  const char* filename_;
  android::properties::PropertyInfoAreaFile property_info_area_file_;
  ContextNode* context_nodes_ = nullptr;
  size_t num_context_nodes_ = 0;
  size_t context_nodes_mmap_size_ = 0;
  prop_area* serial_prop_area_ = nullptr;
};
```

{{< image classes="fancybox center fig-100" src="/property/property_15.png" thumbnail="/property/property_15.png" title="">}}

### 1.2.2Initialize

这个没有定义构造函数，那么就是默认的构造函数`contexts_->Initialize`

```c++
//bionic/libc/system_properties/contexts_serialized.cpp
//初始化的进程是init进程，所有有root读取权限修改，传入的writable是true，传入的filename是"/dev/__properties__"
bool ContextsSerialized::Initialize(bool writable, const char* filename, bool* fsetxattr_failed) {
  filename_ = filename;
  //是否成功初始化属性  
  if (!InitializeProperties()) {
    return false;
  }

  if (writable) {
    mkdir(filename_, S_IRWXU | S_IXGRP | S_IXOTH);
    bool open_failed = false;
    if (fsetxattr_failed) {
      *fsetxattr_failed = false;
    }

    for (size_t i = 0; i < num_context_nodes_; ++i) {
      //开始创建context文件
      if (!context_nodes_[i].Open(true, fsetxattr_failed)) {
        open_failed = true;
      }
    }
    if (open_failed || !MapSerialPropertyArea(true, fsetxattr_failed)) {
      FreeAndUnmap();
      return false;
    }
  } else {
    if (!MapSerialPropertyArea(false, nullptr)) {
      FreeAndUnmap();
      return false;
    }
  }
  return true;
}
```

构造完成之后，`contexts_`的数据结构如下图所示

{{< image classes="fancybox center fig-100" src="/property/property_17.png" thumbnail="/property/property_17.png" title="">}}



#### 1.2.2.1InitializeProperties

```c++
//bionic/libc/system_properties/contexts_serialized.cpp
bool ContextsSerialized::InitializeProperties() {
  //导入1.1中完成序列化的property_info文件
  if (!property_info_area_file_.LoadDefaultPath()) {
    return false;
  }
  //开始创建SElinux安全上下文文件  
  if (!InitializeContextNodes()) {
    FreeAndUnmap();
    return false;
  }

  return true;
}
```

1）`LoadDefaultPath`

```c++
//system/core/property_service/libpropertyinfoparser/property_info_parser.cpp
bool PropertyInfoAreaFile::LoadDefaultPath() {
  return LoadPath("/dev/__properties__/property_info");
}

bool PropertyInfoAreaFile::LoadPath(const char* filename) {
  int fd = open(filename, O_CLOEXEC | O_NOFOLLOW | O_RDONLY);

  struct stat fd_stat;
  if (fstat(fd, &fd_stat) < 0) {
    close(fd);
    return false;
  }
  //判断打开的文件是不是init进程，必须是uid=0，gid=0，且文件大小需要大于PropertyInfoArea类型
  if ((fd_stat.st_uid != 0) || (fd_stat.st_gid != 0) ||
      ((fd_stat.st_mode & (S_IWGRP | S_IWOTH)) != 0) ||
      (fd_stat.st_size < static_cast<off_t>(sizeof(PropertyInfoArea)))) {
    close(fd);
    return false;
  }

  auto mmap_size = fd_stat.st_size;
  //这里做了mmap操作，相当于做了读操作的共享区域的映射
  void* map_result = mmap(nullptr, mmap_size, PROT_READ, MAP_SHARED, fd, 0);
  if (map_result == MAP_FAILED) {
    close(fd);
    return false;
  }
  //共享区域映射之后的指针指向PropertyInfoArea类型
  auto property_info_area = reinterpret_cast<PropertyInfoArea*>(map_result);
  if (property_info_area->minimum_supported_version() > 1 ||
      property_info_area->size() != mmap_size) {
    munmap(map_result, mmap_size);
    close(fd);
    return false;
  }

  close(fd);
  mmap_base_ = map_result;
  mmap_size_ = mmap_size;
  return true;
}
```

> **`mmap`说明**
>
> 关于`mmap`东西很多，这里之前浅谈一些基础`api`用法，具体可点击[这里](https://juejin.cn/post/7119116943256190990)。
>
> ```c++
> void* mmap(void* addr, size_t length, int prot, int flags, int fd, off_t offset);
> ```
>
> - addr 代表映射的虚拟内存起始地址；
> - length 代表该映射长度；
> - prot 描述了这块新的内存区域的访问权限；
> - flags 描述了该映射的类型；
> - fd 代表文件描述符；
> - offset 代表文件内的偏移值。
>
> |  参数   | 参数可选用的值  | 对应的含义                                                   |
> | :-----: | :-------------: | ------------------------------------------------------------ |
> | `prot`  |   `PROT_EXEC`   | 该内存映射有可执行权限，可以看成是代码段，通常存储CPU可执行机器码 |
> |         |   `PROT_READ`   | 该内存映射可读                                               |
> |         |  `PROT_WRITE`   | 该内存映射可写                                               |
> |         |   `PROT_NONE`   | 该内存映射不能被访问                                         |
> | `flags` |  `MAP_SHARED`   | 创建一个共享映射区域                                         |
> |         |  `MAP_PRIVATE`  | 创建一个私有映射区域                                         |
> |         | `MAP_ANONYMOUS` | 创建一个匿名映射区域，该情况只需要传入-1即可                 |
> |         |   `MAP_FIXED`   | 当操作系统以`addr为`起始地址进行内存映射时，如果发现不能满足长度或者权限要求时，将映射失败，如果非`MAP_FIXED`，则系统就会再找其他合适的区域进行映射 |
> |  `fd`   |      大于0      | 内存映射将与文件进行关联                                     |
> |         |       -1        | 匿名映射，此时flags必为`MAP_ANONYMOUS`                       |
>
> {{< image classes="fancybox center fig-100" src="/property/property_10.png" thumbnail="/property/property_10.png" title="">}}

这里的强转的`property_info_area`指针其实对应着上文1.1中的`arena_`指针

{{< image classes="fancybox center fig-100" src="/property/property_11.png" thumbnail="/property/property_11.png" title="">}}

2）`InitializeContextNodes`

生成对应的`SElinux`安全上下文的文件

```c++
//bionic/libc/system_properties/contexts_serialized.cpp
bool ContextsSerialized::InitializeContextNodes() {
  //num_context_nodes这里指的arena_头部结构体PropertyInfoAreaHeader中的contexts_个数
  auto num_context_nodes = property_info_area_file_->num_contexts();
  //计算出一个大小为ContextNode大小*contexts_个数的空间
  auto context_nodes_mmap_size = sizeof(ContextNode) * num_context_nodes;
  //使用mmap分配了一个私有匿名的内存，缓存在context_nodes_中
  void* const map_result = mmap(nullptr, context_nodes_mmap_size, PROT_READ | PROT_WRITE,
                                MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
  if (map_result == MAP_FAILED) {
    return false;
  }
  //调用一次prctl使用PR_SET_VMA参数，功能是给addr和vsize指定的匿名内存区域命名了"System property context nodes"
  prctl(PR_SET_VMA, PR_SET_VMA_ANON_NAME, map_result, context_nodes_mmap_size,
        "System property context nodes");

  context_nodes_ = reinterpret_cast<ContextNode*>(map_result);
  num_context_nodes_ = num_context_nodes;
  context_nodes_mmap_size_ = context_nodes_mmap_size;
  for (size_t i = 0; i < num_context_nodes; ++i) {
    //对每一个Context进行实例化，并且默认的初始值传入
    //这个property_info_area_file_->context(i)为具体的context内容，filename_为/dev/__properties__
    new (&context_nodes_[i]) ContextNode(property_info_area_file_->context(i), filename_);
  }
  return true;
}
```

同`LoadDefaultPath`类似，这里也出现`mmap`操作，不过这里更像是`malloc`分配内存，这个`mmap`操作既是匿名的又是私有的，并且实例化`num_context_nodes`个`ContextNode`。

{{< image classes="fancybox center fig-100" src="/property/property_12.png" thumbnail="/property/property_12.png" title="">}}

#### 1.2.2.2Open

```c++
//bionic/libc/system_properties/context_node.cpp
bool ContextNode::Open(bool access_rw, bool* fsetxattr_failed) {
  //这里的pa_是prop_area类型的指针，如果pa_安全上下文存在说明文件已经存在，直接退出  
  lock_.lock();
  if (pa_) {
    lock_.unlock();
    return true;
  }

  char filename[PROP_FILENAME_MAX];
  //filename 对应的是具体的SElinux的context
  //比如/dev/__properties__/u:object_r:exported_default_prop:s0  
  int len = async_safe_format_buffer(filename, sizeof(filename), "%s/%s", filename_, context_);
  if (len < 0 || len >= PROP_FILENAME_MAX) {
    lock_.unlock();
    return false;
  }

  if (access_rw) {
    //创建对应的文件
    pa_ = prop_area::map_prop_area_rw(filename, context_, fsetxattr_failed);
  } else {
    pa_ = prop_area::map_prop_area(filename);
  }
  lock_.unlock();
  return pa_;
}
```

`map_prop_area_rw`

```c++
//bionic/libc/system_properties/prop_area.cpp
prop_area* prop_area::map_prop_area_rw(const char* filename, const char* context,
                                       bool* fsetxattr_failed) {
  //打开文件，不存在就创建文件
  const int fd = open(filename, O_RDWR | O_CREAT | O_NOFOLLOW | O_CLOEXEC | O_EXCL, 0444);
  ...
  //文件大小设置为PA_SIZE（128 * 1024），多的大小那么就截断    
  if (ftruncate(fd, PA_SIZE) < 0) {
    close(fd);
    return nullptr;
  }

  pa_size_ = PA_SIZE;
  pa_data_size_ = pa_size_ - sizeof(prop_area);
  //通过mmap映射了一个可读可写的共享区域
  void* const memory_area = mmap(nullptr, pa_size_, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
  if (memory_area == MAP_FAILED) {
    close(fd);
    return nullptr;
  }
  //并且设置区域的开始为prop_area类型，且附上初值PROP_AREA_MAGIC = 0x504f5250，PROP_AREA_VERSION = 0xfc6ed0ab
  prop_area* pa = new (memory_area) prop_area(PROP_AREA_MAGIC, PROP_AREA_VERSION);

  close(fd);
  return pa;
}
```

这里第三次出现了`mmap`，而且是通过创建的`SElinux`安全上下文文件来映射对应的共享内存区域

{{< image classes="fancybox center fig-100" src="/property/property_13.png" thumbnail="/property/property_13.png" title="">}}

#### 1.2.2.3MapSerialPropertyArea

```c++
//bionic/libc/system_properties/contexts_serialized.cpp
bool ContextsSerialized::MapSerialPropertyArea(bool access_rw, bool* fsetxattr_failed) {
  char filename[PROP_FILENAME_MAX];
  //这里的filename是/dev/__properties__/properties_serial
  int len = async_safe_format_buffer(filename, sizeof(filename), "%s/properties_serial", filename_);
  if (len < 0 || len >= PROP_FILENAME_MAX) {
    serial_prop_area_ = nullptr;
    return false;
  }

  if (access_rw) {
    //init进程肯定是走可读写权限，创建了对应的/dev/__properties__/properties_serial文件，大小为128K
    //这里传入的context_跟上述不一致，这里是"u:object_r:properties_serial:s0"，主要用于校验作用
    serial_prop_area_ =
        prop_area::map_prop_area_rw(filename, "u:object_r:properties_serial:s0", fsetxattr_failed);
  } else {
    serial_prop_area_ = prop_area::map_prop_area(filename);
  }
  return serial_prop_area_;
}
```

{{< image classes="fancybox center fig-100" src="/property/property_14.png" thumbnail="/property/property_14.png" title="">}}

### 1.3属性安全上下文的文件导入

这里跟1.2中的有重复部分，这里就不具体展开了，作用是为了二次确认是否`/dev/__properties__/property_info`是否映射正常



### 1.4其他的属性初始化

```c++
ProcessKernelDt();
ProcessKernelCmdline();
ExportKernelBootProps();
PropertyLoadBootDefaults();
```

### 1.4.1ProcessKernelDt

```c++
//system/core/init/property_service.cpp
static void ProcessKernelDt() {
    //判断 /proc/device-tree/firmware/android/compatible 文件中的值是否为 android,firmware
    if (!is_android_dt_value_expected("compatible", "android,firmware")) {
        return;
    }
    //get_android_dt_dir()的值为/proc/device-tree/firmware/android
    std::unique_ptr<DIR, int (*)(DIR*)> dir(opendir(get_android_dt_dir().c_str()), closedir);
    if (!dir) return;

    std::string dt_file;
    struct dirent* dp;
    ////遍历dir中的文件
    while ((dp = readdir(dir.get())) != NULL) {
        if (dp->d_type != DT_REG || !strcmp(dp->d_name, "compatible") ||
            !strcmp(dp->d_name, "name")) {
            continue;
        }

        std::string file_name = get_android_dt_dir() + dp->d_name;

        android::base::ReadFileToString(file_name, &dt_file);
        std::replace(dt_file.begin(), dt_file.end(), ',', '.');
        // 将 ro.boot.文件名 作为key，文件内容为value，设置进属性
        InitPropertySet("ro.boot."s + dp->d_name, dt_file);
    }
}
```

### 1.4.2ProcessKernelCmdline

```c++
//system/core/init/property_service.cpp
static void ProcessKernelCmdline() {
    bool for_emulator = false;
    ImportKernelCmdline([&](const std::string& key, const std::string& value) {
        if (key == "qemu") {
            for_emulator = true;
        } else if (StartsWith(key, "androidboot.")) {
            InitPropertySet("ro.boot." + key.substr(12), value);
        }
    });

    if (for_emulator) {
        ImportKernelCmdline([&](const std::string& key, const std::string& value) {
            // In the emulator, export any kernel option with the "ro.kernel." prefix.
            InitPropertySet("ro.kernel." + key, value);
        });
    }
}
```

主要展开讲一下`ProcessKernelCmdline`，主要是处理`kernel`预设的`Cmdline`的一些属性（在Android系统中，可以用`cat /proc/cmdline`查看），并且把`androidboot.`前缀的`key`值找出来，修改成`ro.boot.`的`key`，和把`androidboot.`前缀的对应的`value`值形成预设的属性值。

如果需要自定义一些在kernel阶段预设的属性进行设置，可以在`Uboot`阶段，进行设置

```c++
//类似把androidboot.abc放入到cmdline中
//Uboot代码中的setup_bootenv函数添加下面的语句
Setenv2Bootargs("androidboot.abc=", "yangyang48");
```

然后可以在设备上打印具体的`cmdline`

```txt
vince:/ # cat /proc/cmdline
cat /proc/cmdline
sched_enable_hmp=1 sched_enable_power_aware=1 console=ttyHSL0,115200,n8 androidboot.console=ttyHSL0 androidboot.hardware=qcom msm_rtb.filter=0x237 ehci-hcd.park=3 lpm_levels.sleep_disabled=1 androidboot.bootdevice=7824900.sdhci earlycon=msm_hsl_uart,0x78af000 buildvariant=user androidboot.emmc=true androidboot.verifiedbootstate=orange androidboot.veritymode=enforcing androidboot.keymaster=1 ddr_sorting=undo androidboot.abc=yangyang48 androidboot.serialno=357370480904 androidboot.secureboot=1 androidboot.product.region=cn androidboot.bootloader=MSM8953_DAISY1.0_20190902222450 androidboot.baseband=msm mdss_mdp.panel=1:dsi:0:qcom,mdss_dsi_td4310_fhdplus_video_e7:wpoint=03:1:none:cfg:single_dsi
```

通过上面的`ProcessKernelCmdline`，可以得知属性`ro.boot.abc`会被设置进去

### 1.4.3ExportKernelBootProps

```c++
//system/core/init/property_service.cpp
static void ExportKernelBootProps() {
    constexpr const char* UNSET = "";
    struct {
        const char* src_prop;
        const char* dst_prop;
        const char* default_value;
    } prop_map[] = {
            // clang-format off
        { "ro.boot.serialno",   "ro.serialno",   UNSET, },
        { "ro.boot.mode",       "ro.bootmode",   "unknown", },
        { "ro.boot.baseband",   "ro.baseband",   "unknown", },
        { "ro.boot.bootloader", "ro.bootloader", "unknown", },
        { "ro.boot.hardware",   "ro.hardware",   "unknown", },
        { "ro.boot.revision",   "ro.revision",   "0", },
            // clang-format on
    };
    for (const auto& prop : prop_map) {
        std::string value = GetProperty(prop.src_prop, prop.default_value);
        if (value != UNSET) InitPropertySet(prop.dst_prop, value);
    }
}
```

`ro.serialno`， 可见它是从内核启动参数`ro.boot.serialno`，如果这些`ro`属性没有被设置，那么通过这里的初始化来设置属性。

### 1.4.4PropertyLoadBootDefaults

```c++
void PropertyLoadBootDefaults() {
    std::map<std::string, std::string> properties;
    if (!load_properties_from_file("/system/etc/prop.default", nullptr, &properties)) {
        // Try recovery path
        if (!load_properties_from_file("/prop.default", nullptr, &properties)) {
            // Try legacy path
            load_properties_from_file("/default.prop", nullptr, &properties);
        }
    }
    load_properties_from_file("/system/build.prop", nullptr, &properties);
    load_properties_from_file("/system_ext/build.prop", nullptr, &properties);
    load_properties_from_file("/vendor/default.prop", nullptr, &properties);
    load_properties_from_file("/vendor/build.prop", nullptr, &properties);
    if (SelinuxGetVendorAndroidVersion() >= __ANDROID_API_Q__) {
        load_properties_from_file("/odm/etc/build.prop", nullptr, &properties);
    } else {
        load_properties_from_file("/odm/default.prop", nullptr, &properties);
        load_properties_from_file("/odm/build.prop", nullptr, &properties);
    }
    load_properties_from_file("/product/build.prop", nullptr, &properties);
    load_properties_from_file("/factory/factory.prop", "ro.*", &properties);

    if (access(kDebugRamdiskProp, R_OK) == 0) {
        LOG(INFO) << "Loading " << kDebugRamdiskProp;
        load_properties_from_file(kDebugRamdiskProp, nullptr, &properties);
    }

    for (const auto& [name, value] : properties) {
        std::string error;
        if (PropertySet(name, value, &error) != PROP_SUCCESS) {
            LOG(ERROR) << "Could not set '" << name << "' to '" << value
                       << "' while loading .prop files" << error;
        }
    }
    //关于ro.product.[brand|device|manufacturer|model|name]属性的配置
    property_initialize_ro_product_props();
    //关于ro.build.[fingerprint|brand|name|device|version|id|type|tags]属性的配置
    property_derive_build_fingerprint();
	//persist.sys.usb.config是否是adb连接，笔者连接时[persist.sys.usb.config]: [mtp,adb]
    update_sys_usb_config();
}
```

这个函数是用于导入一些默认的属性配置，一般工程开发会在这里面添加一些关于自定义但需要开机设置的一些属性。可以看到上面的属性都可以在这个阶段来进行初始化设置，属性会被添加到属性对应的`SElinux`安全上下文文件中。

通过键值对`map`来获取这些配置文件里面的属性，然后统一设置**开机预置**属性。(`map`默认按照`key`从小到大进行排序)

{{< image classes="fancybox center fig-100" src="/property/property_3.png" thumbnail="/property/property_3.png" title="">}}



## 2StartPropertyService

用于属性服务的初始化，这个服务是用于管控属性设置的

```c++
//system/core/init/property_service.cpp
void StartPropertyService(int* epoll_socket) {
    InitPropertySet("ro.property_service.version", "2");
    //初始化了双工通信，这里用于本地通信
    int sockets[2];
    if (socketpair(AF_UNIX, SOCK_SEQPACKET | SOCK_CLOEXEC, 0, sockets) != 0) {
        PLOG(FATAL) << "Failed to socketpair() between property_service and init";
    }
    //这里的epoll_socket会被作为双工通信的发送端
    *epoll_socket = from_init_socket = sockets[0];
    //这里的init_socket会被作为双工通信的接收端
    init_socket = sockets[1];
    //这个用于rc文件中属性发生变化的时候的作用
    StartSendingMessages();
    //建立一个socket，用于管理属性服务
    if (auto result = CreateSocket(PROP_SERVICE_NAME, SOCK_STREAM | SOCK_CLOEXEC | SOCK_NONBLOCK,
                                   false, 0666, 0, 0, {});
        result.ok()) {
        property_set_fd = *result;
    } else {
        LOG(FATAL) << "start_property_service socket creation failed: " << result.error();
    }
    //监听socket，accept 队列长度为8，也就是已完成连接建立的队列长度
    listen(property_set_fd, 8);

    auto new_thread = std::thread{PropertyServiceThread};
    property_service_thread.swap(new_thread);
}
```

## 2.1StartSendingMessages

主要是为了设置一个true的一个flag值，用于`rc`文件中的属性变化来起到触发作用

```c++
//system/core/init/property_service.cpp
void StartSendingMessages() {
    auto lock = std::lock_guard{accept_messages_lock};
    accept_messages = true;
}
```

> 关于lock_guard也是一种比较新的特性，更加方便。**构造时候加锁，析构的时候解锁**，具体可以点击[这里](https://blog.csdn.net/gehong3641/article/details/124028976)。
>
> 1. lock_guard具有两种构造方法
>
>    `lock_guard(mutex& m)`，这种构造会加锁
>
>    `lock_guard(mutex& m, adopt_lock)`，这种构造不会加锁
>
> 2. lock_guard默认的析构方法
>
>    ~lock_guard()，用于解锁



## 2.2CreateSocket

```c++
//system/core/init/util.cpp
//传入的name为"property_service"，passcred为false，perm为0666，uid，gid为0
Result<int> CreateSocket(const std::string& name, int type, bool passcred, mode_t perm, uid_t uid,
                         gid_t gid, const std::string& socketcon) {
    if (!socketcon.empty()) {
        if (setsockcreatecon(socketcon.c_str()) == -1) {
            return ErrnoError() << "setsockcreatecon(\"" << socketcon << "\") failed";
        }
    }
    //使用本地协议PF_UNIX(或PF_LOCAL),type中包含SOCK_STREAM，因此创建本地通信的流式套接字为fd
    android::base::unique_fd fd(socket(PF_UNIX, type, 0));
    if (fd < 0) {
        return ErrnoError() << "Failed to open socket '" << name << "'";
    }

    if (!socketcon.empty()) setsockcreatecon(nullptr);

    struct sockaddr_un addr;
    memset(&addr, 0 , sizeof(addr));
    addr.sun_family = AF_UNIX;
    //ANDROID_SOCKET_DIR为"/dev/socket"，那么addr.sun_path为"/dev/socket/property_service"
    snprintf(addr.sun_path, sizeof(addr.sun_path), ANDROID_SOCKET_DIR "/%s", name.c_str());

    if ((unlink(addr.sun_path) != 0) && (errno != ENOENT)) {
        return ErrnoError() << "Failed to unlink old socket '" << name << "'";
    }

    std::string secontext;
    //对路径addr.sun_path的权限检测
    if (SelabelLookupFileContext(addr.sun_path, S_IFSOCK, &secontext) && !secontext.empty()) {
        setfscreatecon(secontext.c_str());
    }

    if (passcred) {
        int on = 1;
        if (setsockopt(fd, SOL_SOCKET, SO_PASSCRED, &on, sizeof(on))) {
            return ErrnoError() << "Failed to set SO_PASSCRED '" << name << "'";
        }
    }

    int ret = bind(fd, (struct sockaddr *) &addr, sizeof (addr));
    int savederrno = errno;
    ...
    return fd.release();
}
```

创建了一个本地的`socket`，且路径为`/dev/socket/property_service`

## 2.3PropertyServiceThread

```c++
//system/core/init/property_service.cpp
static void PropertyServiceThread() {
    Epoll epoll;
    if (auto result = epoll.Open(); !result.ok()) {
        LOG(FATAL) << result.error();
    }
    //这里是封装了一层系统调用的epoll，通过监听fd来完成对函数的调用
    if (auto result = epoll.RegisterHandler(property_set_fd, handle_property_set_fd);
        !result.ok()) {
        LOG(FATAL) << result.error();
    }
    //这里是封装了一层系统调用的epoll，通过监听fd来完成对函数的调用
    if (auto result = epoll.RegisterHandler(init_socket, HandleInitSocket); !result.ok()) {
        LOG(FATAL) << result.error();
    }
    
    while (true) {
        auto pending_functions = epoll.Wait(std::nullopt);
        if (!pending_functions.ok()) {
            LOG(ERROR) << pending_functions.error();
        } else {
            //当上面的fd发生变化的时候，立刻调用对应注册的函数
            for (const auto& function : *pending_functions) {
                (*function)();
            }
        }
    }
}
```

### 2.3.1handle_property_set_fd

```c++
//system/core/init/property_service.cpp
static void handle_property_set_fd() {
    static constexpr uint32_t kDefaultSocketTimeout = 2000; /* ms */
    //SOCK_CLOEXEC，用于防止父进程泄露打开的文件给子进程
    int s = accept4(property_set_fd, nullptr, nullptr, SOCK_CLOEXEC);
    if (s == -1) {
        return;
    }

    ucred cr;
    socklen_t cr_size = sizeof(cr);
    if (getsockopt(s, SOL_SOCKET, SO_PEERCRED, &cr, &cr_size) < 0) {
        close(s);
        PLOG(ERROR) << "sys_prop: unable to get SO_PEERCRED";
        return;
    }
    //将原本socket的句柄转交给SocketConnection，SocketConnection实际上也是用poll机制的封装
    SocketConnection socket(s, cr);
    uint32_t timeout_ms = kDefaultSocketTimeout;

    uint32_t cmd = 0;
    //接收消息，超过2s就自动退出
    //消息体接收分别是cmd，name，value，context
    if (!socket.RecvUint32(&cmd, &timeout_ms)) {
        PLOG(ERROR) << "sys_prop: error while reading command from the socket";
        socket.SendUint32(PROP_ERROR_READ_CMD);
        return;
    }
    switch (cmd) {
    //PROP_MSG_SETPROP这个是老版本的魔数，Android11已经不使用这个api了
    case PROP_MSG_SETPROP: 
         ...
         break;    
    case PROP_MSG_SETPROP2: {
        std::string name;
        std::string value;
        if (!socket.RecvString(&name, &timeout_ms) ||
            !socket.RecvString(&value, &timeout_ms)) {
          PLOG(ERROR) << "sys_prop(PROP_MSG_SETPROP2): error while reading name/value from the socket";
          socket.SendUint32(PROP_ERROR_READ_DATA);
          return;
        }

        std::string source_context;
        if (!socket.GetSourceContext(&source_context)) {
            PLOG(ERROR) << "Unable to set property '" << name << "': getpeercon() failed";
            socket.SendUint32(PROP_ERROR_PERMISSION_DENIED);
            return;
        }

        const auto& cr = socket.cred();
        std::string error;
        //这里的接收比较简单，最终调用到HandlePropertySet这个地方
        uint32_t result = HandlePropertySet(name, value, source_context, cr, &socket, &error);
        if (result != PROP_SUCCESS) {
            LOG(ERROR) << "Unable to set property '" << name << "' from uid:" << cr.uid
                       << " gid:" << cr.gid << " pid:" << cr.pid << ": " << error;
        }
        //把成功的结果PROP_SUCCESS发送给对端
        socket.SendUint32(result);
        break;
      }

    default:
        LOG(ERROR) << "sys_prop: invalid command " << cmd;
        socket.SendUint32(PROP_ERROR_INVALID_CMD);
        break;
    }
}
```

这里主要分成几块，将原本创建的本地通信的流式套接字移交给poll机制封装的`SocketConnection`，通过`SocketConnection`来完成对消息体的接收，然后消息体正常接收会调用到`HandlePropertySet`，这个地方用于设置属性。

### 2.3.2HandleInitSocket

这个函数主要是处理persist属性的初始化，初始化会把文件中的persist属性导入到文件中去，导入完成之后在设置persist属性，并且设置一个完成导入属性的属性，`ro.persistent_properties.ready`的属性值为true，说明persist导入文件和设置属性完成。

```c++
//system/core/init/property_service.cpp
static void HandleInitSocket() {
    auto message = ReadMessage(init_socket);
    if (!message.ok()) {
        LOG(ERROR) << "Could not read message from init_dedicated_recv_socket: " << message.error();
        return;
    }
    //这里的InitMessage其实也是一个类，不过是proto格式转化为对应的c++格式的类
    auto init_message = InitMessage{};
    if (!init_message.ParseFromString(*message)) {
        LOG(ERROR) << "Could not parse message from init";
        return;
    }

    switch (init_message.msg_case()) {
        case InitMessage::kLoadPersistentProperties: {
            load_override_properties();
            //这里的persistent_properties的类是persistent_properties.proto中定义的
            auto persistent_properties = LoadPersistentProperties();
            for (const auto& persistent_property_record : persistent_properties.properties()) {
                InitPropertySet(persistent_property_record.name(),
                                persistent_property_record.value());
            }
            InitPropertySet("ro.persistent_properties.ready", "true");
            //这个变量会在后面的设置属性中用到，如果这里没有初始化成功，后续的persist对应文件将无法保存
            persistent_properties_loaded = true;
            break;
        }
        default:
            LOG(ERROR) << "Unknown message type from init: " << init_message.msg_case();
    }
}
```

`InitMessage`这个类真正定义的地方，可以简单的理解为是一个**序列化的消息体**。而且`protobuf`会将对应的`proto`文件中的变量`load_persistent_properties`，变成`kLoadPersistentProperties`，这就是为什么本文涉及到的好多源码的变量前会带有字母`k`。

```protobuf
//system/core/init/property_service.proto
syntax = "proto2";
option optimize_for = LITE_RUNTIME;

message PropertyMessage {
    message ControlMessage {
        optional string msg = 1;
        optional string name = 2;
        optional int32 pid = 3;
        optional int32 fd = 4;
    }

    message ChangedMessage {
        optional string name = 1;
        optional string value = 2;
    }

    oneof msg {
        ControlMessage control_message = 1;
        ChangedMessage changed_message = 2;
    };
}

message InitMessage {
    //oneof关键字，可以简单的理解为是联合体的数据结构
    oneof msg {
        bool load_persistent_properties = 1;
        bool stop_sending_messages = 2;
        bool start_sending_messages = 3;
    };
}
```

> 关于`proto`之前的[`dumpsys media.camera`](https://yangyang48.github.io/2022/04/dumpsys%E7%AE%80%E4%BB%8B-%E4%BB%A5media.camera%E4%B8%BA%E4%BE%8B/)文章中提到过一次，但没有具体深入，当然本文也不会特别深入。
>
> `Protobuf` 是一种与平台无关、语言无关、可扩展且轻便高效的**序列化数据结构**的协议，可以用于网络通信和**数据存储**。而且由于和语言无关，`proto`文件既可以转化成`c++`文件，也可以转化成`java`，`python`这类的文件。当然在源码中，可以直接理解为定义了一种数据类，专门用于序列化存储。
>
> 目前我们定义的`proto`文件，比如上面的`property_service.proto`，可以在下面的源码路径中找到
>
> ```text
> //其中xxx为具体的架构，有arm架构，arm架构等
> /out/soong/.intermediates/system/core/init/libinit/android_xxx_static/gen/proto/system/core/init/property_service.pb.h
> /out/soong/.intermediates/system/core/init/libinit/android_xxx_static/gen/proto/system/core/init/property_service.pb.cc
> ```
>
> 具体的规则如下，所指定的消息字段修饰符必须是如下之一：
>
> - required：一个格式良好的消息一定要含有1个这种字段。表示该值是必须要设置的；
> - optional：消息格式中该字段可以有0个或1个值（不超过1个）。
> - repeated：在一个格式良好的消息中，这种字段可以重复任意多次（包括0次）。重复的值的顺序会被保留。表示该值可以重复，相当于java中的List。
>
> 上面只是最简单的提到了`proto`，如果需要进一步了解语法规则，可以点击[这里](http://t.zoukankan.com/GarfieldEr007-p-10113328.html)。

#### 2.3.2.1load_override_properties

```c++
//system/core/init/property_service.cpp
static void load_override_properties() {
    //如果存在/data/local.prop，那么就和1.4.4一样的操作即可
    if (ALLOW_LOCAL_PROP_OVERRIDE) {
        std::map<std::string, std::string> properties;
        load_properties_from_file("/data/local.prop", nullptr, &properties);
        for (const auto& [name, value] : properties) {
            std::string error;
            if (PropertySet(name, value, &error) != PROP_SUCCESS) {
                LOG(ERROR) << "Could not set '" << name << "' to '" << value
                           << "' in /data/local.prop: " << error;
            }
        }
    }
}
```

#### 2.3.2.2LoadPersistentProperties

```c++
//system/core/init/persistent_properties.cpp
PersistentProperties LoadPersistentProperties() {
    //如果这里导入正常，直接返回即可
    //这里的persistent_properties结构是定义在persistent_properties.proto中定义的，
    auto persistent_properties = LoadPersistentPropertyFile();

    if (!persistent_properties.ok()) {
        LOG(ERROR) << "Could not load single persistent property file, trying legacy directory: "
                   << persistent_properties.error();
        persistent_properties = LoadLegacyPersistentProperties();
        if (!persistent_properties.ok()) {
            LOG(ERROR) << "Unable to load legacy persistent properties: "
                       << persistent_properties.error();
            return {};
        }
        if (auto result = WritePersistentPropertyFile(*persistent_properties); result.ok()) {
            RemoveLegacyPersistentPropertyFiles();
        } else {
            LOG(ERROR) << "Unable to write single persistent property file: " << result.error();
            // Fall through so that we still set the properties that we've read.
        }
    }

    return *persistent_properties;
}
```

##### 2.3.2.2.1persistent_properties.proto

```protobuf
//system/core/init/persistent_properties.proto
syntax = "proto2";
option optimize_for = LITE_RUNTIME;

message PersistentProperties {
    message PersistentPropertyRecord {
        optional string name = 1;
        optional string value = 2;
    }
    //可以理解为是list<PersistentPropertyRecord>
    //每一个PersistentPropertyRecord中都有name，value,但name，value不一定存在
    repeated PersistentPropertyRecord properties = 1;
}
```

##### 2.3.2.2.2LoadPersistentPropertyFile

```c++
//system/core/init/persistent_properties.cpp
Result<PersistentProperties> LoadPersistentPropertyFile() {
    //读取文件内容，如果内容不存在就直接返回
    auto file_contents = ReadPersistentPropertyFile();
    if (!file_contents.ok()) return file_contents.error();
    //如果内容存在，那么把文件中的内容变成PersistentProperties的形式
    PersistentProperties persistent_properties;
    if (persistent_properties.ParseFromString(*file_contents)) return persistent_properties;
    ...
}
```

`ReadPersistentPropertyFile`

```c++
//system/core/init/persistent_properties.cpp
//persistent_property_filename为"/data/property/persistent_properties"
Result<std::string> ReadPersistentPropertyFile() {
    //查看是否还存在未完成的临时tmp文件，存在就删除
    const std::string temp_filename = persistent_property_filename + ".tmp";
    if (access(temp_filename.c_str(), F_OK) == 0) {
        LOG(INFO)
            << "Found temporary property file while attempting to persistent system properties"
               " a previous persistent property write may have failed";
        unlink(temp_filename.c_str());
    }
    //读取对应的/data/property/persistent_properties，是否存在文件，第一次进来肯定是不存在的
    auto file_contents = ReadFile(persistent_property_filename);
    if (!file_contents.ok()) {
        return Error() << "Unable to read persistent property file: " << file_contents.error();
    }
    return *file_contents;
}
```



##### 2.3.2.2.3LoadLegacyPersistentProperties

因为第一次的时候文件`/data/property/persistent_properties`是不存在的，所以会走到这里，这里的遍历`/data/property`文件夹，实际上是为了兼容以前的`api`，现在的`api`只有一个`persistent_properties`，以前是一个个类似`persist.a.b.c`这样的一个文件组成的`/data/property`文件夹。

```c++
//system/core/init/persistent_properties.cpp
//这里的kLegacyPersistentPropertyDir是/data/property
Result<PersistentProperties> LoadLegacyPersistentProperties() {
    //打开对应的/data/property文件夹
    std::unique_ptr<DIR, decltype(&closedir)> dir(opendir(kLegacyPersistentPropertyDir), closedir);

    PersistentProperties persistent_properties;
    dirent* entry;
    //开始遍历文件夹中的文件，是否以persist开头
    while ((entry = readdir(dir.get())) != nullptr) {
        if (!StartsWith(entry->d_name, "persist.")) {
            continue;
        }
        ...
        std::string value;
        //存在的话，那么将该文件中的key，value写入到对应的persistent_properties中
        if (ReadFdToString(fd, &value)) {
            AddPersistentProperty(entry->d_name, value, &persistent_properties);
        } else {
            PLOG(ERROR) << "Unable to read persistent property file " << entry->d_name;
        }
    }
    return persistent_properties;
}
```



##### 2.3.2.2.4WritePersistentPropertyFile

```c++
//system/core/init/persistent_properties.cpp
Result<void> WritePersistentPropertyFile(const PersistentProperties& persistent_properties) {
    const std::string temp_filename = persistent_property_filename + ".tmp";
    unique_fd fd(TEMP_FAILURE_RETRY(
        open(temp_filename.c_str(), O_WRONLY | O_CREAT | O_NOFOLLOW | O_TRUNC | O_CLOEXEC, 0600)));
    if (fd == -1) {
        return ErrnoError() << "Could not open temporary properties file";
    }
    std::string serialized_string;
    if (!persistent_properties.SerializeToString(&serialized_string)) {
        return Error() << "Unable to serialize properties";
    }
    if (!WriteStringToFd(serialized_string, fd)) {
        return ErrnoError() << "Unable to write file contents";
    }
    fsync(fd);
    fd.reset();
    //将/data/property/persistent_properties.tmp文件更名为/data/property/persistent_properties
    if (rename(temp_filename.c_str(), persistent_property_filename.c_str())) {
        ...
    }
    //文件的目录/data/property
    auto dir = Dirname(persistent_property_filename);
    //判断是否存在这个目录
    auto dir_fd = unique_fd{open(dir.c_str(), O_DIRECTORY | O_RDONLY | O_CLOEXEC)};
    if (dir_fd < 0) {
        return ErrnoError() << "Unable to open persistent properties directory for fsync()";
    }
    fsync(dir_fd);

    return {};
}
```

具体的流程如下

{{< image classes="fancybox center fig-100" src="/property/property_18.png" thumbnail="/property/property_18.png" title="">}}

#### 2.3.2.3InitPropertySet

```c++
//system/core/init/property_service.cpp
uint32_t InitPropertySet(const std::string& name, const std::string& value) {
    uint32_t result = 0;
    ucred cr = {.pid = 1, .uid = 0, .gid = 0};
    std::string error;
    result = HandlePropertySet(name, value, kInitContext, cr, nullptr, &error);
    if (result != PROP_SUCCESS) {
        LOG(ERROR) << "Init cannot set '" << name << "' to '" << value << "': " << error;
    }

    return result;
}
```

这里的`InitPropertySet`函数只增加了`cr`的参数，使得`cr`为第一个进程，且`uid`，`gid`都为0

```c++
uint32_t HandlePropertySet(const std::string& name, const std::string& value,
                           const std::string& source_context, const ucred& cr,
                           SocketConnection* socket, std::string* error) {
    //检查name，value，source_context三者的权限性
    if (auto ret = CheckPermissions(name, value, source_context, cr, error); ret != PROP_SUCCESS) {
        return ret;
    }
    //如果是ctl的属性，那么就调用到SendControlMessage去
    if (StartsWith(name, "ctl.")) {
        return SendControlMessage(name.c_str() + 4, value, cr.pid, socket, error);
    }
    //这个sys.powerctl使用与关机属性的
    if (name == "sys.powerctl") {
        std::string cmdline_path = StringPrintf("proc/%d/cmdline", cr.pid);
        std::string process_cmdline;
        std::string process_log_string;
        if (ReadFileToString(cmdline_path, &process_cmdline)) {
            ...
        }
        if (value == "reboot,userspace" && !is_userspace_reboot_supported().value_or(false)) {
            ...
        }
    }

    //如果init以外的进程正在写入一个非空值，这意味着该进程正在请求init在'value'指定的路径上执行restorecon操作
    //kRestoreconProperty为"selinux.restorecon_recursive"
    if (name == kRestoreconProperty && cr.pid != 1 && !value.empty()) {
        static AsyncRestorecon async_restorecon;
        async_restorecon.TriggerRestorecon(value);
        return PROP_SUCCESS;
    }
    //检验完毕之后，最终会调用到PropertySet函数中
    return PropertySet(name, value, error);
}
```

`PropertySet`

属性设置生效的地方

```c++
//system/core/init/property_service.cpp
static uint32_t PropertySet(const std::string& name, const std::string& value, std::string* error) {
    size_t valuelen = value.size();

    if (!IsLegalPropertyName(name)) {
        *error = "Illegal property name";
        return PROP_ERROR_INVALID_NAME;
    }

    if (auto result = IsLegalPropertyValue(name, value); !result.ok()) {
        *error = result.error().message();
        return PROP_ERROR_INVALID_VALUE;
    }
    //这里的属性设置，是通过name找到对应的prop_info
    prop_info* pi = (prop_info*) __system_property_find(name.c_str());
    if (pi != nullptr) {
        //这里是限制ro属性第二次写入的操作
        if (StartsWith(name, "ro.")) {
            *error = "Read-only property was already set";
            return PROP_ERROR_READ_ONLY_PROPERTY;
        }
        //如果找到的是原本存在的name，直接更新对应的value即可
        __system_property_update(pi, value.c_str(), valuelen);
    } else {
        //如果没有找到对应的name，name直接添加属性，ro属性第一次写入也是这里添加进去的
        int rc = __system_property_add(name.c_str(), name.size(), value.c_str(), valuelen);
        if (rc < 0) {
            *error = "__system_property_add failed";
            return PROP_ERROR_SET_FAILED;
        }
    }
    //能够调用到这里面，需要persistent_properties_loaded为true
    //在2.3.2HandleInitSocket中完成文件导入和设置就会调用到这里
    //也就是说默认的persist属性已经导入进去了，只有自定义的persist属性才会被再次写到persist的文件中
    if (persistent_properties_loaded && StartsWith(name, "persist.")) {
        WritePersistentProperty(name, value);
    }
    //这里的accept_messages是在2.1StartSendingMessages初始化的时候就打开了，用于rc文件中的属性触发
    auto lock = std::lock_guard{accept_messages_lock};
    if (accept_messages) {
        PropertyChanged(name, value);
    }
    return PROP_SUCCESS;
}
```

属性设置的流程如下所示

{{< image classes="fancybox center fig-100" src="/property/property_20.png" thumbnail="/property/property_20.png" title="">}}

##### 2.3.2.3.1__system_property_find

```c++
//bionic/libc/bionic/system_property_api.cpp
__BIONIC_WEAK_FOR_NATIVE_BRIDGE
const prop_info* __system_property_find(const char* name) {
  return system_properties.Find(name);
}
```

`system_properties.Find`

```c++
//bionic/libc/system_properties/system_properties.cpp
const prop_info* SystemProperties::Find(const char* name) {
  //如果没有初始化过，直接返回空指针，这个在1.2中SystemProperties::AreaInit已经初始化  
  if (!initialized_) {
    return nullptr;
  }
  //通过属性名字，找到name对应的SElinux安全上下文
  prop_area* pa = contexts_->GetPropAreaForName(name);
  if (!pa) {
    async_safe_format_log(ANDROID_LOG_ERROR, "libc", "Access denied finding property \"%s\"", name);
    return nullptr;
  }
  //SElinux安全上下文，那么查找这个域中完整的路径
  return pa->find(name);
}
```

1)`contexts_->GetPropAreaForName`

```c++
//bionic/libc/system_properties/contexts_serialized.cpp
prop_area* ContextsSerialized::GetPropAreaForName(const char* name) {
  uint32_t index;
  //从私有匿名区域查找name对应的SElinux安全上下文的ContextNode的编号
  property_info_area_file_->GetPropertyInfoIndexes(name, &index, nullptr);
  ...
  auto* context_node = &context_nodes_[index];
  if (!context_node->pa()) {
    //不检查 no_access_，因为与foreach()的情况不同，希望为该函数中的每个非允许属性访问生成一个selinux
    context_node->Open(false, nullptr);
  }
  //返回对应当前ContextNode指向的pa指针，也就是指向对应的SElinux安全上下文文件  
  return context_node->pa();
}
```

`GetPropertyInfoIndexes`

```c++
//system/core/property_service/libpropertyinfoparser/property_info_parser.cpp
void PropertyInfoArea::GetPropertyInfoIndexes(const char* name, uint32_t* context_index,
                                              uint32_t* type_index) const {
  uint32_t return_context_index = ~0u;
  uint32_t return_type_index = ~0u;
  const char* remaining_name = name;
  //这里的root_node不是prop_bt中的root_node(),这里指的是PropertyInfoAreaHeader中的root_offset
  //对应的还是/dev/__properties__/property_info 
  auto trie_node = root_node();
  //通过分隔符点来循环遍历，直到找到对应的index
  while (true) {
    const char* sep = strchr(remaining_name, '.');

    // Apply prefix match for prefix deliminated with '.'
    if (trie_node.context_index() != ~0u) {
      return_context_index = trie_node.context_index();
    }
    if (trie_node.type_index() != ~0u) {
      return_type_index = trie_node.type_index();
    }

    // Check prefixes at this node.  This comes after the node check since these prefixes are by
    // definition longer than the node itself.
    CheckPrefixMatch(remaining_name, trie_node, &return_context_index, &return_type_index);

    if (sep == nullptr) {
      break;
    }

    const uint32_t substr_size = sep - remaining_name;
    TrieNode child_node;
    if (!trie_node.FindChildForString(remaining_name, substr_size, &child_node)) {
      break;
    }

    trie_node = child_node;
    remaining_name = sep + 1;
  }

  // We've made it to a leaf node, so check contents and return appropriately.
  // Check exact matches
  for (uint32_t i = 0; i < trie_node.num_exact_matches(); ++i) {
    if (!strcmp(c_string(trie_node.exact_match(i)->name_offset), remaining_name)) {
      if (context_index != nullptr) {
        if (trie_node.exact_match(i)->context_index != ~0u) {
          *context_index = trie_node.exact_match(i)->context_index;
        } else {
          *context_index = return_context_index;
        }
      }
      if (type_index != nullptr) {
        if (trie_node.exact_match(i)->type_index != ~0u) {
          *type_index = trie_node.exact_match(i)->type_index;
        } else {
          *type_index = return_type_index;
        }
      }
      return;
    }
  }
  // Check prefix matches for prefixes not deliminated with '.'
  CheckPrefixMatch(remaining_name, trie_node, &return_context_index, &return_type_index);
  // Return previously found prefix match.
  if (context_index != nullptr) *context_index = return_context_index;
  if (type_index != nullptr) *type_index = return_type_index;
  return;
}
```

通过把属性`name`拆分出来分成若干个结点来找到对应的`index`，如果是`ro.boot.hardware.reversion`，那么就会通过一个个结点最终找到`reversion`这个结点对应序列化保存的`index`。

{{< image classes="fancybox center fig-100" src="/property/property_6.png" thumbnail="/property/property_6.png" title="">}}

2)`pa->find`

找到`index`之后，开始找对应的`prop_info`，对应的数据结构如下

{{< image classes="fancybox center fig-100" src="/property/property_16.png" thumbnail="/property/property_16.png" title="">}}

```c++
//bionic/libc/system_properties/prop_area.cpp
const prop_info* prop_area::find(const char* name) {
  //这里的root_node()才是prop_bt中的函数
  return find_property(root_node(), name, strlen(name), nullptr, 0, false);
}
```

`find_property`

```c++
//bionic/libc/system_properties/prop_area.cpp
const prop_info* prop_area::find_property(prop_bt* const trie, const char* name, uint32_t namelen,
                                          const char* value, uint32_t valuelen,
                                          bool alloc_if_needed) {
  if (!trie) return nullptr;

  const char* remaining_name = name;
  prop_bt* current = trie;
  while (true) {
    const char* sep = strchr(remaining_name, '.');
    const bool want_subtree = (sep != nullptr);
    //通过点分割后的字符串和未分割的字符串的长度相差，来判断是否是最后一个结点  
    const uint32_t substr_size = (want_subtree) ? sep - remaining_name : strlen(remaining_name);

    if (!substr_size) {
      return nullptr;
    }

    prop_bt* root = nullptr;
    uint_least32_t children_offset = atomic_load_explicit(&current->children, memory_order_relaxed);
    if (children_offset != 0) {
      //查找到子结点  
      root = to_prop_bt(&current->children);
    } else if (alloc_if_needed) {
      uint_least32_t new_offset;
      root = new_prop_bt(remaining_name, substr_size, &new_offset);
      if (root) {
        atomic_store_explicit(&current->children, new_offset, memory_order_release);
      }
    }

    if (!root) {
      return nullptr;
    }
    //向左向右查找，根据字符串的字母顺序来排列
    current = find_prop_bt(root, remaining_name, substr_size, alloc_if_needed);
    if (!current) {
      return nullptr;
    }

    if (!want_subtree) break;

    remaining_name = sep + 1;
  }
  //退出，说明name只剩下最后一个结点
  //尝试查找对应的prop_info值
  uint_least32_t prop_offset = atomic_load_explicit(&current->prop, memory_order_relaxed);
  if (prop_offset != 0) {
    //找到就返回对应的prop_info
    return to_prop_info(&current->prop);
  } else if (alloc_if_needed) {
    uint_least32_t new_offset;
    prop_info* new_info = new_prop_info(name, namelen, value, valuelen, &new_offset);
    if (new_info) {
      atomic_store_explicit(&current->prop, new_offset, memory_order_release);
    }
    return new_info;
  } else {
    return nullptr;
  }
}
```

如果查找`ro.boot.dynamic_partitions`，可以直接在下图中查找到，最后返回`ro.boot.dynamic_partitions`对应的`prop_info`

{{< image classes="fancybox center fig-100" src="/property/property_22.png" thumbnail="/property/property_22.png" title="">}}

##### 2.3.2.3.2__system_property_update

```c++
//bionic/libc/bionic/system_property_api.cpp
__BIONIC_WEAK_FOR_NATIVE_BRIDGE
int __system_property_update(prop_info* pi, const char* value, unsigned int len) {
  return system_properties.Update(pi, value, len);
}
```

`system_properties.Update`

```c++
//bionic/libc/system_properties/system_properties.cpp
int SystemProperties::Update(prop_info* pi, const char* value, unsigned int len) {
  ...
  //新增了一个serial_pa的prop_area安全上下文，对应文件名是/dev/__properties__/properties_serial  
  prop_area* serial_pa = contexts_->GetSerialPropArea();
  if (!serial_pa) {
    return -1;
  }
  //同前面一样  
  prop_area* pa = contexts_->GetPropAreaForName(pi->name);
  //找到prop_info中的serial，这里的组成是 value长度(8位) + 更新或者添加*2(24位)
  uint32_t serial = atomic_load_explicit(&pi->serial, memory_order_relaxed);
  //SERIAL_VALUE_LEN这个操作是右移运算，右移32位，得到value的长度
  unsigned int old_len = SERIAL_VALUE_LEN(serial);
  //只要设置了脏位，脏备份区域中就有一个未损坏的预脏值副本，确保在看到脏序列之前发布脏区更新
  memcpy(pa->dirty_backup_area(), pi->value, old_len + 1);
  atomic_thread_fence(memory_order_release);
  serial |= 1;
  //更新value的前，序列号为奇数  
  atomic_store_explicit(&pi->serial, serial, memory_order_relaxed);
  //更新value的值  
  strlcpy(pi->value, value, len + 1);
  atomic_thread_fence(memory_order_release);
  //serial,这里的prop_info中的serial，前8位是value的长度，后24位为序列数，如果开始更新则为奇数，更新完成则为偶数
  atomic_store_explicit(&pi->serial, (len << 24) | ((serial + 1) & 0xffffff), memory_order_relaxed);
  __futex_wake(&pi->serial, INT32_MAX);  // Fence by side effect
  //只要有操作update的函数，就会对serial_pa中的serial_加1
  atomic_store_explicit(serial_pa->serial(),
                        atomic_load_explicit(serial_pa->serial(), memory_order_relaxed) + 1,
                        memory_order_release);
  // __futex_wake以及serial值用于控制更新操作的并发进行。
  __futex_wake(serial_pa->serial(), INT32_MAX);

  return 0;
}
```

每次`Update`的时候，会更新几处地方

1. `prop_info`的`value`会更新
2. `prop_info`的`serial`会更新，前8位是新`value`的长度，后24位记录数值，这个数值是被修改次数的2倍，如果为奇数，说明这个`value`正在写入`prop_info`中
3. 会更新`/dev/__properties__/properties_serial`所在的`prop_area`中的`serial_`值，目前有两种情况会变化，`Update`和`Add`的时候，都会增加1。

下面举例说明，在总共只有三个属性的情况下，对于`aaudio.hw_burst_min_usec`这个属性中，第一次更新了一次属性，那么`prop_info`中的`serial`为`0x04000004`(前8位的4是`value`的长度4，后32位的4指的是(`Add`+`Update`)*2)，并且`/dev/__properties__/properties_serial`所在的`prop_area`中的`serial_`值为4。

{{< image classes="fancybox center fig-100" src="/property/property_36.png" thumbnail="/property/property_36.png" title="">}}

> 补充关于Android属性中的脏区，**主要为了防止多线程时候发生读写错误**。
>
> 脏区在属性系统中只有两处地方调用和一处地方定义。
>
> 1）一处读取
>
> ```c++
> //bionic/libc/system_properties/system_properties.cpp
> uint32_t SystemProperties::ReadMutablePropertyValue(const prop_info* pi, char* value) {
>   //获取prop_info中的serial  
>   uint32_t new_serial = load_const_atomic(&pi->serial, memory_order_acquire);
>   uint32_t serial;
>   unsigned int len;
>   for (;;) {
>     serial = new_serial;
>     //获取前8位的值
>     len = SERIAL_VALUE_LEN(serial);
>     //如果serial为奇数，那么读取脏区内容拷贝到value变量，反之直接拷贝prop_info中的value值到value变量  
>     if (__predict_false(SERIAL_DIRTY(serial))) {
>       prop_area* pa = contexts_->GetPropAreaForName(pi->name);
>       memcpy(value, pa->dirty_backup_area(), len + 1);
>     } else {
>       memcpy(value, pi->value, len + 1);
>     }
>     atomic_thread_fence(memory_order_acquire);
>     new_serial = load_const_atomic(&pi->serial, memory_order_relaxed);
>     if (__predict_true(serial == new_serial)) {
>       break;
>     }
>     atomic_thread_fence(memory_order_acquire);
>   }
>   return serial;
> }
> ```
>
> 2）一处写入
>
> 详见上述`SystemProperties::Update`，每次更新都会调用。
>
> ```c++
> memcpy(pa->dirty_backup_area(), pi->value, old_len + 1);
> ```
>
> 3）一处声明
>
> ```c++
> //bionic/libc/system_properties/include/system_properties/prop_area.h
> char* dirty_backup_area() {
>     return data_ + sizeof (prop_bt);
> }
> ```
>
> 关于`memcpy`如果第三个参数小于目标区域，正常把内容拷贝；如果第三个参数大于目前区域，会发生`memcpy`内存越界，造成段错误。这里所谓的"**脏区**"，就是`prop_bt`后面的空间。可以在`map_prop_area_rw`中知道，每一个`prop_area`定义的共享内存大小都为128k，所以**拷贝到脏区不会内存越界**。因此，保证传入的`value`大小小于92长度，必定能保证目前区域有足够的空间。
>
> ```c++
> //bionic/libc/system_properties/prop_area.cpp
> prop_area* prop_area::map_prop_area_rw(const char* filename, const char* context,
>                                        bool* fsetxattr_failed) {
>   const int fd = open(filename, O_RDWR | O_CREAT | O_NOFOLLOW | O_CLOEXEC | O_EXCL, 0444);
>   ...
>   //PA_SIZE大小为128k，sizeof(prop_area)为124字节，pa_data_size_大小为128k-124
>   pa_size_ = PA_SIZE;
>   pa_data_size_ = pa_size_ - sizeof(prop_area);
> 
>   void* const memory_area = mmap(nullptr, pa_size_, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
>   if (memory_area == MAP_FAILED) {
>     close(fd);
>     return nullptr;
>   }
> 
>   prop_area* pa = new (memory_area) prop_area(PROP_AREA_MAGIC, PROP_AREA_VERSION);
> 
>   close(fd);
>   return pa;
> }
> ```



##### 2.3.2.3.3__system_property_add

```c++
//bionic/libc/bionic/system_property_api.cpp
__BIONIC_WEAK_FOR_NATIVE_BRIDGE
int __system_property_add(const char* name, unsigned int namelen, const char* value,
                          unsigned int valuelen) {
  return system_properties.Add(name, namelen, value, valuelen);
}
```

`system_properties.Add`

```c++
//bionic/libc/system_properties/system_properties.cpp
int SystemProperties::Add(const char* name, unsigned int namelen, const char* value,
                          unsigned int valuelen) {
  ...
  //新增了一个serial_pa的prop_area安全上下文，对应文件名是/dev/__properties__/properties_serial    
  prop_area* serial_pa = contexts_->GetSerialPropArea();
  if (serial_pa == nullptr) {
    return -1;
  }
  //这里和前面一模一样，重新找到对应的prop_area安全上下文
  prop_area* pa = contexts_->GetPropAreaForName(name);
  if (!pa) {
    async_safe_format_log(ANDROID_LOG_ERROR, "libc", "Access denied adding property \"%s\"", name);
    return -1;
  }
  //在这个安全上下文文件中添加对应的属性
  bool ret = pa->add(name, namelen, value, valuelen);
  if (!ret) {
    return -1;
  }

  //添加一个新属性，会使这个serial_pa对应的序列增加1
  atomic_store_explicit(serial_pa->serial(),
                        atomic_load_explicit(serial_pa->serial(), memory_order_relaxed) + 1,
                        memory_order_release);
  __futex_wake(serial_pa->serial(), INT32_MAX);
  return 0;
}
```

`pa->add`

```c++
//bionic/libc/system_properties/prop_area.cpp
//这里的添加操作和find方式区别就是，最后一个参数，最后一个参数找不到可以添加
bool prop_area::add(const char* name, unsigned int namelen, const char* value,
                    unsigned int valuelen) {
  return find_property(root_node(), name, namelen, value, valuelen, true);
}
```

比如新增加一个属性`ro.boot.serialno`为`"1234567890"`，那么会更新对应的`prop_info`的`name`和`value`值，并且对应`/dev/__properties__/properties_serial`所在的`prop_area`中的`serial_`值会变成5。

{{< image classes="fancybox center fig-100" src="/property/property_24.png" thumbnail="/property/property_24.png" title="">}}

##### 2.3.2.3.4WritePersistentProperty

```c++
//system/core/init/persistent_properties.cpp
void WritePersistentProperty(const std::string& name, const std::string& value) {
    //先找/data/property/persistent_properties，是否存在这个文件，存在这个文件那么直接导入
    auto persistent_properties = LoadPersistentPropertyFile();

    if (!persistent_properties.ok()) {
        LOG(ERROR) << "Recovering persistent properties from memory: "
                   << persistent_properties.error();
        //没有这个文件从私有匿名内存中找，并且把key，value导入到对应序列化格式中persistent_properties
        persistent_properties = LoadPersistentPropertiesFromMemory();
    }
    //在序列化格式中查找是否存在name
    auto it = std::find_if(persistent_properties->mutable_properties()->begin(),
                           persistent_properties->mutable_properties()->end(),
                           [&name](const auto& record) { return record.name() == name; });
    //如果存在，更新value值
    if (it != persistent_properties->mutable_properties()->end()) {
        it->set_name(name);
        it->set_value(value);
    //不存在的话新添加到序列化格式中
    } else {
        AddPersistentProperty(name, value, &persistent_properties.value());
    }
    //将序列化格式写入到对应的文件中，就是重新写入到/data/property/persistent_properties文件中
    if (auto result = WritePersistentPropertyFile(*persistent_properties); !result.ok()) {
        LOG(ERROR) << "Could not store persistent property: " << result.error();
    }
}
```

`LoadPersistentPropertiesFromMemory`

```c++
//system/core/init/persistent_properties.cpp
//相对来说这是一个比较高阶的用法，里面存在两个匿名的函数指针
//简单说一下值的传递6->5->4->3->2->1
PersistentProperties LoadPersistentPropertiesFromMemory() {
    PersistentProperties persistent_properties;
    __system_property_foreach(
        //这里的cookie实际上就是persistent_properties，设定为2
        [](const prop_info* pi, void* cookie) {
            __system_property_read_callback(
                pi,
                //这里cookie实际上是__system_property_read_callback中第三个参数，设定为4
                [](void* cookie, const char* name, const char* value, unsigned serial) {
                    if (StartsWith(name, "persist.")) {
                        //这里的properties是__system_property_read_callback中的匿名函数指针参数cookie，设定为5
                        auto properties = reinterpret_cast<PersistentProperties*>(cookie);
                        //这里的properties的修改，会对应影响到上面cookie，设定为6
                        AddPersistentProperty(name, value, properties);
                    }
                },
                //这里的cookie实际上是前面__system_property_foreach的匿名函数指针的参数，设定为3
                cookie);
        },
        //这个persistent_properties值是传入的原始值，设定为1
        &persistent_properties);
    return persistent_properties;
}
```

这个函数，实际上是第一次导入`persist`文件内容的函数。读取私有匿名区域中的`ContextNodes`，**遍历所有`ContextNode`中指向的`SElinux`安全上下文文件的`prop_area`所形成的的字典树**。即这里是开始创建`/data/property/persistent_properties`的函数。

{{< image classes="fancybox center fig-100" src="/property/property_19.png" thumbnail="/property/property_19.png" title="">}}

# 3获取属性

这里包括非`init`进程和`init`进程获取属性，这里非`init`进程有两种方式，一种是在`App`调用`Java层`或者`Native层`调用获取属性的方式，另一种是直接`adb`敲命令`getprop`。

## 3.1App读取属性流程分析（Java侧）

```java
//App读取属性
//除了获取字符串属性，还有getInt、getLong、getBoolean，都可以用反射来获取，这里只说明获取字符串属性
import android.os.SystemProperties;
public static String getProperty(String key, String defaultValue) {
    String value = defaultValue;
    try {
        Class<?> c = Class.forName("android.os.SystemProperties");
        Method get = c.getMethod("get", String.class, String.class);
        value = (String) (get.invoke(c, key, defaultValue));
    } catch (Exception e) {
        e.printStackTrace();
    }
    return value;
}

String strValue = getProperty("persist.a.b.c", "0");
```

### 3.1.1`SystemProperties.get`

```java
//frameworks/base/core/java/android/os/SystemProperties.java
//这四个对外提供的接口都是隐藏的，通常需要用反射来调用
//onKeyAccess是用于检查查询属性名的合法性
public static String get(@NonNull String key, @Nullable String def) {
    if (TRACK_KEY_ACCESS) onKeyAccess(key);
    return native_get(key, def);
}

public static int getInt(@NonNull String key, int def) {
    if (TRACK_KEY_ACCESS) onKeyAccess(key);
    return native_get_int(key, def);
}

public static long getLong(@NonNull String key, long def) {
    if (TRACK_KEY_ACCESS) onKeyAccess(key);
    return native_get_long(key, def);
}

public static boolean getBoolean(@NonNull String key, boolean def) {
    if (TRACK_KEY_ACCESS) onKeyAccess(key);
    return native_get_boolean(key, def);
}   
```

### 3.1.2`native_get`

```c++
//frameworks\base\core\jni\android_os_SystemProperties.cpp
int register_android_os_SystemProperties(JNIEnv *env)
{
    const JNINativeMethod method_table[] = {
        //字符串就是对应SystemProperties_getSS这个函数名
        { "native_get",
          "(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;",
          (void*) SystemProperties_getSS },
        { "native_get_int", "(Ljava/lang/String;I)I",
          (void*) SystemProperties_get_integral<jint> },
        { "native_get_long", "(Ljava/lang/String;J)J",
          (void*) SystemProperties_get_integral<jlong> },
        { "native_get_boolean", "(Ljava/lang/String;Z)Z",
          (void*) SystemProperties_get_boolean },
        { "native_find",
          "(Ljava/lang/String;)J",
          (void*) SystemProperties_find },
        ...
    };
    return RegisterMethodsOrDie(env, "android/os/SystemProperties",
                                method_table, NELEM(method_table));
}
//最终根据传入不同的函数指针来完成不同类型的属性获取
template<typename Functor>
void ReadProperty(JNIEnv* env, jstring keyJ, Functor&& functor)
{
    ScopedUtfChars key(env, keyJ);
    if (!key.c_str()) {
        return;
    }
//下面这个__BIONIC__，在一开始编译bionic的时候就已经编译进去了    
#if defined(__BIONIC__)
    //1.寻找到属性名对应的Selinux的安全上下文名
    const prop_info* prop = __system_property_find(key.c_str());
    if (!prop) {
        return;
    }
    ReadProperty(prop, std::forward<Functor>(functor));
#else
    std::forward<Functor>(functor)(
        android::base::GetProperty(key.c_str(), "").c_str());
#endif
}

template<typename Functor>
void ReadProperty(const prop_info* prop, Functor&& functor)
{
#if defined(__BIONIC__)
    auto thunk = [](void* cookie,
                    const char* /*name*/,
                    const char* value,
                    uint32_t /*serial*/) {
        std::forward<Functor>(*static_cast<Functor*>(cookie))(value);
    };
    //2.进行属性名的读取
    __system_property_read_callback(prop, thunk, &functor);
#else
    LOG(FATAL) << "fast property access supported only on device";
#endif
}

//这个是字符串的属性获取
jstring SystemProperties_getSS(JNIEnv* env, jclass clazz, jstring keyJ,
                               jstring defJ)
{
    jstring ret = defJ;
    ReadProperty(env, keyJ, [&](const char* value) {
        if (value[0]) {
            ret = env->NewStringUTF(value);
}
    });
    if (ret == nullptr && !env->ExceptionCheck()) {
      ret = env->NewStringUTF("");  // Legacy behavior
    }
    return ret;
}
//这个是Int和Long的属性获取
template <typename T>
T SystemProperties_get_integral(JNIEnv *env, jclass, jstring keyJ,
                                       T defJ)
{
    T ret = defJ;
    ReadProperty(env, keyJ, [&](const char* value) {
        android::base::ParseInt<T>(value, &ret);
    });
    return ret;
}
//这个是boolean的属性获取
jboolean SystemProperties_get_boolean(JNIEnv *env, jclass, jstring keyJ,
                                      jboolean defJ)
{
    ParseBoolResult parseResult = ParseBoolResult::kError;
    ReadProperty(env, keyJ, [&](const char* value) {
        parseResult = android::base::ParseBool(value);
    });
    return jbooleanFromParseBoolResult(parseResult, defJ);
}
```

上面的逻辑就是如果定义了`__BIONIC__`，那么直接就是先调用`__system_property_find`，后调用`__system_property_read_callback`。没有定义`__BIONIC__`的话，只能调用`GetProperty`。

```c++
//bionic/tools/versioner/current/sys/cdefs.h
#define __BIONIC__ 1	//默认已经定义
```

#### 3.1.2.1`__system_property_find`

```c++
//bionic/libc/bionic/system_property_api.cpp
__BIONIC_WEAK_FOR_NATIVE_BRIDGE
const prop_info* __system_property_find(const char* name) {
  return system_properties.Find(name);
}
```

`system_properties.Find`

```c++
//bionic/libc/system_properties/system_properties.cpp
const prop_info* SystemProperties::Find(const char* name) {
  if (!initialized_) {
    return nullptr;
  }

  prop_area* pa = contexts_->GetPropAreaForName(name);
  if (!pa) {
    async_safe_format_log(ANDROID_LOG_ERROR, "libc", "Access denied finding property \"%s\"", name);
    return nullptr;
  }

  return pa->find(name);
}
```

#### 3.1.2.2`__system_property_read_callback`

```c++
//bionic/libc/bionic/system_property_api.cpp
__BIONIC_WEAK_FOR_NATIVE_BRIDGE
void __system_property_read_callback(const prop_info* pi,
                                     void (*callback)(void* cookie, const char* name,
                                                      const char* value, uint32_t serial),
                                     void* cookie) {
  return system_properties.ReadCallback(pi, callback, cookie);
}
```

`system_properties.ReadCallback`

```c++
//bionic/libc/system_properties/system_properties.cpp
void SystemProperties::ReadCallback(const prop_info* pi,
                                    void (*callback)(void* cookie, const char* name,
                                                     const char* value, uint32_t serial),
                                    void* cookie) {
  ...
  char value_buf[PROP_VALUE_MAX];
  uint32_t serial = ReadMutablePropertyValue(pi, value_buf);
  callback(cookie, pi->name, value_buf, serial);
  }

uint32_t SystemProperties::ReadMutablePropertyValue(const prop_info* pi, char* value) {
  uint32_t new_serial = load_const_atomic(&pi->serial, memory_order_acquire);
  uint32_t serial;
  unsigned int len;
  for (;;) {
    serial = new_serial;
    len = SERIAL_VALUE_LEN(serial);
    if (__predict_false(SERIAL_DIRTY(serial))) {
      ...
    } else {
      memcpy(value, pi->value, len + 1);
  }
    atomic_thread_fence(memory_order_acquire);
    new_serial = load_const_atomic(&pi->serial, memory_order_relaxed);
    if (__predict_true(serial == new_serial)) {
      break;
      }
    atomic_thread_fence(memory_order_acquire);
    }
  return serial;
}
```

{{< image classes="fancybox center fig-100" src="/property/property_27.png" thumbnail="/property/property_27.png" title="">}}

## 3.2App读取属性流程分析（native侧）

```c++
//App通过封装的so进行读取属性
#include <cutils/properties.h>
#define PROPERTY_KEY_MAX   32
#define PROPERTY_VALUE_MAX  92
char *value= new char[PROPERTY_VALUE_MAX];
property_get("persist.a.b.c", value, "0");
property_get_int32("persist.a.b.c.int", 0);
property_get_int64("persist.a.b.c.long", 0L);
property_get_bool("persist.a.b.c.boolean", false);
```

### 3.2.1`property_get`

```c++
//system/core/libcutils/properties.cpp
//获取bool类型的属性，其实也是间接调用property_get，然后自己在封装成bool类型
int8_t property_get_bool(const char *key, int8_t default_value) {
    if (!key) {
        return default_value;
  }

    int8_t result = default_value;
    char buf[PROPERTY_VALUE_MAX] = {'\0'};

    int len = property_get(key, buf, "");
    if (len == 1) {
        char ch = buf[0];
        if (ch == '0' || ch == 'n') {
            result = false;
        } else if (ch == '1' || ch == 'y') {
            result = true;
        }
    } else if (len > 1) {
        if (!strcmp(buf, "no") || !strcmp(buf, "false") || !strcmp(buf, "off")) {
            result = false;
        } else if (!strcmp(buf, "yes") || !strcmp(buf, "true") || !strcmp(buf, "on")) {
            result = true;
  }
}

    return result;
}
//获取Long和Int类型的属性，核心的内容是property_get，然后自己在封装成Long和Int类型
static intmax_t property_get_imax(const char *key, intmax_t lower_bound, intmax_t upper_bound,
                                  intmax_t default_value) {
    ...
    int len = property_get(key, buf, "");
    if (len > 0) {
        int tmp = errno;
        errno = 0;

        // Infer base automatically
        result = strtoimax(buf, &end, /*base*/ 0);
        if ((result == INTMAX_MIN || result == INTMAX_MAX) && errno == ERANGE) {
            // Over or underflow
            result = default_value;
            ALOGV("%s(%s,%" PRIdMAX ") - overflow", __FUNCTION__, key, default_value);
        } else if (result < lower_bound || result > upper_bound) {
            // Out of range of requested bounds
            result = default_value;
            ALOGV("%s(%s,%" PRIdMAX ") - out of range", __FUNCTION__, key, default_value);
        } else if (end == buf) {
            // Numeric conversion failed
            result = default_value;
            ALOGV("%s(%s,%" PRIdMAX ") - numeric conversion failed", __FUNCTION__, key,
                  default_value);
        }

        errno = tmp;
  }

    return result;
  }

//获取Long类型的属性
int64_t property_get_int64(const char *key, int64_t default_value) {
    return (int64_t)property_get_imax(key, INT64_MIN, INT64_MAX, default_value);
}
//获取Int类型的属性
int32_t property_get_int32(const char *key, int32_t default_value) {
    return (int32_t)property_get_imax(key, INT32_MIN, INT32_MAX, default_value);
}
//获取字符串类型的属性
int property_get(const char *key, char *value, const char *default_value) {
    //最终调用的就是__system_property_get函数
    int len = __system_property_get(key, value);
    if (len > 0) {
        return len;
    }
    if (default_value) {
        len = strnlen(default_value, PROPERTY_VALUE_MAX - 1);
        memcpy(value, default_value, len);
        value[len] = '\0';
    }
    return len;
}
```

### 3.2.2`__system_property_get`

```c++
//bionic/libc/bionic/system_property_api.cpp
 __BIONIC_WEAK_FOR_NATIVE_BRIDGE
int __system_property_get(const char* name, char* value) {
    return system_properties.Get(name, value);
}
```

### 3.2.3`system_properties.Get`

```c++
//bionic/libc/system_properties/system_properties.cpp
int SystemProperties::Get(const char* name, char* value) {
  const prop_info* pi = Find(name);

  if (pi != nullptr) {
    return Read(pi, nullptr, value);
  } else {
    value[0] = 0;
    return 0;
  }
  }

const prop_info* SystemProperties::Find(const char* name) {
  if (!initialized_) {
    return nullptr;
  }

  prop_area* pa = contexts_->GetPropAreaForName(name);
  if (!pa) {
    async_safe_format_log(ANDROID_LOG_ERROR, "libc", "Access denied finding property \"%s\"", name);
    return nullptr;
  }

  return pa->find(name);
  }

int SystemProperties::Read(const prop_info* pi, char* name, char* value) {
  uint32_t serial = ReadMutablePropertyValue(pi, value);
  if (name != nullptr) {
    size_t namelen = strlcpy(name, pi->name, PROP_NAME_MAX);
    if (namelen >= PROP_NAME_MAX) {
      async_safe_format_log(ANDROID_LOG_ERROR, "libc",
                            "The property name length for \"%s\" is >= %d;"
                            " please use __system_property_read_callback"
                            " to read this property. (the name is truncated to \"%s\")",
                            pi->name, PROP_NAME_MAX - 1, name);
    }
  }
  if (is_read_only(pi->name) && pi->is_long()) {
    async_safe_format_log(
        ANDROID_LOG_ERROR, "libc",
        "The property \"%s\" has a value with length %zu that is too large for"
        " __system_property_get()/__system_property_read(); use"
        " __system_property_read_callback() instead.",
        pi->name, strlen(pi->long_value()));
  }
  return SERIAL_VALUE_LEN(serial);
  }

uint32_t SystemProperties::ReadMutablePropertyValue(const prop_info* pi, char* value) {
  // We assume the memcpy below gets serialized by the acquire fence.
  uint32_t new_serial = load_const_atomic(&pi->serial, memory_order_acquire);
  uint32_t serial;
  unsigned int len;
  for (;;) {
    serial = new_serial;
    len = SERIAL_VALUE_LEN(serial);
    if (__predict_false(SERIAL_DIRTY(serial))) {
      // See the comment in the prop_area constructor.
      prop_area* pa = contexts_->GetPropAreaForName(pi->name);
      memcpy(value, pa->dirty_backup_area(), len + 1);
    } else {
      memcpy(value, pi->value, len + 1);
}
    atomic_thread_fence(memory_order_acquire);
    new_serial = load_const_atomic(&pi->serial, memory_order_relaxed);
    if (__predict_true(serial == new_serial)) {
      break;
  }
    atomic_thread_fence(memory_order_acquire);
  }
  return serial;
}
```

{{< image classes="fancybox center fig-100" src="/property/property_28.png" thumbnail="/property/property_28.png" title="">}}

> 补充：关于`libc.so`是什么时候初始化的
>
> 关于在Android中的`libc.so`，实际上是`bionic/libc`，根据以下规则来生成对应的`libc.so`
>
> ```makefile
> //bionic/libc/Android.bp
> cc_library_static {
>     name: "libc_init_dynamic",
>     defaults: ["libc_defaults"],
>     srcs: ["bionic/libc_init_dynamic.cpp"],
>     cflags: ["-fno-stack-protector"],
> }
> 
> // ========================================================
> // libc.a + libc.so
> // ========================================================
> cc_library {
>     defaults: [
>         "libc_defaults",
>         "libc_native_allocator_defaults",
>     ],
>     name: "libc",
>     static_ndk_lib: true,
>     export_include_dirs: ["include"],
>     product_variables: {
>         platform_sdk_version: {
>             asflags: ["-DPLATFORM_SDK_VERSION=%d"],
>         },
>     },
>     static: {
>         srcs: [ ":libc_sources_static" ],
>         cflags: ["-DLIBC_STATIC"],
>         whole_static_libs: [
>             "gwp_asan",
>             "libc_init_static",
>             "libc_common_static",
>             "libc_unwind_static",
>         ],
>     },
>     shared: {
>         srcs: [ ":libc_sources_shared" ],
>         whole_static_libs: [
>             "gwp_asan",
>             "libc_init_dynamic",
>             "libc_common_shared",
>         ],
>     },
>     ...
> }   
> ```
>
> 这里面实际上是`libc_init_dynamic.cpp`为`libc_init_dynamic.a`的源文件，而`libc.so`是依赖于这个`libc_init_dynamic.a`的静态库，也就是间接也依赖于`libc_init_dynamic.cpp`文件。`libc_init_dynamic.cpp`文件，是用于初始化属性系统的，用于`Api`访问。
>
> ```c++
> //bionic/libc/bionic/libc_init_dynamic.cpp
> __attribute__((constructor(1))) static void __libc_preinit() {
>   __stack_chk_guard = reinterpret_cast<uintptr_t>(__get_tls()[TLS_SLOT_STACK_GUARD]);
>   __libc_preinit_impl();
> }
> ```
>
> 关于constructor(1)特性。`constructor`参数让系统执行`main()`函数之前调用函数(被`__attribute__((constructor))`修饰的函数).同理, `destructor`让系统在`main()`函数退出或者调用了`exit()`之后,调用我们的函数.带有这些修饰属性的函数,对于我们初始化一些在程序中使用的数据非常有用。具体原理可以点击[这里](https://www.jianshu.com/p/dd425b9dc9db)。
>
> ```c++
> //bionic/libc/bionic/libc_init_dynamic.cpp
> __attribute__((noinline))
> static void __libc_preinit_impl() {
>   TlsModules& tls_modules = __libc_shared_globals()->tls_modules;
>   tls_modules.generation_libc_so = &__libc_tls_generation_copy;
>   __libc_tls_generation_copy = tls_modules.generation;
> 
>   __libc_init_globals();
>   __libc_init_common();
>   ...
>   netdClientInit();
> }
> ```
>
> 用于初始化一些通用的环境变量
>
> ```c++
> //bionic/libc/bionic/libc_init_common.cpp
> void __libc_init_common() {
>   // Initialize various globals.
>   environ = __libc_shared_globals()->init_environ;
>   errno = 0;
>   setprogname(__libc_shared_globals()->init_progname ?: "<unknown>");
> 
> #if !defined(__LP64__)
>   __check_max_thread_id();
> #endif
> 
>   __libc_add_main_thread();
> 
>   __system_properties_init(); // Requires 'environ'.
>   __libc_init_fdsan(); // Requires system properties (for debug.fdsan).
>   __libc_init_fdtrack();
> 
>   SetDefaultHeapTaggingLevel();
> }
> ```
>
> `__system_properties_init`
>
> ```c++
> //bionic/libc/bionic/system_property_api.cpp
> #define PROP_FILENAME "/dev/__properties__"
> int __system_properties_init() {
>   return system_properties.Init(PROP_FILENAME) ? 0 : -1;
> }
> ```
>
> `system_properties.Init`
>
> ```c++
> //bionic/libc/system_properties/system_properties.cpp
> bool SystemProperties::Init(const char* filename) {
>   // This is called from __libc_init_common, and should leave errno at 0 (http://b/37248982).
>   ErrnoRestorer errno_restorer;
> 
>   if (initialized_) {
>     contexts_->ResetAccess();
>     return true;
>   }
> 
>   if (strlen(filename) >= PROP_FILENAME_MAX) {
>     return false;
>   }
>   strcpy(property_filename_, filename);
>   //最重要的从这里开始
>   if (is_dir(property_filename_)) {
>     if (access("/dev/__properties__/property_info", R_OK) == 0) {
>       contexts_ = new (contexts_data_) ContextsSerialized();
>       if (!contexts_->Initialize(false, property_filename_, nullptr)) {
>         return false;
>       }
>     }
>     ...
>   }    
>   initialized_ = true;
>   return true;
> }
> ```
>
> 因为是App调用到`Init`函数，这里的`Initialize`函数所传递的都为`false`，说明是没有可写入权限的。
>
> ContextsSerialized函数，是用于实例化序列化的对象，然后调用序列化对象的`Initialize`函数。
>
> ```c++
> //bionic/libc/system_properties/contexts_serialized.cpp
> bool ContextsSerialized::Initialize(bool writable, const char* filename, bool* fsetxattr_failed) {
>   filename_ = filename;
>   if (!InitializeProperties()) {
>     return false;
>   }
> 
>   if (writable) {
>       ...
>   }
>   return true;
> }
> ```
>
> `InitializeProperties`
>
> ```c++
> //bionic/libc/system_properties/contexts_serialized.cpp
> bool ContextsSerialized::InitializeProperties() {
>   if (!property_info_area_file_.LoadDefaultPath()) {
>     return false;
>   }
> 
>   if (!InitializeContextNodes()) {
>     FreeAndUnmap();
>     return false;
>   }
> 
>   return true;
> }
> ```
>
> 这里的流程跟前面1.2中的`__system_property_area_init`非常相似，但是区别是1.2中的`__system_property_area_init`初始化是在`Init`进程初始化的，具有**可读写权限**，而这里的`__system_properties_init`初始化是在`bionic/libc`这个so库初始化的，用于其他进程的调用，所以是没有**可写权限**的。
>
> {{< image classes="fancybox center fig-100" src="/property/property_29.png" thumbnail="/property/property_29.png" title="">}}



## 3.3adb获取属性

`adb`命令直接输入`getprop`，实际上是调用到一个`toolbox`当中的进程`getprop`，有两个参数`ZT`，不能一起使用，`-Z`是打印对应的`SElinux`安全上下文文件，`-T`是打印对应的value具体的类型，比如`string`，`bool`等。

```c++
//system/core/toolbox/getprop.cpp
extern "C" int getprop_main(int argc, char** argv) {
    auto result_type = ResultType::Value;

    while (true) {
        ...
        switch (arg) {
            case 'h':
                std::cout << "usage: getprop [-TZ] [NAME [DEFAULT]]\n"
                             "\n"
                             "Gets an Android system property, or lists them all.\n"
                             "\n"
                             "-T\tShow property types instead of values\n"
                             "-Z\tShow property contexts instead of values\n"
                          << std::endl;
                return 0;
            case 'T':
                ...
                result_type = ResultType::Type;
                break;
            case 'Z':
                ...
                result_type = ResultType::Context;
                break;
            ...
    }

    if (optind >= argc) {
        //不加任何参数答应所有的属性
        PrintAllProperties(result_type);
        return 0;
    }
    ...
    PrintProperty(argv[optind], (optind == argc - 1) ? "" : argv[optind + 1], result_type);
    return 0;
}
```

`PrintProperty`

```c++
//system/core/toolbox/getprop.cpp
void PrintProperty(const char* name, const char* default_value, ResultType result_type) {
    switch (result_type) {
        case ResultType::Value:
            //实际上调用的GetProperty这个api方法
            std::cout << GetProperty(name, default_value) << std::endl;
            break;
        ...
    }
}
```

到这里的GetProperty函数，就跟App的流程类似了。

`GetProperty`

```c++
//system/core/base/properties.cpp
std::string GetProperty(const std::string& key, const std::string& default_value) {
  std::string property_value;
#if defined(__BIONIC__)
  const prop_info* pi = __system_property_find(key.c_str());
  if (pi == nullptr) return default_value;

  __system_property_read_callback(pi,
                                  [](void* cookie, const char*, const char* value, unsigned) {
                                    auto property_value = reinterpret_cast<std::string*>(cookie);
                                    *property_value = value;
                                  },
                                  &property_value);
#else
  auto it = g_properties.find(key);
  if (it == g_properties.end()) return default_value;
  property_value = it->second;
#endif
  return property_value.empty() ? default_value : property_value;
}
```

`GetProperty`函数的内部逻辑，会根据`__BIONIC__`这个宏是否定义，分两个不同的逻辑。可以看到如果该宏已定义的话，会继续调用`__system_property_find`, 如果该宏未定义，则从全局变量`g_properties`中读取。

## 3.4Init进程获取属性

```c++
//system/core/init/property_service.cpp
//实际上这里获取属性直接使用GetProperty来调用，跟adb方式一致
static void ExportKernelBootProps() {
    constexpr const char* UNSET = "";
    struct {
        const char* src_prop;
        const char* dst_prop;
        const char* default_value;
    } prop_map[] = {
            // clang-format off
        { "ro.boot.serialno",   "ro.serialno",   UNSET, },
        { "ro.boot.mode",       "ro.bootmode",   "unknown", },
        { "ro.boot.baseband",   "ro.baseband",   "unknown", },
        { "ro.boot.bootloader", "ro.bootloader", "unknown", },
        { "ro.boot.hardware",   "ro.hardware",   "unknown", },
        { "ro.boot.revision",   "ro.revision",   "0", },
            // clang-format on
    };
    for (const auto& prop : prop_map) {
        std::string value = GetProperty(prop.src_prop, prop.default_value);
        if (value != UNSET) InitPropertySet(prop.dst_prop, value);
    }
}
```

{{< image classes="fancybox center fig-100" src="/property/property_30.png" thumbnail="/property/property_30.png" title="">}}

## 3.5获取属性总结

不管是`Init`进程还是非`Init`进程，最终都会调用到`/bionic/libc`这个动态库中，且会先后调用`__system_property_find`和`__system_property_read_callback`函数，前者函数是通过属性名，从`/dev/__properties__/property_info`找到属性名对应`Selinux`安全上下文，然后在`/dev/__properties__`目录中找到对应的`Selinux`安全上下文文件，根据字典树的方式来读取全属性名。后者函数在安全上下文文件中，如果能够找到对应的`prop_info`，那么读取对应的`value`值，如果找不到那么就会返回`null`（其中`App`调用的`Java`和`Native`层，在上层封装了默认值，会返回默认值）。

# 4设置属性

这里包括非`init`进程和`init`进程设置属性，这里非`init`进程有两种方式，一种是在`App`调用`Java层`或者`Native层`调用设置属性的方式，另一种是直接`adb`敲命令`setprop`。

## 4.1App写入属性流程分析（Java侧）

```java
//App设置属性
import android.os.SystemProperties;
public static boolean setProperty(String key, String value) {
    try {
        Class<?> c = Class.forName("android.os.SystemProperties");
        Method set = c.getMethod("set", String.class, String.class);
        set.invoke(c, key, value);
        return true;
    } catch (Exception e) {
        e.printStackTrace();
    }
    return false;
}

setProperty("persist.a.b.c", "1");
```

### 4.1.1`setProperty`

```java
//frameworks/base/core/java/android/os/SystemProperties.java
//需要通过反射来设置
public static void set(@NonNull String key, @Nullable String val) {
    if (val != null && !val.startsWith("ro.") && val.length() > PROP_VALUE_MAX) {
        throw new IllegalArgumentException("value of system property '" + key
                                           + "' is longer than " + PROP_VALUE_MAX + " characters: " + val);
    }
    if (TRACK_KEY_ACCESS) onKeyAccess(key);
    native_set(key, val);
}
```

### 4.1.2`native_set`

```c++
//frameworks\base\core\jni\android_os_SystemProperties.cpp
int register_android_os_SystemProperties(JNIEnv *env)
{
    const JNINativeMethod method_table[] = {
        ...
        //设置属性就是这个SystemProperties_set函数 
        { "native_set", "(Ljava/lang/String;Ljava/lang/String;)V",
          (void*) SystemProperties_set },
    };
    return RegisterMethodsOrDie(env, "android/os/SystemProperties",
                                method_table, NELEM(method_table));
}
//这里是设置属性的地方
void SystemProperties_set(JNIEnv *env, jobject clazz, jstring keyJ,
                          jstring valJ)
{
    ScopedUtfChars key(env, keyJ);
    if (!key.c_str()) {
        return;
    }
    std::optional<ScopedUtfChars> value;
    if (valJ != nullptr) {
        value.emplace(env, valJ);
        if (!value->c_str()) {
            return;
        }
    }
    bool success;
    //最终会调用到这里的__system_property_set函数
#if defined(__BIONIC__)
    success = !__system_property_set(key.c_str(), value ? value->c_str() : "");
#else
    success = android::base::SetProperty(key.c_str(), value ? value->c_str() : "");
#endif
    if (!success) {
        jniThrowException(env, "java/lang/RuntimeException",
                          "failed to set system property (check logcat for reason)");
    }
}
```

### 4.1.3`__system_property_set`

```c++
//bionic/libc/bionic/system_property_set.cpp
__BIONIC_WEAK_FOR_NATIVE_BRIDGE
int __system_property_set(const char* key, const char* value) {
  if (key == nullptr) return -1;
  if (value == nullptr) value = "";
  //获取当前属性的版本号
  if (g_propservice_protocol_version == 0) {
    detect_protocol_version();
  }
  //比较当前的版本号，属性系统启动的时候就是设置为ro.property_service.version为2
  //1的话为Android 8或者更早的系统版本
  if (g_propservice_protocol_version == kProtocolVersion1) {
    // Old protocol does not support long names or values
    ...
  } else {
    // New protocol only allows long values for ro. properties only.
    if (strlen(value) >= PROP_VALUE_MAX && strncmp(key, "ro.", 3) != 0) return -1;
    // Use proper protocol
    PropertyServiceConnection connection;
    ...
    //建立socket连接，这里连接的就是Init进程启动时候的Property_Service
    //内部使用了writev，writev面向的是分散的数据块，两个函数的最终结果都是将内容写入连续的空间。
    SocketWriter writer(&connection);
    //发送tcp消息，首先是MSG类型，这里是属性系统版本v2.0，然后追加key和value的值
    if (!writer.WriteUint32(PROP_MSG_SETPROP2).WriteString(key).WriteString(value).Send()) {
    ...
  }
}
```

`SocketWriter`的数据结构如下图所示

{{< image classes="fancybox center fig-100" src="/property/property_31.png" thumbnail="/property/property_31.png" title="">}}

```c++
//bionic/libc/bionic/system_property_set.cpp
SocketWriter& WriteUint32(uint32_t value) {
    uint32_t* ptr = uint_buf_ + uint_buf_index_;
    uint_buf_[uint_buf_index_++] = value;
    iov_[iov_index_].iov_base = ptr;
    iov_[iov_index_].iov_len = sizeof(*ptr);
    ++iov_index_;
    return *this;
}

SocketWriter& WriteString(const char* value) {
    uint32_t valuelen = strlen(value);
    WriteUint32(valuelen);
    if (valuelen == 0) {
    return *this;
  }

    CHECK(iov_index_ < kIovSize);
    iov_[iov_index_].iov_base = const_cast<char*>(value);
    iov_[iov_index_].iov_len = valuelen;
    ++iov_index_;
    return *this;
}
```

#### 4.1.3.1`WriteUint32(PROP_MSG_SETPROP2)`

{{< image classes="fancybox center fig-100" src="/property/property_32.png" thumbnail="/property/property_32.png" title="">}}

#### 4.1.3.2`WriteString("persist.a.b.c")`

{{< image classes="fancybox center fig-100" src="/property/property_33.png" thumbnail="/property/property_33.png" title="">}}

#### 4.3.3.3`WriteString("1")`

{{< image classes="fancybox center fig-100" src="/property/property_34.png" thumbnail="/property/property_34.png" title="">}}

## 4.2App写入属性流程分析（Native侧）

```c++
//App通过封装的so进行读取属性
#include <cutils/properties.h>
#define PROPERTY_KEY_MAX   32
#define PROPERTY_VALUE_MAX  92
property_set("persist.a.b.c", "1");
```

`property_set`

```c++
//system/core/libcutils/properties.cpp
int property_set(const char *key, const char *value) {
    return __system_property_set(key, value);
}
```

`__system_property_set`跟上面4.1的流程一致，这里就不继续展开。

## 4.3adb写入属性

`adb`命令直接输入`setprop`，实际上是调用到一个`toolbox`当中的进程`setprop`，需要有两个参数，`name`和`value`，且`name`和`value`必须符合规范（`name`长度小于32，`value`长度小于92）。

```c++
//system/core/toolbox/setprop.cpp
extern "C" int setprop_main(int argc, char** argv) {
    ...
    auto name = std::string{argv[1]};
    auto value = std::string{argv[2]};
    ...
    if (!SetProperty(name, value)) {
        std::cerr << "Failed to set property '" << name << "' to '" << value
                  << "'.\nSee dmesg for error reason." << std::endl;
        return EXIT_FAILURE;
    }

    return EXIT_SUCCESS;
}
```

`SetProperty`

```c++
//system/core/base/properties.cpp
bool SetProperty(const std::string& key, const std::string& value) {
  return (__system_property_set(key.c_str(), value.c_str()) == 0);
}
```

`__system_property_set`跟上面4.1的流程一致，这里就不继续展开。

## 4.4Init进程写入属性

```c++
static void update_sys_usb_config() {
    ...
    if (config.empty() || config == "none") {
        InitPropertySet("persist.sys.usb.config", is_debuggable ? "adb" : "none");
    } else if (is_debuggable && config.find("adb") == std::string::npos &&
        ...
        InitPropertySet("persist.sys.usb.config", config);
    }
}
```

`InitPropertySet`

```c++
//system/core/init/property_service.cpp
uint32_t InitPropertySet(const std::string& name, const std::string& value) {
    uint32_t result = 0;
    ucred cr = {.pid = 1, .uid = 0, .gid = 0};
    std::string error;
    result = HandlePropertySet(name, value, kInitContext, cr, nullptr, &error);
    if (result != PROP_SUCCESS) {
        LOG(ERROR) << "Init cannot set '" << name << "' to '" << value << "': " << error;
    }

    return result;
}
```

`HandlePropertySet`

```c++
//system/core/init/property_service.cpp
uint32_t HandlePropertySet(const std::string& name, const std::string& value,
                           const std::string& source_context, const ucred& cr,
                           SocketConnection* socket, std::string* error) {
    if (auto ret = CheckPermissions(name, value, source_context, cr, error); ret != PROP_SUCCESS) {
        return ret;
    }

    if (StartsWith(name, "ctl.")) {
        return SendControlMessage(name.c_str() + 4, value, cr.pid, socket, error);
    }
    ...
    return PropertySet(name, value, error);
}
```

这个`PropertySet`又回到了2.3.2.3，具体不继续展开。

{{< image classes="fancybox center fig-100" src="/property/property_35.png" thumbnail="/property/property_35.png" title="">}}

## 4.5设置属性总结

不管是`Init`进程还是非`Init`进程，**设置属性需要调用到`Init`进程去操作**。如果是非`Init`进程，会通过本地`socket`通信，路径为`/dev/socket/property_service`与`Init`中的属性服务建立通信，让所有非`Init`进程的设置操作，全部在`Init`进程中操作。最终都会调用到`/bionic/libc`这个动态库中，且会先调用`__system_property_find`函数，通过属性名，从`/dev/__properties__/property_info`找到属性名对应`Selinux`域，然后在`/dev/__properties__`目录中找到对应的`Selinux`域文件。如果返回的`prop_info`存在，那么接着调用`__system_property_update`函数更新`value`值，如果`prop_info`不存在，那么接着调用`__system_property_Add`函数在域文件中按照字典树规则添加一个属性值。

# 5关于属性变化

在这个基础上Android推出了一个`init.rc`的机制，即**类似通过读取配置文件的方式，来启动不同的进程**，而不是通过传`exec`在代码中一个个的来执行进程。

> `rc`是一个配置文件，内部由Android初始化语言编写（`Android Init Language`）编写的脚本。
>
> 具体可以参考**`/system/core/init/README.md`**这个文件，详细介绍了rc文件的语法。

当属性变化之后，`rc`文件就会执行对应的`Command`，这个流程又是怎么样的呢？

我们知道，其它进程可以直接进行属性的读操作，但是属性系统的写操作只能在` init` 进程中进行，其它进程进行属性的写操作也需要通过 `init `进程。

```c++
//system/core/init/property_service.cpp
static uint32_t PropertySet(const std::string& name, const std::string& value, std::string* error) {
    ...
    auto lock = std::lock_guard{accept_messages_lock};
    if (accept_messages) {
        PropertyChanged(name, value);
    }
    return PROP_SUCCESS;
}
```

其实就是在上述属性设置的末尾有一个判断是否接受信息的判断，**默认是打开的**，在`StartPropertyService`函数的时候就有打开

```c++
void StartPropertyService(int* epoll_socket) {
    ...
    StartSendingMessages();
    ...
}

void StartSendingMessages() {
    auto lock = std::lock_guard{accept_messages_lock};
    accept_messages = true;
}
```

`PropertyChanged`

如果属性是 `sys.powerctl`，我们绕过事件队列并**立即处理**它。这是为了确保 `init` 将始终立即关闭/重新启动，无论是否有其他待处理的事件要处理，或者 `init` 是否正在等待 `exec` 服务或等待属性。在非热关机情况下，将触发“关机”触发器以执行设备特定的命令。

```c++
//system/core/init/init.cpp
void PropertyChanged(const std::string& name, const std::string& value) {

    if (name == "sys.powerctl") {
        trigger_shutdown(value);
    }
	//将rc文件中属性改变发生的命令或者事件添加到ActionManager内部队列中，等待轮询执行
    if (property_triggers_enabled) {
        ActionManager::GetInstance().QueuePropertyChange(name, value);
        WakeMainInitThread();
    }

    prop_waiter_state.CheckAndResetWait(name, value);
}
```

其中`ActionManager`定义是在属性服务初始化之后，立刻初始化

```c++
//system/core/init/init.cpp
//用于存放动作，包括init.cpp和rc文件
int SecondStageMain(int argc, char** argv) {
    ...
    ActionManager& am = ActionManager::GetInstance();
    //用于存放服务，包括init.cpp和rc文件
    ServiceList& sm = ServiceList::GetInstance();
    //解析rc文件，如果没有特殊配置ro.boot.init_rc，则解析./init.rc把/system/etc/init、/product/etc/init、/product_services/etc/init、/odm/etc/init、/vendor/etc/init 这几个路径加入init.rc之后解析的路径，在init.rc解析完成后，解析这些目录里的rc文件。
    LoadBootScripts(am, sm);
    ...
    while (true) {
        ...
        //处理rc文件触发的事件
        if (!(prop_waiter_state.MightBeWaiting() || Service::is_exec_service_running())) {
            m.ExecuteOneCommand();
        }
        //处理关机/重启事件
        if (!IsShuttingDown()) {
            HandleControlMessages();
            SetUsbController();
        }
    }
    return 0;
}
```

从代码中可以看到，`property_triggers_enabled` 是 `<属性变化触发 action>` 的使能点，开启之后每次属性发生变化都会调用 `ActionManager.QueuePropertyChange(name, value)` 函数

```c++
//system/core/init/action_manager.cpp
void ActionManager::QueueEventTrigger(const std::string& trigger) {
    auto lock = std::lock_guard{event_queue_lock_};
    event_queue_.emplace(trigger);
}
```

`QueuePropertyChange` 函数同样填充了 `event_queue_` 队列，将已改变属性的键值对作为参数。

运行队列中已经添加了该属性的变化触发条件，同样通过 `am.ExecuteOneCommand()` 函数遍历所有的 `_actions` 链表，执行相应的 `commands`。



# 总结

本文主要对属性系统的初始化原理`PropertyInit`、`StartPropertyService`，获取属性、设置属性和属性变化这五个方面来展开详解，围绕着属性，`SElinux`，`mmap`，共享内存，匿名私有内存，`Trie`，`rc文件`，`persist`属性做了一个流程的介绍和分析。总的来说，属性看起来简单，内部机制还是比较复杂的。特别是由于笔者水平有限，可能属性系统某些细节的介绍不够深入，这个需要读者发挥自己的主观能动性进一步完善。



# 下载

本文源码[点击这里](https://github.com/YangYang48/project/tree/master/property)



# 几个问题

> 1）SElinux怎么跟属性对应起来的

首先，`SElinux`是一种权限系统，也就是说每个文件都有自己对应的标签，比如每个文件都会有自己的标签。如果需要访问这个文件，需要有对应标签的类型的权限，访问`/sdcard/..ccdid`需要这个`u:object_r:sdcardfs:s0`标签的`dir`和`file`类型的`read`，`search`等权限，因为我需要访问到这个文件。

```shell
# 这里的第五列实际上就是标签，文件会有标签。
picasso:/sdcard $ ls -laZ
total 12946
drwxrwx--x 220 root sdcard_rw u:object_r:sdcardfs:s0   20480 2022-09-10 02:20 .
drwx--x--x   4 root sdcard_rw u:object_r:sdcardfs:s0    3488 2022-09-07 18:41 ..
-rw-rw----   1 root sdcard_rw u:object_r:sdcardfs:s0     301 2022-09-08 08:51 ..ccdid
-rw-rw----   1 root sdcard_rw u:object_r:sdcardfs:s0      57 2022-09-08 08:51 ..ccvid
-rw-rw----   1 root sdcard_rw u:object_r:sdcardfs:s0      96 2020-06-27 10:59 .3deaedc28469674418c57af6b7cc832b
drwxrwx--x   2 root sdcard_rw u:object_r:sdcardfs:s0    3488 2022-03-01 17:48 .6226f7cbe59e99a90b5cef6f94f966fd
drwxrwx--x   3 root sdcard_rw u:object_r:sdcardfs:s0    3488 2020-05-30 14:01 .7934039a
-rw-rw----   1 root sdcard_rw u:object_r:sdcardfs:s0      16 2021-01-25 14:22 .AID_SYS
drwxrwx--x   2 root sdcard_rw u:object_r:sdcardfs:s0    3488 2022-09-10 13:35 .DataStorage
drwxrwx--x   2 root sdcard_rw u:object_r:sdcardfs:s0    3488 2020-12-10 14:03 .Download
drwxrwx--x   3 root sdcard_rw u:object_r:sdcardfs:s0    3488 2020-06-10 11:33 .INSTALLATION
drwxrwx--x   2 root sdcard_rw u:object_r:sdcardfs:s0    3488 2021-11-24 15:27 .RtcDataStorage
-rw-rw----   1 root sdcard_rw u:object_r:sdcardfs:s0      36 2021-01-25 14:22 .SID_SYS
drwxrwx--x   2 root sdcard_rw u:object_r:sdcardfs:s0    3488 2020-12-10 14:03 .Trusfort
-rw-rw----   1 root sdcard_rw u:object_r:sdcardfs:s0     169 2021-01-25 14:22 .UA_SYS
```

而属性安全上下文文件也是一样，所有的属性都会对应到一个标签，然后将默认设置的属性全部导入到一个文件中，也就是在本文1.1序列化的一个操作，`/dev/__properties__/property_info`，这里面保存了默认的所有属性对应的标签。再者，新建了一系列的属性安全上下文文件，包含了标签和对应的属性。访问一个属性的时候，首先会找到这个属性对应的标签，然后在`/dev/__properties__/`找到对应的属性安全上下文文件，找到之后，通过字典树的形式最终找到对应的属性。



> 2）自定义属性怎么添加上去的

自定义属性也是属于设置属性的范畴。其实就是简单的来说，分成三部分。

1. 找到对应属性的标签，如果没有找到那么就是缺省标签，`u:object_r:default_prop:s0`
2. 找到`/dev/__properties__/xxx`对应的文件，`xxx`为对应属性的标签，然后通过字典树访问，没有找到这个属性
3. 重新添加这个属性，根据字典树的添加原理，按照方式添加这个自定义属性



> 3）自定义属性会对应哪个`SElinux`标签呢

这个需要根据设置对前缀来区分，比如`bluetooth.`开头的属性都属于`u:object_r:bluetooth_prop:s0`标签的，访问对应的`bluetooth.`需要有`u:object_r:bluetooth_prop:s0`标签类型的权限。有些默认定义的属性访问权限很高，并不是随意一个进程或者`App`可以自由访问的。

```shell
bluetooth.              u:object_r:bluetooth_prop:s0
config.                 u:object_r:config_prop:s0
```

比如自定义一个属性，没有任何与默认属性定义的冲突，那么就是默认缺省属性

```shell
persist.a.b.c.d         u:object_r:default_prop:s0
```



> 4）persist属性为什么重启存在，我怎么删除自定义的persist属性呢

i）首先回答第一个问题，为什么重启存在，因为除了像其他的属性一样会保存在`/dev/__properties__/`目录中，`persist.`开头的属性还会被保存在`/data/property/persistent_properties`这个文件中。开机的时候除了会重新初始化默认属性，还会将`/data/property/persistent_properties`这个文件中属性读到对应的`/dev/__properties__/`目录对应的安全上下文文件中。和上面的2.3.2.2.4的流程一样，只不过因为在第一次设置属性的时候用到了`LoadPersistentPropertiesFromMemory`，`/data/property/persistent_properties`文件已经创建。所以直接将该文件的内容从`proto`变成字符串的访问来设置即可。

{{< image classes="fancybox center fig-100" src="/property/property_25.png" thumbnail="/property/property_25.png" title="">}}



ii）怎么删除自定义的`persist`属性，比如`persist.a.b.c.d`

`Android8.0`或者更低版本，只需要删除`/data/property/persist.a.b.c.d`的文件，并且删除`/data/property/persist.a.b.c.d`在`/dev/__properties__/`目录中的安全上下文文件。

`Android11`版本，只需要删除或者更名`/data/property/`目录，并且删除`/data/property/persist.a.b.c.d`在`/dev/__properties__/`目录中的安全上下文文件。

删除自定义的persist属性，需要知其所以然，原理是清理掉`/data/property/persistent_properties`，并且不能让属性在下一次启动的时候，还原清理掉的文件`/data/property/persistent_properties`。

> 这里需要了解几件事
>
> - 重启的过程中关机会设置属性`persist.sys.boot.reason`。这里假定我是通过`adb`方式重启，那么
>
>   ```shell
>   [persist.sys.boot.reason]: [reboot,shell]
>   ```
>
> - mmap在删除硬盘文件之后，进程依然能够让映射的内存区域进行读写操作，具体原理可以点击[这里](https://bbs.csdn.net/topics/360058295)。简单的来讲就是，mmap机制，文件映射到逻辑地址空间的内存中，实际上也进程是一个持有文件句柄的过程。在建映射以后、解除映射之前，如果文件被删除，单个进程读写是没有问题，或者是多个进程在建映射以后、解除映射之前，如果文件被删除，多个进程读读没有问题。

高版本中，必须删除或者更名`/data/property/`目录，如果不对这个目录做处理，一旦重启依然会生效。

```c++
//system/core/init/persistent_properties.cpp
//这里才是真正让persist属性起不起作用的函数
Result<void> WritePersistentPropertyFile(const PersistentProperties& persistent_properties) {
    //创建一个临时文件/data/property/persistent_properties.tmp
    const std::string temp_filename = persistent_property_filename + ".tmp";
    //打开这个临时文件
    unique_fd fd(TEMP_FAILURE_RETRY(
        open(temp_filename.c_str(), O_WRONLY | O_CREAT | O_NOFOLLOW | O_TRUNC | O_CLOEXEC, 0600)));
    if (fd == -1) {
        return ErrnoError() << "Could not open temporary properties file";
    }
    std::string serialized_string;
    if (!persistent_properties.SerializeToString(&serialized_string)) {
        return Error() << "Unable to serialize properties";
    }
    //将所有属性，这里的属性依然是包括persist.a.b.c.d的，写入到这个临时文件中
    if (!WriteStringToFd(serialized_string, fd)) {
        return ErrnoError() << "Unable to write file contents";
    }
    fsync(fd);
    fd.reset();
    //将临时文件重命名为/data/property/persistent_properties
    if (rename(temp_filename.c_str(), persistent_property_filename.c_str())) {
        int saved_errno = errno;
        unlink(temp_filename.c_str());
        return Error(saved_errno) << "Unable to rename persistent property file";
    }
    //这里是最关键的，找到/data/property/persistent_properties文件的目录，即/data/property
    //因为我们已经将这个目录更改，所以磁盘中已经找不到这个目录，直接返回添加所有属性失败，属性中包括persist.a.b.c.d
    auto dir = Dirname(persistent_property_filename);
    auto dir_fd = unique_fd{open(dir.c_str(), O_DIRECTORY | O_RDONLY | O_CLOEXEC)};
    if (dir_fd < 0) {
        return ErrnoError() << "Unable to open persistent properties directory for fsync()";
    }
    fsync(dir_fd);

    return {};
}
```

删除`/data/property/persistent_properties`和删除`persist.a.b.c.d`在`/dev/__properties__/`目录中的安全上下文文件。重启关机过程中，会设置关机的原因属性，那么会重新导入`/dev/__properties__/`定义的`Selinux`安全上下文对应的所有的`ContextNode`中的属性(删除`persist.a.b.c.d`在`/dev/__properties__/`目录中的安全上下文文件，**进程依然可以读文件所映射的逻辑地址**)，其中包括`persist.a.b.c.d`属性，那么在关机的过程中会重新生成`/data/property/persistent_properties`文件，那么下一次开机过程中，会从`/data/property/persistent_properties`文件导入所有的`persist.`开头的属性，那么自然`persist.a.b.c.d`属性也会被导入。因此，**必须对`/data/property`目录进行更改。**

{{< image classes="fancybox center fig-100" src="/property/property_26.png" thumbnail="/property/property_26.png" title="">}}

> 5）属性系统预置的`SElinux`规则文件和预置写入的属性文件在哪个目录

`SElinux`规则文件，通常是下面几个目录文件，其中这些带有`selinux`目录是通过`mmm system/sepolicy`生成的，重启系统之后`SElinux`新的规则就能生效，通常对于`SElinux`修改都是在`/device`对应的`sepolicy`目录修改。

```txt
/system/etc/selinux/plat_property_contexts
/system_ext/etc/selinux/system_ext_property_contexts
/vendor/etc/selinux/vendor_property_contexts
/product/etc/selinux/product_property_contexts
/odm/etc/selinux/odm_property_contexts
```

预置写入的属性文件，通常是下面几个目录文件，也是在`/device`对应目录`mk`中修改，修改完成之后更新几个目录文件，重启之后即可生效。当然如果是修改类似`ro.boot`这种属性需要在`uboot`修改，`mk`修改是不起作用的。

```txt
#并不是所有的文件都会存在设置属性的值
/system/etc/prop.default
/prop.default              #recovery path
/default.prop              #legacy path
/system/build.prop
/system_ext/build.prop
/vendor/default.prop
/vendor/build.prop
/odm/etc/build.prop
/product/build.prop
/factory/factory.prop
```



# 参考

[[1] 低调小一, Android init进程——属性服务, 2015.](https://wangzhengyi.blog.csdn.net/article/details/44999103)

[[2] 昨夜星辰_zhangjg, Android是如何使用selinux来保护系统属性的, 2021.](https://blog.csdn.net/zhangjg_blog/article/details/122057945)

[[3] apigfly, 深入Android系统（二）Bionic库, 2020.](https://blog.csdn.net/lijie2664989/article/details/107826733)

[[4] 悠然红茶, 深入讲解Android Property机制, 2015.](https://youranhongcha.blog.csdn.net/article/details/48379239)

[[5] sanchuyayun, SEAndroid策略分析, 2017.](https://blog.csdn.net/sanchuyayun/article/details/54944820)

[[6] thl789, Android属性之build.prop生成过程分析, 2011.](https://haili-tian.blog.csdn.net/article/details/7014300)

[[7] 淡陌羿格, 安卓property service系统分析, 2021.](https://blog.csdn.net/qq_36063677/article/details/116750095)

[[8] BestW2Y, [Android 基础] -- Android 属性系统简介, 2021.](https://blog.csdn.net/u014674293/article/details/119147063)

[[9] YJer, Notepad++ 安装 HexEditor 插件, 2022.](https://blog.csdn.net/qq_33249042/article/details/122972301)

[[10] 流金岁月5789651, Framework学习之旅：init 进程启动过程, 2022.](https://blog.csdn.net/xufei5789651/article/details/125961599)

[[11] 微信终端开发团队, 快速缓解 32 位 Android 环境下虚拟内存地址空间不足的“黑科技”, 2021.](https://cloud.tencent.com/developer/article/1854375)

[[12] canyie, Android Property 实现解析与黑魔法, 2022.](https://blog.canyie.top/2022/04/09/property-implementation-and-isolation/)

[[13] Dufresne, Android 系统属性SystemProperty分析, 2012.](https://www.cnblogs.com/bastard/archive/2012/10/11/2720314.html)

[[14] android系统之属性系统详解.](https://wenku.baidu.com/view/b012e4c049fe04a1b0717fd5360cba1aa8118cdb?pcf=2&bfetype=new&bfetype=new)

[[15] iteye_563, SEAndroid安全机制对Android属性访问的保护分析, 2014.](https://blog.csdn.net/iteye_563/article/details/82601798)

[[16] 岁月斑驳7, Android 8.1 开机流程分析（2）, 2018.](https://blog.csdn.net/qq_19923217/article/details/82014989)

[[17] 流金岁月5789651, Framework学习之旅：init 进程启动过程, 2022.](https://blog.csdn.net/xufei5789651/article/details/125961599)

[[18] 静思心远, C语言变长数组(柔性数组) struct中char data[0]的用法, 2020.](https://qingsong.blog.csdn.net/article/details/108438071)

[[19] itzyjr, ❥关于C++之私有继承, 2022.](https://blog.csdn.net/itzyjr/article/details/123544141)

[[20] 2013我笑了, android property属性property_set()&& property_get() selinux权限问题, 2019.](https://blog.csdn.net/u012542095/article/details/85850434)

[[21] 水墨长天, std::lock_guard的原理和应用, 2022.](https://blog.csdn.net/gehong3641/article/details/124028976)

[[22] i校长, 深入理解MMAP原理，大厂爱不释手的技术手段, 2022.](https://juejin.cn/post/7119116943256190990)

[[23] beOkWithAnything, mmap 父子进程共享内存通信, 2020.](https://blog.csdn.net/swq463/article/details/109738256)

[[24] GarfieldEr007, Protobuf 语法指南, 2018.](https://www.cnblogs.com/GarfieldEr007/p/10113328.html)

[[25] yaoyz105, Protobuf 学习（一）proto文件, 2022.](https://yaoyz.blog.csdn.net/article/details/93189219)

[[26] 私房菜, Android protobuf 生成c++ 文件详解, 2021.](https://justinwei.blog.csdn.net/article/details/120560637)

[[27] OrangeAdmin, 专注于中台化代码生成器, 2013.](https://www.cnblogs.com/orangeform/archive/2013/01/02/2841485.html)

[[28] 水墨长天, std::lock_guard的原理和应用, 2022.](https://blog.csdn.net/gehong3641/article/details/124028976)

[[29] 深度Java, ACCEPT()和ACCEPT4(), 2012.](https://blog.csdn.net/21aspnet/article/details/8196671)

[[30] konga, open中的 O_CLOEXEC 标志, 2014.](https://blog.csdn.net/konga/article/details/39062691)

[[31] Thinbug, 删除文件后删除mmap（）, 2017.](https://www.thinbug.com/q/42961339)

[[32] cxz7531, mmap函数建立文件的内存映射后，删除文件，能正常读取内容吗？, 2011.](https://bbs.csdn.net/topics/360058295)

[[33] weaiken, `__attribute__ `机制详解, 2019.](https://blog.csdn.net/weaiken/article/details/88085360)

[[34] kenny肉桂, `__attribute__`((constructor))用法解析, 2016.](https://www.jianshu.com/p/dd425b9dc9db)

[[35] linuxheik, writev, 2017.](https://blog.csdn.net/linuxheik/article/details/76125411)

[[36] winsonCCCC, Linux网络编程之sockaddr与sockaddr_in,sockaddr_un分析, 2021.](https://blog.csdn.net/weixin_43829445/article/details/116497954)
