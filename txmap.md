## 项目展示
![dcc84efba921bdfdee58e8dd559ab233](https://github.com/user-attachments/assets/72c9c87e-03b5-445f-bbdf-fa113659f535)

### Server
```c++
//对外暴露的接口 用来增删要调试的变量
#define HTTP_FLAGS(X) \
X(bool, drawA, true) \
X(bool, drawB, true) \
X(int, m_a, 100);


//-----------服务器实现的细节---

//通过X宏技巧将待调试变量注册到结构体中
#define EXP_FLAG(type, var, val) type var = val;
struct HttpDebug {
	HTTP_FLAGS(EXP_FLAG)
};
#undef EXP_FLAG

//模拟引擎环境
class world {
public:
	HttpDebug a;

	HttpDebug& get() {
		return a;
	}
};
world w;//当前世界

template <typename FlagType>
inline void handleSetFlag(FlagType HttpDebugFlags::*pFlag,
		      const httplib::Request& req, httplib::Response& res) {
	//......
	if (cur_world) {
	    auto& flag = cur_world->debug_flags_.get()->*pFlag;
	    flag = std::move(v);
	    cur_world->setNeedRedraw(true);
	}
	//......
}

//通过X宏技巧结合模板 将待调试变量注册Http的API 
#define REGISTER_FLAG_API(pType, pFlag)                           \
  do {                                                            \
    const char* set_path = "/set/" #pFlag;                        \
    svr_.Get(set_path, [](const Request& req, Response& res) {    \
      handleSetFlag<pType>(&HttpDebugFlags::pFlag, req, res);     \
    });                                                           \
  } while (false)

#define REG_FLAG(type, var, defaultVal) \
  REGISTER_FLAG_API(type, var);
            EXP_FLAG(REG_FLAG)

int main() {
	HttpDebug a;
	cout << a.drawA << '\n';
	HTTP_FLAGS(REG_FLAG)
	return 0;
}

```
### 客户端效果
![image](https://github.com/user-attachments/assets/b8b347f1-12e0-4e18-9510-757d2bddd9a5)



打开地图后，通过点击“刷新world"按钮发送http请求来获取所有world并动态的生成world相应的checkbox

![image](https://github.com/user-attachments/assets/26f036dc-933e-454c-bc45-c1457c759a81)



勾选其中一个world选项，发送http请求获取当前world下需要debug的变量并动态生成相应的控件(左侧bool变量对应checkbox,右侧其他变量通过修改文本框来实现对变量的修改)

![image](https://github.com/user-attachments/assets/4fd11fe5-e515-4918-9e6e-d414e11547a1)


通过勾选左侧选项来实时的更新地图渲染

![image](https://github.com/user-attachments/assets/9249544a-c5b8-45b0-9a08-5034311b5115)


![image](https://github.com/user-attachments/assets/ce7ebd5a-6021-488e-8cac-3269d4391b37)


![image](https://github.com/user-attachments/assets/0545701e-ae0c-481a-8e41-d702d5e35f49)


直接在右侧文本框中修改变量的值（点击sure按钮提交）

![image](https://github.com/user-attachments/assets/6af30f43-9714-4a73-b19e-6992f8c8a4bf)
### Lua虚拟机
将cpp中api注册到Lua虚拟机示意图
![12b426f9921cc02a3380bb452f767989](https://github.com/user-attachments/assets/86b2496a-3bd8-48cd-945c-2c4e8f4964bb)

使用sol2库将cpp代码注册到Lua虚拟机中

![image](https://github.com/user-attachments/assets/85f141a3-8d5e-4637-9e11-52ca122b7889)

使用Lua原生API仅仅注册一个构造函数所需的代码(整个胶水代码长100多行)

![image](https://github.com/user-attachments/assets/92b912b7-286e-45df-9500-f824e5ef33fe)

#### [libclang](https://github.com/llvm/llvm-project/blob/main/clang/include/clang-c/Index.h)解析c++代码
[clang接口](https://clang.llvm.org/doxygen/group__CINDEX__CPP.html)
通过libclang逐个解析cpp头文件，
```c++
//-----部分示例代码
//-----------输出解析到的函数
std::string DebugUtils::toString(ClangFunctionCursor cursor)
{
    std::stringstream ss;
    if (cursor.isInlined()) {
        ss << "inline ";
    }
    ss << cursor.returnTypeString() << " " << cursor.fullNameString() << "(";
    int n = cursor.getNumArguments();
    for (int i = 0; i < n; ++i) {
        if (i > 0) {
            ss << ", ";
        }
        auto arg = cursor.getArgument(i);
        ss << arg.typeString() << " " << arg.cursorSpelling();
    }
    ss << ")";
    return ss.str();
}
std::string DebugUtils::toString(ClangConversionCursor cursor)
{
    std::stringstream ss;
    if (cursor.isVirtual()) {
        ss << "virtual ";
    }
    ss << cursor.cursorSpelling() << "()";
    if (cursor.isConst()) {
        ss << " const";
    }
    if (cursor.isRVReference()) {
        ss << " &&";
    }
    if (cursor.isPureVirtual()) {
        ss << " = 0";
    }
    return ss.str();
}
//-----------输出解析到的类
std::string DebugUtils::toString(ClangClassCursor cursor)
{
    std::stringstream ss;

    auto baseClasses = cursor.getChildrenOfKind<ClangBaseClassSpecifierCursor>(CXCursor_CXXBaseSpecifier);

    bool isStruct = cursor.kind() == CXCursor_StructDecl;
    ss << (isStruct ? "struct " : "class ") << cursor.fullNameString();

    bool isFirstBase = true;
    for (auto base : baseClasses) {
        if (isFirstBase) {
            ss << " : ";
            isFirstBase = false;
        } else {
            ss << ", ";
        }
        if (base.isVirtualBase()) {
            ss << "virtual ";
        }
        ss << accessSpecifierToString(base.getAccessSpecifier()) << " " << base.fullNameString();
    }
    ss << " {\n"
       << (isStruct ? "" : "public:\n");

    cursor.visitClassPublicChildren([&ss](ClangCursor child) {
        switch (child.kind()) {
        case CXCursor_FieldDecl:
            ss << "    " << child.type().spelling() << " " << child.cursorSpelling() << ";\n";
            break;
        case CXCursor_VarDecl:
            ss << "    static " << child.type().spelling() << " " << child.cursorSpelling() << ";\n";
            break;
        case CXCursor_CXXMethod:
            ss << "    " << toString(ClangMethodCursor { child }) << ";\n";
            break;
        case CXCursor_Constructor:
            ss << "    " << toString(ClangConstructorCursor { child }) << ";\n";
            break;
        case CXCursor_Destructor:
            ss << "    " << child.cursorSpelling() << "();\n";
            break;
        case CXCursor_ConversionFunction:
            ss << "    " << toString(ClangConversionCursor { child }) << ";\n";
            break;
        default:
            break;
        }
    });
    ss << "}";
    return ss.str();
}
```
将cpp代码注册到lua中（这部分通过将libclang注册到lua虚拟机中，用lua写胶水代码）
```c++
//示例
static sol::state g_lua;//创建一个全局的lua虚拟机

static void registerApis(const HeaderParser& parser)
{
	auto m = g_lua["clang"].get_or_create<sol::table>();
	m["clang_createIndex"] = clang_createIndex;
	m["clang_disposeIndex"] = clang_disposeIndex;
	m["clang_createIndexWithOptions"] = clang_createIndexWithOptions;
	//m["clang_XXX"] = clang_XXX;
	//......提取 clang-c/Index.h 中的所有函数声明
}
void initLua(const HeaderParser& parser)
{
	g_lua.open_libraries(
		sol::lib::base,
		sol::lib::package,
		sol::lib::string);
	registerApis(parser);
}

void LuaCodeGenerator::generateNative(const HeaderParser& parser)
{
    auto& functions = parser.getFunctions();
    auto& variables = parser.getVariables();
    auto& constants = parser.getConstants();
    auto& classes = parser.getClasses();
    auto& macros = parser.getMacros();

    std::stringstream ss;
    for (auto& f : functions) {
        ss << "m[\"" << f.cursorSpelling() << "\"] = " << f.cursorSpelling() << ";\n";
    }

    for (auto& c : classes) {
        ss << "m.new_usertype<" << c.cursorSpelling() << ">(\"" << c.cursorSpelling() << "\", sol::constructors<()>";
        for (auto& m : c.getMemberVariables()) {
            ss << ",\n\t\"" << m.cursorSpelling() << "\", &" << c.cursorSpelling() << "::" << m.cursorSpelling();
        }
        ss << ");\n";
    }

    std::cout << ss.str() << std::endl;
}
```

### 通过hook malloc函数族的方式集成[tracy](https://github.com/wolfpld/tracy)
使用preload方法示意
```c++
#include <dlfcn.h>
#include <stdio.h>
#include <stdlib.h>

void *(*original_malloc)(size_t) = NULL;

// hook的malloc函数
void *malloc(size_t size)
{//printf会调用malloc 不要加printf，可以使用puts
    if (original_malloc == NULL)
    {
        original_malloc = (void *(*)(size_t))dlsym(RTLD_NEXT, "malloc");
    }
    return original_malloc(size);
}
```
动态链接器： 工作于程序运行阶段 工作时需要提供动态库所在目录位置
```
export LD_LIBRARY_PATH=/path/to/libmylib.so (临时奏效)
```
preload
```
export LD_PRELOAD=/path/to/libmylib.so
```
```
g++ main.cpp -o demo -lmylib -L./lib    //-l/L后直接带参数 没有空格
```
在本地找到合适版本的libc++_shared.so
```
find /Users/amaz1ng/Library/Android/sdk/ndk/21.4.7075529/ | grep libc++_shared
```
将demo、libc++_shared.so和libmylib.so发送到虚拟机中
```
adb push /path/to/demo /data/local/tmp
adb push /path/to/libmylib.so /data/local/tmp
adb push /path/to/libc++_shared.so /data/local/tmp
```
在虚拟机下运行
```
adb shell
cd /data/local/tmp
export LD_LIBRARY_PATH=/data/local/tmp
./demo
exit
```
使用[xhook](https://github.com/iqiyi/xHook)
```c++
#include <stdio.h>
#include <stdlib.h>
#include "tracy/Tracy.hpp"
#include "jni/xhook.h"
#include <iostream>
#include <unistd.h>

void *my_malloc(size_t size)
{ 
    write(1, "my_malloc\n", 11);
    auto ptr = malloc(size);//调用原始malloc
    TracyAllocS(ptr, size, 20);//使用第三方tracy来进行内存分析
    return ptr;
}

void my_free(void *ptr)
{
    write(1, "my_free\n", 9);
    free(ptr);
    TracyFreeS(ptr, 20);
}

void do_xhook()
{
    // 替换所有动态库下的malloc和free函数
    int res1 = xhook_register(".*\\.so$", "malloc", (void *)my_malloc, NULL);
    int res2 = xhook_register(".*\\.so$", "free", (void *)my_free, NULL);
    // 忽略当前库mylib.so下的malloc和free函数
    int res_ignore1 = xhook_ignore("mylib\\.so$", "malloc");
    int res_ignore2 = xhook_ignore("mylib\\.so$", "free");
}

static struct SoMain
{
    SoMain()
    {
        do_xhook();//将除了当前库以外的所有动态库都hook上
	write(1, "Hook\n", 5);
    }
} _soMain;
```
### 内存分析效果
![image](https://github.com/user-attachments/assets/f58e87f1-a2c5-4d25-9d6f-5cd54f97955c)


![image](https://github.com/user-attachments/assets/d03ea143-94d2-4a53-b5a7-ad6d12887f70)


![image](https://github.com/user-attachments/assets/40d43b02-a5e9-48b6-8fd7-9f5a482c3f2d)

