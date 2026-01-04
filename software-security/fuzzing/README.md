# 模糊测试（Fuzzing）学习

[TOC]

## 1. 模糊测试概念梳理

### 1.1 什么是模糊测试（Fuzzing）

- **模糊测试**又称为 `Fuzzing`，是一种软件测试技术。其核心概念为**自动**产生**随机输入**到一个程序中，并监视程序异常，如崩溃、断言失败，以发现可能的程序错误（[模糊测试 - 维基百科，自由的百科全书](https://zh.wikipedia.org/wiki/模糊测试)）

### 1.2 一个简单的示例：test.c

- `test.c` 是你要测试的程序，从 `stdin` 读取 `8` 个字节后打印，但打印前会判断 `input` 的前两个字节是否为`AB`。如果是的话就会执行到代码 `(2)` ，并触发**段错误**结束程序

  ```c
  // test.c
  // gcc -o test test.c
  #include <unistd.h>
  
  int main()
  {
      char input[8] = {0};
      read(STDIN_FILENO, input, 8);
      if (input[0] == 'A' && input[1] == 'B') // (1)
          *((unsigned int *)0) = 0xdeadbeef;  // (2)
      write(STDOUT_FILENO, input, 8);
      return 0;
  }
  ```

- 但实际上大型项目往往有**上百万行代码**，中小型至少也有**数千数万行代码**，人工逐行分析或测试几乎不可行，将这种测试自动化才是可行之举，因此出现了**模糊测试**的出现

- 模糊就是去：
  1. 自动测试执行目标项目
  2. 自动生成或枚举不同的输入数据
  3. 根据程序的返回值、崩溃状态等判断是否发生异常

- 而负责做这些事情的程序称为**模糊器**（`Fuzzer`），并且根据开发或执行效率，会选择使用不同的语言来实际工作

### 1.3 实现一个简单的 Fuzzer

- 针对 `test.c`，我们可以用 Python 编写一个简单的 `fuzzer` `fuzzer.py` ，自动向程序喂入不同输入，并由退出状态 `(1)` 判断 `test` 是否发生执行异常，最终会找到元素 `'AB'` 会触发行为异常

  ```python
  # fuzzer.py
  import subprocess
  
  target = './test'
  inps = ['AA', 'BB', 'BA', 'AB']
  
  for inp in inps:
      try:
          subprocess.run([target], input=inp.encode(), capture_output=True, check=True)
      except subprocess.CalledProcessError: # (1)
          print(f"bug found with input: '{inp}'")
  
  # (output)
  # bug found with input: 'AB'
  ```

- 在模糊测试之前，为了检测程序的功能是否正常，通常会写一些测试脚本或者**单元测试**

  | 测试方式 | 关注点                               |
  | -------- | ------------------------------------ |
  | 单元测试 | 程序在**正常使用场景**下是否正确     |
  | 模糊测试 | 程序在**非预期、异常输入**下是否安全 |

- 在实际使用中，正常用户通常会按照预期方式使用服务，若单元测试通过，往往意味着功能满足需求；但攻击者并不会遵循正常使用方式，会输入非法、畸形或极端数据，如果程序未对这些情况进行检查，可能导致服务崩溃（DoS），甚至被获取主机控制权
- 模糊测试正是从**攻击者视角**出发，自动生成异常输入，不断触发不同执行路径，判断哪些输入能够触发漏洞

## 2. 内部架构模糊测试

### 2.1 Basic Block 与 Control Flow Graph（CFG）

- 在程序执行过程中，会因为不同的条件执行不同的代码路径，而这些条件通常由 `if / else` 语句控制

  ```c
  if (a == 1 && b == 2)
  	puts("condition 1");
  else
      puts("condition 2");
  ```

- 如果将上述执行逻辑画成图，就可以得到**控制流图**（`Control Flow Graph`，`CFG`）。`CFG` 中的每一个节点称为一个 `Basic Block` ，**`Basic Block` 是不可再拆分的最小执行单元**。每个 `Basic Block` 具有以下特性：
  1. 一定从第一条指令开始执行
  2. 不会从中间被其他分支跳入
  3. 只要进入该 `Basic Block`，其中的所有指令都会被顺序执行

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);
    width: 30%; height: auto;" 
    src="./images/2-0.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">CFG 和 Basic Block</div>
</center>

- 下图是使用反汇编工具（如 `IDA`）生成的指令级 `CFG`，对照 `Basic Block` 的定义，可以清楚看到指令是如何被划分到不同 `Basic Block` 中的

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);
    width: 75%; height: auto;" 
    src="./images/2-1.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">指令级 CFG</div>
</center>

### 2.2 Fuzzing 的整体流程概览

- 从宏观上看，一个完整的 `Fuzzing` 流程可以拆解为三个核心组件：
  1. `Seed Selection`（种子选择）
  2. `Mutation`（变异）
  3. `Coverage`（衡量覆盖率）

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);
    width: 60%; height: auto;" 
    src="./images/2-2.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">Fuzzing 整体流程</div>
</center>
- `Corpus` （语料库）指的是曾经产生过 **新 `coverage`** 的输入集合，或在初始化阶段提供的初始 `seed`

  1. 以 `fuzzer.py` 为例，`inps = ['A', 'B']` 即是所有 `seed`，可以视为 `corpus`

     ```python
     inps = ['A', 'B']
     ```

- 不同的 `seed` 会具有不同属性，如执行速度、覆盖的代码路径（`coverage`）。`Seed Selection` 就是根据这些属性，决定下一次使用哪一个 `seed` 作为输入的算法

  1. `fuzzer.py` 的种子选择算法比较简单，始终选择最旧的 `seed ` `inps[0]`作为下一次测试的输入

     ```python
     inp = inps[0]  # (1)
     ```

- 选定 `seed` 后，`fuzzer` 会对其进行 `Mutation`，以增加输入的随机性

  1. 在 `fuzzer.py` 中，变异演算会随机挑选出来一个字符，追加在原 `seed` 后面，生成最终送入目标程序的 `input`

     ```python
     inp += random.choice(['A', 'B', 'C'])  # (2)
     ```

  2. 在本例中，`test.c` 的逻辑是：当输入满足不同条件时，会输出 `"AAA"` 或 `"BBB"`；输出越多，说明执行路径越“深入”。在 `fuzzer.py` 中：如果程序有输出，说明该输入执行到了更深的路径，该输入被认为是 `interesting input`，并被加入 corpus。所以在这里，可以将**输出的有无**视为一种非常简化的 `coverage` 形式

- 在实际的 `fuzzing` 中，`coverage` 通常指**程序执行过程中覆盖了多少代码**。而程序代码本质上是由多个 `Basic Block` 组成，因此覆盖的 `Basic Block` 越多，越有可能触发潜在漏洞

- 为了统计哪些 `Basic Block` 被执行，需要在程序中插入记录代码，这种技术称为**插桩**（`Instrumentation`）。`Instrumentation` 会在 `Basic Block` 执行前插入统计逻辑，用于记录执行路径

- `Mutation` 有时效果并不好，可能导致长时间无法获得新的 `coverage`，因此，`fuzzer` 通常会对 `corpus` 做一些“扰动”操作，如定期重置 `corpus`、打乱 `seed` 顺序、强制引入初始 `seed`

  1. `fuzzer.py` 的扰动操作为定期重置 `corpus` ，这种做法可以避免 `fuzzing` 陷入局部最优状态

     ```python
     if count % 100 == 0 or len(inps) == 0:  # (4)
         inps = ['A', 'B']
     ```

- 示例代码：

```c
// test.c
// gcc -o test test.c
#include <unistd.h>
#include <stdio.h>

int main()
{
    char input[8] = {0};
    read(STDIN_FILENO, input, 8);

    if (input[0] == 'A') {
        puts("AAA");
        if (input[1] == 'B') {
            puts("BBB");
            if (input[2] == 'C') {
                *((unsigned int *)0) = 0xdeadbeef; // bug
            }
        }
    }
    return 0;
}
```

```python
# fuzzer.py
import subprocess
import random

target = './test'
inps = ['A', 'B'] # 语料库
count = 1

while True:
    inp = inps[0] # (1) 种子选择算法比较简单，始终选择最旧的 seed 作为下一次测试的输入
    inp += random.choice(['A', 'B', 'C']) # (2) 变异演算会随机挑选出来一个字符，追加在原 seed 后面
    del inps[0]
    count += 1

    try:
        comp = subprocess.run([target], input=inp.encode(), capture_output=True, check=True)
        if comp.stdout != b'': # 如果输出不为空
            inps.append(inp) # (3) 那么代表此输入为有趣 (interesting)
    except subprocess.CalledProcessError:
        print(f"bug found with input: '{inp}'")
        break

    if count % 100 == 0 or len(inps) == 0: # (4) 定期打乱语料库，避免变异效果不好导致输入无法取得新的覆盖范围
        inps = ['A', 'B']
```

- 因此一个 `Fuzzer` 的好坏通常从以下三个方面进行评估：
  1. `Seed Selection`：是否能选出真正有价值的 `seed`
  2. `Mutation`：随机策略是否高效、是否能持续探索新路径
  3. `Coverage`：覆盖率获取方式是否精确、是否引入过高的运行开销（`overhead`）

- 一些比较有名的**模糊测试工具**
  1. `American Fuzzy Lop`（`AFL`）：`AFL` 是一个高效的模糊测试工具
  2. `libFuzzer`：`libFuzzer` 是 `LLVM/Clang` 提供的一个模糊测试引擎，它可以轻松地集成到现有的代码中
  3. `Syzkaller`：`Syzkaller` 是一个专注于操作系统系统调用接口的模糊测试工具，它可以自动生成各种系统调用序列，并对内核进行测试以发现漏洞和错误
  4. `OSS-Fuzz`：`OSS-Fuzz` 旨在通过自动化模糊测试发现开源软件中的安全漏洞和错误

## 3. OS fuzzer - syzkaller - 介绍

