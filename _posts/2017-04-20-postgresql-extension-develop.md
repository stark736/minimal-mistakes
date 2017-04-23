---
title: PostgreSQL 扩展开发基础教程
date: "2017-04-20 20:00:00"
tag: 
	PostgreSQL
---

由于业务需要，我们实现了解析客户定义的伪代码并计算的功能。所需计算数据大多存储在 PostgreSQL 中，因而我们需要利用 PostgreSQL 函数实现一部分计算。但有时，原生的函数的行为并不完全贴合我们的需求，同时也无法通过函数的组合来达到目的。因此，我们决定扩展 PostgreSQL 的函数。

我将我们实现扩展的过程记录下，整理成下面一篇扩展开发的基础教程，以供与我们有同样需求的参考。

本篇文章中 PostgreSQL 使用9.6.2的版本。

## 开发前的准备

我们为这次开发的扩展命名为`array_ext`。我们希望这个扩展能够增强 PostgreSQL 处理数组的能力。

### 搭建基础结构

首先，我们必须为了安装扩展做一些准备。为了能够在 PostgreSQL 中使用`CREATE EXTENSION`命令加载扩展，我们的扩展需要两个必需的文件：

- `extension_name.control` 控制文件，声明该扩展的基础信息。
- `extension--version.sql` 加载扩展所需要执行的SQL文件。

我们创建 `array_ext.control` 文件，内容如下：

```bash
# array_ext extension
comment = 'extend array'
default_version = '0.0.1'
relocatable = true
```

同样，我们需要创建 `array_ext--0.0.1.sql` 文件，不过这里先不写入内容，因为该文件的内容和我们后面要编写的函数息息相关，这里先按下不表。

### 准备安装文件

为了让我们开发的扩展能方便的安装到 PostgreSQL 服务器上，我们打算使用`make install`命令让整个安装的过程变得简单并且统一。这样，我们需要一个`Makefile`文件。在这个文件中，我们会复用服务器上的`pg_config`工具里的环境变量，让我们开发时，不需要关心安装时服务器上的 PostgreSQL 是什么状态。

我们为扩展创建一个最简单`Makefile`文件，内容如下：

```bash
EXTENSION = array_ext        # 扩展的名称
DATA = array_ext--0.0.1.sql  # 扩展安装的SQL文件

# 以下是 PostgreSQL 构建扩展相关的命令，保留就可以
PG_CONFIG = pg_config
PGXS := $(shell $(PG_CONFIG) --pgxs)
include $(PGXS)
```

## 开发扩展

准备工作做完后，我们进入扩展开发的环节。

### 我们要开发什么

我们打算在这个扩展中支持一个新的函数，用于为给定的数组中插入元素。如果插入的元素已经存在数组中，则不再重复插入，否则插入到数组的末尾。设计的函数如下：`jsonb array_ext_append(jsonb arr, int elm)`。这是我们扩展函数中最简单的一个，用于基础的教程再合适不过。

### 决定使用的语言

PostgreSQL 支持使用`PL/pgSQL`语言或者原生的C语言开发扩展。`PL/pgSQL`开发简单，然而性能上较原生的C语言要逊色不少。有不少人已经做过相关的性能测试，这里就不再重复说明。我们开发的扩展的目的是为了增强生产的 PostgreSQL，自然要选择性能更好的C语言。

### 编写代码

接下来，我们开始编写`jsonb array_ext_append(jsonb arr, int elm)`函数的实现代码。

```c
#include "postgres.h"
#include "fmgr.h"
#include "utils/jsonb.h"

PG_MODULE_MAGIC;

JsonbValue *IteratorAppend(JsonbIterator **, Numeric, JsonbParseState **);

PG_FUNCTION_INFO_V1(array_ext_append);

Datum
array_ext_append(PG_FUNCTION_ARGS)
{
    Numeric     elm;
    Jsonb       *arr = NULL;
    JsonbValue  *res = NULL;
    JsonbIterator   *it;
    JsonbParseState *st = NULL;

    arr = PG_GETARG_JSONB(0);
    elm = PG_GETARG_NUMERIC(1);

    it = JsonbIteratorInit(&arr->root);
    res = IteratorAppend(&it, elm, &st);

    PG_RETURN_JSONB(JsonbValueToJsonb(res));
}

JsonbValue *
IteratorAppend(JsonbIterator **it, Numeric value, JsonbParseState  **state)
{
    uint32          r, rk;
    bool            t;
    JsonbValue      v, *res = NULL;
    t = true;
    r = rk = JsonbIteratorNext(it, &v, false);
    if (rk == WJB_BEGIN_ARRAY) {
        res = pushJsonbValue(state, r, NULL);
        for(;;) {
            r = JsonbIteratorNext(it, &v, true);
            if (r == WJB_END_OBJECT || r == WJB_END_ARRAY)
                break;

            if (strcmp(numeric_normalize(v.val.numeric), numeric_normalize(value)) == 0) {
                t = false;
            }

            pushJsonbValue(state, r, &v);
        }

        if (t) {
            v.type = jbvNumeric;
            v.val.numeric = value;
            pushJsonbValue(state, WJB_ELEM, &v);
        }

        res = pushJsonbValue(state, WJB_END_ARRAY, NULL);
    }

    return res;
}
```

我将完整的代码摘录下来，方便大家参考和测试。我选择部分重要的代码一一说明。

`#include "postgres.h"` 包含 PostgreSQL 基础的接口。这是开发 PostgreSQL 扩展必需包含的头文件。

`#include "fmgr.h"` 包含了`PG_GETARG_XX`和`PG_RETURN_XX`等获取参数和返回结果的重要的宏，基本上是必需的。

`PG_MODULE_MAGIC` 是一个从 PostgreSQL 8.2版本后就必须的宏，必须写在`#include "fmgr.h"`之后。

`PG_FUNCTION_INFO_V1` 宏声明了我们所定义的函数为 Version-1 约定的函数。我们选择了 Version-1 的开发约定，所以在定义方法之前，需要调用 `PG_FUNCTION_INFO_V1(array_ext_append)`宏对函数声明。对 Version-1 有兴趣的同学可以移步 [Version 1 Calling Conventions](https://www.postgresql.org/docs/9.4/static/xfunc-c.html#AEN55804) 查看详情。

`Datum` 等同于`void *`，表示函数返回任意类型的数据。

`arr = PG_GETARG_JSONB(0);` 获取函数的第一个参数的值，并且将其转换为 jsonb 类型。

`PG_RETURN_JSONB(JsonbValueToJsonb(res));` 将结果转换为 jsonb 类型并返回。

### 声明扩展函数

除了在源码中实现我们自定义的函数，我们还需要在扩展中声明函数，这样才能在 PostgreSQL 中使用。这时，我们就需要用到我们之前创建但没有使用的`array_ext–0.0.1.sql`。在文件中，我们声明了函数的参数和返回值的类型。

```sql
-- complain if script is sourced in psql, rather than via CREATE EXTENSION
\echo Use "CREATE EXTENSION array_ext" to load this file. \quit
CREATE FUNCTION array_ext_append(jsonb, numeric) RETURNS jsonb
AS '$libdir/array_ext'
LANGUAGE C IMMUTABLE CALLED ON NULL INPUT;
```

## 安装扩展

到此，我们的扩展的代码就全部完成，很简单，不是吗？我想你现在一定迫不及待想让自己开发的代码运行起来。别急，我们一步一步来。

### 编译安装

确保你的机器上已经安装了 PostgreSQL，并且 PostgreSQL 服务已经启动运行。通过`pg_config`我们能看到本机的 PostgreSQL 运行的参数。

```bash
➜  array_ext git:(master) ✗ pg_config
BINDIR = /usr/local/Cellar/postgresql/9.6.2/bin
DOCDIR = /usr/local/Cellar/postgresql/9.6.2/share/doc/postgresql
HTMLDIR = /usr/local/Cellar/postgresql/9.6.2/share/doc/postgresql
INCLUDEDIR = /usr/local/Cellar/postgresql/9.6.2/include
PKGINCLUDEDIR = /usr/local/Cellar/postgresql/9.6.2/include
...
```

进入我们的开发目录。我们已经定义了`Makefile`文件，所以我们可以直接编译和安装。

```bash
➜  array_ext git:(master) ✗ make && make install
clang -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Wendif-labels -Wmissing-format-attribute -Wformat-security -fno-strict-aliasing -fwrapv -Wno-unused-command-line-argument -O2  -I. -I./ -I/usr/local/Cellar/postgresql/9.6.2/include/server -I/usr/local/Cellar/postgresql/9.6.2/include/internal -I/usr/local/opt/openssl/include -I/usr/local/opt/readline/include -I/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.12.sdk/usr/include/libxml2   -c -o array_ext.o array_ext.c
clang -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Wendif-labels -Wmissing-format-attribute -Wformat-security -fno-strict-aliasing -fwrapv -Wno-unused-command-line-argument -O2  -L/usr/local/lib -L/usr/local/opt/openssl/lib -L/usr/local/opt/readline/lib  -Wl,-dead_strip_dylibs   -bundle -bundle_loader /usr/local/Cellar/postgresql/9.6.2/bin/postgres -o array_ext.so array_ext.o
/bin/sh /usr/local/lib/postgresql/pgxs/src/makefiles/../../config/install-sh -c -d '/usr/local/share/postgresql/extension'
/bin/sh /usr/local/lib/postgresql/pgxs/src/makefiles/../../config/install-sh -c -d '/usr/local/share/postgresql/extension'
/bin/sh /usr/local/lib/postgresql/pgxs/src/makefiles/../../config/install-sh -c -d '/usr/local/lib/postgresql'
/usr/bin/install -c -m 644 .//array_ext.control '/usr/local/share/postgresql/extension/'
/usr/bin/install -c -m 644 .//array_ext--0.0.1.sql  '/usr/local/share/postgresql/extension/'
/usr/bin/install -c -m 755  array_ext.so '/usr/local/lib/postgresql/'
```

可以看到，我们编写的代码被编译成`array_ext.so`文件，拷贝到了本机的 PostgreSQL 的库目录。PostgreSQL 使用动态链接库，所以不需要重启服务即可以加载到扩展的函数。

### 加载扩展

我们使用访问 PostgreSQL 的用户连接到 PostgreSQL 服务，加载我们开发的扩展。

```sql
➜  ~ psql -U joshua
psql (9.6.2)
Type "help" for help.

joshua=# CREATE EXTENSION array_ext;
CREATE EXTENSION
```

加载完毕，我们马上试试新的函数是不是可以运行吧。

```sql
joshua=# select array_ext_append('[1,2]', 3);
 array_ext_append
------------------
 [1, 2, 3]
(1 row)

joshua=# select array_ext_append('[1,2]', 2);
 array_ext_append
------------------
 [1, 2]
(1 row)
```

直接输出了我们预期的结果，大功告成！

## 测试

我们为 PostgreSQL 扩展开发开了一个不错的头，但接下来持续的开发将会面对一个问题：如何快速的回归测试。这里，我介绍一个简单的回归测试的方法：`make installcheck`。

首先，我们把测试写成一个 SQL 文件，保存为`sql/array_ext_test.sql`。我们把之前用于测试的 SQL 写入到这个文件中，并且在开头添加加载扩展的语句（如果没有，在运行测试时就会返回错误）。

```sql
CREATE EXTENSION array_ext;

select array_ext_append('[1,2]', 3);
select array_ext_append('[1,2]', 2);
```

然后在`Makefile`文件中增加测试的命令。

```bash
...
EXTENSION = array_ext        # 扩展的名称
DATA = array_ext--0.0.1.sql  # 扩展安装的SQL文件
REGRESS = array_ext_test     # 扩展测试的SQL文件
...
```

我们运行`make installcheck`看看会发生什么。

```bash
➜  array_ext git:(master) ✗ make installcheck
/usr/local/lib/postgresql/pgxs/src/makefiles/../../src/test/regress/pg_regress --inputdir=./ --bindir='/usr/local/Cellar/postgresql/9.6.2/bin'    --dbname=contrib_regression array_ext_test
(using postmaster on Unix socket, default port)
============== dropping database "contrib_regression" ==============
DROP DATABASE
============== creating database "contrib_regression" ==============
CREATE DATABASE
ALTER DATABASE
============== running regression test queries        ==============
test array_ext_test           ... diff: /Users/joshua/Programs/local/test/postgresql/array_ext/expected/array_ext_test.out: No such file or directory
diff command failed with status 512: diff  "/Users/joshua/Programs/local/test/postgresql/array_ext/expected/array_ext_test.out" "/Users/joshua/Programs/local/test/postgresql/array_ext/results/array_ext_test.out" > "/Users/joshua/Programs/local/test/postgresql/array_ext/results/array_ext_test.out.diff"
make: *** [installcheck] Error 2
```

出现了一个错误。检查错误后发现，缺少两个目录`results`和`expected`。我们要将期望的结果写入`expected/array_ext_test.out`，执行测试 SQL 输出的结果会写入`results/array_ext_test.out`。这样，通过 diff 这两个文件的内容差异，就能验证结果是否符合预期。

我们创建`expected/array_ext_test.out`文件，这里可以把刚才错误执行时生成的`results/array_ext_test.out`内容拷贝到`expected/array_ext_test.out`文件中（我们通过肉眼验证过结果是正确的）。然后，我们再执行测试命令，屏幕上输出了这次测试的结果。

```bash
➜  array_ext git:(master) ✗ make installcheck
/usr/local/lib/postgresql/pgxs/src/makefiles/../../src/test/regress/pg_regress --inputdir=./ --bindir='/usr/local/Cellar/postgresql/9.6.2/bin'    --dbname=contrib_regression array_ext_test
(using postmaster on Unix socket, default port)
============== dropping database "contrib_regression" ==============
DROP DATABASE
============== creating database "contrib_regression" ==============
CREATE DATABASE
ALTER DATABASE
============== running regression test queries        ==============
test array_ext_test           ... ok

=====================
 All 1 tests passed.
=====================
```

## 结语

好了，以上就是开发一个 PostgreSQL 扩展的所需的基本要素。可以看到，开发并不繁琐复杂。但如果想要开发一个稳定运行的高效的扩展，则需要对 PostgreSQL 内核有更深入的了解。本篇作为一个入门教程，希望能抛砖引玉，提供给有兴趣的同学做为参考。


## 参考资料

http://big-elephants.com/2015-10/writing-postgres-extensions-part-i/