# bechmark

## 说明

```
BenchmarkRandInt-8   	68453040	        17.8 ns/op
```

b.N run 了 68453040 次，平均每次for loop 耗时 17.8 ns



## flags 

[go doc](https://golang.org/cmd/go/#hdr-Testing_flags)

`-run`: 指定测试文件 

`go test -run=.Wav. -bench=. `

```
goos: darwin
goarch: amd64
pkg: wavtest
Benchmark_generateWav-8              146           8048843 ns/op
PASS
ok      wavtest 2.519s

```



`-count`: 运行 benchmark n 次，默认为1次

```
go test -run=._generateWav --bench=. -count=2
```

```
goos: darwin
goarch: amd64
pkg: wavtest
Benchmark_generateWav-8              139           8296433 ns/op
Benchmark_generateWav-8              139           8271049 ns/op
PASS
ok      wavtest 4.298s
```



### 分析和测试的指令

`-benchmem`: 打印内存分配信息

```
 go test -run=._generateWav --bench=. --benchmem
```

```
goos: darwin
goarch: amd64
pkg: wavtest
Benchmark_generateWav-8              140           8304707 ns/op         7246342 B/op     161268 allocs/op
PASS
ok      wavtest 2.132s
```

每次循环中分配内存 161268 次

`-memprofile mem.out`:所有测试通过后，将内存分配的信息写入文件