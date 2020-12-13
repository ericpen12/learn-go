# pprof

## pprof 采集数据的方式

-   **runtime/pprof**: 手动调用`runtime.StartCPUProfile`或者`runtime.StopCPUProfile`等 API来生成和写入采样文件，灵活性高
-   **net/http/pprof**: 通过 http 服务获取Profile采样文件，简单易用，适用于对应用程序的整体监控。通过 runtime/pprof 实现
-   **go test**: 通过 `go test -bench . -cpuprofile prof.cpu`生成采样文件 适用对函数进行针对性测试



## pprof 分析

```
go tool pprof binary profile	
```



## 常用命令

*   top(n): 查看前 n位信息，默认为10 可以使用 -cum 以 cum 排序
*   list func: 查看函数的信息
*   web: 生成 svg 可在浏览器中打开

## 参考资料

https://juejin.cn/post/6844904079525675016#heading-9