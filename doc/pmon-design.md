# Platform monitor refactoring design #

### Rev 0.1 ###

### Revision
 | Rev |     Date    |       Author       | Change Description                |
 |:---:|:-----------:|:------------------:|-----------------------------------|
 | 0.1 |             | Liu Kebo/Kevin Wang | Initial version                   |

## 1. New platform API implememtation ##
Old platform base APIs will be replaced by new designed API gradually. New API is well structed in a hierarchy style, a root "Platform" class include all the chassis in it, and each chassis will containe all the peripheral devices: PSUs, FANs, SFPs, etc.

As for the vendors, the way to implement the new API will be very similiar, the difference is that individual plugins will be replaced by a "sonic_platform" python package.

Besides existing eeprom, SFP, fan, led and PSU APIs, new base APIs were added for platform, chassis and watchdog.

New platfrom API is still under developing in this PR: https://github.com/Azure/sonic-platform-common/pull/13

## 2. Export platform related data to DB ##
Currently when user try to fetch switch peripheral devices related data with CLI, underneath it will directly access hardware via platfrom plugins, in some case it could be very slow, to improvement the performance of these CLI and also for the SNMP maybe, we can collect these data before hand and store them to the DB, and CLI/SNMP will access cached data(DB) instead,  which will be much faster.

Now we already have a Xcvrd daemon which collect SFP related data periodly from SFP eeprom, we may take Xcvrd as reference and add new deamons(like for PSU, fan, etc.). 

During the boot up of the daemons, it will collect the constant data like serial number, manufature name,.... and for the variable ones (tempreture, voltage, fan speed ....) need to be collected periodically. A common data collection flow for these deamons can be like below picture:

![](https://github.com/keboliu/SONiC/blob/gh-pages/images/daemon-flow.svg)

Besides data collection, some daemon also need to responese to the request to set device status, for example, fan daemon may need to adjust the fan speed to expected value.

PSU daemon need to collect PSU numbers, PSU status, and PSU fan speed, PSU fan direction


### 2.1 DB Schema ###
#### 2.1.1 Chassis Table ####

    mac_addr                = STRING                
    reboot_cause            = STRING                
    fan_num                 = INT                   
    fan_list                = STRING_ARRAY   
    psu_num                 = INT
    psu_list                = STRING_ARRAY
    watchdog                = WATCHDOG_CLASS
    
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
    
#### 2.1.5 Psu Table ####
    disarm_flag              = INT
    arm_status               = INT

## 3. Platform monitor related CLI refactor ##
This design is not intend to add new CLI commands or change the original CLI command paramertes and output, it mainly for change the way that CLI get the data.

Take "show platform psustatus" as an example, behind the scene it's calling psu plugin to access the hardware and get the psu status and print out.  In the new design, psu daemon will fetch the psu status and update to DB before hand, thus CLI only need to make use of redis DB APIs and get the informations from the related DB entries.

Transceiver and PSU related CLIs will be refactored. 

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
