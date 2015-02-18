## Tools for MT3339 Based GPS

It may seem slightly counter-intuitive, but in order for a GPS unit to produce an accurate fix,
it has to know the time, the location of the GPS satellites, and it's own rough location.
Odd, given that it's job is to tell YOU those things.  Since the satellites broadcast the time and
their positions, the GPS unit can get the info but **it must have a strong signal from 4 satellites 
for about a minute.**  If the signal level drops or a satellite goes out of view in that minute,
the GPS unit has to start over again.  Outdoors with a clear view of the sky, this isn't usually
a problem.  Indoors or when the outdoor view is obstructed by trees, terrain, buildings, etc.
it can be impossible to get an initial fix.

"But wait, my cell phone gets a fast fix indoors!".  Yep, and there's a reason for that.  
Being network connected, the phone already knows the exact time and it's rough location as
the cell network provides both.  As for the location of the satellites at any particular time
(the ephemeris data), that's public information and the GPS manufacturers thoughtfully
provide files, usually updated nightly and good for 7, 14 or 30  days, that can be downloaded
to their GPS units.  The whole process is called Assisted-GPS and doesn't require as strong
a signal to get and maintain a fix.

If you going to use your MT3339 based unit only outdoors, no need to read further.  Otherwise...

The Adafruit Ultimate GPS is based on the GlobalTop PA6H GPS unit which is in turn based on the
MediaTek MT3339 chip.  GlobalTop provides the ephemeris data in the form of an
EPO (Extended Prediction Orbit) file available from their FTP site (see eporetrieve, below).
There are other sources for EPO data but there are 2 formats, EPO and EPO-II.
The MT3339 in this applications needs the original EPO format and can handle only 7 or 14 day files.
That's what's on the GlobalTop site.

So now what?  Use the tools...

### Get the EPO file:

#### eporetrieve:  Retrieves the current EPO file from the GlobalTop FTP Server
```
eporetrieve [ <file_name> [ <output_file> ] ]

Example:
# eporetrieve MTK14.EPO /tmp/MTK14.EPO
 
```
It's generally best to do this in the morning UTC to make sure the file is valid for the current time.

### Inspect the EPO file:

#### epoinfo: Displays information about an EPO file
```
epoinfo <EPO_File>

Example:
# epoinfo /tmp/MTK14.EPO
Opening EPO Type I file
Set:    1.  GPS Wk: 1832  Hr:  48  Sec:  172800  2015-02-16 23:59:46 to 2015-02-17 05:59:46 UTC
Set:    2.  GPS Wk: 1832  Hr:  54  Sec:  194400  2015-02-17 05:59:46 to 2015-02-17 11:59:46 UTC
Set:    3.  GPS Wk: 1832  Hr:  60  Sec:  216000  2015-02-17 11:59:46 to 2015-02-17 17:59:46 UTC
Set:    4.  GPS Wk: 1832  Hr:  66  Sec:  237600  2015-02-17 17:59:46 to 2015-02-17 23:59:46 UTC
Set:    5.  GPS Wk: 1832  Hr:  72  Sec:  259200  2015-02-17 23:59:46 to 2015-02-18 05:59:46 UTC
<snip>
Set:   53.  GPS Wk: 1834  Hr:  24  Sec:   86400  2015-03-01 23:59:46 to 2015-03-02 05:59:46 UTC
Set:   54.  GPS Wk: 1834  Hr:  30  Sec:  108000  2015-03-02 05:59:46 to 2015-03-02 11:59:46 UTC
Set:   55.  GPS Wk: 1834  Hr:  36  Sec:  129600  2015-03-02 11:59:46 to 2015-03-02 17:59:46 UTC
Set:   56.  GPS Wk: 1834  Hr:  42  Sec:  151200  2015-03-02 17:59:46 to 2015-03-02 23:59:46 UTC
  56 EPO sets.  Valid from 2015-02-16 23:59:46 to 2015-03-02 23:59:46 UTC

```
Each set covers a 6 hour period so for a 14 day file, there should be 56 sets in the file.
Make sure the file is a Type I file and that the "Valid from" span includes "now". 

### Load the EPO file to the GPS:

#### epoloader:  Loads an EPO file to the GPS
```
epoloader [-h] [-t yyyy,mm,dd,hh,mm,ss | - ] [-l lat.dddddd,lon.dddddd,alt]
	[-s {4800,9600,19200,38400,57600,115200}] [-k] [-c]
	<EPO_File> <gps_device>
	
'epoloader --help' for more information.
```

Before you start this step, make sure you're unit is connected and that the port speed
matches the GPS speed.  You can use the gpsinit utility (see below) to do this.
If you can 'cat' the port where the GPS is connected and get valid NMEA data,
you should be OK.
```
# stty -F /dev/tty<something> raw
# cat /dev/tty<something>
$GPGGA,000221.000,3899.0037,N,10499.9070,W,1,10,1.01,1795.4,M,-20.7,M,,*6C
$GPGLL,3899.0037,N,10499.9070,W,000221.000,A,A*4E
$GPRMC,000221.000,A,3899.0037,N,10499.9070,W,0.16,224.89,180215,,,A*74
$GPVTG,224.89,T,,M,0.16,N,0.30,K,A*3C
$GPZDA,000221.000,18,02,2015,,*5A
```

If you unit is at factory state, you'll need to supply your location and the time
in addition to the EPO file.  If you're unit already has a fix and you just need to update
EPO data, you don't need to supply the location and time.  

You location and altitude must be specified in decimal degrees and meters.  Google Earth
is a good source.  The more accurate the better but it needs to be within 30km.

For the time, just let epoloader get it from your host system, assuming it's
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

NOTE:  It's pretty much impossible to mess up the GPS unit with this process as it doesn't touch
the unit's firmware.  If you get into a bad communications state, just try again or remove the
power and battery for a few seconds to reset back to factory state.    

Now you're ready.  Assuming your location and time is in epoloader.conf, the EPO file is
/tmp/MTK14.EPO, your device is connected to /dev/ttyUSB1 and you want to use the default 115200 speed, then...
```
# epoloader @epoloader.conf /tmp/MTK14.EPO /dev/ttyUSB1
Opening EPO Type I file
Current port speed: 9600
Set port speed: 115200
Setting known values: 38.889468,-77.352420,5  2015,02,17,23,15,35
Clearing existing EPO data
Setting binary mode, speed: 115200
Sending 56 EPO sets
Sending set    1.  Valid from 2015-02-16 23:59:46 UTC
Sending set    2.  Valid from 2015-02-17 05:59:46 UTC
Sending set    3.  Valid from 2015-02-17 11:59:46 UTC
Sending set    4.  Valid from 2015-02-17 17:59:46 UTC
Sending set    5.  Valid from 2015-02-17 23:59:46 UTC
<snip>
Sending set   52.  Valid from 2015-03-01 17:59:46 UTC
Sending set   53.  Valid from 2015-03-01 23:59:46 UTC
Sending set   54.  Valid from 2015-03-02 05:59:46 UTC
Sending set   55.  Valid from 2015-03-02 11:59:46 UTC
Sending set   56.  Valid from 2015-03-02 17:59:46 UTC
  56 sets sent.  Valid from 2015-02-16 23:59:46 to 2015-03-02 23:59:46 UTC
Resetting NMEA Mode
Verified EPO in NVRAM matches file

```
At 115200, the process should take about 17 seconds.  If you didn't specify the '-k' option,
the unit and the port will be returned to their previous speed. 

### Does it make a difference??

Oh yes...  Look at the times for internal antenna/inside.  The difference is amazing.

	UnitLocation:	Always inside. Too cold to be outside.
	Antenna:  		Internal or External Active
	AntLocation:
		ClearSky:	Outside, clear sky
		Inside:		First floor room with window
		DeepInside:	First floor interior room, no window
	EPO: 			Y - loaded, N - not loaded
	StartFrom:
		Factory:	Factory reset
		Reset:		Hot start or hold enable pin low for 1 sec
		PowerCycle: Remove VCC for 5 seconds (with backup battery connected)
	TTFF:			Time to First Fix in seconds (3 tests)
	 

| UnitLocation | Antenna  | AntLocation | EPO | StartFrom  | TTFF1 | TTFF2 | TTFF3 |
|--------------|----------|-------------|-----|------------|------:|------:|------:|
| Inside       | External | Clear Sky   | N   | Factory    | 68    | 72    | 150   |
| Inside       | External | Clear Sky   | N   | Reset      | 8     | 10    | 12    |
| Inside       | External | Clear Sky   | N   | PowerCycle | 13    | 15    | 9     |
| Inside       | External | Clear Sky   | **Y** | Factory    | 2     | 2     | 2     |
| Inside       | External | Clear Sky   | **Y**   | Reset      | 7     | 7     | 7     |
| Inside       | External | Clear Sky   | **Y**   | PowerCycle | 1     | 8     | 2     |
| Inside       | External | Inside      | N   | Factory    | 590   | 367   | 283   |
| Inside       | External | Inside      | N   | Reset      | 6     | 4     | 5     |
| Inside       | External | Inside      | N   | PowerCycle | 8     | 6     | 15    |
| Inside       | External | Inside      | **Y**   | Factory    | 10    | 6     | 14    |
| Inside       | External | Inside      | **Y**   | Reset      | 5     | 5     | 5     |
| Inside       | External | Inside      | **Y**   | PowerCycle | 13    | 10    | 7     |
| Inside       | External | Deep Inside | N   | Factory    | 420   | 35    | 102   |
| Inside       | External | Deep Inside | N   | Reset      | 3     | 6     | 6     |
| Inside       | External | Deep Inside | N   | PowerCycle | 2     | 6     | 2     |
| Inside       | External | Deep Inside | **Y**   | Factory    | 14    | 20    | 17    |
| Inside       | External | Deep Inside | **Y**   | Reset      | 7     | 8     | 5     |
| Inside       | External | Deep Inside | **Y**   | PowerCycle | 5     | 19    | 10    |
| Inside       | Internal | Inside      | N   | Factory    | 2422  | 3600* | Gave Up |
| Inside       | Internal | Inside      | N   | Reset      | 6     | 4     | 11    |
| Inside       | Internal | Inside      | N   | PowerCycle | 10    | 9     | 9     |
| Inside       | Internal | Inside      | **Y**   | Factory    | 14    | 33    | 15    |
| Inside       | Internal | Inside      | **Y**   | Reset      | 7     | 4     | 4     |
| Inside       | Internal | Inside      | **Y**   | PowerCycle | 20    | 25    | 16    |


### gpsinit:  Initializes GPS configuration
```
gpsinit -i <init_command> [ -s <speed> ] | -f <init_file>  <gps_device>
```
### gpssend:  Sends a single command to the GPS
```
gpssend <command_string> <gps_device>
```
###gpsstatus:  Continuously displays simple GPS output
```
gpsstatus <gps_device>
```	
	



