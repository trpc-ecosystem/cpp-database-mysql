

# cpp-database-mysql

MySQL数据库已经成为构建互联网世界的基石，尤其是其支持事务的特性，支撑了很多互联网的很多应用场景，比如电商、支付等。我们在tRPC-cpp中提供了以客户端的方式访问下游MySQL服务。

该组件基于MySQL 8.0的c api(libmysqlclient)，对MySQL server进行调用。并使用框架内的线程池进行驱动，因此对api的调用不会阻塞工作线程和fiber协程。以下相关类均在 **namespace trpc::mysql** 之内。


- [如何引入](#如何引入)
- [简单使用示例](#简单使用示例)
- [查询结果类](#查询结果类)
- [用户接口](#用户接口)
- [错误信息](#错误信息)
- [更多例子](#更多例子)
    - [日期和Blob](#更多类型)
    - [插入和更新](#插入和更新)
    - [事务](#事务)
    - [异步接口](#异步接口)


## 如何引入

### Bazel

1. 引入仓库

   在项目的WORKSPACE文件中，引入cpp-database-mysql仓库及其依赖：
   ```python
   load("@bazel_tools//tools/build_defs/repo:git.bzl", "git_repository")

   git_repository(
       name = "trpc_cpp",
       remote = "https://github.com/trpc-group/trpc-cpp.git",
       branch = "main",
   )
   
   load("@trpc_cpp//trpc:workspace.bzl", "trpc_workspace")
   trpc_workspace()
      
   git_repository(
       name = "trpc_cpp_database",
       remote = "https://github.com/trpc-ecosystem/cpp-database.git",
       branch = "main",
   )

   load("@trpc_cpp_database//trpc:workspace.bzl", "trpc_cpp_database_workspace")
   trpc_cpp_database_workspace()
   ```
2. 引入依赖

   在需要用到此插件的目标中引入“`trpc/client/mysql:mysql_service_proxy`”依赖。例如：

   ```python
   cc_binary(
    name = "fiber_client",
    srcs = ["fiber_client.cc"],
    deps = [
        "@trpc_cpp_database//trpc/client/mysql:mysql_plugin",
        ...
    ],
   )
   ```

### CMake

参考如下的 CMakeLists.txt 来组织项目：

```bash
# First, import trpc-cpp.
include(FetchContent)
FetchContent_Declare(
    trpc-cpp
    GIT_REPOSITORY    https://github.com/trpc-group/trpc-cpp.git
    GIT_TAG           main
    SOURCE_DIR        ${CMAKE_CURRENT_SOURCE_DIR}/cmake_third_party/trpc-cpp
)
FetchContent_MakeAvailable(trpc-cpp)

# Then, import trpc_cpp_database
FetchContent_Declare(
    trpc_cpp_database
    GIT_REPOSITORY    https://github.com/trpc-ecosystem/cpp-database.git
    GIT_TAG           main
    SOURCE_DIR        ${CMAKE_CURRENT_SOURCE_DIR}/cmake_third_party/trpc_cpp_database
)
FetchContent_MakeAvailable(trpc_cpp_database)

# Last, link to your target
target_link_libraries(your_target trpc
                                  trpc_cpp_database_mysql)
```




## 简单使用示例

### 获取 MysqlServiceProxy 实例

获取一个下游服务的访问代理对象，类型 trpc::mysql::MysqlServiceProxy。推荐使用 TrpcClient::GetProxy 方式获取，参考 [client_guide.md](https://github.com/trpc-group/trpc-cpp/blob/main/docs/zh/client_guide.md)。
如果你希望脱离TrpcClient单独使用实例，请注意手动初始化（不推荐单独使用，因为各种初始化函数成员是protected，你需要继承后才能手动初始化），可以参考 [mysql_service_proxy_test.cc](../../trpc/client/mysql/mysql_service_proxy_test.cc)

#### 初始化配置

在client中配置一个service，需要指定MySQL的专门配置选项，示例如下。这些 MySQL 配置参数是特定于服务的，因此每个服务都必须单独配置自己的 MySQL 选项。示例如下

```yaml
# ...

client:
  service:
    - name: mysql_server
      protocol: mysql
      timeout: 4000
      target: 127.0.0.1:3306
      selector_name: direct
      max_conn_num: 16
      idle_time: 40000

      mysql:
        user_name: "root"             # 数据库用户名
        password: "abc123"            # 用户密码
        dbname: "test"                # 数据库名，一个服务对应一个数据库
        char_set: "utf8mb4"           # 使用的字符集，默认utf8m4
        num_shard_group: 4            # 设置连接队列shard的组数，默认为4
        thread_num: 4                 # 查询任务IO线程池的线程数，默认为4
        thread_bind_core: ""          # 工作线程是否绑定处理核心，默认为不绑定，空字符串也表示不绑定
        # thread_bind_core: "1,2-4"   # 目标核心用逗号隔开，左侧配置表示绑定到处理器1,2,3,4号逻辑核心，等价于"1,2,3,4"


# ...

```

#### 初始化插件
在你使用相关组件之前，请用一下代码做初始化（只需要全局调用一次）
```c++
#include "trpc/client/mysql/mysql_plugin.h"

// Other code ....

::trpc::mysql::InitPlugin();

// Then you can use this plugin
// ...
```

#### 访问过程

1. 获取proxy, 使用 `::trpc::GetTrpcClient()->GetProxy<MysqlServiceProxy>(service_name)`。

   ```c++
   auto proxy = ::trpc::GetTrpcClient()->GetProxy<::trpc::mysql::MysqlServiceProxy>("mysql_server");
   ```
   注意，虽然直接使用 `GetProxy(service_name)` 能够自动读取配置文件中的MySQL相关配置并初始化，但由于插件MySQL配置选项并不属于 `ServiceProxyOption`，`GetProxy(service_proxy, option)`这样传参的方式去初始化proxy只能初始化非MySQL相关的配置。
   如果你希望通过代码指定MySQL的配置，请使用 `SetMysqlConfig`，例如：
   ```c++
   MysqlClientConf mysql_conf;
   mysql_conf.dbname = "test";
   mysql_conf.password = "abc123";
   mysql_conf.user_name = "root";
   mysql_conf.thread_num = 8;
   mysql_conf.thread_bind_core = "1,2-4";
   auto proxy = ::trpc::GetTrpcClient()->GetProxy<::trpc::mysql::MysqlServiceProxy>("mysql_server");
   proxy->SetMysqlConfig(mysql_conf);
   ```

2. 创建 `ClientContextPtr` 对象 `context`：使用 `MakeClientContext(proxy)`。

   ```cpp
   auto ctx = ::trpc::MakeClientContext(proxy);
   ```

3. 调用 `Query` 执行查询语句。

   ```c++
   MysqlResults<int, std::string> res;
   trpc::Status s = proxy->Query(ctx, res, "select id, username from users where id = ? and username = ?", 3, "carol");
   
   if(!s.OK()) {
     TRPC_FMT_ERROR("status: {}", s.ToString());
   } else {
     const std::vector<std::tuple<int, std::string>>& res_set = res.ResultSet();
     int id = std::get<0>(res_set[0]);
     std::string username = std::get<1>(res_set[0]);
     std::cout << "id: " << id << ", username: " << username << std::endl;
   }
   ```

    - 这里我们用模板参数指定字段类型，创建一个 `MysqlResults` 来接收结果，然后作为参数传递给 `Query` 。

    - 传递 SQL 语句，语句中的值变量可以使用 **"?"**  作为占位符，并将占位符对应的变量依次作为参数传递。

    - 最后检查Status。 无误后，通过 `ResultSet()` 获取结果集的引用，实际数据以 `vector<tuple>` 的形式获取，其中tuple模板类型为MysqlResults的模板类型。


#  原理

##  查询结果类

[mysql_results.h](../../trpc/client/mysql/executor/mysql_results.h)

```c++
template <typename... Args>
class MysqlResults
```

`MysqlResults` 是一个模板类，用于存储 MySQL 查询结果。它支持多种模式来处理查询结果，包括：

- **OnlyExec**: 无结果集的执行模式
- **NativeString**:  `std::string_view` 数据
- **BindType**: 将查询结果绑定到特定类型，**下文将除了以上两种模式之外的其它情况称为 BindType**。

### 结果集类型

`MysqlResults` 使用 `std::vector` 保存结果集（除了 `OnlyExec`），每一个元素为一行，其类型与模板参数有关

| 模板参数       | 结果集类型                                   |
| -------------- | -------------------------------------------- |
| `NativeString` | `std::vector<std::vector<std::string_view>>` |
| `Args...`      | `std::vector<std::tuple<Args...>>`           |


### 函数成员

|                                                        | **描述**                                                     |
| ------------------------------------------------------ | ------------------------------------------------------------ |
| `bool OK()`                                            | 查询是否成功。                                               |
| `const std::string& GetErrorMessage()`                 | 如果查询有错误，返回错误原因，否则为空。                     |
| `const auto& ResultSet()`                              | 返回结果集**只读**引用（除 `OnlyExec`），结果集是一个 `std::vector`，每一个元素代表一行，其具体类型和模板参数有关。 |
| `bool GetResultSet(T& res)`                            | 将结果集移动到到参数 `res` 中。当模板参数为NativeString时，应当传入 `std::vector<std::vector<std::string>>` （注意不是 string_view）；当模板参数为BindType时，函数参数类型应与**结果集类型**相同。只能调用一次，返回值代表是否成功。 |
| `size_t GetAffectedRowNum()`                           | 返回查询影响的行数                                           |
| `bool IsValueNull(size_t row_index, size_t col_index)` | 结果集中某个值是否为空                                       |
| `const std::vector<std::string>& GetFieldsName()`      | 获取结果表字段名                                             |

对于无结果集查询(OnlyExec)，请不要使用 MysqlResult<OnlyExec> 中的 `GetResultSet` 和 `ResultSet`。同时 `MysqlResult<OnlyExec>::GetFieldsName()` 返回是空的 vector。

### 使用示例

```
/*
mysql> select * from users;
+----+----------+-------------------+---------------------+------------+
| id | username | email             | created_at          | meta       |
+----+----------+-------------------+---------------------+------------+
|  1 | alice    | alice@example.com | 2024-09-08 13:16:24 | NULL       |
|  2 | bob      | bob@abc.com       | 2024-09-08 13:16:24 | NULL       |
|  3 | carol    | carol@example.com | 2024-09-08 13:16:24 | NULL       |
|  4 | rose     | NULL              | 2024-09-08 13:16:53 | NULL       |
+----+----------+-------------------+---------------------+------------+
*/
```



#### class MysqlResults\<OnlyExec\>

```c++
// example
MysqlResults<OnlyExec> exec_res;
Status s = proxy->Execute(client_context, exec_res,
                 "insert into users (username)"
                 "values (?)", "Jack");
if(s.OK())
    size_t n_rows = exec_res.GetAffectedRowNum();
else
    std::string error = s.ToString();
```

用于像 `INSERT`、`UPDATE` 这类不会返回结果集的查询操作。通过 `GetAffectedRowNum()` 获取影响的行。



#### class MysqlResults\<NativeString\>

```c++

MysqlResults<NativeString> query_res;
proxy->Execute(client_context, query_res,
             "select * from users");

using ResSetType = std::vector<std::vector<std::string_view>>;
// ResSetType& res_data = query_res.ResultSet();
auto& res_data = query_res.ResultSet();

std::cout << res_data[0][1] << std::endl;  // alice@example.com
if(query_res.IsValueNull(3, 2) != 0)
    std::cout << "rose's email is null" << std::endl;

```

如果模板参数指定为 `NativeString` ，则结果集会以 `std::vector<std::vector<std::string_view>>` 返回，每一个 `std::vector<std::string_view>` 表示一行，里面每一个元素是该行的对应字段值。其中 `std::string_view` 对应的内存资源在 MysqlResults析构后会被释放， 你可以通过 `GetResultSet` 将结果拷贝出来。



#### class MysqlResults\<Args...\>

```c++

// example
MysqlResults<int, std::string, MysqlTime> query_res;
Status s = proxy->Execute(client_context, query_res,
                  "select id, email, created_at from users "
                  "where id = 1 or username = \"bob\")";

// using ResSetType = std::vector<std::tuple<int, std::string, MysqlTime>>;
if(s.OK()) {
    // ResSetType& res_set = query_res.ResultSet();
    auto& res_set = query_res.ResultSet();
    int id = std::get<0>(res_set[0]);
    std::string email = std::get<1>(res_set[1]);
    MysqlTime mtime = std::get<2>(res_set[1]);
}         
else
    std::string error = s.ToString();
```

使用类型绑定，如果模板指定为除 `OnlyExec`，`NativeString`之外的其它类型，则结果集的每一行是一个tuple。（这里关于 `MysqlTime` 的使用见[日期和Blob](#更多类型)）

注意模板参数需要和查询结果匹配，特别是使用 select * 时：

- 模板参数个数需要与实际查询结果字段个数一致。
- 模板参数的类型需要与实际字段类型匹配（例如不能用int去接收MySQL的字符串）。

可以从Status中获取错误信息，例如下面的情况：
  ```c++
  MysqlResults<int, int, double> res;
  Status s = proxy->Query(client_context, res,
                          "select id, email, created_at from users");
  std::string error_msg = s.ErrorMessage();
  // error_msg: "Bind output type warning for fields: (email, created_at)."
  ```
具体匹配关系见下表：

| MySQL Type                   | Template Output Type                              |
|------------------------------|---------------------------------------------------|
| TINYINT                      | `int8_t`, `uint8_t`                               |
| SMALLINT                     | `int16_t`, `uint16_t`                             |
| MEDIUMINT, INT               | `int32_t`, `uint32_t`                             |
| BIGINT                       | `int64_t`, `uint64_t`                             |
| FLOAT                        | `float`                                           |
| DOUBLE                       | `double`                                          |
| DECIMAL                      | `std::string`                                     |
| YEAR                         | `int16_t`, `uint16_t`, `std::string`, `MysqlTime` |
| TIME, DATE, DATETIME, TIMESTAMP | `MysqlTime`, `std::string`                        |
| CHAR, BINARY                 | `std::string`                                     |
| VARCHAR, VARBINARY           | `std::string`                                     |
| TINYBLOB, TINYTEXT          | `std::string`, `MysqlBlob`                        |
| BLOB, TEXT                   | `std::string`, `MysqlBlob`                        |
| MEDIUMBLOB, MEDIUMTEXT       | `std::string`, `MysqlBlob`                        |
| LONGBLOB, LONGTEXT           | `std::string`, `MysqlBlob`                        |
| BIT                          | `std::string`, `MysqlBlob`                        |




## 用户接口

[mysql_service_proxy.h](./trpc/client/mysql/mysql_service_proxy.h)

用户可以使用 `trpc::mysql::MysqlServiceProxy` 对 MySQL server进行访问，并可以通过框架 yaml 配置文件对访问进行定制。proxy 支持通过 SQL 语句进行数据库的读写和事务，并同时提供同步和异步两种方式。

| 方法名                     | 描述                                | 参数                                                                                                                           | 返回类型                        | 备注                      |
| -------------------------- | ----------------------------------- |------------------------------------------------------------------------------------------------------------------------------|-----------------------------| ------------------------- |
| `Query`                    | 执行 SQL 查询并检索所有结果行。     | `context`: 客户端上下文;   `res`: 用于返回查询结果的 MysqlResults 对象；  `sql_str`: 带有占位符 "?" 的 SQL 查询字符串  `args`: 输入参数，可以为空                  | `Status`                    |                           |
| `AsyncQuery`               | 异步执行 SQL 查询并检索所有结果行。 | `context`: 客户端上下文；  `sql_str`: 带有占位符 "?" 的 SQL 查询字符串；  `args`: 输入参数，可以为空                                                     | `Future<MysqlResults>`      |                           |
| `Execute`                  | 同步执行 SQL 查询，用于OnlyExec。   | `context`: 客户端上下文；  `res`: 用于返回查询结果的 MysqlResults 对象；  `sql_str`: 带有占位符 "?" 的 SQL 查询字符串；  `args`: 输入参数，可以为空                  | `Status`                    | 可被`Query` 完全替代      |
| `AsyncExecute`             | 异步执行 SQL 查询，用于OnlyExec。   | `context`: 客户端上下文；  `sql_str`: 带有占位符 "?" 的 SQL 查询字符串；  `args`: 输入参数，可以为空                                                     | `Future<MysqlResults>`      | 可被`AsyncQuery` 完全替代 |
| `Query`（事务支持）        | 在事务中执行 SQL 查询。             | `context`: 客户端上下文 ； `handle`: 事务标识；  `res`: 用于返回查询结果的 MysqlResults 对象；  `sql_str`: 带有占位符 "?" 的 SQL 查询字符串；  `args`: 输入参数，可以为空 | `Status`                    |                           |
| `Execute`（事务支持）      | 在事务中执行 SQL 查询。             | `context`: 客户端上下文；  `handle`: 事务标识；  `res`: 用于返回查询结果的 MysqlResults 对象；  `sql_str`: 带有占位符 "?" 的 SQL 查询字符串；  `args`: 输入参数，可以为空 | `Status`                    |                           |
| `AsyncQuery`（事务支持）   | 异步在事务中执行 SQL 查询。         | `context`: 客户端上下文；  `handle`: 事务标识 ； `sql_str`: 带有占位符 "?" 的 SQL 查询字符串；  `args`: 输入参数，可以为空                                    | `Future<MysqlResults>`      |                           |
| `AsyncExecute`（事务支持） | 异步在事务中执行 SQL 查询。         | `context`: 客户端上下文 ； `handle`: 事务标识；  `sql_str`: 带有占位符 "?" 的 SQL 查询字符串；  `args`: 输入参数，可以为空                                    | `Future<MysqlResults>`      |                           |
| `Begin`                    | 开始一个事务。                      | `context`: 客户端上下文；  `handle`: 空事务标识                                                                                          | `Status`                    |                           |
| `Commit`                   | 提交一个事务。                      | `context`: 客户端上下文；  `handle`: 事务标识                                                                                           | `Status`                    |                           |
| `Rollback`                 | 回滚一个事务。                      | `context`: 客户端上下文；  `handle`: 事务标识                                                                                           | `Status`                    |                           |
| `AsyncBegin`               | 异步开始一个事务。                  | `context`: 客户端上下文                                                                                                            | `Future<TxHandlePtr>`       |                           |
| `AsyncCommit`              | 异步提交一个事务。                  | `context`: 客户端上下文；  `handle`: 事务标识                                                                                           | `Future<>` |                           |
| `AsyncRollback`            | 异步回滚一个事务。                  | `context`: 客户端上下文；  `handle`: 事务标识                                                                                           | `Future<>` |                           |

- 当`MysqlResults` 使用 `NativeString`  时，仅仅使用字符串格式化来处理占位符，转为一个完整的SQL语句进行查询。除此之外的情况会使用 MySQL prepared statement，可以防止SQL注入，关于重复使用同一个prepared statement接口目前暂时还未提供。

- 由于mysql api 的调用由线程池完成，因此：

    - 在同步接口中，如果外部逻辑处于fiber协程下，该协程会等待，但不会阻塞线程。
    - 在异步接口中，如果是在fiber环境中请使用 `::trpc::fiber` 里的future相关接口。



## 错误信息

- 同步接口的 `Status` 返回调用过程的错误信息，同时也可以通过`MysqlResult` 通过 `OK` 和 `GetErrorMessage` 以字符串形式描述MySQL查询错误
  。两者的区别是所有错误都会返回到Status，而MysqlResult中只会保存MySQL查询相关的错误，例如上下文Timeout和被Filter拒绝的情况下，MysqlResult不会有错误。
  （总之 MysqlResult 的错误信息一定会被设置到返回的 Status，因此**完全可以只检查 Status**）。
- 异步接口如果出错，会返回的 exception future 对象，因为异常future不包含值，因此无法通过 future 里面的 value 即  `MysqlResults` 对象获取MySQL查询错误。因此在异步接口中，只能从异常future的exception返回。



## 更多例子

### 更多类型

[mysql_type.h](../../trpc/client/mysql/executor/mysql_type.h)

对于 `MysqlResults<NativeString>` ，不需要关心查询结果的字段类型，因为都是以字符串的形式返回。如果希望将结果绑定到类型，对于日期和Blob，也可以使用 `MysqlTime` 和 `MysqlBlob` (当然你也同样可以使用 `std::string` 作为通用类型来接收)。

#### MysqlTime

**将MysqlTime作为BindType的模板参数类型**

```c++
// Use MysqlTime
MysqlResults<MysqlTime> time_res;

proxy->Query(ctx, time_res, "select created_at from users");

MysqlTime my_time = std::get<0>(time_res.ResultSet()[0]);

std::cout << "NativeString: " << str_time << std::endl;

std::string fmt_str_time = trpc::util::FormatString("{}-{}-{} {}:{}:{}",
my_time.GetYear(),
my_time.GetMonth(),
my_time.GetDay(),
my_time.GetHour(),
my_time.GetMinute(),
my_time.GetSecond());

std::cout << "MysqlTime: " << fmt_str_time << std::endl;

// Or use ToString
std::cout << "MysqlTime: " << my_time.ToString() << std::endl;
```

**创建一个 MysqlTime :**

```c++
// Build a MysqlTime
MysqlTime new_time;
new_time.SetYear(2024).SetMonth(9).SetDay(10).SetHour(21);
std::cout << new_time.ToString() << std::endl;

MysqlTime new_time_from_str;
new_time_from_str.FromString(new_time.ToString());
std::cout << new_time_from_str.ToString() << std::endl;
```

MySQL中各种日期类型均可以使用 `MysqlTime` 接收，只是结果都被结构化为 year, mouth, day等 （例如如果你想获取原始时间戳，请使用字符串）。

`MysqlTime` 可以通过 `SetXXX` 和 `GetXXX` 分别设置和获取对应值，或者可以通过 `FromString` 和 `ToString` 分别从字符串创建和导出为字符串，格式为 **"YYYY-MM-DD HH:MM:SS"**。

需要注意其中的无效字段，例如 MySQL 的 **YEAR** 类型，返回的结果中只有  `GetYear()` 是有效的，不要使用其它字段。

#### MysqlBlob

```c++
// field "meta" type: BLOB
MysqlResults<std::string, MysqlBlob> query_res;
proxy->Query(ctx, query_res, "select username, meta from users"
"where id = ?", 1);
MysqlBlob& blob = std::get<1>(query_res.ResultSet()[0]);
std::string_view data_view = blob.AsStringView();
```

MysqlBlob底层使用 `std::string` 。

以上两种类型也可以在插入或更新时作为占位符参数传入。

### 插入和更新

```c++
MysqlResults<std::string, MysqlTime> query_res;
MysqlResults<OnlyExec> exec_res;
std::string jack_email = "jack@abc.com";
MysqlTime mtime;
mtime.SetYear(2024).SetMonth(9).SetDay(10);

proxy->Execute(ctx, exec_res,
                         "insert into users (username, email, created_at)"
                         "values (\"jack\", ?, ?)",
                         jack_email, mtime);


ctx = trpc::MakeClientContext(proxy);
proxy->Execute(ctx, exec_res, "delete from users where username = \"jack\"");

```

和select使用方法相同，只是插入和更新语句由于不返回结果集，请使用 `MysqlResult<OnlyExec>`。同样，也可以使用占位符，对应的参数也支持 `MysqlBlob` 和 `MysqlTime` 。

### 事务

[transaction.h](../../trpc/client/mysql/transaction.h)

`TransactionHandle` 标识一个事务，实际应使用其引用计数指针 `TxHandlePtr`，下面将介绍如何使用 proxy 进行一个事务。

1. **开始一个事务**

   ```c++
   trpc::ClientContextPtr ctx = trpc::MakeClientContext(proxy);
   TxHandlePtr handle = nullptr;
   trpc::Status s = proxy->Begin(ctx, handle);
   
   if(!s.OK()) {
     TRPC_FMT_ERROR("status: {}", s.ToString());
     return;
   }
   ```

   创建一个 `TransactionHandle` 变量，作为 `Begin`参数，来开始一个事务。

2. **在事务中执行查询**

   ```c++
   ...
       
   // Insert a row
   s = proxy->Execute(ctx, handle, exec_res,
                        "insert into users (username, email, created_at)"
                        "values (\"jack\", \"jack@abc.com\", ?)", mtime);
   if(!s.OK() || (exec_res.GetAffectedRowNum() != 1)) {
     TRPC_FMT_ERROR("status: {}", s.ToString());
     return;
   }
   
   // Update a row
   s = proxy->Execute(ctx, handle, exec_res,
                        "update users set email = ? where username = ?",
                        "jack@gmail.com",
                        "jack");
   if(!s.OK() || (exec_res.GetAffectedRowNum() != 1)) {
     TRPC_FMT_ERROR("status: {}", s.ToString());
     return;
   }
   
   ...
   ```

   将 handle 作为第二个参数传入 `Query` 或 `Execute` ，其余和非事务接口一致。

3. **结束事务**

   ```c++
   ...
   s = proxy->Commit(ctx, handle);
   if(!s.OK()) {
     TRPC_FMT_ERROR("status: {}", s.ToString());
     return;
   }
   ...
   ```

   使用 `Commit` 提交一个事务，如果 Status 正常，那么该事务结束，此后handle将不可用；如果 Status 不正常，请检查错误信息后，之后可以再次尝试调用 `Commit` 。

   ```c++
   ...
   s = proxy->RollBack(ctx, handle);
   if(!s.OK()) {
     TRPC_FMT_ERROR("status: {}", s.ToString());
     return;
   }
   ...
   ```

   同样的，也可以 `RollBack` 。
4. **关于事务中的异常**
    - 事务中的异常检查和非事务接口的异常检查一致，都是通过检查返回的Status或Future的异常信息。
    - 如果开始了一个事务，但最后没有显式调用Commit或者Rollback，那么TransactionHandle会在析构时断开连接，而连接断开时会**自动回滚**。此外，查询过程中因为其他原因连接断开
      也会回滚，注意检查返回的Status信息。
    - 可以通过 handle.GetState() 检查当前状态

| TransactionHandle::TxState | 描述                                   |
|----------------------------|--------------------------------------|
| `kNotInited`                 | 未初始化(空事务)                            |
| `kStarted`                     | 事务已开始                                |
| `kRollBacked`                | 事务已回滚                                |
| `kCommitted`                 | 事务已提交                                |
| `kInValid`                   | 事务无效（对象被move时的状态，并非指rollback或commit） |



### 异步接口

和同步接口不同，异步接口并非通过传递 `MysqlResult` 来接收查询结果，而是返回 `Future<MysqlResult<Arg...>>` 。其中模板参数在调用 proxy 的函数成员时显示指定

```c++
trpc::ClientContextPtr ctx = trpc::MakeClientContext(proxy);
MysqlResults<int, std::string> res;

using ResultType = MysqlResults<int, std::string>;
auto future = proxy->AsyncQuery<int, std::string>(ctx, "select id, username from users where id = ?", 3)
                        .Then([](trpc::Future<ResultType>&& f){
                          if(f.IsReady()) {
                            auto res = f.GetValue0();  // MysqlResults<int, std::string>
                            printResult(res.ResultSet());
                            return trpc::MakeReadyFuture();
                          }
                          return trpc::MakeExceptionFuture<>(f.GetException());
                        });
std::cout << "do something\n";
trpc::future::BlockingGet(std::move(future));
```

这里 `proxy->AsyncQuery<int, std::string>(ctx, "select id, username from users where id = ?", 3)` 应当返回 `Future<MysqlResults<int, std::string>>` ，然后通过 `Then`  回调处理查询结果， 将 `AsyncQuery` 返回的 future 中的值获取出来，然后打印，最后返回一个就绪的空future。

**对于异步事务接口**，参考 [future_client.cc](../../examples/mysql/client/future/future_client.cc)

```c++
MysqlResults<OnlyExec> exec_res;
MysqlResults<NativeString> query_res;
TxHandlePtr handle = nullptr;

trpc::ClientContextPtr ctx = trpc::MakeClientContext(proxy);

auto fut = proxy->AsyncBegin(ctx)
          .Then([&handle](Future<TxHandlePtr>&& f) mutable {
            if(f.IsFailed())
              return trpc::MakeExceptionFuture<>(f.GetException());

            // Get the ref counted handle ptr here.
            // Or you can return Future<TxHandlePtr> and get the handle outside.
            handle = f.GetValue0();
            return trpc::MakeReadyFuture<>();
          });
trpc::future::BlockingGet(std::move(fut));

auto fut2 = proxy
          ->AsyncQuery<NativeString>(ctx, handle, "select username from users where username = ?", "alice")
          .Then([](Future<MysqlResults<NativeString>>&& f) {
            if(f.IsFailed())
              return trpc::MakeExceptionFuture<>(f.GetException());
            auto t = f.GetValue();
            auto res = std::move(std::get<0>(t));

            std::cout << "\n>>> select username from users where username = alice\n";
            PrintResultTable(res);
            return trpc::MakeReadyFuture<>();
          });

auto fu3 = trpc::future::BlockingGet(std::move(fu2));
...
```

通过 `AsyncBegin` 开始一个事务，它会返回 `Future<TxHandlePtr>` ，这里使用一个 `handle` 在回调里取出返回的 TxHandlePtr，然后赋值给它（事实上也可以不用回调，直接在外面从返回的Future去获取里面的handle）。之后为了在事务中进行查询，我们需要传递 handle 参数给 `AsyncQuery` 和`AsyncExecute` 。后续在事务上的所有操作只需要传递 handle 即可。

只要有 handle，后续这个事务也可以使用同步事务接口。

你也可以直接在Then Chain里面完成整个事务，并没有什么特别的地方，详情见  [future_client.cc](./examples/mysql/client/future/future_client.cc)


