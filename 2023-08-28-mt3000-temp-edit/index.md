# GL-MT3000自定义固件风扇阈值调节


## 概述

GL-MT3000刷自编译lede固件，调节风扇起转温度



## 步骤

```bash
cd /sys/devices/virtual/thermal/thermal_zone0/
ll 

-r--r--r--    1 root     root          4096 Aug 28 14:42 available_policies
lrwxrwxrwx    1 root     root             0 Aug 28 14:42 cdev0 -> ../cooling_device0/
-r--r--r--    1 root     root          4096 Aug 28 14:42 cdev0_trip_point
-rw-r--r--    1 root     root          4096 Aug 28 14:42 cdev0_weight
lrwxrwxrwx    1 root     root             0 Aug 28 14:42 cdev1 -> ../cooling_device0/
-r--r--r--    1 root     root          4096 Aug 28 14:42 cdev1_trip_point
-rw-r--r--    1 root     root          4096 Aug 28 14:42 cdev1_weight
lrwxrwxrwx    1 root     root             0 Aug 28 14:42 cdev2 -> ../cooling_device0/
-r--r--r--    1 root     root          4096 Aug 28 14:42 cdev2_trip_point
-rw-r--r--    1 root     root          4096 Aug 28 14:42 cdev2_weight
drwxr-xr-x    3 root     root             0 Jan  1  1970 hwmon0/
-rw-r--r--    1 root     root          4096 Aug 28 14:42 integral_cutoff
-rw-r--r--    1 root     root          4096 Aug 28 14:42 k_d
-rw-r--r--    1 root     root          4096 Aug 28 14:42 k_i
-rw-r--r--    1 root     root          4096 Aug 28 14:42 k_po
-rw-r--r--    1 root     root          4096 Aug 28 14:42 k_pu
-rw-r--r--    1 root     root          4096 Aug 28 14:42 mode
-rw-r--r--    1 root     root          4096 Aug 28 14:42 offset
-rw-r--r--    1 root     root          4096 Aug 28 14:42 policy
drwxr-xr-x    2 root     root             0 Aug 28 14:42 power/
-rw-r--r--    1 root     root          4096 Aug 28 14:42 slope
lrwxrwxrwx    1 root     root             0 Aug 28 14:42 subsystem -> ../../../../class/thermal/
-rw-r--r--    1 root     root          4096 Aug 28 14:42 sustainable_power
-r--r--r--    1 root     root          4096 Aug 28 14:21 temp
-rw-r--r--    1 root     root          4096 Aug 28 14:42 trip_point_0_hyst
-rw-r--r--    1 root     root          4096 Aug 28 14:42 trip_point_0_temp
-r--r--r--    1 root     root          4096 Aug 28 14:42 trip_point_0_type
-rw-r--r--    1 root     root          4096 Aug 28 14:42 trip_point_1_hyst
-rw-r--r--    1 root     root          4096 Aug 28 14:42 trip_point_1_temp
-r--r--r--    1 root     root          4096 Aug 28 14:42 trip_point_1_type
-rw-r--r--    1 root     root          4096 Aug 28 14:42 trip_point_2_hyst
-rw-r--r--    1 root     root          4096 Aug 28 14:42 trip_point_2_temp
-r--r--r--    1 root     root          4096 Aug 28 14:42 trip_point_2_type
-rw-r--r--    1 root     root          4096 Aug 28 14:42 trip_point_3_hyst
-rw-r--r--    1 root     root          4096 Aug 28 14:42 trip_point_3_temp
-r--r--r--    1 root     root          4096 Aug 28 14:42 trip_point_3_type
-rw-r--r--    1 root     root          4096 Aug 28 14:42 trip_point_4_hyst
-rw-r--r--    1 root     root          4096 Aug 28 14:05 trip_point_4_temp
-r--r--r--    1 root     root          4096 Aug 28 14:42 trip_point_4_type
-r--r--r--    1 root     root          4096 Aug 28 14:42 type
-rw-r--r--    1 root     root          4096 Aug 28 14:42 uevent
```

主要关注以下文件：

```
-rw-r--r--    1 root     root          4096 Aug 28 14:42 trip_point_0_hyst
-rw-r--r--    1 root     root          4096 Aug 28 14:42 trip_point_0_temp
-r--r--r--    1 root     root          4096 Aug 28 14:42 trip_point_0_type
-rw-r--r--    1 root     root          4096 Aug 28 14:42 trip_point_1_hyst
-rw-r--r--    1 root     root          4096 Aug 28 14:42 trip_point_1_temp
-r--r--r--    1 root     root          4096 Aug 28 14:42 trip_point_1_type
-rw-r--r--    1 root     root          4096 Aug 28 14:42 trip_point_2_hyst
-rw-r--r--    1 root     root          4096 Aug 28 14:42 trip_point_2_temp
-r--r--r--    1 root     root          4096 Aug 28 14:42 trip_point_2_type
-rw-r--r--    1 root     root          4096 Aug 28 14:42 trip_point_3_hyst
-rw-r--r--    1 root     root          4096 Aug 28 14:42 trip_point_3_temp
-r--r--r--    1 root     root          4096 Aug 28 14:42 trip_point_3_type
-rw-r--r--    1 root     root          4096 Aug 28 14:42 trip_point_4_hyst
-rw-r--r--    1 root     root          4096 Aug 28 14:05 trip_point_4_temp
-r--r--r--    1 root     root          4096 Aug 28 14:42 trip_point_4_type
```

`trip_point_×_temp` 是几个风扇阈值，比如：

```
root@OpenWrt:/sys/devices/virtual/thermal/thermal_zone0# cat trip_point_4_temp
60000
```

说明风扇从 60 度开始转。

将温度设置为65度。

```
echo 65000 > /sys/devices/virtual/thermal/thermal_zone0/trip_point_4_temp
```


