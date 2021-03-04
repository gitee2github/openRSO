驱动接口操作说明
=========

# 1 内核启动参数
- mpam启动需要ACPI支持，**cmdline**需要添加**mpam=acpi**参数。

# 2 接口总览
驱动接口位于 **/sys/fs/resctrl**。

```
/sys/fs/resctrl(根分组)
 ├── cpus                      # bitmask方式显示根分组关联的vcpu
 ├── cpus_list                 # cpu list方式显示根分组关联的vcpu
 ├── info                      # 用于显示属性信息及错误提示信息
 │   ├── L3
 │   │   ├── cbm_mask          # 系统所支持的最大cache way bitmask，一个bit代表一个cache way
 │   │   ├── features          # 挂载参数时，显示该挂载功能的配置范围（最大值，整型）
 │   │   ├── min_cbm_bits      # 使用schemata所能配置的最小cache way bitmask
 │   │   ├── num_closids       # L3能够提供创建控制组的最大数量
 │   │   └── shareable_bits    # 当前所有cbm_mask全部shareable，支持后续扩展
 │   ├── L3_MON
 │   │   ├── num_monitors      # 可同时监控控制组或监控组的数量
 │   │   └── num_rmids         # 可创建控制组和监控组的总数
 │   ├── last_cmd_status       # 操作错误提示
 │   ├── MB
 │   │   ├── bandwidth_gran    # 带宽百分比配置粒度
 │   │   ├── features          # 同L3 features
 │   │   ├── min_bandwidth     # 最小带宽配置百分比
 │   │   └── num_closids       # 同L3 num_closid
 │   └── MB_MON
 │       ├── num_monitors      # 同L3 num_monitors
 │       └── num_rmids         # 同L3 num_rmids
 ├── mon_data
 │   ├── mon_L3_00             # 标号代表NUMA node id，表示当前分组所关联的pid/vcpu在NUMA0上实际占用L3 Cache大小，下同
 │   ├── mon_L3_01
 │   ├── mon_L3_02
 │   ├── mon_L3_03
 │   ├── mon_MB_00
 │   ├── mon_MB_01
 │   ├── mon_MB_02
 │   └── mon_MB_03
 ├── mon_groups                # 创建监控组目录
 ├── rmid                      # 指示控制组或监控组标识，标识唯一
 ├── schemata                  # 资源使用配置接口
 └── tasks                     # 显示与根组关联的pid
```
> 备注 1：创建控制组在根分组下，创建监控组在控制组的mon_groups目录下
> 备注 2：可创建控制组的数量取所有资源的num_closids最小值
> 备注 3：控制组和监控组总数超过L3 num_monitors时会影响L3 Cache监控的准确性
> 备注 4：resctrl支持L3 Cache CODE和DATA分开配置，参考下文，若使能，mon_data目录亦相应变化

资源配置通过schemata接口进行控制，如下：
```
[root@localhost]# cat schemata
L3:0=7fff;1=7fff;2=7fff;3=7fff       # 标号代表NUMA node id
MB:0=100;1=100;2=100;3=100
```

# 3 接口挂载方式
当前挂载参数包括：**cdpl3**，**caPbm**，**mbMax**，**mbMin**，**mbHdl**，可互相搭配组合使用,
参数含义参考备注5。

- 使用默认参数：
```
[root@localhost fs]# mount -t resctrl resctrl /sys/fs/resctrl/
[root@localhost fs]# cat /sys/fs/resctrl/schemata
L3:0=7fff;1=7fff;2=7fff;3=7fff # L3选择caPbm控制方式
MB:0=100;1=100;2=100;3=100     # MB默认选择mbMax控制方式
```

- 开启L3 Code Data Prioritization（仅软件行为）：
```
[root@localhost fs]# mount -t resctrl resctrl /sys/fs/resctrl/ -o cdpl3
[root@localhost fs]# cat /sys/fs/resctrl/schemata
L3CODE:0=7fff;1=7fff;2=7fff;3=7fff # Code控制
L3DATA:0=7fff;1=7fff;2=7fff;3=7fff # Data控制
MB:0=100;1=100;2=100;3=100
```

- 手动选择L3/MB控制方式，支持后续扩展：
```
[root@localhost fs]# mount -t resctrl resctrl /sys/fs/resctrl/ -o caPbm,mbMax,mbMin
[root@localhost fs]# cat /sys/fs/resctrl/schemata
L3PBM:0=7fff;1=7fff;2=7fff;3=7fff # L3手动选择CPBM控制方式
MBMAX:0=100;1=100;2=100;3=100     # MB手动选择mbMax控制方式
MBMIN:0=0;1=0;2=0;3=0             # MB手动选择mbMin控制方式

[root@localhost resctrl]# cat info/MB/features # 查看控制方式的上限
caPbm@7fff
mbMax@100
mbMin@100
```

- 手动选择L3/MB控制方式，同时开启cdpl3
```
[root@localhost fs]# mount -t resctrl resctrl /sys/fs/resctrl/ -o caPbm,mbMax,mbMin,cdpl3
[root@localhost fs]# cat /sys/fs/resctrl/schemata
L3CODEPBM:0=7fff;1=7fff;2=7fff;3=7fff
L3DATAPBM:0=7fff;1=7fff;2=7fff;3=7fff
MBMAX:0=100;1=100;2=100;3=100
MBMIN:0=0;1=0;2=0;3=0
```

- 使能MB hardlimit，手动配置流量强制约束
```
[root@localhost fs]# mount -t resctrl resctrl /sys/fs/resctrl/ -o mbHdl
[root@localhost fs]# cat /sys/fs/resctrl/schemata
L3:0=7fff;1=7fff;2=7fff;3=7fff
MB:0=100;1=100;2=100;3=100
MBHDL:0=1;1=1;2=1;3=1 #手动配置hardlimit，更加灵活限制流量
```

## 接口配置和使用

/sys/fs/resctrl默认为根分组，根分组可以创建若干个控制组，一个控制组可以关联一组pid/vcpu，
通过配置控制组可达到限制L3 Cache/Memory Bandwidth使用的效果，通过读取对应分组mon_data接口可以获取该组资源占用情况。

- 创建一个新的控制组，关联vcpu/pid
```
[root@localhost resctrl]# cd /sys/fs/resctrl/ && mkdir p1
[root@localhost resctrl]# cd p1 && echo $$ > tasks # 关联当前shell进程pid到p1组
[root@localhost p1]# cat tasks # 可查看关联pid
29190
29607

[root@localhost resctrl]# cd p1 && echo '0-1' > cpus_list
[root@localhost p1]# cat cpus_list   #可查看关联cpu
0-1

[root@localhost resctrl]# cat info/L3/num_closids # 可在info对应的资源的目录下查看closid数量，即可以创建控制组的数量
16
[root@localhost resctrl]# cat info/MB/num_closids
16

[root@localhost resctrl]# cat info/L3_MON/num_rmids # 创建每一个控制组或监控组均需要消耗一个rmid
16
[root@localhost resctrl]# cat info/MB_MON/num_rmids
16
```

- 对一个控制组限制L3 Cache使用
```
[root@localhost resctrl]# cat info/L3/cbm_mask # 查看info目录下对应资源的属性
7fff
[root@localhost resctrl]# cat info/L3/min_cbm_bits
1

[root@localhost resctrl]# cd /sys/fs/resctrl/p1
[root@localhost p1]# cat schemata
L3:0=7fff;1=7fff;2=7fff;3=7fff
MB:0=100;1=100;2=100;3=100
[root@localhost p1]# echo 'L3:0=1' > schemata   #配置1条cache way给p1分组
[root@localhost p1]# cat schemata
L3:0=1;1=7fff;2=7fff;3=7fff #若此时改组关联了pid/cpu，那么该pid/cpu产生的访存请求只能够allocate到这个cache way
MB:0=100;1=100;2=100;3=100
```

- 对一个控制组限制MB使用
```
[root@localhost resctrl]# cat info/MB/min_bandwidth #和配置L3类似，也可以查看MB的相关信息
1
[root@localhost resctrl]# cat info/MB/bandwidth_gran
2
#可知，配置带宽最小百分比是1%，粒度是2%，注意尽管可配置1%，因为粒度是2%，所以实际可配置最小带宽是2%。
[root@localhost p1]# cat schemata
L3:0=1;1=7fff;2=7fff;3=7fff
MB:0=100;1=100;2=100;3=100
[root@localhost p1]# echo 'MB:0=1' > schemata
[root@localhost p1]# cat schemata
L3:0=1;1=7fff;2=7fff;3=7fff
MB:0=2;1=100;2=100;3=100
```

- 读取控制组的监控数据

```
[root@localhost resctrl]# cd /sys/fs/resctrl/
[root@localhost resctrl]# grep . mon_data/* # 直接使用mon_data即可获取
mon_data/mon_L3_00:48165888
mon_data/mon_L3_01:57714688
mon_data/mon_L3_02:46344192
mon_data/mon_L3_03:47719424
mon_data/mon_MB_00:1048
mon_data/mon_MB_01:0
mon_data/mon_MB_02:0
mon_data/mon_MB_03:0

[root@localhost resctrl]# cat info/L3_MON/num_monitors # 需要注意创建了超过总数为8组的控制组/监控组之后可能后续创建的分组监控数据不准
8
```

- 创建监控组
```
[root@localhost resctrl]# cd /sys/fs/resctrl/p1
[root@localhost p1]# cd mon_groups/ && mkdir m1 # 监控组只能监控，m1分组资源配置跟随p1分组
[root@localhost mon_groups]# ls m1/
cpus  cpus_list  mon_data  rmid  tasks
[root@localhost m1]# echo '0-1' > cpus_list
[root@localhost m1]# cat cpus_list
0-1
#此时p1控制组监控的是p1控制组本身及所有子监控组的监控值之和。
```

> 备注 5：关键字解释，具体含义参考[arm MPAM 手册][1]：
**cdpl3**: L3 Code Data Prioritization;
**caPbm**: Cache Portion Bit Map，按照位图控制分配特定容量和特定位置的Cache（包括L2和L3），其中每个bit代表一条Cache way；
**mbMax**: Memory Bandwidth Maximum Partition，按照能够通过受控DDR通道最大带宽的百分比进行访存流量限制；
**mbMin**: Memory Bandwidth Minimum Partition，提供最小带宽百分比表示允许通过受控DDR通道的容量，小于最小百分比将享受较高优先级的通过权；
**mbHdl**: Memory Bandwidth Hard Limit，开启会使得分区的带宽使用率降至最大带宽控制的范围之内，参考Max，否则，只有在通道拥挤时才会做适当限制。

[1]: https://developer.arm.com/documentation/ddi0598/latest
