# Gigabyte X870E Aorus Elite WiFi7 Linux Shenanigans
This exists as I recently built a system based on this motherboard (Rev 1.1) running Fedora 41 and was encouraged by a friend (Thanks, Jason) to documnet the little tidbits needed to get some of the extras working, and hopefully tracking the progress of making all this work as time goes on. 

_Note: This is all for my system, running the Rev 1.1 version of this board, 9950X CPU, 192GB Corsair Dominator Platnium RAM, 2x WD850X M.2 4TB, and a RTX 3080 FE, BIOS F4a._

## What works out of the box? 
Short answer, nearly everything "works." 

There's just some small deeper things that need some help. Most notably, audio output control, RGB control, and FAN RPM reporting (control isn't possible yet, see below.)  

## Getting Things Working
### Audio Output Control
First primary issue I had was there's no way to simultaneously output to both the Analog Line Out jack and the Digital S/PDIF jack. In my use case I run my headphones through a bluetooth DAC that properly support sthe higher quality audio codec and my speakers are usually just for system noises. When in a focus mode I like to funnel only music and meeting/chat/etc... to the headset and having the system dings/beeps for Slack, Teams, EMail, etc... were distracting. This was a fairly simple fix: 

From [this](https://unix.stackexchange.com/questions/655767/simultaneous-digital-and-analog-output-on-pipewire) StackExchange post I placed the following in the `9999-custom.conf` file located in `/usr/share/alsa-card-profile/mixer/profile-sets/`:
```
[Profile output:analog-stereo+output:iec958-stereo+input:analog-stereo]
description = Digital Stereo (IEC958) Output + Analog Stereo Duplex
output-mappings = analog-stereo iec958-stereo
input-mappings = analog-stereo
```
_NOTE: I did have some update change where this file was located which broke my audio devices. I ultimately ended up creating a new `9999-custom.conf` file in `~/.config/alsa-card-profile` and symlinking it to the above location to hopefully preserve the file if something messes with it later._

You'll need to restart the audio stack (`systemctl --user restart pipewire`) to get the changes to take place, but once you do you'll be able to select the Digital (S/PDIF) and Analog (Line Out) outputs separately for individual apps without issue. 

### Getting fan RPM to display in tools like `sensors` or `CoolerControl`
Shoutout to [bakman2's gist](https://gist.github.com/bakman2/e801f342aaa7cade62d7bd54fd3eabd8) which lead me to [frankcrawford's it87 repo](https://github.com/frankcrawford/it87) that solved this issue. 

First run `sensors` at the terminal and verify that newer updates (if you're reading this in the future) haven't solved this issue out of the box. Following the instructions in the first link did produce the below message when running `sensor-detect` (as they said it would):
```
Trying family `ITE'...                                      Yes
Found unknown chip with ID 0x8688
```
Then following the rest of the instructions provided by `bakman2` with some slide modifications (as there's a slight change needed in the steps):
```
git clone https://github.com/frankcrawford/it87
cd it87
make clean
make
sudo make install
sudo modprobe it87 ignore_resource_conflict=1 force_id=0x8622
```
_NOTE: The `ignore_resource_conflict=1` can be omitted if you're running with the kernel argument `acpi_enforce_resources=lax` which some folks might be doing for RGB shenanigans or other similar low level shenanigans. You can try it with/without before doign t he below steps to determine what you need. You can run `modprobe -r it87` to unload it when doing testing without having to reboot._

Once you've got it working in "testing" go ahead and set it up to work every time on boot with the following commands:
```
sudo touch /etc/modprobe.d/it87.conf
sudo echo "it87 ignore_resource_conflict=1 force_id=0x8622" > /etc/modprobe.d/it87.conf
sudo touch /etc/modules-load.d/it87.conf
sudo echo "it87" > /etc/modules-load.d/it87.conf
```
_NOTE: On that 2nd line, be sure to reflect the appropriate command if you do or do not use the `ignore_resource_conflict=1` bit._

Now, when you run `sensors` you should see not only the fans, but some voltages, temperatures, and the status of the intrusion header:
```
it8622-isa-0a40
Adapter: ISA adapter
in0:           1.10 V  (min =  +0.00 V, max =  +3.06 V)
in1:           1.99 V  (min =  +0.00 V, max =  +3.06 V)
in2:           2.00 V  (min =  +0.00 V, max =  +3.06 V)
in3:           2.00 V  (min =  +0.00 V, max =  +3.06 V)
in4:         960.00 mV (min =  +0.00 V, max =  +3.06 V)
in5:           1.15 V  (min =  +0.00 V, max =  +3.06 V)  ALARM
in6:           1.13 V  (min =  +0.00 V, max =  +3.06 V)  ALARM
3VSB:          3.31 V  (min =  +0.00 V, max =  +6.12 V)
Vbat:          3.17 V  
fan1:        1175 RPM  (min =   10 RPM)
fan2:         467 RPM  (min =   10 RPM)
fan3:         807 RPM  (min =   10 RPM)
fan4:           0 RPM  (min =    0 RPM)
fan5:           0 RPM  (min =    0 RPM)
temp1:        +39.0°C  (low  = +127.0°C, high = +127.0°C)  sensor = thermistor
temp2:        +67.0°C  (low  = +127.0°C, high = +127.0°C)  sensor = thermistor
temp3:        +49.0°C  (low  = +127.0°C, high = +127.0°C)
temp4:        +51.0°C  
intrusion0:  ALARM
```
For reference, fan1 is `CPU_FAN`, fan2-4 are the `SYS_FAN1` thru `SYS_FAN3` headers, and I presume fan5 is the `FAN4_PUMP` header but I didn't verify it in my setup. I am unclear on all of the temperature sensors (and if someone put a PR in to update it I'd take it) but I am pretty sure `temp2` is the AMD chipset as it's warmer than the rest. Checking [the manual](https://www.gigabyte.com/Motherboard/X870E-AORUS-ELITE-WIFI7-rev-10-11/support#support-manual) it looks like there are 3 temp sensors on the board indicated (see "1-1 Motherboard Layout" on page 4) but this module reports 4 temp sensors while the `gigabyte_wmi` one reports 5. I am unsure what they all represent. 

### RGB Control (via OpenRGB)
Information coming soon. I need to hit save before I lose everything I've already typed. 
