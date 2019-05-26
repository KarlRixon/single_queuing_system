## 一、实验内容和目的

### 1.修理店空闲概率

在出队事件中，如果队列为空，则开始空闲状态，记录此时间。

```C
if (num_in_q == 0) {

    /* The queue is empty so make the server idle and eliminate the
        departure (service completion) event from consideration. */

    // 记录空闲状态的开始时间
    last_idle = sim_time;

    server_status      = IDLE;
    time_next_event[2] = 1.0e+30;
}
```

在到达事件中，结束空闲状态，更新空闲总时长。

```C
++num_custs_delayed;
server_status = BUSY;

// 结束空闲状态，刷新空闲总时长
area_idle += sim_time - last_idle;
```

将结果输出到文件

```C
fprintf(outfile, "Idle probability%17.3f\n\n", area_idle / sim_time);
```

此外可以直接用1-服务利用率来计算空闲率，结果相同。

### 2.店内有3个顾客的概率

在顾客到达事件中判断并记录店内有三个顾客的开始时间
如果队列中人数超过2个即店内顾客超过3人，则更新店内有3位顾客的总时间。

```C
// 记录店内有三个顾客的开始时间
if(num_in_q == 2){
    last_three = sim_time;
}
if(num_in_q == 3){
    area_three += sim_time - last_three;
}
```

在顾客出队事件中，同样进行是否有三个顾客的判断，并更新累计时间。

```C
--num_in_q;

// 判断店内顾客小于3人或从多于3人回到3人
if(num_in_q == 1){
    area_three += sim_time - last_three;
}
if(num_in_q == 2){
    last_three = sim_time;
}
```

输出结果

```C
fprintf(outfile, "Three customers probability%6.3f\n\n", area_three / sim_time);
```

### 3.店内至少有一个顾客的概率

即1-空闲概率

### 4.在店内顾客的平均数

在timing后的统计函数中判断，如果处于BUSY状态，说明有顾客在店内，则更新顾客在店内总时长。

```C
/* Compute time since last event, and update last-event-time marker. */

time_since_last_event = sim_time - time_last_event;
time_last_event       = sim_time;

// 如果处于忙碌状态则更新顾客在店内总时长
if(server_status == BUSY){
    total_custs_time += (num_in_q+1) * time_since_last_event;
}
```

最后用所有顾客在店内的时长除以sim_time即为店内顾客平均数

```C
fprintf(outfile, "Average number of customer%7.3f\n\n", total_custs_time / sim_time);
```

此外店内至少有一人的概率等于服务利用率。

### 5.顾客在店内的平均逗留时间

用第四题中算出的所有顾客在店内逗留时间除以顾客数

### 6.顾客必须在店内消耗15min的概率

顾客在店内消耗时间包括服务时间和等待时间，由于可能有多个用户同时等待，所以需要建一个数组对每个用户在店内消耗的时间进行统计，然后在离开事件中判断如果消耗时间大于15min则做记录。
在到达事件中对顾客到达时间做记录

```C
// 不管是否处于忙状态，都要对顾客到达时间做记录
cust_time[num_in_q] = sim_time;
```

顾客离开，结算逗留时间，若大于15分钟，做记录，并更新时间记录列表。

```C
// 对离开的顾客结算逗留时间，判断是否达到15min，并更新记录数组
if((sim_time - cust_time[0]) >= 15){
    time_le_15min ++;
}
for(i = 0; i<=num_in_q; i++){
    cust_time[i] = cust_time[i+1];
}
```

输出结果

```C
fprintf(outfile, "Time more than 15min probability%6.3f\n\n", time_le_15min / num_delays_required);
```

## 二、实验结果

输入

```txt
       12.0       4.0      1000
```

```txt
Single-server queueing system

Mean interarrival time     12.000 minutes

Mean service time           4.000 minutes

Number of customers          1000



Average delay in queue      2.122 minutes

Average number in queue     0.173

Server utilization          0.339

Idle probability            0.661

Three customers probability 0.028

At least one customer       0.339

Average number of customer  0.512

Average time of customer    6.265

Time more than 15min probability 0.070

Time simulation ended   12231.693 minutes
```

## 三、运行情况分析与取得的经验

在main函数中，timing负责在下一个到达事件与下一个出队事件的时间中选出最小的，用这个时间更新当前模拟时间，来推进模拟进度，并设置下一个事件的标志，如果下一个事件为到达事件则执行arrive函数，否则执行depart函数。Timing推进当前时刻，而arrive与depart则是为下一次timing做准备，因为每次调用timing都要看最近的下一个事件是到达还是出队。
如果执行到达操作，先判断当前状态是都为忙碌，若是，则将该顾客加入等待队列，否则直接对顾客服务。
如果执行出队操作，先判断队列中是否有等待者，若没有说明该顾客已接受完服务准备离店。这里可以做一些统计工作。若队列中还有人，说明轮到该顾客接受服务。
模拟的总时间可以分为多个时间段，每个时间段末尾是一个到达或出队事件，timing推进一个时间片段后，紧接着执行update_time_avg_stats函数来统计一些数据。此外也可以在到达和出队操作中的某一步后进行统计。

经验：我觉得这个程序最妙的一个地方是将顾客离店和顾客出队用一个对等待队列是否为空的判断联系起来。
此外使用到达时间的指数分布来模拟顾客到达次数的泊松分布，以及指数分布随机数生成模块蕴含大量概率论及数学知识。需要仔细搞清楚才能完成题目要求。