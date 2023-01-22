## Tools for MT3339/PA6H Based GPS

It may seem slightly counter-intuitive, but in order for a GPS unit to produce an accurate fix,
it has to know the time, the location of the GPS satellites, and it's own rough location.
Odd, given that it's job is to tell YOU those things.  Since the satellites broadcast the time and
their positions, the GPS unit can get the info but **it must have a strong signal (>25dBHz) from 4 satellites 
for about a minute.**  If the signal level drops a little or a satellite goes out of view in that minute,
the GPS unit has to start over again.  Outdoors with a clear view of the sky, this isn't usually
a problem.  Indoors or when the outdoor view is obstructed by trees, terrain, buildings, etc.
it can be impossible to get an initial fix because you just can't get a strong signal from the same
4 satellites for enough time.

"But wait, my cell phone gets a fast fix indoors!".  Yep, and there's a reason for that.
Being network connected, the phone already knows the exact time and it's rough location as
the cell network provides both.  As for the location of the satellites at any particular time
(the ephemeris data), that's public information and the GPS manufacturers thoughtfully
provide files, usually updated nightly and good for 7, 14 or 30  days, that the phone can download.
The whole process is called Assisted-GPS and doesn't require as strong
a signal (~15dBHz) to get and maintain a fix.

If you going to use your MT3339 based unit only outdoors, no need to read further.  Otherwise...

The Adafruit Ultimate GPS is based on the GlobalTop PA6H GPS unit which is in turn based on the
MediaTek MT3339 chip.  GlobalTop provides the ephemeris data in the form of an
EPO (Extended Prediction Orbit) file available from their FTP site (see eporetrieve, below).
There are other sources for EPO data, including directly from MediaTek, but there are 2 formats,
EPO ( 1920 byte sets) and EPO-II ( 2304 byte sets). Currently there are only EPO-II files
available for download so that's the only format this program now supports.  Unfortunately though,
while the 3339 can use EPO-II files, it can't store as many sets since the sets are larger.
Only 23 sets can be uploaded reliably which is only about 5.75 days so make sure you use the
`-m` option to limit the sets to 23 and adjust your cron job accordingly.

So now what?  Use the tools...

### Get the EPO file:

#### eporetrieve:  Retrieves the current EPO file from the GlobalTop FTP Server
```
eporetrieve [ <file_name> [ <output_file> ] ]

Example:
# eporetrieve EPO.DAT /tmp/EPO.DAT
 
```
This script is just a wrapper around wget.  It's generally best to do this in the morning UTC
to make sure the file is valid for the current time.  If you pull it late in the UTC day, you may
get a file that starts the next day and is not yet valid.

### Inspect the EPO file:

#### epoinfo: Displays information about an EPO file
```
usage: epoinfo [-h] <EPO_File>

Prints EPO file information

positional arguments:
  <EPO_File>  EPO data file

optional arguments:
  -h, --help  show this help message and exit

You can use '@filename' to read arguments from a file.
```
To examine the file /tmp/EPO.DAT...
```
# epoinfo /tmp/EPO.DAT
Opening EPO Type II file
Set:    1.  GPS Wk: 2194  Hr:  24  Sec:   86400  2022-01-23 23:59:46 to 2022-01-24 05:59:46 UTC
Set:    2.  GPS Wk: 2194  Hr:  30  Sec:  108000  2022-01-24 05:59:46 to 2022-01-24 11:59:46 UTC
Set:    3.  GPS Wk: 2194  Hr:  36  Sec:  129600  2022-01-24 11:59:46 to 2022-01-24 17:59:46 UTC
Set:    4.  GPS Wk: 2194  Hr:  42  Sec:  151200  2022-01-24 17:59:46 to 2022-01-24 23:59:46 UTC
Set:    5.  GPS Wk: 2194  Hr:  48  Sec:  172800  2022-01-24 23:59:46 to 2022-01-25 05:59:46 UTC
<snip>
Set:  115.  GPS Wk: 2198  Hr:  36  Sec:  129600  2022-02-21 11:59:46 to 2022-02-21 17:59:46 UTC
Set:  116.  GPS Wk: 2198  Hr:  42  Sec:  151200  2022-02-21 17:59:46 to 2022-02-21 23:59:46 UTC
Set:  117.  GPS Wk: 2198  Hr:  48  Sec:  172800  2022-02-21 23:59:46 to 2022-02-22 05:59:46 UTC
Set:  118.  GPS Wk: 2198  Hr:  54  Sec:  194400  2022-02-22 05:59:46 to 2022-02-22 11:59:46 UTC
Set:  119.  GPS Wk: 2198  Hr:  60  Sec:  216000  2022-02-22 11:59:46 to 2022-02-22 17:59:46 UTC
Set:  120.  GPS Wk: 2198  Hr:  66  Sec:  237600  2022-02-22 17:59:46 to 2022-02-22 23:59:46 UTC
 120 EPO sets.  Valid from 2022-01-23 23:59:46 to 2022-02-22 23:59:46 UTC

```
The current EPO.DAT provides by MediaTek is a type II file covering 30 days.  Since each
set covers 6 hours, the file will contain 120 sets.  However, as noted above, the MT3999 can
only keep 23 sets of Type II data, or about 5.75 days worth, in it's NVRAM.  See below.

### Load the EPO file to the GPS:

#### epoloader:  Loads an EPO file to the GPS
```
usage: epoloader [-h] [-t yyyy,mm,dd,hh,mm,ss | -] [-l lat.dddddd,lon.dddddd,alt]
                 [-s {4800,9600,19200,38400,57600,115200}] [-k] [-c] [-n] [-m MAX_SETS]
                 <EPO_File> <gps_device>

Loads EPO data sets to MT3339 GPS

positional arguments:
  <EPO_File>            EPO File or '-' to just set the known parameters
  <gps_device>          GPS serial device such as '/dev/ttyUSB0'

options:
  -h, --help            show this help message and exit
  -s {4800,9600,19200,38400,57600,115200}, --speed {4800,9600,19200,38400,57600,115200}
                        Interface speed
  -k, --keep-new-speed  Don't restore the old speed on exit
  -c, --clear           Clears the existing EPO data from the unit
  -n, --no-init         Skips the initialization
  -m MAX_SETS, --max-sets MAX_SETS
                        Send only n sets

optional known time and location parameters:
  -t yyyy,mm,dd,hh,mm,ss | - , --time yyyy,mm,dd,hh,mm,ss | - 
                        Current UTC or UTC from host
  -l lat.dddddd,lon.dddddd,alt, --location lat.dddddd,lon.dddddd,alt
                        Current location specified in decimal degrees and meters

You can use '@filename' to read arguments from a file.
```

Before you start this step, make sure you're unit is connected and that the port speed
matches the GPS speed.  You can use the gpsinit utility (see below) to do this.
If you can 'cat' the port where the GPS is connected and get valid NMEA data,
you should be OK.
```
# ## Set the port to raw mode first
# stty -F /dev/tty<something> raw
# cat /dev/tty<something>
$GPGGA,000221.000,3899.0037,N,10499.9070,W,1,10,1.01,1795.4,M,-20.7,M,,*6C
$GPGLL,3899.0037,N,10499.9070,W,000221.000,A,A*4E
$GPRMC,000221.000,A,3899.0037,N,10499.9070,W,0.16,224.89,180215,,,A*74
$GPVTG,224.89,T,,M,0.16,N,0.30,K,A*3C
$GPZDA,000221.000,18,02,2015,,*5A
```

If your unit is at factory state or it's time and last known location are off significantly, 
you'll need to supply your location and the time in addition to the EPO file.
If you're unit already has a fix and you just need to update EPO data, you don't need to
supply the location and time.  

You location and altitude must be specified in decimal degrees, using negative numbers for West and South,
and meters. Google Earth is a good source.  The more accurate the better but the location needs to be within 30km.

For the time, just let epoloader get it from your host system using the '-t-' option, assuming it's
accurate.  If not, specify it as shown above in UTC.

Since you can specify options to epoloader in a file, you may want to create a file
that has the location and time options in it.  For instance, if you're on the lawn
outside the Washington Monument...
```
echo "-l 38.889468,-77.352420,5 -t-" > epoloader.conf
```
Finally a last word on port speed.  By now you've already made sure that the port
speed and the unit speed match.  If the unit is at factory state, it's probably
at 9600 baud which is just a touch slow.  By supplying the '-s' option to epoloader,
you can specify the speed to use for the load and the program will take care of 
setting both the unit and the port to the correct speed, then resetting both to their
original speed on completion.  Use 115200 (the default) unless you have a flaky setup.  You can also
specify the '-k' option to leave the unit and port at the new speed.

It's pretty much impossible to mess up the GPS unit with this process as it doesn't touch
the unit's firmware.  If you get into a bad communications state, just try again, use gpsinit, 
try holding the EN signal low for a second, or remove the
power and battery for a few seconds to reset back to factory state.

Almost there... Since the MT3339 can only hold 23 EPO sets, the default maximum number
of sets to send is 26.  You can change this with the `-m / --max-sets` option but
attempting to send more than 23 will cause the validation to fail 

NOW you're ready.  Assuming your location and time is in epoloader.conf, the EPO file is
/tmp/EPO.DAT, your device is connected to /dev/ttyUSB1 and you want to use the default 115200 speed, then...
```
$ ./epoloader -s 115200 -m 23 -c -l 39.754598,-105.236353,1780 -t- /tmp/EPO.DAT /dev/ttyUSB1
Opening EPO Type II file with 120 sets
Setting NMEA at 115200 baud.
   Unit is currently in NMEA mode at 115200 baud.
GPS and port are now synchronized at 115200
GPS Version:  ['PMTK705', 'AXN_2.31_3339_13101700', '5632', 'PA6H', '1.0']
Clearing existing EPO data
Setting binary mode, speed: 115200
Sending 23 EPO sets of 120
Sending set    1.  Valid from 2023-01-20 23:59:46 UTC to 2023-01-21 05:59:46 UTC
Sending set    2.  Valid from 2023-01-21 05:59:46 UTC to 2023-01-21 11:59:46 UTC
Sending set    3.  Valid from 2023-01-21 11:59:46 UTC to 2023-01-21 17:59:46 UTC
Sending set    4.  Valid from 2023-01-21 17:59:46 UTC to 2023-01-21 23:59:46 UTC
Sending set    5.  Valid from 2023-01-21 23:59:46 UTC to 2023-01-22 05:59:46 UTC
Sending set    6.  Valid from 2023-01-22 05:59:46 UTC to 2023-01-22 11:59:46 UTC
Sending set    7.  Valid from 2023-01-22 11:59:46 UTC to 2023-01-22 17:59:46 UTC
Sending set    8.  Valid from 2023-01-22 17:59:46 UTC to 2023-01-22 23:59:46 UTC
Sending set    9.  Valid from 2023-01-22 23:59:46 UTC to 2023-01-23 05:59:46 UTC
Sending set   10.  Valid from 2023-01-23 05:59:46 UTC to 2023-01-23 11:59:46 UTC
Sending set   11.  Valid from 2023-01-23 11:59:46 UTC to 2023-01-23 17:59:46 UTC
Sending set   12.  Valid from 2023-01-23 17:59:46 UTC to 2023-01-23 23:59:46 UTC
Sending set   13.  Valid from 2023-01-23 23:59:46 UTC to 2023-01-24 05:59:46 UTC
Sending set   14.  Valid from 2023-01-24 05:59:46 UTC to 2023-01-24 11:59:46 UTC
Sending set   15.  Valid from 2023-01-24 11:59:46 UTC to 2023-01-24 17:59:46 UTC
Sending set   16.  Valid from 2023-01-24 17:59:46 UTC to 2023-01-24 23:59:46 UTC
Sending set   17.  Valid from 2023-01-24 23:59:46 UTC to 2023-01-25 05:59:46 UTC
Sending set   18.  Valid from 2023-01-25 05:59:46 UTC to 2023-01-25 11:59:46 UTC
Sending set   19.  Valid from 2023-01-25 11:59:46 UTC to 2023-01-25 17:59:46 UTC
Sending set   20.  Valid from 2023-01-25 17:59:46 UTC to 2023-01-25 23:59:46 UTC
Sending set   21.  Valid from 2023-01-25 23:59:46 UTC to 2023-01-26 05:59:46 UTC
Sending set   22.  Valid from 2023-01-26 05:59:46 UTC to 2023-01-26 11:59:46 UTC
Sending set   23.  Valid from 2023-01-26 11:59:46 UTC to 2023-01-26 17:59:46 UTC
================================================================================
 23 sets sent.     Valid from 2023-01-20 23:59:46 UTC to 2023-01-26 17:59:46 UTC
    sets in NVRAM: Valid from 2023-01-20 23:59:46 UTC to 2023-01-26 17:59:46 UTC
Verified EPO in NVRAM matches file

```
At 115200, the process should take about 17 seconds.  If you didn't specify the '-k' option,
the unit and the port will be returned to their previous speed. 

### Does it make a difference??

Oh yes...  Look at the times for Factory Reset.  The difference is amazing.

	StartFrom:
		Factory:	Factory reset
		Reset:		After first fix, hot start or hold enable pin low for 1 sec
		PowerCycle: After first fix, remove VCC for 5 seconds (with backup battery connected)
	Antenna:  		Internal or External Active
	AntLocation:
		ClearSky:	Outside, clear sky
		Inside:		First floor room with window
		DeepInside:	First floor interior room, no window
	EPO: 			Y - loaded, N - not loaded
	TTFF:			Time to First Fix in seconds (3 tests)
	 

| StartFrom  | Antenna  | AntLocation | EPO |                          | TTFF1 | TTFF2 | TTFF3 |
|------------|----------|-------------|-----|--------------------------|------:|------:|------:|
| Factory    | External | Clear Sky   | N   | :heavy_exclamation_mark: | 68    | 72    | 150   |
| Factory    | External | Clear Sky   | Y   | :white_check_mark:       | 2     | 2     | 2     |
| Factory    | External | Deep Inside | N   | :bangbang:               | 420   | 335   | 504   |
| Factory    | External | Deep Inside | Y   | :white_check_mark:       | 14    | 20    | 17    |
| Factory    | External | Inside      | N   | :bangbang:               | 590   | 367   | 283   |
| Factory    | External | Inside      | Y   | :white_check_mark:       | 10    | 6     | 14    |
| Factory    | Internal | Inside      | N   | :x:                      | 2422  | 3600* | 7200* |
| Factory    | Internal | Inside      | Y   | :white_check_mark:       | 14    | 33    | 15    |
| PowerCycle | External | Clear Sky   | N   |                          | 13    | 15    | 9     |
| PowerCycle | External | Clear Sky   | Y   |                          | 1     | 8     | 2     |
| PowerCycle | External | Deep Inside | N   |                          | 2     | 6     | 2     |
| PowerCycle | External | Deep Inside | Y   |                          | 5     | 19    | 10    |
| PowerCycle | External | Inside      | N   |                          | 8     | 6     | 15    |
| PowerCycle | External | Inside      | Y   |                          | 13    | 10    | 7     |
| PowerCycle | Internal | Inside      | N   |                          | 10    | 9     | 9     |
| PowerCycle | Internal | Inside      | Y   |                          | 20    | 25    | 16    |
| Reset      | External | Clear Sky   | N   |                          | 8     | 10    | 12    |
| Reset      | External | Clear Sky   | Y   |                          | 7     | 7     | 7     |
| Reset      | External | Deep Inside | N   |                          | 3     | 6     | 6     |
| Reset      | External | Deep Inside | Y   |                          | 7     | 8     | 5     |
| Reset      | External | Inside      | N   |                          | 6     | 4     | 5     |
| Reset      | External | Inside      | Y   |                          | 5     | 5     | 5     |
| Reset      | Internal | Inside      | N   |                          | 6     | 4     | 11    |
| Reset      | Internal | Inside      | Y   |                          | 7     | 4     | 4     |
|            |          |             |     |                          |       | *     | Gave Up|


### gpsinit:  Initializes GPS configuration
```
usage: gpsinit [-h] [-s {4800,9600,19200,38400,57600,115200}]
               [-i <init_command> | -f <command_file>]
               <gps_device>

Initialized GPS to known state

positional arguments:
  <gps_device>          GPS serial device such as '/dev/ttyUSB0'

optional arguments:
  -h, --help            show this help message and exit
  -s {4800,9600,19200,38400,57600,115200}, --speed {4800,9600,19200,38400,57600,115200}
                        Interface speed
  -i <init_command>, --init_command <init_command>
                        A single initialization command such as 'PMTK101'
  -f <command_file>, --command_file <command_file>
                        A file containing commands used to initialize the GPS

You can use '@filename' to read arguments from a file.
```
gpsinit can help get the GPS unit to a known state.  It will attempt to get the port
and GPS unit to the same speed, then send commands of your choosing to set update rate,
enable/disable NMEA sentences, etc.  You can specify the desired speed and a single command on the
command line with the '-i' and '-s' options or the speed and a list of commands in a file specified
by the '-f' option.  

This is an example gpsinit_reset.conf that clears the EPO data, does a factory reset, sets some defaults,
sets NMEA sentences and frequency and finally does an epo load.
```
# You can use the following variables in this file...
#  ${SPEED}
#  ${DEVICE}
#
#  The following commands are recognized...
#
#  sleep <seconds>                 Sleep for n.nnn seconds
#  set_system_clock                Gets the date/time from the GPS unit and sets the system clock
#  setspeed <newspeed>             Set new port and unit speed to <newspeed>
#  clear_epo                       Clear the existing EPO from the device
#  eporetrieve <eporetrieve_args>  Run eporetrieve
#  epoloader <epoloader_args>      Run epoloader
#  factory_reset                   Clear system/user configurations then cold start
#                                  The unit and port speed will be reset to 9600

#  The following commands can be prefixed with '-' to not wait for an ACK.
#  hot_start          Restart using all available data
#  warm_start         Don't use Ephemeris at restart
#  cold_start         Don't use Time, Position, Almanacs and Ephemeris
#	                    data at re‐start
#  PMTKnnn,<args>     Send the specified PMTK command
#

# Clear the EPO NVRAM
clear_epo
# Factory reset
factory_reset
# Now set to 115200
setspeed 115200
# Set DGPS mode to SBAS
PMTK301,2
# Set SBAS Enabled
PMTK313,1
# Set NMEA Sentence Output
PMTK314,1,1,1,1,1,1,0,0,0,0,0,0,0,0,0,0,0,1,0
# Set SBAS 'Integrity' Mode
PMTK319,1
# Disable Periodic Mode
PMTK225,0
# Enable AIC Mode
PMTK286,1
# Enable EASY Mode
PMTK869,1,1
# Set output rate to 1 second
PMTK220,1000
# Load the EPO. 
epoloader -c -s ${SPEED} @gpsinit_loc.conf /tmp/MTK14.EPO ${DEVICE}

# You shouldn't put anything after the epolaoder.
#  It can interfere with the unit getting it's initlal fix.
```
```
# gpsinit -f gpsinit_reset.conf /dev/ttyUSB0
```

### gpssend:  Sends a single command to the GPS
```
gpssend <command_string> <gps_device>

Example:
# ## Send a Hot Start command to the GPS unit
# gpssend PMTK101 /dev/ttyUSB0
# ##  Writes '$PMTK101*32\r\n'

```
Sometimes you just want to send a single command to the GPS unit but having to calculate the 
required checksum is a pain.  gpssend is a simple shell script that does the calculation for
you and writes the result to the GPS device.

###gpsstatus:  Continuously displays simple GPS output
```
gpsstatus <gps_device>
```	
This shell script just displays the output of the GPS unit and calculates the time to first fix.


