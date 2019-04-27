# Installing "real" linux on a Google Pixelbook

This repo documents the process of replacing ChromeOs on a stock [Google Pixelbook][pixelbook_product_page]
with a "real" linux distribution. It also contains an automated configuration script that will fix things
that are broken in a stock install, like sound, display and keyboard backlights, touchpad sensitivity, etc.

A very nice feature of the method described here is that it **does not require taking the machine apart**!
Previous resources I've come across have instructed people to disassemble their Pixelbook and disconnect
the battery cable to disable the firmware write protect. This method avoids the need for that, although
you will need to spend ~$20 USD on a special USB cable. See the [installation instructions](#installation)
for details.

The automated configuration targets Ubuntu 19.04 (Disco Dingo), although it's likely that the basic techniques
used will work for any distribution. I initially tried using Fedora Workstation 29, but ran into an issue where
the system would crash immediately after resuming from suspend. I assumed this was due to my tweaks, but decided
to give another distro a shot and found Ubuntu worked without issue. As a nice bonus, bluetooth works out of the
box on Ubuntu, whereas Fedora required some fiddling post-install.


## Current Status

Here's what's working at the moment:

| Feature            | Stock Ubuntu         | After modifications                     |
|--------------------|----------------------|-----------------------------------------|
| WiFi               | Working              | Working                                 |
| Bluetooth          | Working              | Working                                 |
| Touchscreen        | Working              | Working                                 |
| Suspend            | Working              | Working                                 |
| Touchpad           | Working, but awkward | Working (tweaked to feel nice)          |
| Display backlight  | Always on at 100%    | Adjustable using standard controls      |
| Sound              | Broken               | Working ([see below](#audio-support))   |
| Keyboard backlight | Broken               | Working (using helper script to adjust) |
| Swap               | Working              | Unsupported, see [swap](#swap-support)  |
| Hibernate          | Untested             | Unsupported                             |

## Implementation details

If you're not interested in the details, you can skip ahead to the [installation instructions](#installation), but
it might be worhth giving this section a skim so you can see what's going on and fix things if they break later.

This section refers to the modifications performed by the automatic configuration script in this repo. For details
on unlocking the write protect setting and installing linux in the first place, see the [installation instructions](#installation).

### Kernel

The kernel is compiled from Google's ChromiumOS fork of the linux kernel, which enables support for the display
backlight controls and the audio hardware.

This approach was heavily inspired by [@megabytefisher's brilliant hack](https://github.com/megabytefisher/eve-linux-hacks),
the key insight of which is to compile the chromium fork of the linux kernel with a configuration that's as close to the
one used by the stock Pixelbook as possible. Many thanks to [@megabytefisher](https://github.com/megabytefisher), and I
hope that my work will be useful to them as well.

The configuration is copied from `@megabytefisher's`, with some very small modifications since I'm using a slightly newer
version of the kernel (`4.4.178` vs `4.4.164`).

### Firmware

Again, this is inspired by `@megabytefisher`. To enable the audio hardware, we need some firmware files that can be
extracted from a Pixelbook recovery image. The automated configuration script will pull all the firmware files from
the recovery image and install them to the correct place, and also pull some configuration needed for full audio
support.

### Audio support

Once we're running the chromium-flavored kernel with the firware files in place, we can access the raw audio hardware,
and `aplay -l` should show some output devices. However, the Pixelbook uses an audio chip that's not well supported by
the linux audio "userland" components (ALSA and pulseaudio). As a result, both ALSA and pulseaudio can only playback
through the internal speakers, and there's no way to switch to the headphones. Recording is also unsupported out of
the box, as none of the capture devices are recognized.

This was a deal-breaker for me, since my hope for this machine was to use it for work, where I often need to participate
in video calls.

#### Background info

Before getting into the solution, let me give a quick overview oonf how audio on linux works, filtered through my super
limited understanding.

At the bottom is the hardware driver, which takes the form of a kernel module, which in our case also requires some
special firmware files. Once those are in place, the audio hardware is avalable outside of the kernel. However,
the "interface" for using the hardware is extremely low-level and not directly usable by most linux programs and
end users.

The layer just above the kernel is [ALSA](https://www.alsa-project.org/wiki/Main_Page), the Advanced Linux Sound Architecture.
This provides a common interface to the many, many kinds of audio drivers exposed by the kernel. It lets you do things
like adjust the volume and other parameters exposed by the driver, and provides a high-level API that applications can
use to play and record audio.

Above ALSA is [pulseaudio](https://www.freedesktop.org/wiki/Software/PulseAudio/), which serves as an "audio server", allowing
many "client" applications to all share the same audio hardware, something which is quite difficult with ALSA alone.  PulseAudio
is used by Gnome and other desktop environments; when you move the volume slider in the Gnome UI, it tells the PulseAudio server
to adjust the volume on the current output device. PulseAudio uses ALSA for the acutal playback and recording, although I think
it also supports other backends.

#### How to get "perfect" audio on the pixelbook

The problem with audio on the Pixelbook is that neither ALSA nor PulseAudio are configured to use the specific controls and devices
provided by the kernel audio driver. ALSA tries to map the driver interfaces to a known configurations, but since there's no special
mapping for the Pixelbook chip, it falls back on the default "Analog Stereo" profile, which only supports the internal speakers.

Everything works great on ChromeOS, of course, because Google has put in the engineering work to get everything playing nice with
the hardware. However, while ChromeOS uses ALSA, it doesn't use it directly, and it also doesn't use PulseAudio. Instead, Google
wrote their own audio server, [`cras`](https://www.chromium.org/chromium-os/chromiumos-design-docs/cras-chromeos-audio-server),
the Chromium OS Audio Server. This serves the same purpose as PulseAudio but consumes fewer resources.

I spent a little while trying to figure out how to get ALSA and PulseAudio to play nice with the Pixelbook audio driver, but I
didn't manage to make any headway.

The breakthrough came when I saw that [crouton](https://github.com/dnschneid/crouton) supports audio by actually compiling `cras` inside the
`chroot` environment and letting `cras` talk to ALSA and manage the hardware. I had already discovered that the eve recovery image contained
configuration files for `cras` that are specific to the pixelbook hardware, so I figured that the best shot of getting everything to work
would be to get `cras` in the mix.

I decided to try compiling `cras` and running it inside the standard linux environment to see if it could make sense of the weird
hardware interface provided by the driver and exposed by ALSA. Some hours later, it works!

After compiling `cras` and running it (with the config files in the right place), the `cras_test_client` included with `cras` can
switch between speakers & headphones (and also HDMI 1 and HDMI 2, but I haven't tried using those yet). It can also switch between
a headset mic and the internal mic, and exposes "loopback" capture devices to record whatever is playing through the output device.

You can also use `cras_test_client` to test audio playback and recording - to see if things are working try
`cras_test_client --playback_file /usr/share/sounds/alsa/Front_Left.wav`.

Included with `cras` are some ALSA plugins that create a new virtual ALSA device named `cras` - if those are in the right place
(`/usr/lib/x86_64-linux-gnu/alsa-lib/` on Ubuntu), apps that support ALSA can target that device and the audio will be routed
to `cras`, which will then send it back to ALSA, this time targeting the actual hardware devices.

The final piece of the puzzle is PulseAudio, which will let us use the standard Gnome volume controls and basically let the
rest of the system pretend that we're using a standard audio setup.

Once again, crouton shows the way - the [crouton pulseaudio config](https://github.com/dnschneid/crouton/blob/master/chroot-etc/pulseaudio-default.pa)
routes audio through the special `cras` ALSA device, which then sends audio to `cras`, which sends it back to ALSA, which
finally sends it to the kernel, where it hits the hardware at last.

Here's a picture of this Rube Goldberg contraption:

```
                    cras uses ALSA's real audio
                    devices for the hardware         +---------------+
                    exposed by the driver            |               |
                                                     |     cras      |
                               +---------------------+               |
                               |                     |               |
                               v                     +-------+-------+
+--------------+      +--------+------+                      ^  Audio sent to the "cras"
|              |      |               |                      |  device gets routed to cras,
|    Audio     +<-----+     ALSA      +----------------------+  which sends it back to ALSA
|    Driver    |      |               |                         using the real hardware device
|              |      |               |              +---------------+
+--------------+      +-------+-------+              |               |
                              ^                      |  pulseaudio   |
                              +----------------------+               |
                                                     |               |
                            pulse uses               +---------------+
                            virtual "cras" ALSA device
                            
```

#### Switching audio devices

I was feeling pretty smug when I got audio working through the headphones, but the process of switching devices wasn't very
pleasant. The basic process works like this:

1. run `cras_test_client` with no arguments to get a status report, including the names and IDs of all the input and output devices
2. find the ID of the output device you want to target (will be two numbers separated by `:`, e.g. `7:0`)
3. run `cras_test_client --select_output $id_from_step_2`

This seemed like a job for a hacky script, so I wrote one up. The `eve-audio-ctl.py` script that gets installed when you run the
automated setup script wraps `cras_test_client` and parses the device IDs, so you can just do e.g. `eve-audio-ctl.py -o headphone`.

Running `eve-audio-ctl.py` with no arguments will show a list of available audio devices and indicate which is active.

The final piece of the puzzle is automatically switching inputs when headphones are plugged or unplugged. For this, I used the
`acpi_listen` command, which writes messages to stdout when headphones and microphones are plugged in or removed. If you run
`eve-audio-ctl.py -j`, the script will listen for events and switch to headphones when they're plugged in and speakers when
the headphones are unplugged. It will also switch the input to the headset mic if one is plugged, and switch back to the
internal mic if removed.

The automatic install also creates a systemd service called `eve-headphone-jack-listener`, which runs the script automatically
in the background, so everything should "just work" as expected without having to explicitly run the script.

### Keyboard backlight

When running the chromium-flavored kernel, the keyboard backlight can be controlled by writing an integer value between 0 and 100
to `/sys/class/leds/chromeos::kbd_backlight/brightness`, e.g. `echo 100 > '/sys/class/leds/chromeos::kbd_backlight/brightness'`.
By default only `root` has permission to do this, so the automatic install script installs a udev rules file to grant permission
to an `leds` group, and also makes sure the group exists and the main user account is a member.

There's also a `eve-keyboard-brightness.sh` script installd in `/usr/local/bin` that you can use to more easily set the brightness:
`eve-keyboard-brightness.sh 100`.

Right now the script only supports absolute brightness values between 0 and 100. At some point I'll probably add support for relative 
increments / decrements, so that you can assign a keyboard shortcut to adjust the brightness by some amount with each keypress.


ansible: https://ansible.io # FIXME: is this right?
pixelbook_product_page: http://fixme.before.merge
