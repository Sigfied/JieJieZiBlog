# Doris 实践

- What -> Why -> How -> Never

## 修改表结构失败
### 无法变更问题？
- 在这张 Mini_apm_err_with_index 上试图修改 err_errmsg (varchar 255 ) 字段的类型为 text 类型
  ![](表结构-1.PNG)
  执行 Doris 的修改表结构语句,出现异常

```sql   
  ALTER TABLE db_demo.mini_apm_err_with_index
  MODIFY COLUMN err_errmsg text
  doris Table[xxxx]‘s state is not NORMAL. Do not allow doing ALTER ops
```

### 原因及修复手段
doris 建好表后对表结构进行修改，使用alter语句修改，但多个alter执行就会报 
> **Table[xxxx]'s state is not NORMAL. Do not allow doing ALTER ops** 。

这是因为一个表同时只能进行一个schema chanage任务。

可以使用 show 命令查看正在执行的表结构变更任务

```sql
SHOW ALTER TABLE COLUMN
```

![](插入执行情况-1.PNG)

<br/>
Mini_apm_err_with_index 表在执行修改语句前有大量数据，新增一个字段（列）需要一定时间等待变更，所以在变更期间再次变更则会报错，等待一段时间就可。

## 数据插入失败
### Java 层程序
```Java
public int dorisStreamLoad(List<?> list, String tableName) {

    try {
        JSONObject res = JSONObject.parseObject(sendData(JSON.toJSONString(list), tableName));
        return res.getInteger("NumberLoadedRows") == list.size() ? list.size() : 0;
    } catch (Exception e) {
     
    }
    return 0;
}
```

调用 Doris 的 HTTP 接口，实现数据的批量导入，只需要把数据存在一个 List 里面。在发送的时候选择使用 JSON 的格式发送。
目前的使用过程中，发送 JSON 的数据，长度为 10000 的 JSON 数组。耗时在2.7s ~ 3.8s 之间

> 后续的优化思路 : 可以尝试试用多线程去做数据的导入，留一个线程池来做这个的调度。
```Java
private String sendData(String content, String tableName) throws Exception {
    final String loadUrl = String.format("http://%s:%s/api/%s/%s/_stream_load",
                              DORIS_HOST,
                              DORIS_HTTP_PORT,
                              DORIS_DB,
                              tableName);

    final HttpClientBuilder httpClientBuilder = HttpClients
            .custom()
            .setRedirectStrategy(new DefaultRedirectStrategy() {
                @Override
                protected boolean isRedirectable(String method) {
                    return true;
                }
            });

    try (CloseableHttpClient client = httpClientBuilder.build()) {
        HttpPut put = new HttpPut(loadUrl);
        StringEntity entity = new StringEntity(content, "UTF-8");
        put.setHeader(HttpHeaders.EXPECT, "100-continue");
        put.setHeader(HttpHeaders.AUTHORIZATION, basicAuthHeader(DORIS_USER, DORIS_PASSWORD));

        //使用 JSON 格式
        put.setHeader("format", "json");
        put.setHeader("strip_outer_array", "true");


        put.setEntity(entity);

        try (CloseableHttpResponse response = client.execute(put)) {
            String loadResult = "";
            if (response.getEntity() != null) {
                loadResult = EntityUtils.toString(response.getEntity());
            }
            final int statusCode = response.getStatusLine().getStatusCode();
            if (statusCode != 200) {
                log.error(String.format("Stream load failed, statusCode=%s load result=%s", statusCode, loadResult));
            }
            log.info("Stream load result: {}", loadResult);
            System.out.println(loadResult);
            return JSONObject.toJSONString(loadResult.replace("\n", ""));
        }
    }
}

```

有时会出现数据插入失败的情况

```JSON
{
  "TxnId": 135783,
  "Label": "baa81be2-80e7-4745-a523-2df3e1f9d508",
  "Comment": "",
  "TwoPhaseCommit": "false",
  "Status": "Fail",
  "Message": "[INTERNAL_ERROR]too many filtered rows\n0. /opt/tiger/compile_path/src/code.byted.org/emr/doris/be/src/common/stack_trace.cpp:302: StackTrace::tryCapture() @ 0x000000000ba2b457 in /opt/emr/current/doris2/be/lib/doris_be\n1. /opt/tiger/compile_path/src/code.byted.org/emr/doris/be/src/common/stack_trace.h:0: doris::get_stack_trace[abi:cxx11]() @ 0x000000000ba299ed in /opt/emr/current/doris2/be/lib/doris_be\n2. /var/local/ldb-toolchain/bin/../lib/gcc/x86_64-linux-gnu/11/../../../../include/c++/11/bits/basic_string.h:187: doris::Status doris::Status::Error<true>(int, std::basic_string_view<char, std::char_traits<char> >) @ 0x000000000aec30eb in /opt/emr/current/doris2/be/lib/doris_be\n3. /opt/tiger/compile_path/src/code.byted.org/emr/doris/be/src/common/status.h:348: std::_Function_handler<void (doris::RuntimeState*, doris::Status*), doris::StreamLoadExecutor::execute_plan_fragment(std::shared_ptr<doris::StreamLoadContext>)::$_0>::_M_invoke(std::_Any_data const&, doris::RuntimeState*&&, doris::Status*&&) @ 0x000000000b91ccc9 in /opt/emr/current/doris2/be/lib/doris_be\n4. /var/local/ldb-toolchain/bin/../lib/gcc/x86_64-linux-gnu/11/../../../../include/c++/11/bits/unique_ptr.h:360: doris::FragmentMgr::_exec_actual(std::shared_ptr<doris::FragmentExecState>, std::function<void (doris::RuntimeState*, doris::Status*)> const&) @ 0x000000000b82662c in /opt/emr/current/doris2/be/lib/doris_be\n5. /var/local/ldb-toolchain/bin/../lib/gcc/x86_64-linux-gnu/11/../../../../include/c++/11/bits/shared_ptr_base.h:701: std::_Function_handler<void (), doris::FragmentMgr::exec_plan_fragment(doris::TExecPlanFragmentParams const&, std::function<void (doris::RuntimeState*, doris::Status*)> const&)::$_0>::_M_invoke(std::_Any_data const&) @ 0x000000000b831e79 in /opt/emr/current/doris2/be/lib/doris_be\n6. /opt/tiger/compile_path/src/code.byted.org/emr/doris/be/src/util/threadpool.cpp:0: doris::ThreadPool::dispatch_thread() @ 0x000000000ba6806f in /opt/emr/current/doris2/be/lib/doris_be\n7. /var/local/ldb-toolchain/bin/../usr/include/pthread.h:562: doris::Thread::supervise_thread(void*) @ 0x000000000ba5dffc in /opt/emr/current/doris2/be/lib/doris_be\n8. start_thread @ 0x0000000000007fa3 in /usr/lib/x86_64-linux-gnu/libpthread-2.28.so\n9. clone @ 0x00000000000f906f in /usr/lib/x86_64-linux-gnu/libc-2.28.so\n",
  "NumberTotalRows": 4064,
  "NumberLoadedRows": 4012,
  "NumberFilteredRows": 52,
  "NumberUnselectedRows": 0,
  "LoadBytes": 28064265,
  "LoadTimeMs": 383,
  "BeginTxnTimeMs": 0,
  "StreamLoadPutTimeMs": 7,
  "ReadDataTimeMs": 18,
  "WriteDataTimeMs": 374,
  "CommitAndPublishTimeMs": 0,
  "ErrorURL": "http://10.86.0.16:8040/api/_load_error_log?file=__shard_368/error_log_insert_stmt_a84b646af54b5d12-6acaf52ab86c70aa_a84b646af54b5d12_6acaf52ab86c70aa"
}
```

![image.png](image.png)

根据的报错中提示的 URL 查看日志，不难发现，是因为表中没有这个分区才导致数据的插入失败。
Doris 在没有分区的情况下，做这个分区的插入会出现异常而无法插入

### 解决

![image_1.png](创建分区.png)

在调度系统中重新创建分区即可。

## 创建分区
首先 Doris 创建的分区是左闭右开的区间 [  dt ，dt+++ ) 类似这样的
所以可以写出创建分区的 SQL，在调度系统里面传递参数 : dt 和 dt_int

```SQL
ALTER TABLE db_demo.perf_apm_err_with_index_new
ADD PARTITION IF NOT EXISTS p${dt}
VALUES  [("${dt}"),("${dt_int}"))
```

${dt_int} 为当前日期的 + 1 天，也就是后一天。
比如今天 2024-02-05，则创建 **[ 2024-02-05 , 2024-02-06 )** 区间的分区，相当于精确到了天

分区区间重叠导致创建失败
在我们早期的分区创建尝试中，使用了 less than 关键字，这可以批量创建分区

### Less than ｜ 仅定义分区上界。下界由上一个分区的上界决定

```SQL
PARTITION BY RANGE(col1[, col2, ...])
(
  PARTITION partition_name1 VALUES LESS THAN MAXVALUE|("value1", "value2", ...),
  PARTITION partition_name2 VALUES LESS THAN MAXVALUE|("value1", "value2", ...)
)
```
当我们将其放在调度系统上运行时，将分区名改成  { p+时间 }的做法，后续我们在 debug 的时候，查看分区的时候很难通过分区名来看分区区间。

而且这样也不能满足咱们每天的数据放在当天的分区上。在后续的调试中，会遇到分区区间因为重叠而导致创建失败
比如：已有分区区间 **P1  :  [ 2024-02-05 , 2024-02-10 )** ，
创建一新的分区区间 **P2  :  [ 2024-02-06 , 2024-02-11 )** ，则会报出分区区间重叠而导致数据无法插入


### MULTI RANGE ｜ 批量创建RANGE分区，定义分区的左闭右开区间，设定时间单位和步长

时间单位支持年、月、日、周和小时，这个方式看起来还不错哈，可以批量的创建分区，还能精确到更细的场景
```SQL
PARTITION BY RANGE(col)
(
  FROM ("2000-11-14") TO ("2021-11-14") INTERVAL 1 YEAR,
  FROM ("2021-11-14") TO ("2022-11-14") INTERVAL 1 MONTH,
  FROM ("2022-11-14") TO ("2023-01-03") INTERVAL 1 WEEK,
  FROM ("2023-01-03") TO ("2023-01-14") INTERVAL 1 DAY
)
```
但是，这个创建分区的方式仅能在 Create Table 的时候创建，不能在调度系统中以 Alter Table 的方式新增分区
