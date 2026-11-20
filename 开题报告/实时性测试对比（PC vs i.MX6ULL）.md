# 实时性测试对比（PC Ubuntu vs i.MX6ULL）

> 日期：2026-11-20
> 用途：记录两个平台的实时延迟实测数据，作为课题"确定性部署"的基线材料
> 核心指标：延迟 min / avg / max（看 max，即最坏情况 / WCET 近似）

---

## 一、测试目的

验证"AI 推理慢而不确定 vs 控制环快而确定"中，**控制侧（实时任务）在不同平台上的确定性差异**，为后续异构平台（Orin/RK3588 + STM32）的时延预算提供参照。

---

## 二、测试平台

| | PC | i.MX6ULL |
|---|---|---|
| 芯片/架构 | x86-64 | ARM Cortex-A7（armv7l，**单核**） |
| 系统 | Ubuntu 24.04 | 正点原子 Linux 4.1.15 |
| 内核配置 | lowlatency（PREEMPT） | CONFIG_PREEMPT=y，**CONFIG_HZ=100** |
| 内存/算力 | 20 核，强 | 512MB，BogoMIPS 12（弱） |
| glibc | 2.39 | **2.23** |

---

## 三、PC（x86 Ubuntu）测试

### 3.1 操作命令

```bash
# 安装低延迟内核与实时测试工具
sudo apt install linux-lowlatency
sudo apt install rt-tests

# 切换内核后确认
uname -r                      # 应为 6.8.0-xxx-lowlatency

# 不绑核，测基线
sudo cyclictest -l100000 -m -p99 -i1000

# 绑核 3，消除线程迁移
sudo cyclictest -l100000 -m -p99 -i1000 -a 3

# 加直方图，看延迟分布
sudo cyclictest -l100000 -m -p99 -i1000 -a 3 -h 200
```

### 3.2 实测结果

| 配置 | Min | Avg | Max |
|---|---|---|---|
| 普通内核，不绑核（10 万次） | 2 μs | 3 μs | 195 μs |
| 普通内核，绑核 3（10 万次） | 1 μs | 2 μs | 80 μs |
| **lowlatency，绑核 3（直方图）** | **3 μs** | **3 μs** | **34 μs** |

> 直方图关键结论：**99.5% 的周期延迟落在 3~8 μs**，10 万次里仅 2 次飞到 24/34 μs。

---

## 四、i.MX6ULL 测试

### 4.1 环境要点

- 板子：正点原子 i.MX6ULL MINI（单核 Cortex-A7）
- 内核：Linux 4.1.15，`CONFIG_PREEMPT=y`、**`CONFIG_HZ=100`**
- glibc 2.23（老）→ **必须 `-static` 静态编译**，否则报 `GLIBC_2.34 not found`

### 4.2 操作命令

```bash
# ---- PC 端：交叉编译（静态） ----
arm-linux-gnueabihf-gcc -O2 -static min_cyc.c -o min_cyc
file min_cyc      # 确认 ARM + statically linked

# ---- 传入板子：串口 rz ----
# 板子执行 rz；PC 端 picocom 按 Ctrl+A Ctrl+S 选择 min_cyc 传输

# ---- 板子端：运行与诊断 ----
chmod +x min_cyc
./min_cyc                    # 裸跑（普通调度）
chrt -f 99 ./min_cyc         # SCHED_FIFO 实时优先级

# 关键诊断命令
ldd --version                                  # glibc 2.23
uname -r                                       # 4.1.15
cat /proc/cpuinfo                              # 单核 processor:0
zcat /proc/config.gz | grep -E "CONFIG_PREEMPT|CONFIG_HZ="
cat /proc/interrupts                           # 中断分布（串口中断极高）
```

### 4.3 实测结果（1kHz 周期，1 万次）

| 配置 | Min | Avg | Max |
|---|---|---|---|
| 裸跑（普通调度） | 70.3 μs | 76.1 μs | **2153.7 μs（2.15 ms）** |
| `chrt -f 99`（SCHED_FIFO） | 20.0 μs | 25.1 μs | **1006.7 μs（≈1 ms）** |

### 4.4 关键诊断

- **`CONFIG_HZ=100`**：内核 tick 10ms 一次，对 1ms 实时任务粒度太粗（主要硬伤）。
- **`CONFIG_PREEMPT=y`**：内核可抢占，此项不是瓶颈。
- **单核**：所有中断压在 CPU0，无法绑核/隔离中断。
- **串口中断极高**（87 万次）：使用串口控制台期间会偶发打断实时线程。

---

## 五、对比与分析

| 指标 | PC（lowlatency+绑核3） | i.MX6ULL（chrt -f 99） | 差距 |
|---|---|---|---|
| Min | 3 μs | 20 μs | ~7 倍 |
| Avg | 3 μs | 25 μs | ~8 倍 |
| **Max（WCET 近似）** | **34 μs** | **1007 μs** | **~30 倍** |

**分析**：

1. **PC 能压到 μs 级**：多核 + 可绑核 + 高 HZ + lowlatency，最坏 34μs；
2. **i.MX6ULL 最坏 1~2ms**：单核 + HZ=100 + 中断不可隔离，最坏接近/超过 1ms，对 1kHz 控制环是"错过一个周期"；
3. **SCHED_FIFO 的作用**：i.MX6ULL 上加实时优先级后 avg 从 76μs→25μs，明显改善，但 max 仍被 HZ=100 卡住；
4. **确定性不只靠优先级**，还受"内核配置（HZ）+ 核数（能否绑核）+ 中断路由"共同制约。

---

## 六、结论与对课题的启示

1. **单核 A7 平台的实时极限**：i.MX6ULL 上 avg 25μs / max ~1ms 已是该平台（HZ=100）的真实上限，非代码问题；
2. **最终平台应多核**：Orin/RK3588 多核，可 **绑核 + 中断亲和性隔离 + 高 HZ**，才能把 max 压到可控范围；
3. **本文"确定性部署"的手段**：系统级实时优化（CPU 隔离、绑核、SCHED_FIFO、中断亲和、mlockall），而非依赖 PREEMPT_RT（与 Orin GPU 驱动冲突）；
4. **论文可用数据**：本章数据可直接用于"系统平台与时延特性"部分的基线对比。

---

## 附录：min_cyc.c 核心代码

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#define PERIOD_NS 1000000LL   /* 1ms = 1kHz */
#define LOOPS     10000       /* 约10秒 */

int main(int argc, char **argv) {
    long loops = (argc > 1) ? atol(argv[1]) : LOOPS;
    struct timespec next;
    clock_gettime(CLOCK_MONOTONIC, &next);

    long long min = 1LL << 60, max = 0, sum = 0;
    for (long i = 0; i < loops; i++) {
        next.tv_nsec += PERIOD_NS;
        while (next.tv_nsec >= 1000000000LL) { next.tv_nsec -= 1000000000LL; next.tv_sec++; }
        clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, &next, NULL);

        struct timespec now;
        clock_gettime(CLOCK_MONOTONIC, &now);
        long long lat = (now.tv_sec - next.tv_sec) * 1000000000LL + (now.tv_nsec - next.tv_nsec);
        if (lat < 0) lat = -lat;
        if (lat < min) min = lat;
        if (lat > max) max = lat;
        sum += lat;
    }
    printf("cycles = %ld\n", loops);
    printf("min    = %lld ns (%.3f us)\n", min, min / 1000.0);
    printf("avg    = %.1f ns (%.2f us)\n", (double)sum / loops, (double)sum / loops / 1000.0);
    printf("max    = %lld ns (%.3f us)\n", max, max / 1000.0);
    return 0;
}
```
