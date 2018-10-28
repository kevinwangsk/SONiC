# Platform monitor refactoring design #

### Rev 0.1 ###

### Revision
 | Rev |     Date    |       Author       | Change Description                |
 |:---:|:-----------:|:------------------:|-----------------------------------|
 | 0.1 |             | Liu Kebo/Kevin Wang | Initial version                  |

## 1. New platform API implememtation ##
Old platform base APIs will be replaced by new designed API gradually. New API is well structed in a hierarchy style, a root "Platform" class include all the chassis in it, and each chassis will containe all the peripheral devices: PSUs, FANs, SFPs, etc.

As for the vendors, the way to implement the new API will be very similiar, the difference is that individual plugins will be replaced by a "sonic_platform" python package.

Besides existing eeprom, SFP, fan, led and PSU APIs, new base APIs were added for platform, chassis and watchdog. All the APIs defined in the base classes need to be implemented unless there is a limitation(like hardware not support it.) 

Previously we have an issue with the old implementation, when adding a new platform API to the base class, have to implement it in all the platform plugins, or at least add a dummy stub to them, or it will fail on the platform that doesn't have it. This will be addressed in the new platfrom API design, not part of the work here. 

New platfrom API is defined in this PR: https://github.com/Azure/sonic-platform-common/pull/13

## 2. Export platform related data to DB ##
Currently when user try to fetch switch peripheral devices related data with CLI, underneath it will directly access hardware via platfrom plugins, in some case it could be very slow, to improvement the performance of these CLI and also for the SNMP maybe, we can collect these data before hand and store them to the DB, and CLI/SNMP will access cached data(DB) instead,  which will be much faster.

A common data collection flow for these deamons can be like this: during the boot up of the daemons, it will collect the constant data like serial number, manufature name,.... and for the variable ones (tempreture, voltage, fan speed ....) need to be collected periodically. See below picture.

![](https://github.com/keboliu/SONiC/blob/gh-pages/images/daemon-flow.svg)

Now we already have a Xcvrd daemon which collect SFP related data periodly from SFP eeprom, we may take Xcvrd as reference and add new deamons(like for PSU, fan, etc.). 

PSU daemon need to collect PSU module name, PSU status, and PSU fan speed, PSU fan direction, etc. PSU deamon will also update the current avalaible PSU numbers and PSU list when there is a PSU change. Fan deamon perform similiar activities as PSU daemon.

Transceiver related new API not defined yet in the first phase. Xcvrd may be need some change with the new platform API in the future.

Detail datas that need to be collected please see the below DB Schema section.  

Besides data collection, some daemon also need to responese to the request to set device status, for example, fan daemon may need to adjust the fan speed to expected value. (For this part see open questions.)


### 2.1 DB Schema ###
#### 2.1.1 Chassis Table ####

    mac_addr                = STRING                
    reboot_cause            = STRING                
    fan_num                 = INT                   
    fan_list                = STRING_ARRAY   
    psu_num                 = INT
    psu_list                = STRING_ARRAY
    
#### 2.1.2 Device Table ####

    presence                = BOOLEAN                
    model_num               = STRING                
    serial_num              = STRING                   
    status                  = BOOLEAN
    change_event            = STRING        

#### 2.1.3 Fan Table ####

    direction                = STRING                
    speed                    = INT                
    speed_tolerance          = INT                   
    speed_target             = INT
    led_status               = STRING 
    

#### 2.1.3 Platform Table ####

    chassis_num              = INT                
    chassis_list             = STRING_ARRAY               

#### 2.1.4 Psu Table ####
    fan_direction            = STRING                
    fan_speed                = INT                
    fan_speed_tolerance      = INT                   
    fan_speed_target         = INT
    fan_led_status           = STRING 
    
#### 2.1.5 Watchdog Table ####
    disarm_flag              = INT
    arm_status               = INT

## 3. Platform monitor related CLI refactor ##
### 3.1 change the way that CLI get the data ##
As described previously, we want to change the way that CLI get the data. Take "show platform psustatus" as an example, behind the scene it's calling psu plugin to access the hardware and get the psu status and print out.  In the new design, psu daemon will fetch the psu status and update to DB before hand, thus CLI only need to make use of redis DB APIs and get the informations from the related DB entries.

### 3.2 more output for psu show CLI ###
original PSU show CLI only provide PSU work status, we should add PSU fan status as well.

Original output:
```
admin@sonic# show platform psustatus
PSU    Status   
-----  -------- 
PSU 1  OK       
PSU 2  NOT OK 
```

New output:
```
admin@sonic# show platform psustatus
PSU    Status   FAN SPEED  FAN DIRECTION
-----  -------- ---------- --------------
PSU 1  OK       13417 RPM  Intake 
PSU 2  NOT OK   12320 RPM  Exhaust
```

### 3.3 new show CLI for fan status ###
We don't have a CLI for fan status geting yet, new CLI for fan status could be like below, it's adding a new sub command to the "show platform":

```
admin@sonic# show platform ?
Usage: show platform [OPTIONS] COMMAND [ARGS]...

  Show platform-specific hardware info

Options:
  -?, -h, --help  Show this message and exit.

Commands:
  fanstatus  Show fan status information
  mlnx       Mellanox platform specific configuration...
  psustatus  Show PSU status information
  summary    Show hardware platform information
  syseeprom  Show system EEPROM information
```

The output of the command is like below:
```
admin@sonic# show platform fanstatus
FAN    SPEED      Direction
-----  ---------  ---------
FAN 1  12919 RPM  Intake
FAN 2  13043 RPM  Exhaust
```

### NOTE ###
Tranceiver related platform base API not defined in this phase, transceiver related CLI will be updated after that.
for the sfputility, psuutility, user may want to keep a way to get real-time data from hardware rather than from DB for debug purpose, so we may keep sfputility, psuutility and only install them to pmon. 

## 4. Pmon daemons dynamically loading ##

We have multi pmon daemons for different peripheral devices, like xcvrd for transceivers, ledd for front panel LEDs, etc. Later on we may add more for PSU, fan. 

But not all the platfrom can support all of these daemons due to hardware limitation, in some case some platform may need some special daemons which are not a common. 

Thus if we only load the common pmon daemons in all the platfrom without any determination, may encounter into some errors. To avoid this, pmon need a capability to load the daemons dynamically based on specific platfrom.

The starting of the daemons inside pmon is controlled by supervisord, to dynamically control the starting of daemons, an approach is to manipulate supervisord.

For now pmon superviosrd have a default configuration file which applied to all the platforms by default.

To let vedors have different behavior on their platform, can add a customized suervisord configuration file in the platform folder,  to change the behavior of supervisord to have different running daemon set than normal case. 

## 5. Move hw-management from host to pmon

To make the pmon as the only point that can access devices, we want to move the hw-mgmt from host to pmon container.
Lots of items need to be find out before we know the feasibility, see the open questions.

## 6. Open Questions
1. hw-mgmt package prerequisites? if move it to pmon, need to install all the required packages to pmon.
2. need to sort out the start chain, if we move hw-mgmt to pmon, means each time we reload config hw-mgmt will restart with pmon,
some other tasks in other container may have dependncy on it, need more controll on the starting sequence? 
3. other vendors? the way to manage devices are various on different platforms.
4. For the daemon to response the setting request, need support SNMP set and then daemon maybe need to listen to the DB change which come from SNMP. 
