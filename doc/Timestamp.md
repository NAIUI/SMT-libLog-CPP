# Timestamp类设计

## 数据成员：microSecondsSinceEpoch_

用来来表示从 Epoch时间到目前为止的微妙数，初值0（也表示无效值）。microSecondsSinceEpoch_的数据类型为什么是int64_t，而不是int32_t或者uint64_t？因为32位连一年的微妙数都不能表示，而int64_t可以表示290余年的微妙数（一年按365*24*3600*100000计算），未来还能表示一百余年，也就是说，其范围满足目前日常需求。而有符号的int64_t可以用来让2个时间戳进行差值计算，从而表示先后顺序。当然，时间戳本身为负数没有意义。

## 对象有效性

```C++
// Timestamp.h
    bool valid() const
    { return microSecondsSinceEpoch_ > 0; }

    static Timestamp invalid()
    {
        return Timestamp();
    }
```

1. 为什么需要invalid()？
   有时，我们需要一个临时的Timestamp对象，并不会用它来表示真实的时间，而是表示无用Timestamp对象。此时，用invalid()类函数自然是没问题的。当然，也可以用default ctor来构造一个临时Timestamp对象也是可以的，invalid()实现也是这么做的，但invalid()语义更清晰，而且不依赖于default ctor实现。假设哪天修改了microSecondsSinceEpoch_含义，那么用default ctor来代表无效Timestamp对象，也就失效了，然而invalid()接口却可以不变，客户端不用修改代码。
2. 为什么invalid()是static？
   因为要构造一个Timestamp对象，也就是说，此时还没有Timestamp对象，也就无法通过对象的成员函数来构造自身对象。

## 获取当前时间

```C++
// 获取当前时间戳
Timestamp Timestamp::now()
{
    struct timeval tv;
    // 获取微妙和秒
    // 在x86-64平台gettimeofday()已不是系统调用,不会陷入内核, 多次调用不会有性能损失.
    gettimeofday(&tv, NULL);
    int64_t seconds = tv.tv_sec;
    // 转换为微妙
    return Timestamp(seconds * kMicroSecondsPerSecond + tv.tv_usec);
}
```

## 获取时间戳

```C++
//返回当前时间戳的微妙
int64_t microSecondsSinceEpoch() const { return microSecondsSinceEpoch_; }
//返回当前时间戳的秒数
time_t secondsSinceEpoch() const
{ 
    return static_cast<time_t>(microSecondsSinceEpoch_ / kMicroSecondsPerSecond); 
}
```

## 获取可打印字符串

```C++
    //用std::string形式返回,格式[millisec].[microsec]
    std::string toString() const;
    //格式, "%4d年%02d月%02d日 星期%d %02d:%02d:%02d.%06d",时分秒.微秒
    std::string toFormattedString(bool showMicroseconds = false) const;
```

## 辅助函数（非class member函数）

```C++
/**
 * 定时器需要比较时间戳，因此需要重载运算符
 */
inline bool operator<(Timestamp lhs, Timestamp rhs)
{
    return lhs.microSecondsSinceEpoch() < rhs.microSecondsSinceEpoch();
}

inline bool operator==(Timestamp lhs, Timestamp rhs)
{
    return lhs.microSecondsSinceEpoch() == rhs.microSecondsSinceEpoch();
}

// 如果是重复定时任务就会对此时间戳进行增加。
inline Timestamp addTime(Timestamp timestamp, double seconds)
{
    // 将延时的秒数转换为微妙
    int64_t delta = static_cast<int64_t>(seconds * Timestamp::kMicroSecondsPerSecond);
    // 返回新增时后的时间戳
    return Timestamp(timestamp.microSecondsSinceEpoch() + delta);
}
```

# 单元测试

单元测试测什么？
muduo是以class为单位，根据提供给用户的功能点进行测试。有些进行的是覆盖测试。

Timestamp主要功能点：

* 构造对象：默认对象，无效对象；
* 值语义，即引用传递、值传递对象；
* now()获取当前时间；
* microSecondsSinceEpoch()获取微秒数，secondsSinceEpoch获取秒数()；
* valid()判断对象是否有效；
* fromUnixTime() 将Epoch时间转换为Timestamp对象；
* toString() 将时间戳转换为string类型；
* toFormattedString() 将时间戳转换为格式化字符串string类型；

辅助函数主要功能点：

* operator< 比较2个Timestamp对象大小；
* timeDifference() 计算2个Timestamp对象差值；
* addTime() 将一个Timestamp加上指定时间；

由于toString()和 toFormattedString() 可以输出类的信息，因此可以作为测试时判断的依据。
