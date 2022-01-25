C++ 左值和右值引用

引言

```c++
class Student{
private:
    string info = "AAA";
//todo 第一种情况 getInfo的函数info与main函数的result是旧与新的两个变量，属于值传递    
public:
    string getInfo(){
        return this->info;
    }
//todo 第二种情况 getInfo的函数info与main函数的result是引用关系，一块内存的多个别名     
public:
    string& getInfo_front(){
        return this->info;
    }
};

int main(){
    Student stu;
    //todo 第一种情况 值传递
    stu.getInfo() = "BBBB";
    string res = stu.getInfo();
    cout << "第一种情况：" << res << endl;
    
    //todo 第二种情况 引用传递
    stu.getInfo_front() = "BBBB";
    res = stu.getInfo_front();
    cout << "第二种情况：" << res << endl;
    return 0;
}
```

左值引用是用来获取值，右值引用是用来修改值
