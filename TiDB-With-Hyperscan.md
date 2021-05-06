# TiDB + Hyperscan

## 缘起

之前在研究 Suricata 的时候才知道有多模匹配这么个概念，然后接触到了 Hyperscan 这个库。其实对于多模匹配来说，非常成熟的开源库并不多，而且用起来还是需要动手写一些代码。后来就想到能不能在 TiDB 上面集成 Hyperscan，增加一些内建函数来简化多模匹配的使用。

当然，集成 Hyperscan 库的数据库也是有的，比如 Clickhouse。

## 设想

Hyperscan 的用法主要分为2步：

1. 创建数据库
2. 使用数据库进行匹配

如果想把这两步放到数据库里面，就需要两类函数，一类用来创建 Hyperscan 数据库，另一类用来使用数据库对每行的指定列进行匹配，如果换成 SQL 的话，下面这样的例子可能更直观一些：

```sql
SELECT HS_MATCH(`data`, (SELECT HS_BUILDDB(`id`, `pattern`) FROM `pattern_table`)) as `matched` FROM `data_table`;
```
基于上面这个设想，就需要在 TiDB 中加入一个聚合函数来创建 Hyperscan 的数据库，然后提供一组 match 的标量函数来计算每行是否匹配。

## Scalar Function

在 TiDB 的 Blog 中已经有一篇文章讲述如何添加一个标量函数了。这里需要做的就是加入一系列 match 相关的函数实现。然后注册到全局变量 `funcs` 这个 map 中即可。

另外一个需要注意的是，每个 Scalar Function 都需要给他分配一个 ProtocolBuffer 的 Code。当然，按照正常的流程，需要调整 `tipb` 这个库，为新加入的 Scalar Function 指定一个值。如果不想动 `tipb` 这个库，可以参考 `tipb` 库里面定义的其他函数的 Code 值，然后在 TiDB 这边直接自定义即可。

## Aggregation Function

相较于 Scalar Function 已经在 Parser 层面做好了工作，如果要添加 Aggregation Function，则需要对 `parser` 库进行修改，从而让 Parser 认为 `hs_builddb` 是个聚合函数而不是一个标量函数。这需要在 `parser.y` 中给 `SumExpr` 加入新的函数描述：

```go
|	"HS_BUILDDB" '(' ExpressionList ')'
	{
		$$ = &ast.AggregateFuncExpr{F: ast.AggFuncHSBuildDB, Args: $3.([]ast.ExprNode)}
	}
```

当一个函数是聚合函数时，TiDB 的 Planner 在创建 Plan 的时候会使用 Agg 系列的任务（比如 StreamAgg）。另外，聚合函数的逻辑代码是在 `executor/aggfuncs` 包里，而不是在 `expression` 包里面。

在调整好 Parser 之后，就可以在 TiDB 这边实现 `hs_builddb` 函数了。相较于 Scalar Function 来说，Aggregation Function 的代码实现起来会比较分散。对于 `hs_builddb` 函数的逻辑代码需要实现 `AggFunc` 接口即可。这个接口只有 4 个函数，并不复杂，可以参考一些其他的聚合函数和接口函数的注释就可以解决。

而说 Aggregation Function 实现起来比较分散主要是在周边的配套设施：

1. Aggregation Function Builder (在`executor/aggfuncs/builder.go`中的`Build`函数)
2. Aggregation Function 的 Protocol Buffer 的互相转换 (在`expression/aggregation/agg_to_pb.go`的`AggFuncToPBExpr`和`PBExprToAggFuncDesc`函数) 
3. Aggregation Function 的执行类型推断（在`expression/aggregation/base_func.go`的`typeInfer`函数)
4. 针对 Aggregation Function Not Null 的处理（在`expression/aggregation/descriptor.go`的`UpdateNotNullFlag4RetType`函数）
5. 指定 Aggregation Function 是否支持 TiKV 下推（在`planner/core/rule_aggregation_push_down.go`的`isDecomposableWithJoin`和`isDecomposableWithUnion`函数)

在实现完 `AggFunc` 接口之后，只有找到这些配套设施的位置，加以修改才能让聚合函数正常运行。而且以上绝大部分的函数都是用 `swtich` 语句来区分聚合函数的。其实这对于条件编译并不友好。

## 条件编译

当然，并不是所有人都需要 Hyperscan，或者不是所有编译环境都有 Hyperscan 库，所以条件编译是必须的。好在 Golang 支持 `-tags` 参数来提供条件编译的可能性。那么剩下的是如何设计代码让 TiDB 可以根据环境变量开启或者关闭 Hyperscan 函数的支持。

首先，可以在 go 文件中加入以下注释来让这个文件在有对应的 tag 之后才进行编译：

``` go
// +build hyperscan
```

毕竟不像 C 那样可以通过 `#ifdef` 来对某行代码进行条件编译，Go 能控制的粒度只在文件级别。所以配合 `init` 函数向一个全局的 map 变量增加和减少某些 key 实现条件编译是一个比较合适的办法。

对于标量函数来说，全局变量 `funcs` map 可以通过 `init` 函数配合条件编译来选择性加入或者不加入 Hyperscan 函数的支持。

``` go
// Add hyperscan builtin functions
func init() {
	funcs[HSBuildDBJSON] = &hsBuildDbJSONFunctionClass{baseFunctionClass{HSBuildDBJSON, 1, 2}}
	funcs[HSMatch] = &hsMatchFunctionClass{baseFunctionClass{HSMatch, 2, 3}}
	funcs[HSMatchJSON] = &hsMatchJSONFunctionClass{baseFunctionClass{HSMatchJSON, 2, 2}}
	funcs[HSMatchAll] = &hsMatchAllFunctionClass{baseFunctionClass{HSMatchAll, 2, 3}}
	funcs[HSMatchAllJSON] = &hsMatchAllJSONFunctionClass{baseFunctionClass{HSMatchAllJSON, 2, 2}}
	funcs[HSMatchIds] = &hsMatchIdsFunctionClass{baseFunctionClass{HSMatchIds, 2, 3}}
	funcs[HSMatchIdsJSON] = &hsMatchIdsJSONFunctionClass{baseFunctionClass{HSMatchIdsJSON, 2, 2}}
}
```

但对于聚合函数来说，需要对某些 “**周边的配套设施**” 进行一些修改才好支持条件编译。把 `switch` 语句换成 map 是一种方法，比如针对 Builder 可以在最终返回的时候加入一个针对扩展函数的处理，然后用一个 map 的全局变量来进行注册扩展的聚合函数：

``` go
type aggFuncBuilderFunc func(ctx sessionctx.Context, aggFuncDesc *aggregation.AggFuncDesc, ordinal int) AggFunc

var (
	extensionAggFuncBuilders = map[string]aggFuncBuilderFunc{}
)

func buildFromExtensionAggFuncs(ctx sessionctx.Context, aggFuncDesc *aggregation.AggFuncDesc, ordinal int) AggFunc {
	buildFunc, have := extensionAggFuncBuilders[aggFuncDesc.Name]
	if !have {
		return nil
	}
	return buildFunc(ctx, aggFuncDesc, ordinal)
}

// Build is used to build a specific AggFunc implementation according to the
// input aggFuncDesc.
func Build(ctx sessionctx.Context, aggFuncDesc *aggregation.AggFuncDesc, ordinal int) AggFunc {
	switch aggFuncDesc.Name {
	case ast.AggFuncCount:
		return buildCount(aggFuncDesc, ordinal)
	...
	case ast.AggFuncStddevSamp:
		return buildStddevSamp(aggFuncDesc, ordinal)
	}
	return buildFromExtensionAggFuncs(ctx, aggFuncDesc, ordinal)
}
```

## Scratch 复用

Hyperscan 有一个概念叫 Scratch，每个线程可以创建一个 Thread Local 的 Scratch 对象，可以在循环调用 match 族函数的时候复用，以提高匹配的性能。所以在 Scalar Function 针对多行数据做匹配操作的时候，一次性创建一个 Scratch 对象然后在循环中使用这个 Scratch 对象可以大大提高匹配的速度（实测每次匹配时创建 Scratch 然后销毁对性能的影响非常大）。

但是 Scratch 的文档中明确说明不能多线程共享，好在 TiDB 并不会用多 goroutine 并行执行 Scalar Function（与 PingCAP 的同学确认的），所以并不存在 Scratch 被多线程共享的问题。所以在 Scalar Function 在第一次调用 evalXXX 函数的时候初始化好 Scratch 对象以供后续调用复用就可以了。

## 内存清理
重复分配 Scratch 的性能问题解决了，但是还有个问题：Scalar Function 对象本身并不知道什么时候 SQL 执行完毕，也就没法做确定性资源清理。由于 TiDB 的其他 Scalar Function 并不涉及到资源回收的问题，也就没有对应的回调函数告知 Scalar Function 对象它所对应的 SQL 语句已经执行完毕。但是 Hyperscan 的 Database 和 Scratch 是要明确调用 `Close()` 和 `Free()` 函数来回收资源的。放任不管肯定是会内存泄漏的。

好在 Golang 的 runtime 包提供了 `SetFinalizer` 函数可以在 GC 销毁某个对象时调用一个回调函数。因此对于 Hyperscan 的 Database 和 Scratch 就可以使用下面的方式进行资源的回收：

```go
func (b *baseBuiltinHsSig) initScratch() error {
	if b.scratch != nil {
		return nil
	}
	scratch, err := hs.NewScratch(b.db)
	if err != nil {
		return err
	}
	b.scratch = scratch
	runtime.SetFinalizer(scratch, func(hsScratch *hs.Scratch) {
		hsScratch.Free()
	})
	return nil
}

func buildBlockDBFromBase64(base64Data string) (hs.BlockDatabase, int, error) {
	data, err := base64.StdEncoding.DecodeString(base64Data)
	if err != nil {
		return nil, 0, err
	}
	db, err := hs.UnmarshalBlockDatabase(data)
	runtime.SetFinalizer(db, func(hsdb hs.BlockDatabase) {
		hsdb.Close()
	})
	return db, 0, err
}
```

## 静态链接

默认情况下，通过 Homebrew 或者 rpm，apt 安装的 Hyperscan 的开发库一般启用的都是动态链接。但对于 Golang 来说静态链接才叫“讲究”（总不想在分发和部署的时候还得为 hyperscan 的 so 库文件所在位置操心吧）。

为了能做静态链接，就需要自己编译一下 Hyperscan 的代码，并在编译时指定使用静态链接。但问题并不会这么简单，在做 TiDB 的 Hyperscan 集成时，使用了 `github.com/flier/gohs` 这个库。这个库对于动态链接来说，一点问题都没有。但是用静态链接模式编译 TiDB 的时候就会在连接时候报错。然后你就会看到连接器打印了一大堆的 C++ 的符号找不到的日志。

从报错信息上可以了解到 Go 的连接过程用的是 C 的连接器，而不是 C++，但是找了一圈没有发现有什么命令行参数让 Go 在连接的时候用 g++，直到看到了 gohs 的 [Issue](https://github.com/flier/gohs/issues/27), 然后自己 fork 一个 gohs，加入了一个 dummy.cxx 的空文件，一切正常了。

后来翻了一些文档和帖子才知道 Golang 会通过文件扩展名来判断用 C 还是 C++ 的连接器。

## 总结

最终可以得到一个 TiDB 支持：

* 条件编译可以选择是否让 TiDB 在运行时支持 Hyperscan 系列函数
* 静态链接可以不用跟随发布 Hyperscan 的动态链接库
* 一组 Scalar Function 可以对字符串进行多模式匹配
* 一个 Aggregation Function 可以方便的构建 Hyperscan 的数据库

当然，用起来会是下面的样子：

```sql
mysql> select * from patterns;
+------+---------+
| id   | pattern |
+------+---------+
|    1 | abc     |
|    2 | def     |
|    3 | test    |
+------+---------+
3 rows in set (0.00 sec)

mysql> select line, hs_match_ids(line, (select hs_builddb(id, pattern) from patterns), 'base64') as `matched_ids` from data;
+-----------+-------------+
| line      | matched_ids |
+-----------+-------------+
| abc def   | 1,2         |
| some test | 3           |
| no matche |             |
+-----------+-------------+
3 rows in set (0.01 sec)
```

如果想看到更详细的实现，可以通过这个 [PR](https://github.com/pingcap/tidb/pull/23497) 查看。