# k8s_ceph_benchmarks

This is a benchmark report for running several [fio](https://github.com/axboe/fio) benchmarks against a [CEPH RBD](https://docs.ceph.com/en/pacific/rbd/index.html) image which is deployed within a k8s.
The benchmarks are performed and plotted with the tool [bench-io](https://github.com/louwrentius/fio-plot/tree/master/bench_fio)

Host specs:
- [Intel(R) Xeon(R) W-2295 CPU @ 3.00GHz](https://www.intel.com/content/www/us/en/products/sku/198011/intel-xeon-w2295-processor-24-75m-cache-3-00-ghz/specifications.html)
- 128 GB RAM
- 3x SSDs: [Micron_5300_MTFD](https://www.micron.com/products/ssd/bus-interfaces/sata-ssds/part-catalog/mtfddak3t8tds-1aw1zab)
- 10Gbps NIC adapter
- [Rocky Linux 8.5](https://rockylinux.org/)
- Linux kernel `4.18.0-348.7.1.el8_5.x86_64`
- k8s `1.23.5`

CEPH cluster specs:
- ceph `16.2.7` (pacific)
- 3 hosts
- 9 [CEPH OSDs](https://docs.ceph.com/en/latest/man/8/ceph-osd/) in total (1 per disk)
- default CRUSH map

The following test parameters have been applied to benchmark the metrics `IOPS` and `latency` for a benchmark duration of 60 seconds per combination:
- `block_size`: 4k | 128k | 1M
- `iodepth`: 1 | 2 | 4 | 8 | 16 | 32 | 64
- `numjobs`: 1 | 2 | 4 | 8 | 16 | 32 | 64
- `access_mode`: randread | randwrite

There are three benchmarks which have individual results and some grouped results for a comparison.
- [SSD RAW performance](#ssd-raw-performance)
- [CEPH initial performance](#ceph-initial-performance)
- [CEPH TCP Tuning performance](#ceph-tcp-tuning-performance)

## Comparison between SSD RAW | CEPH initial | CEPH TCP Tuning
This summarizes how the three benchmarks compare against each other.
It can be seen that the CEPH cluster clearly wins against the RAW SSD for `randread` for both `IOPS` and `latency`.

For `randwrite` it's the other way round and the RAW SSD has better results.
This data can be used to inspect the CEPH cluster config to see why there is such a significant drop in IOPS and a relatively high latency.

### IOPS / latency randread
![](group_plots/grouped_4k_randread.png)
![](group_plots/grouped_128k_randread.png)
![](group_plots/grouped_1M_randread.png)

### IOPS / latency randwrite
![](group_plots/grouped_4k_randwrite.png)
![](group_plots/grouped_128k_randwrite.png)
![](group_plots/grouped_1M_randwrite.png)

## SSD RAW performance
This is the benchmark against a local file without any network I/O.

### IOPS randread
![](base_plots/RAW_60_iops_4k_randread.png)
![](base_plots/RAW_60_iops_128k_randread.png)

### IOPS randwrite
![](base_plots/RAW_60_iops_4k_randwrite.png)
![](base_plots/RAW_60_iops_128k_randwrite.png)
![](base_plots/RAW_60_iops_1M_randwrite.png)

### Latency randread
![](base_plots/RAW_60_lat_4k_randread.png)
![](base_plots/RAW_60_lat_128k_randread.png)

### Latency randwrite
![](base_plots/RAW_60_lat_4k_randwrite.png)
![](base_plots/RAW_60_lat_128k_randwrite.png)
![](base_plots/RAW_60_lat_1M_randwrite.png)

## CEPH INITIAL performance
This is the benchmark against an initial CEPH cluster without doing any system changes.
### IOPS randread
![](base_plots/CEPH_INIT_60_iops_4k_randread.png)
![](base_plots/CEPH_INIT_60_iops_128k_randread.png)
![](base_plots/CEPH_INIT_60_iops_1M_randread.png)

### IOPS randwrite
![](base_plots/CEPH_INIT_60_iops_4k_randwrite.png)
![](base_plots/CEPH_INIT_60_iops_128k_randwrite.png)
![](base_plots/CEPH_INIT_60_iops_1M_randwrite.png)

### Latency randread
![](base_plots/CEPH_INIT_60_lat_4k_randread.png)
![](base_plots/CEPH_INIT_60_lat_128k_randread.png)
![](base_plots/CEPH_INIT_60_lat_1M_randread.png)

### Latency randwrite
![](base_plots/CEPH_INIT_60_lat_4k_randwrite.png)
![](base_plots/CEPH_INIT_60_lat_128k_randwrite.png)
![](base_plots/CEPH_INIT_60_lat_1M_randwrite.png)


## CEPH TCP TUNING performance
In this benchmark several TCP kernel settings have been changed to try to improve the latency / IOPS.
However, no significant improvements could be observed. This is quite surprising and needs further analysis.
It might be related to the relative short test duration of 60 seconds and might have more impact when the duration is longer.

```
# 10GE/32MB (33554432)
net.core.rmem_max = 33554432
net.core.wmem_max = 33554432
net.core.rmem_default = 33554432
net.core.wmem_default = 33554432
net.core.optmem_max = 40960
net.ipv4.tcp_rmem = 4096 87380 33554432
net.ipv4.tcp_wmem = 4096 65536 33554432

# Increase number of incoming connections. The value can be raised to bursts of request, default is 128
net.core.somaxconn = 40000

# Increase number of incoming connections backlog, default is 1000
net.core.netdev_max_backlog = 300000

# Maximum number of remembered connection requests, default is 128
net.ipv4.tcp_max_syn_backlog = 30000

# Increase the tcp-time-wait buckets pool size to prevent simple DOS attacks, default is 8192
net.ipv4.tcp_max_tw_buckets = 2000000

# Recycle and Reuse TIME_WAIT sockets faster, default is 0 for both
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1

# Decrease TIME_WAIT seconds, default is 30 seconds
net.ipv4.tcp_fin_timeout = 10

# Tells the system whether it should start at the default window size only for TCP connections
# that have been idle for too long, default is 1
net.ipv4.tcp_slow_start_after_idle = 0

net.ipv4.tcp_syncookies = 0
```

### IOPS randread
![](base_plots/CEPH_TCP_TUNING_60_iops_4k_randread.png)
![](base_plots/CEPH_TCP_TUNING_60_iops_128k_randread.png)
![](base_plots/CEPH_TCP_TUNING_60_iops_1M_randread.png)


### IOPS randwrite
![](base_plots/CEPH_TCP_TUNING_60_iops_4k_randwrite.png)
![](base_plots/CEPH_TCP_TUNING_60_iops_128k_randwrite.png)
![](base_plots/CEPH_TCP_TUNING_60_iops_1M_randwrite.png)

### Latency randread
![](base_plots/CEPH_TCP_TUNING_60_lat_4k_randread.png)
![](base_plots/CEPH_TCP_TUNING_60_lat_128k_randread.png)
![](base_plots/CEPH_TCP_TUNING_60_lat_1M_randread.png)

### Latency randwrite
![](base_plots/CEPH_TCP_TUNING_60_lat_4k_randwrite.png)
![](base_plots/CEPH_TCP_TUNING_60_lat_128k_randwrite.png)
![](base_plots/CEPH_TCP_TUNING_60_lat_1M_randwrite.png)
