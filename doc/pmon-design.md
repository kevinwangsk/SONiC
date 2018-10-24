# Platform monitor refactoring design #
## 1. migrate platform API from plugin to new platform API ##
### 1.1 PSU utility migration ###
### 1.2 SFP utility migration ###
### 1.3 EEPROM utility migration ###
### 1.4 LED utility migration ###
### 1.5 New Chassis platfrom API implementaion ###
### 1.6 New Fan platfrom API implementation
## 2. Export platform related data to DB ##
Currently switch peripheral devices related CLI fetch data directly from hardware via platfrom plugins, in some case it could be very slow to get the data directly from the hardware, to improvement the performance of these CLI and also for the SNMP maybe, we can collect these data and store them to the DB, and CLI/SNMP will access DB instead,  which will be much faster.

Now we already have a Xcvrd daemon which collect SFP related data periodly from SFP eeprom, we may take Xcvrd as reference and add other deamons(like PSU, fan, etc.). 

During the start of the daemon, it will collect the constant datas like serial number, manufature name,.... and for the variable ones (tempreture, voltage, fan speed ....) need to collect in a certain period. A common flow for these deamons can be like below picture:

### 2.1 DB Schema ###
## 3. Platform monitor related CLI refactor ##
## 4. Pmon daemons dynamically load per platfrom capability ##

We have multi pmon daemons for different peripheral devices, like xcvrd for transceivers, ledd for front panel LEDs, etc. Later on we may add more for PSU, fan. 

But not all the platfrom can support all of these daemons due to hardware limitation, in some case some platform may need some special daemons which are not a common. 

Thus if we only load the common pmon daemons in all the platfrom without any determination, may encounter into some errors. To avoid this, pmon need a capability to load the daemons dynamically based on specific platfrom.

The starting of the daemons inside pmon is controlled by supervisord, to dynamically control the starting of daemons, an approach is to manipulate supervisord.

For now pmon superviosrd have a default configuration file which applied to all the platforms by default.

To let vedors have different behavior on their platform, can add a customized suervisord configuration file in the platform folder,  to change the behavior of supervisord to have different running daemon set than normal case. 

## 5. Move hw-management from host to pmon
