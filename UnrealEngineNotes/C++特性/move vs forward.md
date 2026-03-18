- std::move：无条件转为右值（即使传入的是右值，也继续保持右值属性）。（用于“窃取”资源）。
- std::forward：**有条件**保留原值类别（仅用于万能引用）。

| 场景        | 用std::move                | 用std::forward< T >                | 目的          |
| --------- | ------------------------- | --------------------------------- | ----------- |
| 普通具名变量    | f(std::move(obj));        | 不能用（必须指定模板参数）                     | 无条件转为右值，偷资源 |
| 万能引用参数内部  | f(std::move(u));（错误！总是移动） | f(std::forward< T >(u));（正确）      | 保留原始值类别     |
| 非模板函数里想移动 | 推荐用 std::move             | 也可以，但要写 std::forward(decltype(x)) | 更清晰用 move   |

std::move 本身什么都不偷，它只是一个纯编译期转换（static_cast），把左值强制标记为“可以被偷”的右值。真正“偷资源”的是移动构造函数（move constructor）或移动赋值运算符（move assignment operator）。

# 为什么需要偷资源
假设有一个管理大块堆内存的类（如简化版的 std::string 或 std::vector）：
- **拷贝构造**（深拷贝）：把对方的所有数据完整复制一份 → 非常慢，内存占用翻倍。
- **移动构造**（偷资源）：直接把对方的指针、长度等“偷过来”，对方指针置空 → 几乎零开销，只复制几个字节。

形象比喻：
- 拷贝 = 复印一份文件（花时间、占新空间）
- 移动 = 把文件直接搬到你家，对方手里只剩一个空文件夹

# 将亡值
在 C++03 时代，表达式只有 左值（lvalue）和 右值（rvalue）两大类。C++11 为了区分“可以偷资源”的对象，把右值进一步拆成 纯右值（prvalue）和将亡值（xvalue）。

每一种表达式**只属于**以下三大**主要类别**之一：
- **lvalue**（左值）：有身份（identity）、可取地址、持久存在的对象。
- **prvalue**（纯右值）：纯临时值、无身份、用于初始化或计算。
- **xvalue**（将亡值 / expiring value）：**即将消亡**的对象，有身份但资源**可以被偷走**（移动）。

```cpp
#include<iostream>
#include<utility>
#include<string>

std::string getTemp() { return "temporary"; } // 返回 prvalue

std::string&& getExpiring(std::string& s) { 
	return std::move(s); // 返回 xvalue 
}

int main() { 
	std::string s = "hello";

	// prvalue 示例
	std::string a = getTemp(); // prvalue → 可能移动构造 
	std::string b = "literal"; // 字面量相关 prvalue

	// xvalue 示例 
	std::string c = getExpiring(s); // xvalue → 移动 
	std::string d = std::move(s); // std::move 产生 xvalue

	// 取地址测试（prvalue 非法，xvalue 通常可但不推荐） 
	// & (42); // 错误：prvalue 不能取地址 
	// & std::move(s); // 某些情况下可，但危险 
}
```