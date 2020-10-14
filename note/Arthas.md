### trace

`trace` 命令能主动搜索 `class-pattern`／`method-pattern` 对应的方法调用路径，渲染和统计整个调用链路上的所有性能开销和追踪调用链路。

```sh
$ trace demo.MathGame run
# -n 100  捕捉到100次调用就退出
$ trace demo.MathGame run  -n 100
# --skipJDKMethod false  trace jdk里的函数
$ trace --skipJDKMethod false demo.MathGame run
# '#cost > 10' 展示耗时大于10ms的调用路径
$ trace demo.MathGame run '#cost > 10'
```

### JAD

反编译

```shell
jad --source-only com.nike.ctm.replenish.application.impl.ReplenishProposalServiceImpl
```



### redefine

重新加载class

~~~sh
redefine /tmp/com/example/demo/arthas/user/UserController.class
~~~

