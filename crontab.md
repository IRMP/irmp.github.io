# linux相关

## Crontab

用法：

**crontab** [选项]

| -e   | 编辑定时任务             |
| ---- | ------------------------ |
| -l   | 查询定时任务             |
| -r   | 删除当前用户所有定时任务 |

service crond restart 重启调度任务

**5个占位符说明**

| 项目    | 含义                 | 范围                    |
| ------- | -------------------- | ----------------------- |
| 第一个* | 一小时当中的第几分钟 | 0-59                    |
| 第二个* | 一天当中的第几个小时 | 0-23                    |
| 第三个* | 一个月当中的第几天   | 1-31                    |
| 第四个* | 一年当中的第几个月   | 1-12                    |
| 第五个* | 一周当中的星期几     | 0-7（0和7都代表星期日） |

**特殊符号说明**

| 特殊符号 | 含义                 |
| -------- | -------------------- |
| *        | 代表任何时间         |
| ，       | 代表不连续的时间     |
| -        | 代表连续时间         |
| */n      | 代表没隔多久执行一次 |

