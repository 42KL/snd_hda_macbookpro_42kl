# snd_hda_macbookpro: 42 Kuala Lumpur fork

This is [davidjo/snd_hda_macbookpro](https://github.com/davidjo/snd_hda_macbookpro) with one fix on top: a lock on the headset event queue, which stops a kernel crash at logout. The fix changes `patch_cirrus/patch_cirrus_new84.h` only, which the driver uses on every kernel version. Everything else is upstream as of [`89b22ff`](https://github.com/davidjo/snd_hda_macbookpro/commit/89b22ff90b86468b186706861dd18663562defa7) (2026-09-06), and the upstream README follows below.

## The crash

On iMac19,2 (Cirrus Logic CS8409 with a CS42L83 headset codec), the driver could crash the kernel:

```
BUG: kernel NULL pointer dereference
RIP: cs_8409_perform_external_device_unsolicited_responses
```

It only happened in sessions where the 3.5 mm headphone jack was used (about 6% of them, and 0 of 2002 sessions without jack activity). Most crashes came 0–7 s after logout, when pulseaudio closes its streams. The user's pulseaudio was then stuck in D state, so their home directory stayed mounted until the machine was rebooted. Upstream still has the bug at `89b22ff`.

Cause: `cs_8409_perform_external_device_unsolicited_responses()` runs both from the jack event handler and from the playback and capture PCM hooks. Each walked `spec->unsol_list` and `memset` its entries with no lock, so one could follow a `next` pointer the other had just zeroed.

Fix: a spinlock around every update of the list and its slots. The drain functions move the pending entries to a local list under the lock and handle them outside it, because handling an entry sends codec verbs and may sleep.

## Tested on

iMac19,2, Ubuntu 22.04, kernel `6.5.0-44-generic`.

- The fix was first built on the older upstream commit `259cc39` (module md5 `ba23564f9f3e4756b5c4d91383849f92`, srcversion `8F7574D0F55B9DA0FCFA570`). Speakers to internal mic loopback works the same as without the fix, and 40 simultaneous playback and capture open/close cycles passed with no kernel warnings.
- That build has run on about 130 machines since 2026-10-09. From 2026-10-09 to 2026-10-12 they logged over 8,600 headset jack events and no crashes. Before the fix there were 25 crashes in 21 days.
- This branch (`89b22ff` plus the fix) builds on `6.5.0-44-generic` (srcversion `35698FC3334828B08EA50B2`, 2026-10-13) but has not been loaded on a machine yet. For kernels before 6.17, the only driver changes since `259cc39` are braces around debug prints and a mic change behind the `INTERNAL_MIKE_ONLY` build flag, which is off by default.

## Building

Same as upstream. On Ubuntu the script takes the sound sources from the kernel source package:

```sh
sudo apt install linux-source-6.5.0
sudo ./install.cirrus.driver.sh
```

This installs `/lib/modules/$(uname -r)/updates/snd-hda-codec-cs8409.ko`, which loads at the next boot.

## Taking new upstream changes

The 42KL workstations build from `master` of this repo every time the Ansible role runs, so build and test on one machine before pushing:

```sh
git remote add upstream https://github.com/davidjo/snd_hda_macbookpro.git
git fetch upstream
git merge upstream/master
```

---

# snd_hda_macbookpro

This is a kernel driver for sound on Macs with Cirrus 8409 HDA chips.
Sound output is now reasonably complete and integrated with Linux.
Sound input still needs work.


It will play audio through Internal speakers or headphones.

The primary audio should be set to Analogue Stereo Output in the Settings Audio dialog. Alternatively, if you want to use the internal microphone, set it to Analogue Stereo Duplex.

Sound recording from internal mike and headset mike is not yet fully interfaced with Linux user side.

The recorded sound level is very low but this is the sound level as returned in OSX.
Amplification will be required eg using something like PulseEffects.


The hardware device sound format is limited to 2/4 channel 44.1 kHz S24_LE S32_LE.
As long as use the default device volume control, other formats, frequencies work.


NOTA BENE: The direct hardware device (hw:0,0) and plughw:0,0 device have NO volume control so will be VERY loud!


Currently this works with MAX98706, SSM3515 and TAS5764L amplifiers.
It will NOT work with other amplifiers as each amplifier requires specific programming.


Power down/sleep completely unknown and untested.
At the moment everything is permanently powered on.


The Apple speaker setup is 4 speakers as a left tweeter, left woofer, right tweeter and right woofer
so this is actually a classic HiFi stereo (ie 2 channel) speaker system.
(These names are listed in the layout files under AppleHDA.kext/Contents/Resources).

The channel order for Linux has been modified to left tweeter, right tweeter and left woofer, right woofer
as this fits in with the Linux way much better.

The driver also has been modified to duplicate a stereo sound source onto the second stereo channel so all
speakers are driven (this essentially replicates the snd_hda_multi_out_analog_prepare function).

This will not sound the same as Apple (which is known to be using specific digital filter effects in CoreAudio).

To create a more Apple-like sound requires creating eg an Alsa pseudo device to channel duplicate a stereo sound
and apply different digital filters to the tweeter and woofer channels.


NOTE. My primary testing kernel is now Ubuntu LTS 24.04 6.8.


NOTA BENE. As of linux kernel 6.17 the sound kernel source directory has been completely re-organized.
           The installation script now works for 6.17 kernel versions (and later when they arrive).
           The old installation script is now called install.cirrus.driver.pre617.sh.
           The new version of the install.cirrus.driver.sh script will detect your kernel version and exec
           the old installation script as needed.
           For older kernel version you can just run the old installation script directly
           ie install.cirrus.driver.pre617.sh.
           Note that for kernel version 6.17 new files and directories have been added to the repo
           rather than attempting to update the pre 6.17 versions (as the kernel source changes also
           involved name changes and the new files are more consistent with the new kernel names).


The following installation setup provided by leifliddy.



Compiling and installing driver:
-------------

**fedora package install**
```
dnf install gcc kernel-devel make patch wget
```
**ubuntu package install**  
```
apt install gcc linux-headers-generic make patch wget
```
**arch package install**
```
pacman -S gcc linux-headers make patch wget
```
**void package install**
```
xbps-install -S gcc make linux-headers patch wget
```

**build driver**  
```
git clone https://github.com/davidjo/snd_hda_macbookpro.git
cd snd_hda_macbookpro/
#run the following command as root or with sudo
./install.cirrus.driver.sh
reboot
```

**Deleting driver**
```
# Check your kernel version
uname -a
# delete the ko file
sudo rm /lib/modules/{kernel version}/updates/snd-hda-codec-cs8409.ko
sudo depmod -a
```

Dynamic Kernel Module Support (dkms):
-------------

dkms is a framework which allows kernel modules to be dynamically built for each kernel on your system.
See here for more details: https://github.com/dell/dkms
You will need to first install dkms on your system

**install driver via dkms**
```
sudo ./install.cirrus.driver.sh -i
```

**remove driver from dkms**
```
sudo ./install.cirrus.driver.sh -r
```

