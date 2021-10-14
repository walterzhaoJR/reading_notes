# brpc中bvar的介绍
## 1. 讨论
我们首先考虑一个这样的需求：在多线程场景中，需要统计线上服务的一些qps、latency、error等数据。本质上就是需要实现一个多线程场景下的计数器类库，用于记录和查询程序中的各类统计数值。

在实现这个计数器类库时，可能会遇到一些性能的问题：多个线程在统计累加一些值的时候，可能需要使用锁来保护临界区，这样可能产生严重的竞争问题。这里我们讨论一些背景知识：为了以较低的成本大幅提高性能，现代CPU都会有cache。目前一般是三级缓存结构，其中L1和L2 cache为每个核独有，L3则所有核共享。为了保证所有的核看到正确的内存数据，一个核在写入自己的L1 cache后，CPU会执行Cache一致性算法把对应的cacheline(一般是64字节)同步到其他核。这个过程并不很快，是微秒级的，相比之下写入L1 cache只需要若干纳秒。当很多线程在频繁修改某个字段时，这个字段所在的cacheline被不停地同步到不同的核上，就像在核间弹来弹去，这个现象就叫做cache bouncing。由于实现cache一致性往往有硬件锁，cache bouncing可以看做是一种全局竞争。cache bouncing使访问频繁修改的变量的开销陡增，甚至还会使访问同一个cacheline中不常修改的变量也变慢，这个现象就是false sharing。按cacheline对齐能避免false sharing，但在某些情况下，我们甚至还能避免修改“必须”修改的变量。当很多线程都在累加一个计数器时，我们让每个线程累加私有的变量而不参与全局竞争，在读取时我们累加所有线程的私有变量。虽然读比之前慢多了，但由于这类计数器的读多为低频展现，慢点无所谓。而写就快多了，从微秒到纳秒，几百倍的差距。

如果以多线程累加器这个例子来说，我们可以将累加全局的一个变量改变为每个线程累加自己的一个变量，等到需要的时候再做一次合并，让每个线程累加私有的变量而不参与全局竞争注意，这种实现方式的本质是把写时的竞争转移到了读：读得合并所有写过的线程中的数据，这里是不可避免地变慢了。当读写都很频繁并得基于数值做一些逻辑判断时，就不是很适用了，但是对于写多，读少的场景还是有性能提升的。这里所说的线程局部的变量，也就是我们常说的 thread local存储。

## 2. bvar
bvar是多线程环境下的计数器类库，方便记录和查看用户程序中的各类数值，它利用了thread local存储减少了cache bouncing，brpc集成了bvar，/vars可查看所有曝光的bvar，/vars/VARNAME可查阅某个bvar。brpc大量使用了bvar提供统计数值，当你需要在多线程环境中计数并展现时，可以很好的使用bvar。但bvar不能代替所有的计数器，它的本质是把写时的竞争转移到了读。当你读写都很频繁或得基于最新值做一些逻辑判断时，你不应该用bvar。

bvar是一个相对独立的组件，不依赖外部，你可以只使用bvar而不用将整个brpc库接入自己的工程。并且可以在brpc的web页面上查看bvar的数值，注意这里还聚合统计了过去一分钟的每一秒，每一分钟，每一小时，每一天（做多30天）的值。如图：
![](/Users/walterzhao/Desktop/bvar_web.png)

## 3. bvar代码细节
bvar有两个最基本的两个成员，一个是name，另一个是统计值，存储统计值的内存空间是从本类型的TLS内存池申请而来。

### 3.1 基类 Variable
他主要提供全局key map注册自定义的bvar，并且加入了列举，查询等功能，主要的成员：std::string _name。主要的一些方法：
```c++
    // Implement this method to print the variable into ostream.
    virtual void describe(std::ostream&, bool quote_string) const = 0; // 实现这个方法将bvar输出到ostream中

    // Describe saved series as a json-string into the stream.
    // The output will be ploted by flot.js
    // Returns 0 on success, 1 otherwise(this variable does not save series).
    virtual int describe_series(std::ostream&, const SeriesOptions&) const // 输出一些可以用于绘图的指标的数据（见上图）主要是表了过去60秒、过去60分钟、过去24小时、过去30天一共174个点的的统计值

    // 将bvar对外可见，也就是‘曝光’
    int expose(const butil::StringPiece& name,
               DisplayFilter display_filter = DISPLAY_ON_ALL) {
        return expose_impl(butil::StringPiece(), name, display_filter);
    }

    // Find an exposed variable by `name' and put its description into `os'.
    // Returns 0 on found, -1 otherwise.
    static int describe_exposed(const std::string& name,
                                std::ostream& os,
                                bool quote_string = false,
                                DisplayFilter = DISPLAY_ON_ALL); // 通过名字将曝光的bvar输出到os中

    // Describe saved series of variable `name' as a json-string into `os'.
    // The output will be ploted by flot.js
    // Returns 0 on success, 1 when the variable does not save series, -1
    // otherwise (no variable named so).
    static int describe_series_exposed(const std::string& name,
                                       std::ostream&,
                                       const SeriesOptions&); // 通过名字将保存的bvar的统计值数组输出到os

    inline ostream& operator<<(ostream &os, const ::bvar::Variable &var) { // 重载了 << 运算符，接受并聚合统计值
    var.describe(os, false);
    return os;
}
```

### 3.2 Variable的主要派生类
### 3.2.1 bvar::Reducer 
Reducer用二元运算符把多个值合并为一个值，运算符需满足结合律，交换律，没有副作用。只有满足这三点，才能确保合并的结果不受线程私有数据如何分布的影响。像减法就不满足结合律和交换律，它无法作为此处的运算符.

常见的Redcuer子类有bvar::Adder, bvar::Maxer, bvar::Miner。
* bvar::Adder 用于累加，主要实现了：
```c++
    template <typename Tp>
    struct AddTo {
    void operator()(Tp & lhs, 
                    typename butil::add_cr_non_integral<Tp>::type rhs) const
    { lhs += rhs; }
    };
    struct MinusFrom {
    void operator()(Tp & lhs, 
                    typename butil::add_cr_non_integral<Tp>::type rhs) const
    { lhs -= rhs; }
    };
    // bvar::Adder<int> sum;
    // sum << 1 << 2 << 3 << 4;
    // LOG(INFO) << sum.get_value(); // 10
```
Adder<>可用于非基本类型，对应的类型至少要重载`T operator+(T, T)`。一个已经存在的例子是std::string

* bvar::Maxer 用于取最大值，运算符为std::max。
```c++
    template <typename Tp> 
    struct MaxTo {
    void operator()(Tp & lhs, 
                    typename butil::add_cr_non_integral<Tp>::type rhs) const {
        // Use operator< as well.
        if (lhs < rhs) {
            lhs = rhs;
        }
        }
    };
    // bvar::Maxer<int> value;
    // value<< 1 << 2 << 3 << -4;
    // CHECK_EQ(3, value.get_value());
```
也可以重载operator<

* bvar::Miner 求最小值，类似于Maxer。

### 3.2.2 bvar::IntRecorder

主要用于求平均值
```c++
// For calculating average of numbers.
// Example:
//   IntRecorder latency;
//   latency << 1 << 3 << 5;
//   CHECK_EQ(3, latency.average());
```

其中Stat主要累加了总和和数据个数
```c++
struct Stat {
    Stat() : sum(0), num(0) {};
    Stat(int64_t sum2, int64_t num2) : sum(sum2), num(num2) {}
    int64_t sum;
    int64_t num;
        
    int64_t get_average_int() const {
        //num can be changed by sampling thread, use tmp_num
        int64_t tmp_num = num;
        if (tmp_num == 0) {
            return 0;
        }
        return sum / (int64_t)tmp_num;
    }
    double get_average_double() const {
        int64_t tmp_num = num;
        if (tmp_num == 0) {
            return 0.0;
        }
        return (double)sum / (double)tmp_num;
    }
    Stat operator-(const Stat& rhs) const {
        return Stat(sum - rhs.sum, num - rhs.num);
    }
    void operator-=(const Stat& rhs) {
        sum -= rhs.sum;
        num -= rhs.num;
    }
    Stat operator+(const Stat& rhs) const {
        return Stat(sum + rhs.sum, num + rhs.num);
    }
    void operator+=(const Stat& rhs) {
        sum += rhs.sum;
        num += rhs.num;
    }
};
```

### 3.2.3 bvar::Status
用于显示很少或定期更新的值。不同线程修改后立即可见。额外提供了**set_value** 方法可以修改变量值。

### 3.2.4 bvar::PassiveStatus
显示按需更新的值。这里通过传入一个用户回调函数getfn()，该回调被调用以生成值。按需显示值。在一些场合中，我们无法set_value或不知道以何种频率set_value，更适合的方式也许是当需要显示时才打印。用户传入的回调函数实现这个目的。

```c++
PassiveStatus(const butil::StringPiece& name,
                  Tp (*getfn)(void*), void* arg)
        : _getfn(getfn)
        , _arg(arg)
        , _sampler(NULL)
        , _series_sampler(NULL) {
        expose(name);
    }
```

PassiveStatus很有用，因为很多统计量已经存在，并且是不怎么变化的，我们不需要再次存储它们，而只要按需获取。比如下面的代码声明了一个在linux下显示进程用户名的bvar：

```c++
static void get_username(std::ostream& os, void*) {
    char buf[32];
    if (getlogin_r(buf, sizeof(buf)) == 0) {
        buf[sizeof(buf)-1] = '\0';
        os << buf;
    } else {
        os << "unknown";
    }
}
PassiveStatus<std::string> g_username("process_username", get_username, NULL);
```

### 3.2.5 bvar::Window
获得之前一段时间内的统计值。Window不能独立存在，必须依赖于一个已有的计数器。Window会自动更新，不用给它发送数据。出于性能考虑，Window的数据来自于每秒一次对原计数器的采样，在最差情况下，Window的返回值有1秒的延时。Window本质上可以看做是若干Sample（样本）的集合。

```c++
 bool get_span(time_t window_size, detail::Sample<value_type>* result) const {
        return _sampler->get_value(window_size, result);
    }

    bool get_span(detail::Sample<value_type>* result) const {
        return get_span(_window_size, result);
    }

    virtual value_type get_value(time_t window_size) const {
        detail::Sample<value_type> tmp;
        if (get_span(window_size, &tmp)) {
            return tmp.data;
        }
        return value_type();
    }

    value_type get_value() const { return get_value(_window_size); }
```

可以想象以下场景：Adder是累加的，任何时候拿到的是累计值。如果需要统计一段时间內的数据，就可以使用windows来实现。

### 3.2.6 bvar::PerSecond
获得之前一段时间内平均每秒的统计值。它和Window基本相同，除了返回值会除以时间窗口之外。这里需要注意下PerSecond的使用场景，否则计算的数值是无意义的。例如：
```c++
bvar::Maxer<int> max_value;

// 错误！最大值除以时间是没有意义的
bvar::PerSecond<bvar::Maxer<int> > max_value_per_second_wrong(&max_value);

// 正确，把Window的时间窗口设为1秒才是正确的做法
bvar::Window<bvar::Maxer<int> > max_value_per_second(&max_value, 1);
```
bvar::PerSecond和bvar::Window的主要区别：Window统计一段时间內的数值。PerSecond计算Variabl每秒的平均值。比如要统计内存在上一分钟内的变化，用Window<>的话，返回值的含义是：上一分钟内存增加了18M，用PerSecond<>的话，返回值的含义是：上一分钟平均每秒增加了0.3M。Window的优点是精确值，适合一些比较小的量，比如：上一分钟的错误数，如果这用PerSecond的话，就会得到：上一分钟平均每秒产生了0.016个错误，这样并不合理。另外一些和时间无关的量也要用Window。Window 要求套用在其上的类型要实现get_sampler方法以获取采样器对象，从而获取样本值。
```c++
// R must:
// - have get_sampler() (not require thread-safe)
// - defined value_type and sampler_type
template <typename R, SeriesFrequency series_freq = SERIES_IN_WINDOW>
class Window : public detail::WindowBase<R, series_freq> {
    typedef detail::WindowBase<R, series_freq> Base;
    typedef typename R::value_type value_type;
    typedef typename R::sampler_type sampler_type;
public:
    // Different from PerSecond, we require window_size here because get_value
    // of Window is largely affected by window_size while PerSecond is not.
    Window(R* var, time_t window_size) : Base(var, window_size) {}
    Window(const butil::StringPiece& name, R* var, time_t window_size)
        : Base(var, window_size) {
        this->expose(name);
    }
    Window(const butil::StringPiece& prefix,
           const butil::StringPiece& name, R* var, time_t window_size)
        : Base(var, window_size) {
        this->expose_as(prefix, name);
    }
};
```

### 3.2.7 bvar::LatencyRecorder
LatencyRecorder由以上介绍的若干类型组合而成，（具有类似于FIO的长尾IO分布功能，80%、99%、99.5%、99.9%、99.99，以及CDF）是专用于计算latency和qps的计数器。只需填入latency数据，就能获得latency / max_latency / qps / count。
```c++
class LatencyRecorderBase {
public:
    explicit LatencyRecorderBase(time_t window_size);
    time_t window_size() const { return _latency_window.window_size(); }
protected:
    IntRecorder _latency;
    Maxer<int64_t> _max_latency;
    Percentile _latency_percentile;

    RecorderWindow _latency_window;
    MaxWindow _max_latency_window;
    PassiveStatus<int64_t> _count;
    PassiveStatus<int64_t> _qps;
    PercentileWindow _latency_percentile_window;
    PassiveStatus<int64_t> _latency_p1;
    PassiveStatus<int64_t> _latency_p2;
    PassiveStatus<int64_t> _latency_p3;
    PassiveStatus<int64_t> _latency_999;  // 99.9%
    PassiveStatus<int64_t> _latency_9999; // 99.99%
    CDF _latency_cdf;
    PassiveStatus<Vector<int64_t, 4> > _latency_percentiles;
};
```

## 4. bvar的采集
Sample 是具体bvar的采集样本，结构很简单，包含一个数据和一个时间戳。
```c++
template <typename T>
struct Sample {
    T data;
    int64_t time_us;

    Sample() : data(), time_us(0) {}
    Sample(const T& data2, int64_t time2) : data(data2), time_us(time2) {}  
};
```

Sampler 采样器的基类，通过定期的调用take_sample函数来收集采样值。
```c++
// The base class for all samplers whose take_sample() are called periodically.
class Sampler : public butil::LinkNode<Sampler> {
public:
    Sampler();
        
    // This function will be called every second(approximately) in a
    // dedicated thread if schedule() is called.
    virtual void take_sample() = 0;

    // Register this sampler globally so that take_sample() will be called
    // periodically.
    void schedule();

    // Call this function instead of delete to destroy the sampler. Deletion
    // of the sampler may be delayed for seconds.
    void destroy();
        
protected:
    virtual ~Sampler();
    
friend class SamplerCollector;
    bool _used;
    // Sync destroy() and take_sample().
    butil::Mutex _mutex;
};
```

SamplerCollector bvar将Variable的ReducerSampler成员挂载在SamplerCollector的队列中，启动后台线程，定期遍历队列，驱动Sampler采集Variable的值。bvar这里在实现的时候做了一个优化：每个采样器是本质是一个双向链接，因此可以循环地将多个采样器减少为一个双向链表，将多个列表合并成更大的列表。创建一个专用线程来定期 get_value()。
```c++
void SamplerCollector::run() {
    butil::LinkNode<Sampler> root;
    int consecutive_nosleep = 0;
#ifndef UNIT_TEST
    PassiveStatus<double> cumulated_time(get_cumulated_time, this);
    bvar::PerSecond<bvar::PassiveStatus<double> > usage(
            "bvar_sampler_collector_usage", &cumulated_time, 10);
#endif
    while (!_stop) {
        int64_t abstime = butil::gettimeofday_us();
        Sampler* s = this->reset();
        if (s) {
            s->InsertBeforeAsList(&root);
        }
        int nremoved = 0;
        int nsampled = 0;
        for (butil::LinkNode<Sampler>* p = root.next(); p != &root;) {
            // We may remove p from the list, save next first.
            butil::LinkNode<Sampler>* saved_next = p->next();
            Sampler* s = p->value();
            s->_mutex.lock();
            if (!s->_used) {
                s->_mutex.unlock();
                p->RemoveFromList();
                delete s;
                ++nremoved;
            } else {
                s->take_sample();
                s->_mutex.unlock();
                ++nsampled;
            }
            p = saved_next;
        }
        bool slept = false;
        int64_t now = butil::gettimeofday_us();
        _cumulated_time_us += now - abstime;
        abstime += 1000000L;
        while (abstime > now) {
            ::usleep(abstime - now);
            slept = true;
            now = butil::gettimeofday_us();
        }
        if (slept) {
            consecutive_nosleep = 0;
        } else {            
            if (++consecutive_nosleep >= WARN_NOSLEEP_THRESHOLD) {
                consecutive_nosleep = 0;
                LOG(WARNING) << "bvar is busy at sampling for "
                             << WARN_NOSLEEP_THRESHOLD << " seconds!";
            }
        }
    }
}
```

ReducerSampler 是Sampler的派生类，也是bvar Variable （针对Reducer、IntRecorder和Window类型）的成员，最重要的成员变量如下，ReducerSampler类型对Variable做每秒采样，后台线程调用ReducerSampler::take_sample()方法，驱动Reducer调用get_value()等将各线程聚合出结果，将结果存入自身确定大小的BoundedQueue中。

```c++
    time_t _window_size;
    butil::BoundedQueue<Sample<T> > _q;
```

这里可以结合ReducerSampler再解释下Window的使用。Window，本质上是将一段范围的历史值存起来，存就存在Variable的Sampler的Queue中，并且可以增大Queue的size。大部分Variable使用时不考虑Window方式，因此size设置为1就可以了。 对一个bvar::Maxer<int> max，获取从现在到之前1秒内的最大值，可以设置窗口为1，即bvar::Window<bvar::Maxer<int>> max_per_second(&max, 1)。如果获取从现在到之前1分钟内的最大值，设置窗口60，即bvar::Window<bvar::Maxer<int>> max_per_minute(&max, 60)。bvar::Window的默认窗口大小是bvar_dump_interval，默认10s，可自定义。窗口大小将设置ReducerSampler的BoundedQueue的大小，且只会增大，不会缩小window。

借助Window可以实现很多操作，例如要统计流量，首选可以使用bvar::Adder<uint64_t>进行过累加，我们拿到的是累计值。如果我们使用Window，则可以得到一段时间内的流量，因此Window类型的get_value(time_t window_size)方法是要指定时间维度的。

另外计算Variable的一段时间内每秒的平均值使用类型bvar::PerSecond<>。比如bvar::PerSecond<bvar::Adder<int>> average_per_second(&adder, 60)可以得到最近一分钟内的每秒平均值。

SeriesSampler SeriesSampler将Variable采样的值存成序列，内部Series分配60+60+24+30大小的空间，用于存储最近一分钟的每一秒，一小时中的每一分钟，一天中的每一小时和30天的采样值，很方便用于观察和统计。
```c++
struct Data {
    public:
        Data() {
            // is_pod does not work for gcc 3.4
            if (butil::is_integral<T>::value ||
                butil::is_floating_point<T>::value) {
                memset(_array, 0, sizeof(_array));
            }
        }
        
        T& second(int index) { return _array[index]; }
        const T& second(int index) const { return _array[index]; }

        T& minute(int index) { return _array[60 + index]; }
        const T& minute(int index) const { return _array[60 + index]; }

        T& hour(int index) { return _array[120 + index]; }
        const T& hour(int index) const { return _array[120 + index]; }

        T& day(int index) { return _array[144 + index]; }
        const T& day(int index) const { return _array[144 + index]; }
    private:
        T _array[60 + 60 + 24 + 30];
    };
```


## 5. 最后 
参考文档：https://github.com/apache/incubator-brpc/blob/master/docs/cn/bvar.md

bvar是一个性能高、功能全、设计好的性能采集库，本文主要对bvar中的一些数据结构进行了介绍，bvar还有很多细节值得探究，它的设计和实现值得深入学习。