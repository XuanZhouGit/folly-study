
# 1 简介
Folly(Facebook Open-source Library)是facebook的类似std, boost的基础组件库，其中使用了很多c++ 11的新特性，主要目的是为了满足大规模高性能的需求。其中promise/future是folly实现的一个高性能的异步组件, c++ 11中std, boost都有promise/future的实现， 但std, boost中提供的接口都比较简单，Folly提供的接口比较多，支持via, then等方法，使链式调用更加方便，可读性更强

# 2 例子
## 2.1 makeFuture
```
string func1(int x) {
    return to_string(x);
}
void func2(const string& str) {
    cout << str << endl;
}
  
cout << "makeFuture" << endl;
auto f = makeFuture<int>(42).thenMulti(func1, func2);
cout << "Future ready? " << f.isReady()  << endl;
```

## 2.2 Future with Promise
```
cout << "making Promise" << endl;
Promise<int> p;
auto f = p.getSemiFuture();
folly::InlineExecutor e;
f.via(&e).then([](int x){ cout << "foo(" << x << ")" << endl; });
cout << "Future chain made" << endl;
cout << "Future ready? " << f.isReady() << endl;
cout << "fulfilling Promise" << endl;
p.setValue(42);
cout << "Future ready? " << f.isReady() << endl;
```
## 2.3 map/reduce
```
vector<int> vec = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
vector<Future<int>> futures;
for (int i : vec) {
    futures.push_back(makeFuture<int>(std::move(i)));
}
auto results = futures::map(futures, [](int val) {
    return val * 2;
});
auto f = reduce(results, 0, [](int sum, int&& val){
    return sum + val;
});
cout << f.get() << endl;
```
## 2.4 collect
```
vector<int> vec = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
SharedPromise<vector<int>> p;
CPUThreadPoolExecutor pool(0, 2);
auto f1 = p.getFuture().via(&pool).then([](const vector<int>& vals) {
    string str;
    for (int val : vals) {
        str += to_string(val) + " + ";
    }
    str.resize(str.size() - 3);
    str += " = ";
    return str;
});
auto f2 = p.getFuture().via(&pool).then([](const vector<int>& vals) {
    auto sum = 0;
    for (int val : vals) {
        sum += val;
    }
    return sum;
});
collectAll(f1, f2).then([](const tuple<Try<string>, Try<int>>& results){
    auto str = get<0>(results).value();
    str += to_string(get<1>(results).value());
    cout << str << endl;
});
p.setValue(vec);
pool.setNumThreads(2);
pool.join();
```

# 3 数据结构
## 3.1 总体结构
Format: ![structure](https://github.com/XuanZhouGit/folly-study/blob/master/folly1.PNG)
