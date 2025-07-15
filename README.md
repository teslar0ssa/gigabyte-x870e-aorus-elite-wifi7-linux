# Gigabyte X870E Aorus Elite WiFi7 Linux Shenanigans
This exists as I recently built a system based on this motherboard (Rev 1.1) running Fedora 41 and was encouraged by a friend (Thanks, Jason) to document the little tidbits needed to get some of the extras working, and hopefully tracking the progress of making all this work as time goes on. 

> [!NOTE]
> _Specifications for referece: Rev 1.1 board, 9950X CPU, 192GB Corsair Dominator Platnium RAM, 2x WD850X M.2 4TB, nVidia RTX 3080 FE, and BIOS F4a._

## What works out of the box? 
Short answer, nearly everything "works." 

There's just some small deeper things that need some help. Most notably, audio output control, RGB control, and FAN RPM reporting (control isn't possible yet, see below.)  

## Getting Things Working
### Audio Output Control
#### UPDATE 15 JUL 25: I don't recall when, but sometime in the last couple of months an update enabled `pavucontrol` to enable "Pro Mode" on the "Family 17h/19h/1ah HD Audio Controller." Doing this split out the analog and digital outputs rendering the need to do the below obselete. You can now just flip to "Pro Mode" and the two outputs are split to "Pro" and "Pro 1" giving me all the control I wanted. This also seems to survive updates and reboots without an issue. You can still do the below, but IDK why you would. I'd like to rename the devices, but I haven't dug into how yet. Will update when/if I do...I'm mostly just lazy.

First primary issue I had was there's no way to simultaneously output to both the Analog Line Out jack and the Digital S/PDIF jack. In my use case I run my headphones through a bluetooth DAC that properly support sthe higher quality audio codec and my speakers are usually just for system noises. When in a focus mode I like to funnel only music and meeting/chat/etc... to the headset and having the system dings/beeps for Slack, Teams, EMail, etc... were distracting. This was a fairly simple fix: 

From [this](https://unix.stackexchange.com/questions/655767/simultaneous-digital-and-analog-output-on-pipewire) StackExchange post I placed the following in the `9999-custom.conf` file located in `/usr/share/alsa-card-profile/mixer/profile-sets/`:
```
[Profile output:analog-stereo+output:iec958-stereo+input:analog-stereo]
description = Digital Stereo (IEC958) Output + Analog Stereo Duplex
output-mappings = analog-stereo iec958-stereo
input-mappings = analog-stereo
```
> [!TIP]
> _I did have some update change where this file was located which broke my audio devices. I ultimately ended up creating a new `9999-custom.conf` file in `~/.config/alsa-card-profile` and symlinking it to the above location to hopefully preserve the file if something messes with it later._

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
> [!TIP]
> _The `ignore_resource_conflict=1` can be omitted if you're running with the kernel argument `acpi_enforce_resources=lax` which some folks might be doing for RGB control or other similar low level shenanigans. You can try it with/without before doign the below steps to determine what you need. You can run `modprobe -r it87` to unload it when doing testing without having to reboot._

Once you've got it working in "testing" go ahead and set it up to work every time on boot with the following commands:
```
sudo touch /etc/modprobe.d/it87.conf
sudo echo "it87 ignore_resource_conflict=1 force_id=0x8622" > /etc/modprobe.d/it87.conf
sudo touch /etc/modules-load.d/it87.conf
sudo echo "it87" > /etc/modules-load.d/it87.conf
```
> [!IMPORTANT]
> _On that 2nd line, be sure to reflect the appropriate command if you do or do not use the `ignore_resource_conflict=1` bit._

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
Here's a reference table for the name returned by `sensors` with the actual header name as printed on the motherboard's silkscreen: 

| `sensors` name | mobo header |
|--------------|-------------|
| fan1         | CPU_FAN     |
| fan2         | SYS_FAN1    |
| fan3         | SYS_FAN2    |
| fan4         | SYS_FAN3    |
| fan5         | FAN4_PUMP   |

It doesn't appear `CPU_OPT` has a reporting sensor, but I suspect it's rarely used and not critical here. I am also unclear on all of the temperature sensors (and if someone put a PR in to update it I'd take it) but I am pretty sure `temp2` is the AMD chipset as it's warmer than the rest. Checking [the manual](https://www.gigabyte.com/Motherboard/X870E-AORUS-ELITE-WIFI7-rev-10-11/support#support-manual) it looks like there are 3 temp sensors on the board indicated (see "1-1 Motherboard Layout" on page 4) but this module reports 4 temp sensors while the `gigabyte_wmi` one reports 5. I am unsure what they all represent or why there's a discrepancy between the two. 

### RGB Control (via OpenRGB) ...and RAM/SPD temps too!
> [!TIP]
> _If you don't want RGB, you might still want to do the `i2c` stuff so you can view the RAM module temps._

I tried to avoid RGB in this build as much as possible, unlike my gaming rig which looks like unicorn vomit, but alas the market dictated otherwise. I ended up with some Corsair Dominator Platnium RGB RAM as it was something like $60 or $80 cheaper than non-RGB RAM. (Also, after getting this PC working I learned that my cat seemed to miss the RGB of my old system that used to be here as he seemed pleased to stare at the RAM in unicorn vomit mode while dozing off...many apologies to you, Sagan, for taking away your joy for a few months.) As a side effect of wanting to be able to control the RGB of this RAM, I also enabled the ability see the RAM module temp sensors and dim the annoying GEFORCE RTX logo on my 3080 FE card that defaults to a brightness level I can only describe as "the sun." 

First off I'll note that I had quite a bit of issues with the RPM Fusion abd Flatpak/Flathub versions. I just snagged the latest Linux amd64 build [from their OpenRGB.org home page](https://openrgb.org/releases.html) and it worked fine, the latest version may change over time of course. I'm running `openrgb_1.0rc1_x86_64_f40_1fbacde.rpm` currently and everything works as expected, for whatever that's worth.

The RGB/temp sensors for these seem to live ont eh `i2c` bus and just need a litle help getting enabled:
```
sudo touch /etc/modules-load.d/i2c.conf
sudo echo "i2c-dev" > /etc/modules-load.d/i2c.conf
sudo echo "i2c-piix4" > /etc/modules-load.d/i2c.conf
```
This should start the necessary i2c components on boot, but you can fire them up manually with `modprobe i2c-dev` and `modprobe i2c-piix4`. This was all that was needed to start having the RAM/SPD temps show in `sensors` and the RGB control to become available in `OpenRGB`. I didn't need to install anything extra, but I did install `i2c-tools` via `dnf` to be able to run diagnostic commands like `i2cdetect`. You may be able to get by without these. 

One annoyance is there's some RGB on the motherboard it self that doesn't appear to be controllable. There's [an open issue](https://gitlab.com/CalcProgrammer1/OpenRGB/-/issues/4306) for it (many other Gigabyte boards included) and have been merged into this issue. I signed up for notifications and hope to solve it once it's merged as it appears they do have it sorted. Until then, that one LED under a heatsink shines bright white. 
