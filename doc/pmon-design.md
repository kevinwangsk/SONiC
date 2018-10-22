# Platform monitor refactoring design #
## 1. migrate platform API from plugin to new platform API ##
### 1.1 PSU utility migration ###
### 1.2 SFP utility migration ###
### 1.3 EEPROM utility migration ###
### 1.4 LED utility migration ###
### 1.5 New Chassis platfrom API implementaion ###
### 1.6 New Fan platfrom API implementation
## 2. Export platform related data to DB ##
### 2.1 DB Schema ###
## 3. Platform monitor related CLI refactor ##
## 4. Pmon daemons dynamically load per platfrom capability ##

we have multi daemons for different hardwares, like xcvrd for transceivers, ledd for front panel LEDs. Later on we may add more for PSU, 
fan, etc. When it comes to a specific platfrom, it maybe not able to support all of these daemons due to hardware limitation or not implement
some platfrom API yet due to effots. Thus if we load all these pmon daemons in this platfrom without any determination, may encounter into
some errors. To avoid these error, can load the pmon daemons dynamically based on the capability of the platfrom.

An approach is to manipulate the config file of pmon supervisord to disable the starting of some daemon on some platform if the related 
platfrom API not implemented yet on it.  
