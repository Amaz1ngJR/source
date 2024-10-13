## 项目展示
![dcc84efba921bdfdee58e8dd559ab233](https://github.com/user-attachments/assets/72c9c87e-03b5-445f-bbdf-fa113659f535)

### 服务器的细节演示
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


//通过X宏技巧结合模板 将待调试变量注册Http的API 
template <typename pType>
void func(pType HttpDebug::* pFlag) {
	//通过Htpp发送请求 来修改给定世界下的结构体对象中的成员变量
	auto& val = w.get().*pFlag;
	cout << val << endl;
}

#define REG_FLAG(type, var, val) \
func<type>(&HttpDebug::var);

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

将cpp中api注册到Lua虚拟机示意图
![12b426f9921cc02a3380bb452f767989](https://github.com/user-attachments/assets/86b2496a-3bd8-48cd-945c-2c4e8f4964bb)


### 内存分析效果
![image](https://github.com/user-attachments/assets/f58e87f1-a2c5-4d25-9d6f-5cd54f97955c)


![image](https://github.com/user-attachments/assets/d03ea143-94d2-4a53-b5a7-ad6d12887f70)


![image](https://github.com/user-attachments/assets/40d43b02-a5e9-48b6-8fd7-9f5a482c3f2d)

