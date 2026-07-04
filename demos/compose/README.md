# Compose Demo

**▶ [Run it live in Wokwi](https://wokwi.com/experimental/viewer?diagram=https://raw.githubusercontent.com/pglewis/os-demo/main/demos/compose/diagram.json&firmware=https://raw.githubusercontent.com/pglewis/os-demo/main/demos/compose/kernel.elf)**: runs in your browser, nothing to install.

## What this is

This is a small operating system written from scratch for RISC-V microcontrollers, running here on a simulated ESP32-C3.

## The prompt

When the firmware finishes booting, you land at the cli prompt:

```
>
```

**Note:** the alpha Wokwi viewer uses an HTML input at the very bottom of the page, not a real serial terminal. It won't transmit control or escape characters, so to send a Ctrl-C, type the literal characters `^c` and press Enter as a workaround.

Quick orientation: the kernel is heavily inspired by OS-9, so if you're familiar with it you'll recognize the bones. It's Unix-y. If you're comfortable at a Linux shell, it will feel right at home. You run programs by name, pipe them together, and redirect their input and output, like you'd expect.

## Modules all the way down

The primary building block of the system is a **module**: a small, self-contained, named part you can point at and use. The command line interface is a module. So is every command you run, every piece that knows how to talk to the hardware, even the descriptions of what's wired to which pin.

RAM is scarce and you don't always need everything resident. Modules live in one of two places:

1. **Boot**: built into the boot image, resident in RAM the moment you power on.
2. **Stored**: kept in a flash store, loaded into RAM the first time it's needed.

Either way they end up resident in RAM; the only difference is whether it was there from boot or paged in on demand.

`mdir` lists the program modules resident in RAM right now; `mstore` lists the programs in the flash store, showing which are already loaded into resident RAM and which are still only on flash:

```
> mdir
bar               program
cli               program
mdir              program
pixbar            program
rangepct          program
sample            program
servosink         program
threshold         program

> mstore
cli               program     resident  5425
echo              program     stored    438
cat               program     stored    1677
mdir              program     resident  2249
mstore            program     resident  2265
msave             program     stored    1657
minfo             program     stored    3501
ps                program     stored    1825
kill              program     stored    1881
wr                program     stored    1997
mkdev             program     stored    3437
```

## Piping programs together

Each of these programs does one small job. A program reads from its input and writes to its output, and the `|` symbol connects one program's output to the next one's input.

Start dry, no hardware involved. `echo` just prints whatever you hand it, and `bar` draws a number from 0 to 100 as a meter. Pipe one into the other:

```
> echo 34 | bar
[######--------------]  34%
```

`bar` wants a number from 0 to 100, but real inputs rarely arrive in that range. The demo includes a `rangepct` utility as the adapter: give it a low and a high bound and it maps whatever comes in onto 0 to 100. Drop it in the middle for a three-stage pipe:

```
> echo 150 | rangepct 100 200 | bar
[##########----------]  50%
```

150 sits halfway between 100 and 200, so the meter reads 50%.

## Devices

`mdir` only lists programs by default. This board also has hardware wired to it, each piece defined by a **device descriptor**: a small module that describes one piece of hardware. Some are inputs, like the potentiometer you turn; others are outputs, like the servo. Ask `mdir` to list them with `-d`:

```
> mdir -d
pot               descriptor
servo             descriptor
sonar             descriptor
strip             descriptor
```

Pick one and look closer. `minfo` prints what a module's header knows, and for a device that includes how it's wired and what handles it:

```
> minfo pot
pot
  version 1
  type descriptor
  size 104
  attributes 0x00
  revision 0
  mem 0
  fm gpio
  drv adc
  init gpio-v1
  init_size 20
  pin 3
  pin2 0
  dir in
  active high
  flags 0x00000000
  debounce 0
```

Its type is `descriptor`: data naming which file manager and driver handle the hardware, plus any config they need. Here that's an input on pin 3, bound to the `gpio` file manager and the `adc` driver. The file manager and driver are stock, reusable modules.

To actually use a device you open it by path: its name with a leading slash, like `/pot`, `/sonar`, and `/strip`. That leading slash is how every program names the piece of hardware it wants to talk to.

## Reading a device

A device isn't a program, so it can't sit at the head of a pipe on its own. A `sample` utility bridges the gap: hand it a device path and it opens the device, reads the current value, and prints it over and over, one number per line.

```
> sample /pot
```

Turn the knob and the numbers follow; it runs until you stop it with `^c`. Now the pot is a stream, which is exactly what a pipe wants to eat.

## From knob to meter

Now you have all three pieces. `sample /pot` turns the knob into a stream of numbers, `rangepct 0 4095` maps that raw 0-to-4095 range onto 0 to 100, and `bar` draws the result as a meter, the exact shape from the dry run now aimed at a live sensor. Snap them together, turn the knob, and the meter tracks it live:

```
> sample /pot | rangepct 0 4095 | bar
```

Three programs you already met, each doing its one job. The kernel does the plumbing between them, and `^c` stops the pipe.

## Swapping the ends

Because every stage speaks the same simple language, numbers in and numbers out, the pieces are interchangeable. Swap the stage at either end for one that speaks the same numbers, and the pipe keeps running.

Swap the sink. There's a `pixbar` utility, just another meter, except it draws on the WS2812 strip instead of the terminal. Point the same pot stream at it:

```
> sample /pot | rangepct 0 4095 | pixbar
```

Turn the knob and the strip fills, more LEDs and brighter as the value climbs. Nothing in the pipe changed but the last program.

Swap the source. The sonar reports distance in centimeters, 2 to 400 on the sim's slider. Write the bounds high-to-low, `rangepct 400 2`, so a far reading maps to 0% and a close one to 100%, and feed the same strip:

```
> sample /sonar | rangepct 400 2 | pixbar
```

Click the sonar in the simulation's diagram to open its distance slider and drag it. The closer the reading, the more lights up. Same sink, different source.

## Developer experience

This is a good spot to step aside: there's no hidden magic or heavy lifting tucked away in these little utilities. The same unified IO that lets you snap programs together at the prompt is what makes them easy to write.

Here is the entire `sample` program:

```c
#define DEFAULT_INTERVAL 5

static void usage(void) {
	printf("usage: sample /device [interval_ticks]\n");
}

void sample_main(int argc, char **argv) {
	if (argc < 2 || argc > 3) {
		usage();
		return;
	}

	int interval = DEFAULT_INTERVAL;
	if (argc == 3 && (parse_int(argv[2], &interval) < 0 || interval < 0)) {
		printf("sample: bad interval: %s\n", argv[2]);
		return;
	}

	int fd = open(argv[1]);
	if (fd < 0) {
		printf("sample: cannot open %s\n", argv[1]);
		return;
	}

	for (;;) {
		uint32_t raw = 0;
		if (getstat(fd, GS_RAW_LEVEL, &raw, sizeof raw) < 0) {
			printf("sample: getstat failed\n");
			break;
		}
		printf("%u\n", raw);
		sleep(interval);
	}
	close(fd);
}
```

It opens whatever path you name, asks it for a value over and over, and prints each one. It never learns what's on the other end, and doesn't have to. A device, a pipe, or the terminal all answer to the same handful of calls: `open`, `read`, `write`, `getstat`, `close` (`printf` calls `write`). The kernel does the rest.

You compose simple programs into a complex system, not a monolithic web of parts wired to know about each other. You can compose a subsystem test in isolation or just tinker with gadgets live at the command line, without as many change-compile-flash-test round trips.

## Ambient multitasking

You've been multitasking since the very first pipe. Each stage is its own program, running concurrently, each waking to handle the next number and passing it down the line. Multitasking isn't a sidecar you write plumbing for, the pipe is the plumbing.

What you couldn't do yet was keep working while a pipe ran. A foreground pipe owns the terminal until it ends, and the cli won't prompt you again until it does. Put `&` at the end of the line: the pipe runs in the background and your prompt comes right back.

```
> sample /sonar | rangepct 400 2 | pixbar &
```

The strip keeps tracking the sonar on its own, just out of the foreground. `ps` shows everything that's running:

```
> ps
pid  name              state   prio
0    -                 run     128
1    sample            sleep   128
2    cli               sleep   128
3    rangepct          iowait  128
4    pixbar            iowait  128
5    ps                run     128
```

Your prompt is free, so start a second subsystem, the pot driving the servo, and background it too:

```
> sample /pot | rangepct 0 4095 | servosink &
```

Now the sonar drives the strip while the knob drives the servo in the background, two independent chains, six programs running, none of them even aware the rest exist. Check `ps` again and all six are there.

## Make your own device

Every device you've used so far was described by a module baked into the boot image: `pot`, `sonar`, `strip`, `servo`. But a descriptor is just data, naming a file manager, a driver, and any device-specific information. Nothing about it has to ship in the image. You can author one at the command line.

There's a red LED wired to GPIO5 that the system knows nothing about, and we'd like to use it as a panic light when something gets too close. No descriptor names it, and `mdir -d` doesn't list it. Author one with `mkdev`: give it a name, the file manager and driver it needs, and how it's wired.

```
> mkdev redled gpio gpio pin=5
```

`mdir -d` lists it now:

```
> mdir -d
pot               descriptor
redled            descriptor
servo             descriptor
sonar             descriptor
strip             descriptor
```

Check your work with `minfo redled` to see the header you just authored.

Prove it the blunt way, write a byte straight to the pin. 1 lights it, 0 clears it:

```
> wr /redled 1
> wr /redled 0
```

Now give it a job. The demo has a `threshold` utility that watches a stream and emits 1 when a value is on the firing side of a limit, 0 when it isn't. Point it at the sonar with `below 50` so it fires when something comes within 50 cm, and redirect that on/off stream straight onto the pin with `>`, in the background:

```
> sample /sonar | threshold below 50 > /redled &
```

Drag the sonar's distance below 50 cm and the LED lights; pull it back above and the LED goes dark.

That same sonar is now driving two outputs at once: the strip you left running earlier and the LED you added just now. And you're still at the command line, free to test another subsystem, recombine some pieces, tinker around.
