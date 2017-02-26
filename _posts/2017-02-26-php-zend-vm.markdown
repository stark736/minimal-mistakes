---
title: PHP 内核分析：Zend 虚拟机
tags:
    - PHP 内核
---

PHP 是一门解释型的语言。诸如 Java、Python、Ruby、Javascript 等解释型语言，我们编写的代码不会被编译成机器码运行，而是会被编译中间码运行在虚拟机（VM）上。运行 PHP 的虚拟机，称之为 Zend 虚拟机，今天我们将深入内核，探究 Zend 虚拟机运行的原理。

## OPCODE

什么是 OPCODE？它是一种虚拟机能够识别并处理的指令。Zend 虚拟机包含了一系列的 OPCODE，通过 OPCODE 虚拟机能够做很多事情，列举几个 OPCODE 的例子：

- `ZEND_ADD` 将两个操作数相加。
- `ZEND_NEW` 创建一个 PHP 对象。
- `ZEND_ECHO` 将内容输出到标准输出中。
- `ZEND_EXIT` 退出 PHP。

诸如此类的操作，PHP 定义了186个（随着 PHP 的更新，肯定会支持更多种类的 OPCODE），所有的 OPCODE 的定义和实现都可以在源码的 `zend/zend_vm_def.h` 文件（这个文件的内容并不是原生的 C 代码，而是一个模板，后面会说明原因）中查阅到。

我们来看下 PHP 是如何设计 OPCODE 数据结构：

```c
struct _zend_op {
	const void *handler;
	znode_op op1;
	znode_op op2;
	znode_op result;
	uint32_t extended_value;
	uint32_t lineno;
	zend_uchar opcode;
	zend_uchar op1_type;
	zend_uchar op2_type;
	zend_uchar result_type;
};
```

仔细观察 OPCODE 的数据结构，是不是能找到汇编语言的感觉。每一个 OPCODE 都包含两个操作数，`op1`和 `op2`，`handler` 指针则指向了执行该 OPCODE 操作的函数，函数处理后的结果，会被保存在 `result` 中。

我们举一个简单的例子：

```php
<?php
$b = 1;
$a = $b + 2;
```

我们通过 vld 扩展看到，经过编译的后，上面的代码生成了 ZEND_ADD 指令的 OPCODE。

```bash
compiled vars:  !0 = $b, !1 = $a
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   2     0  E >   ASSIGN                                                   !0, 1
   3     1        ADD                                              ~3      !0, 2
         2        ASSIGN                                                   !1, ~3
   8     3      > RETURN                                                   1
```

其中，第二行是 `ZEND_ADD` 指令的 OPCODE。我们看到，它接收2个操作数，`op1` 是变量 `$b`，`op2` 是数字常量1，返回的结果存入了临时变量中。在 `zend/zend_vm_def.h` 文件中，我们可以找到 ZEND_ADD 指令对应的函数实现：

```c
ZEND_VM_HANDLER(1, ZEND_ADD, CONST|TMPVAR|CV, CONST|TMPVAR|CV)
{
	USE_OPLINE
	zend_free_op free_op1, free_op2;
	zval *op1, *op2, *result;

	op1 = GET_OP1_ZVAL_PTR_UNDEF(BP_VAR_R);
	op2 = GET_OP2_ZVAL_PTR_UNDEF(BP_VAR_R);
	if (EXPECTED(Z_TYPE_INFO_P(op1) == IS_LONG)) {
		if (EXPECTED(Z_TYPE_INFO_P(op2) == IS_LONG)) {
			result = EX_VAR(opline->result.var);
			fast_long_add_function(result, op1, op2);
			ZEND_VM_NEXT_OPCODE();
		} else if (EXPECTED(Z_TYPE_INFO_P(op2) == IS_DOUBLE)) {
			result = EX_VAR(opline->result.var);
			ZVAL_DOUBLE(result, ((double)Z_LVAL_P(op1)) + Z_DVAL_P(op2));
			ZEND_VM_NEXT_OPCODE();
		}
	} else if (EXPECTED(Z_TYPE_INFO_P(op1) == IS_DOUBLE)) {

	...
}
```

上面的代码并不是原生的 C 代码，而是一种模板。

为什么这样做？因为 PHP 是弱类型语言，而其实现的 C 则是强类型语言。弱类型语言支持自动类型匹配，而自动类型匹配的实现方式，就像上述代码一样，通过判断来处理不同类型的参数。试想一下，如果每一个 OPCODE 处理的时候都需要判断传入的参数类型，那么性能势必成为极大的问题（一次请求需要处理的 OPCODE 可能能达到成千上万个）。

哪有什么办法吗？我们发现在编译的时候，已经能够确定每个操作数的类型（可能是常量还是变量）。所以，PHP 真正执行时的 C 代码，不同类型操作数将分成不同的函数，供虚拟机直接调用。这部分代码放在了 `zend/zend_vm_execute.h` 中，展开后的文件相当大，而且我们注意到还有这样的代码：

```c
if (IS_CONST == IS_CV) {
```

完全没有什么意义是吧？不过没有关系，C 的编译器会自动优化这样判断。大多数情况，我们希望了解某个 OPCODE 处理的逻辑，还是通过阅读模板文件 `zend/zend_vm_def.h` 比较容易。顺便说一下，根据模板生成 C 代码的程序就是用 PHP 实现的。

## 执行过程

准确的来说，PHP 的执行分成了两大部分：编译和执行。这里我将不会详细展开编译的部分，而是把焦点放在执行的过程。

通过语法、词法分析等一系列的编译过程后，我们得到了一个名为 OPArray 的数据，其结构如下：

```c
struct _zend_op_array {
	/* Common elements */
	zend_uchar type;
	zend_uchar arg_flags[3]; /* bitset of arg_info.pass_by_reference */
	uint32_t fn_flags;
	zend_string *function_name;
	zend_class_entry *scope;
	zend_function *prototype;
	uint32_t num_args;
	uint32_t required_num_args;
	zend_arg_info *arg_info;
	/* END of common elements */

	uint32_t *refcount;

	uint32_t last;
	zend_op *opcodes;

	int last_var;
	uint32_t T;
	zend_string **vars;

	int last_live_range;
	int last_try_catch;
	zend_live_range *live_range;
	zend_try_catch_element *try_catch_array;

	/* static variables support */
	HashTable *static_variables;

	zend_string *filename;
	uint32_t line_start;
	uint32_t line_end;
	zend_string *doc_comment;
	uint32_t early_binding; /* the linked list of delayed declarations */

	int last_literal;
	zval *literals;

	int  cache_size;
	void **run_time_cache;

	void *reserved[ZEND_MAX_RESERVED_RESOURCES];
};
```

内容超多对吧？简单的理解，其本质就是一个 OPCODE 数组外加执行过程中所需要的环境数据的集合。介绍几个相对来说比较重要的字段：

- `opcodes` 存放 OPCODE 的数组。
- `filename` 当前执行的脚本的文件名。
- `function_name` 当前执行的方法名称。
- `static_variables` 静态变量列表。
- `last_try_catch` `try_catch_array` 当前上下文中，如果出现异常 try-catch-finally 跳转所需的信息。
- `literals` 所有诸如字符串 foo 或者数字23，这样的常量字面量集合。

为什么需要生成这样庞大的数据？因为编译时期生成的信息越多，执行时期所需要的时间就越少。

接下来，我们看下 PHP 是如何执行 OPCODE。OPCODE 的执行被放在一个大循环中，这个循环位于 `zend/zend_vm_execute.h` 中的 `execute_ex` 函数：

```c
ZEND_API void execute_ex(zend_execute_data *ex)
{
	DCL_OPLINE

	zend_execute_data *execute_data = ex;

	LOAD_OPLINE();
	ZEND_VM_LOOP_INTERRUPT_CHECK();

	while (1) {
		if (UNEXPECTED((ret = ((opcode_handler_t)OPLINE->handler)(ZEND_OPCODE_HANDLER_ARGS_PASSTHRU)) != 0)) {
			if (EXPECTED(ret > 0)) {
				execute_data = EG(current_execute_data);
				ZEND_VM_LOOP_INTERRUPT_CHECK();
			} else {
				return;
			}
		}
	}

	zend_error_noreturn(E_CORE_ERROR, "Arrived at end of main loop which shouldn't happen");
}
```

这里，我去掉了一些环境变量判断分支，保留了运行的主流程。可以看到，在一个无限循环中，虚拟机会不断调用 OPCODE 指定的 `handler` 函数处理指令集，直到某次指令处理的结果 `ret` 小于0。注意到，在主流程中并没有移动 OPCODE 数组的当前指针，而是把这个过程放到指令执行的具体函数的结尾。所以，我们在大多数 OPCODE 的实现函数的末尾，都能看到调用这个宏：

```c
ZEND_VM_NEXT_OPCODE_CHECK_EXCEPTION();
```

在之前那个简单例子中，我们看到 vld 打印出的执行 OPCODE 数组中，最后有一项指令为 `ZEND_RETURN` 的 OPCODE。但我们编写的 PHP 代码中并没有这样的语句。在编译时期，虚拟机会自动将这个指令加到 OPCODE 数组的结尾。`ZEND_RETURN` 指令对应的函数会返回 -1，判断执行的结果小于0时，就会退出循环，从而结束程序的运行。

## 方法调用

如果我们调用一个自定义的函数，虚拟机会如何处理呢？

```php
<?php
function foo() {
    echo 'test';
}

foo();
```

我们通过 vld 查看生成的 OPCODE。出现了两个 OPCODE 指令执行栈，是因为我们自定义了一个 PHP 函数。在第一个执行栈上，调用自定义函数会执行两个 OPCODE 指令：`INIT_FCALL` 和 `DO_FCALL`。

```bash
compiled vars:  none
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   2     0  E >   NOP
   6     1        INIT_FCALL                                               'foo'
         2        DO_FCALL                                      0
         3      > RETURN                                                   1

compiled vars:  none
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   3     0  E >   ECHO                                                     'test'
   4     1      > RETURN                                                   null
```


其中，`INIT_FCALL` 准备了执行函数时所需要的上下文数据。`DO_FCALL` 负责执行函数。`DO_FCALL` 的处理函数根据不同的调用情况处理了大量逻辑，我摘取了其中执行用户定义的函数的逻辑部分：

```c
ZEND_VM_HANDLER(60, ZEND_DO_FCALL, ANY, ANY, SPEC(RETVAL))
{
    USE_OPLINE
    zend_execute_data *call = EX(call);
    zend_function *fbc = call->func;
    zend_object *object;
    zval *ret;

    ...

    if (EXPECTED(fbc->type == ZEND_USER_FUNCTION)) {
        ret = NULL;
        if (RETURN_VALUE_USED(opline)) {
            ret = EX_VAR(opline->result.var);
            ZVAL_NULL(ret);
        }

        call->prev_execute_data = execute_data;
        i_init_func_execute_data(call, &fbc->op_array, ret);

        if (EXPECTED(zend_execute_ex == execute_ex)) {
            ZEND_VM_ENTER();
        } else {
            ZEND_ADD_CALL_FLAG(call, ZEND_CALL_TOP);
            zend_execute_ex(call);
        }
    }

    ...

    ZEND_VM_SET_OPCODE(opline + 1);
    ZEND_VM_CONTINUE();
}
```

可以看到，`DO_FCALL` 首先将调用函数前的上下文数据保存到 `call->prev_execute_data`，然后调用 `i_init_func_execute_data` 函数，将自定义函数对象中的 `op_array`（每个自定义函数会在编译的时候生成对应的数据，其数据结构中包含了函数的 OPCODE 数组） 赋值给新的执行上下文对象。

然后，调用 `zend_execute_ex` 函数，开始执行自定义的函数。`zend_execute_ex` 实际上就是前面提到的 `execute_ex` 函数（默认是这样，但扩展可能重写 `zend_execute_ex` 指针，这个 API 让 PHP 扩展开发者可以通过覆写函数达到扩展功能的目的，不是本篇的主题，不准备深入探讨），只是上下文数据被替换成当前函数所在的上下文数据。

我们可以这样理解，最外层的代码就是一个默认存在的函数（类似 C 语言中的 `main()` 函数），和用户自定义的函数本质上是没有区别的。

## 逻辑跳转

我们知道指令都是顺序执行的，而我们的程序，一般都包含不少的逻辑判断和循环，这部分又是如何通过 OPCODE 实现的呢？

```php
<?php
$a = 10;
if ($a == 10) {
    echo 'success';
} else {
    echo 'failure';
}
```

我们还是通过 vld 查看 OPCODE（不得不说 vld 扩展是分析 PHP 的神器）。

```bash
compiled vars:  !0 = $a
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   2     0  E >   ASSIGN                                                   !0, 10
   3     1        IS_EQUAL                                         ~2      !0, 10
         2      > JMPZ                                                     ~2, ->5
   4     3    >   ECHO                                                     'success'
         4      > JMP                                                      ->6
   6     5    >   ECHO                                                     'failure'
   7     6    > > RETURN                                                   1
```
我们看到，`JMPZ` 和 `JMP` 控制了执行流程。`JMP` 的逻辑非常简单，将当前的 OPCODE 指针指向需要跳转的 OPCODE。

```c
ZEND_VM_HANDLER(42, ZEND_JMP, JMP_ADDR, ANY)
{
	USE_OPLINE

	ZEND_VM_SET_OPCODE(OP_JMP_ADDR(opline, opline->op1));
	ZEND_VM_CONTINUE();
}
```

`JMPZ` 仅仅是多了一次判断，根据结果选择是否跳转，这里就不再重复列举了。而处理循环的方式与判断基本上是类似的。

```php
<?php
$a = [1, 2, 3];
foreach ($a as $n) {
    echo $n;
}
```

```bash
compiled vars:  !0 = $a, !1 = $n
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   2     0  E >   ASSIGN                                                   !0, <array>
   3     1      > FE_RESET_R                                       $3      !0, ->5
         2    > > FE_FETCH_R                                               $3, !1, ->5
   4     3    >   ECHO                                                     !1
         4      > JMP                                                      ->2
         5    >   FE_FREE                                                  $3
   5     6      > RETURN                                                   1
```

循环只需要 `JMP` 指令即可完成，通过 `FE_FETCH_R` 指令判断是否已经到达数组的结尾，如果到达则退出循环。

## 结语

通过了解 Zend 虚拟机，相信你对 PHP 是如何运行的，会有更深刻的理解。想到我们写的一行行代码，最后机器执行的时候会变成数不胜数的指令，每个指令又建立在复杂的处理逻辑之上。那些从前随意写下的代码，现在会不会在脑海里不自觉的转换成 OPCODE 再品味一番呢？
