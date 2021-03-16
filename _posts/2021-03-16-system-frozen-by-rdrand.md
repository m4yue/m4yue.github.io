---
layout: default
title: 随机指令RDRAND导致的系统挂起
---

# 随机指令RDRAND导致的系统挂起

## 起因
程序`sway`在amd的处理器上运行时，系统[无响应](https://theavid.dev/sway-rng)

调查后发现是因为json-c库在用`RDRAND`取随机数时，得到了返回值-1，从而进入了无限循环。

`https://github.com/json-c/json-c/blob/31f1ab2be1142d982fc9b8c8913bfd1233599ebb/random_seed.c#L72`
```c
/* get_rdrand_seed - GCC x86 and X64 */

#if defined __GNUC__ && (defined __i386__ || defined __x86_64__)

#define HAVE_RDRAND 1

static int get_rdrand_seed(void)
{
    DEBUG_SEED("get_rdrand_seed");
    int _eax;
    // rdrand eax
    __asm__ __volatile__("1: .byte 0x0F\n"
                         "   .byte 0xC7\n"
                         "   .byte 0xF0\n"
                         "   jnc 1b;\n"
                         : "=a" (_eax));
    return _eax;
}

#endif
```

调查中使用的技巧
1. linux kernel的`sysreq`机制，可以将无响应的系统控制权转换给用户
2. gcore命令产生指定进程的coredump，以便分析程序挂起的位置
3. gdb分析core file

## 处理

讨论见https://github.com/json-c/json-c/pull/589

1. 在该处取随机数的位置增加判断，如果为AMD的处理器，则该指令不可用，同时多次判断取到的值是否为-1
    1. 此方案不通用，是针对特定的bug的补丁，未来可能其他处理器也会有类似问题
2. 多次采样，如果他们的值都是一样的，那么很大可能，该指令不可用
    1. 此方案会更加通用，不针对特定的处理器，也可以兼容未来可能出现问题的其他处理器(比如不返回-1，而是返回0或其他特殊值)
    2. 采样次数可以为8/10次，linux kernel也是这样做的
    3. 此方案有个问题是会存在误报，对于正常工作的cpu，理论上也会出现多次`RDRAND`同一个值
    ```
    对于RDRAND获取32位的随机数，获取到A的概率为1/(2^32)，连续取10次均获取到A的概率为'1/(2^32) * 1/(2^32) * 1/(2^32) *1/(2^32) *1/(2^32) *1/(2^32) *1/(2^32) *1/(2^32) *1/(2^32) *1/(2^32)'，约为4.6816763546921983e-97
    ```
    4. 在2020年5月的时候，json-c的库采用了此方案
3. 在[hackernews](https://news.ycombinator.com/item?id=26463688)讨论发现，上述方案2存在争议
    1. 为什么需要在用户空间去使用这类特殊的cpu指令，为什么不用操作系统提供的接口
        1. 因为这个库的使用时机早于系统的/dev/random
        2. json-c中需要使用随机数的原因是为了获取随机种子，以便对字符串hash，并抵御hash denial-of-service attacks
    2. 由于在初始化后检查了`rdrand`是否可用，后续不再检查，在特定的情况下仍会有问题
        1. 如vm随眠并恢复后，或者vm迁移
    3. 该处取随机数时如果没法使用rdrand，则会回退使用其他方法，如当前时间。为什么不直接使用回退方法，会更加健壮。而且rdrand很慢，一般为110多纳秒