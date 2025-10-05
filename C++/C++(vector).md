# 动态数组：vector

- 头文件：` #include <vector> `
- 创建(1)：std::vector<数据类型/int/float> 自定义名称；
- 创建(2)：std::vector<数据类型/int/float> 自定义名称(3,1)；   //1，1，1
- 创建(3): std::vector<数据类型/int/float> 自定义名称{2,7,1};  //2, 7, 1

### vector<int> v{1, 2, 3};
- 添加元素（尾）: v.push_back(4);   // 现在v是{1,2,3,4}
- 删除元素（尾）: v.pop_back();
- 获取数组大小  ：v.size(); 
- 判断容器是否为空：v.empty();      //空返回true
- 清空容器 ：v.clear();
- 访问元素 ：v.at[2];           //返回v[2]的值，并具有越界检查的功能。
- 正常访问 ：v[2];
- 访问首元素 ：v.front();
- 访问尾元素 ：v.back();  

- 查看容量 ：v.capacity();
- 调整容量 ：v.resize(n)        //大小变为n
---

