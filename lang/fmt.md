
# Dynamically pass values to Sprintf or Printf

原打算在[httplog](https://github.com/bingoohuang/httplog)中，也像[java版本](https://github.com/gobars/httplog)那样，在日志文件中输出时，增加一个abbrevMaxSize支持（默认64）。

对代码分析了一下，这个不是标准动作，不是简单的几行改动就可以的，得改动格式化的地方，或者JSON序列化的地方才行。打算暂时不动，等真正有需求再说。

分析的过程中，顺便了解了一下fmt包的实现。然后想到上次动态宽度的实现。

以前都是"曲线救国"，先拼fmt，然后再打印。知道看到[这段代码](https://github.com/bingoohuang/sqlite3perf/blob/master/generate.go#L187)

```go
log.Printf("%*d/%*d (%6.2f%%) written in %s, avg: %s/record, %2.2f records/s",
    l, g.NumRecs, l, g.NumRecs, p*float64(g.NumRecs), dur,
    time.Duration(dur.Nanoseconds()/int64(g.NumRecs)), float64(g.NumRecs)/dur.Seconds())
```

才发现，不需要曲线救国，直接一行代码就可以干了，一般的fmt的代码示例，还真没有这种。

## 取两位精度后，自动去除.00

一般也是曲线救国来完成，搜索了一下，fmt没办法直接支持这种。

https://play.golang.org/p/BeFr0ixJR3i

```go
// FormatFloat format a float64 with precision without tailing zeros and dot.
// eg:
// FormatFloat(1.000001, 2) => 1
// FormatFloat(1.123456, 2) => 1.12
// FormatFloat(0.123456, 2) => 0.12
// FormatFloat(0.100001, 2) => 0.1
func FormatFloat(num float64, prc int) string {
	return strings.TrimRight(strings.TrimRight(fmt.Sprintf("%.*f", prc, num), "0"), ".")
}
```

## 更多解释

[StackOverflow](https://stackoverflow.com/a/42308274)

Go to: https://golang.org/pkg/fmt/ and scroll down until you find this:

`fmt.Sprintf("%[3]*.[2]*[1]f", 12.0, 2, 6)`

equivalent to

`fmt.Sprintf("%6.2f", 12.0)`

will yield " 12.00". Because an explicit index affects subsequent verbs, this notation can be used to print the same values multiple times by resetting the index for the first argument to be repeated

The real core of the description of using arguments to set field width and precision occurs further above:

Width and precision are measured in units of Unicode code points, that is, runes. (This differs from C's printf where the units are always measured in bytes.) Either or both of the flags may be replaced with the character '*', causing their values to be obtained from the next operand, which must be of type int.

The example above is just using explicit indexing into the argument list in addition, which is sometimes nice to have and allows you to reuse the same width and precision values for more conversions.

So you could also write:

`fmt.Sprintf("*.*f", 6, 2, 12.0)`
