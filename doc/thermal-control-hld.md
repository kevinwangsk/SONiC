# Thermal control HLD on Mellanox platform #

### Rev 0.1 ###

### Revision
 | Rev |     Date    |       Author       | Change Description                |
 |:---:|:-----------:|:------------------:|-----------------------------------|
 | 0.1 |             |    Kevin Wang      | Initial version                   |

## About This Manual ##

This document is intend to provide high level design about thermal control on Mellanox platform.

## 1. Thermal control design ##
Linux kernel provides a framework for thermal control. It defines a set of concepts, such as thermal zones, trip points, cooling devices, thermal instances, thermal governors, to work together for the thermal control. The kernel thermal algorithm uses step_wise policy which set FANs according to the thermal trends (high temperature = faster fan; lower temperature = slower fan). Refer to kernel documentation file: Documentation/thermal/sysfs-api.txt.
But currently on Mellanox platform, hw-mgmt will disable the kernel's thermal and set PWM to a consistent value(60% of full speed). With the low fan speed, the temperature may raise higher once some unexpected things happened.
With the latest hw-mgmt package, it will not disable the kernel's thermal. That means the step_wise policy will be applied to the board, and the fan speed will be changed by kernel according to the temperature. Besides kernel's functions, hw-mgmt will cover two more cases:
    ###Setting PWM to full speed if one of PS units is not present (in such case thermal monitoring in kernel is set to disabled state until the problem is not recovered). Such events will be reported to systemd journaling system.
    ###Setting PWM to full speed if one of FAN drawers is not present or one of tachometers is broken present (in such case thermal monitoring in kernel is set to disabled state until the problem is not recovered). Such events will be reported to systemd journaling system.

### 1.1  ###



#### 1.1.1  ####

	

### 1.2  ###

### 1.3  ###


## 2.  ##



## 3. Open Questions ##


      
