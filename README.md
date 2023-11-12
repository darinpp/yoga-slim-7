## Info
from `dmidecode`
```
System Information
	Manufacturer: LENOVO
	Product Name: 83AA
	Version: YogaAir 14s APU8
	Wake-up Type: Power Switch
	SKU Number: LENOVO_MT_83AA_BU_idea_FM_YogaAir 14s APU8
	Family: YogaAir 14s APU8
```
```
cat /sys/class/sound/hwC1D0/subsystem_id 
0x17aa38bb
```

## Sound
It uses TI 2781 for the subwoofer speakers. 
The support is in kernel 6.6+. The driver's design has two flaws that makes the runtime PM not work very well.
On each runtime suspend, the driver resets the current program and upon resume it reloads all of the DSP's firmware. 
The firmware (`/lib/firmware/TIAS2781RCA4.bin`) supplied is also flawed as it does hardware resets (instead of software ones) each time the play pauses. 
This causes the DSP to shutdown. So on play resume, all the FW (`/lib/firmware/TAS2XXX38BB.bin`) needs to be uploaded again. 
That takes more than 2 secs before the play starts. Even after it starts, it does few pauses.
The easiest solution for this problem is to replace the hardware resets with software ones. That is done by replacing all
occurences of `00 00 5c d9` in the `TIAS2781RCA4.bin` file with `00 00 5c 19`. Check the data sheet for the description of register `5c`.

To make the subwoofer's work need the following:
#### Disable driver runtime suspend/resume PM
as root (if rc.local doesn't exit - create, make executable and add `#!/bin/sh`)
```
cat << EOF >> /etc/rc.local
# Disable runtime suspend/resume for tas 2781 as it is not working
echo on > /sys/bus/i2c/drivers/tas2781-hda/i2c-TIAS2781\:00/power/control
EOF
```
On Fedora the rc.local file is `/etc/rc.d/rc.local`

#### Place the firmware in /lib/firmware
The files `TIAS2781RCA4.bin` and `TAS2XXX38BB.bin` need to be copies in `/lib/firmware`

#### Controling subwoofer speakers volume
By default the normal speaker volume controls don't control the volume of the subwoofer's. 
The 2781 speakers are controlled with a separate control
```
amixer cget numid=3
numid=3,iface=CARD,name='Speaker Digital Gain'
  ; type=INTEGER,access=rw---R--,values=1,min=0,max=200,step=0
  : values=148
  | dBscale-min=-100.00dB,step=1.00dB,mute=0
```
To make the amp speakers change volume in sync with the volume of the main speakers - use https://github.com/darinpp/alsa-controller
It will update the digital gain for the amp speakers when the main speaker's volume changes.
The 2781 driver doesn't expose changing the volume of the L/R speakers separately (even though that should work). 
So changing the L/R balance won't afect the amp speakers.
