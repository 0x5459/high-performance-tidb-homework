# 课程01 作业

## 环境
- Ubuntu 20.04
- go 1.15
- rust 1.45.1

## 下载源代码
```shell
make tidb && cd tidb
git clone https://github.com/pingcap/tidb.git
git clone https://github.com/pingcap/pd.git
git clone https://github.com/tikv/tikv.git
```

## 改写
打开 tidb 源代码， 尝试搜索 Begin，发现 kv/kv.go 中存在一个 Storage 接口， 该接口中包含 `Begin() (Transaction, error)` 和 `BeginWithStartTS(startTS uint64) (Transaction, error)` 方法。 

于是寻找包含这个接口的结构体，找到了 session/session.go 中的 session 结构体。研究了一会 session.go。发现 session 结构体中的 `txn TxnState` 字段很可疑。遂打开其源码，看到一行注释:

> It's a lazy transaction, that means it's a txnFuture before StartTS() is really need.

感觉很可能就需要改动这里。
```go
// session/txn.go 
func (tf *txnFuture) wait() (kv.Transaction, error) {
	startTS, err := tf.future.Wait()
	if err == nil {
        // 在这里打印 hello transaction
        logutil.BgLogger().Info("hello transaction")
		return tf.store.BeginWithStartTS(startTS)
	} else if config.GetGlobalConfig().Store == "unistore" {
		return nil, err
	}

	logutil.BgLogger().Warn("wait tso failed", zap.Error(err))
	// It would retry get timestamp.
	return tf.store.Begin()
}
```

## 编译

```shell
cd tidb
make

cd ../pd
make

cd ../tikv
make build
```

## 运行
结合 [官方文档](https://docs.pingcap.com/zh/tidb/stable/command-line-flags-for-tidb-configuration) 和自己摸索源代码，才最终能够成功运行。

### 1. 运行 pd
```shell
nohup pd/bin/pd-server --name="pd" \
          --data-dir="data/pd" \
          --client-urls="http://127.0.0.1:2379" \
          --peer-urls="http://127.0.0.11:2380" \
          --log-file="log/pd.log" >/dev/null 2>&1 &
```

### 2. 运行 tikv

```shell
nohup tikv/target/debug/tikv-server --log-file="log/tikv1.log" \
                --addr="127.0.0.1:20160" \
                --data-dir="data/tikv1" \
                --pd-endpoints="http://127.0.0.1:2379" >/dev/null 2>&1 &

nohup tikv/target/debug/tikv-server --log-file="log/tikv2.log" \
                --addr="127.0.0.1:20161" \
                --data-dir="data/tikv2" \
                --pd-endpoints="http://127.0.0.1:2379" >/dev/null 2>&1 &

nohup tikv/target/debug/tikv-server --log-file="log/tikv3.log" \
                --addr="127.0.0.1:20162" \
                --data-dir="data/tikv3" \
                --pd-endpoints="http://127.0.0.1:2379" >/dev/null 2>&1 &
```

### 3. 运行 tidb
```shell
nohup ./tidb/bin/tidb-server --store="tikv" \
                --path="127.0.0.1:2379" \
                --log-file="log/tidb.log" >/dev/null 2>&1 &
```

## 测试

查看 TIDB 日志 `tail -f log/tidb.log` 发现每个几秒就会打印若干 `hello transaction` 日志。 猜测可能是 TIDB 内部执行的 sql 语句。忽略之~

连接 TIDB `mysql -h127.0.0.1 -P 4000 -uroot -proot` 任意执行 sql 能看到日志输出。
```
[2020/08/15 20:29:07.905 +08:00] [INFO] [txn.go:372] ["hello transaction"]
```
