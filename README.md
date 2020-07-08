# How to fix `mtrr_cleanup: can not find optimal value`

On some systems, Linux will log `mtrr_cleanup: can not find optimal value` and ask you to `please specify mtrr_gran_size/mtrr_chunk_size`.
This primarily happens on legacy BIOS booting systems with both CPU integrated and discrete graphics chipsets with muxless graphics switching ("Optimus").

## Editors note
Adam Helbing had a nice article written up on how to fix this, originally published on 2012-05-23 under CC-BY-NC-SA 3.0 license at http://my-fuzzy-logic.de/blog/index.php?/archives/41-Solving-linux-MTRR-problems.html

This article is still available in Internet archives:
* [Archive.today](http://archive.vn/4xibn)
* [Internet Archive Wayback Machine](http://web.archive.org/web/20190904223631/http://my-fuzzy-logic.de/blog/index.php?/archives/41-Solving-linux-MTRR-problems.html)

To preserve this how-to and make it re-usable, Adam kindly re-licensed it to me - Tom Reynolds (@tomreyn) - (and by me re-publishing it here, to you, too) under [Creative Commons Attribution Share-Alike 4.0 International (CC-BY-SA 4.0)](COPYING) license ([at CreativeCommons.org](https://creativecommons.org/licenses/by/4.0/)). Proof of relicensing is available in this Git repository at [relicensing_proof/](relicensing_proof/) which also contains a copy of the original article. Having access to the re-licensed copy available here, you are able and welcome to create derivative works (providing credit and not claiming endorsement on your derivative work - see the license deed for details).

A copy of the original article, converted to Markdown, with only minimal changes applied (formatting, Wikipedia link, ortography) is available below. Since the comments published with the original article were useful to me, but they were contributions under the articles' original CC-BY-NC-SA 3.0 license, I have paraphrased them instead.

# Solving linux MTRR problems
> by Adam Helbing
> Originally published on 2012-05-23

If you find one or several of the following lines in your kernel message buffer, your system suffers from memory management problems:

```
mtrr_cleanup: can not find optimal value
please specify mtrr_gran_size/mtrr_chunk_size
MTRR allocation failed. Graphics performance may suffer.
```


Keep on reading for some background info and a few steps to solve that problem.

## What is a MTRR? 

[Memory Type Range Registers](https://en.wikipedia.org/wiki/Memory_type_range_register) are basically a table which tells the system how to cache which ranges of installed memory. It is set up by bios initially, but can be altered anytime by the OS if needed. Whenever you change the amount of memory installed or flash a new bios, a new mtrr table is computed. Here is a sane mtrr table from my Lenovo T400 notebook with 6 GiB of ram:

```
fk ~ $ cat /proc/mtrr 
reg00: base=0x000000000 (    0MB), size= 2048MB, count=1: write-back
reg01: base=0x080000000 ( 2048MB), size= 1024MB, count=1: write-back
reg02: base=0x0bc000000 ( 3008MB), size=   64MB, count=1: uncachable
reg03: base=0x100000000 ( 4096MB), size= 2048MB, count=1: write-back
reg04: base=0x180000000 ( 6144MB), size= 1024MB, count=1: write-back
reg05: base=0x1bc000000 ( 7104MB), size=   64MB, count=1: uncachable
reg06: base=0x0d0000000 ( 3328MB), size=  256MB, count=1: write-combining
```

As you see, my Laptop has 7 registers available, there are some small uncachable areas defined (I don't know why, some bios stuff I suppose..), but the big chunks are set to write back. The last is a special one: It is the Intel onboard graphic cards memory, which supports write-combining for best performance.

## So, what can go wrong?

Propably a lot. In my case, I found the following deeply disturbing lines inside the kernel buffer:

```
mtrr: no more MTRRs available
[drm] MTRR allocation failed.  Graphics performance may suffer.
```

What happened? The drm module tried to grab a free mtrr to set up a write-combined cache area for the graphic cards memory, but all 7 register were occupied by some other cache-declarations already, so that failed. However, the machine works fine without, 3D and compositing work, as do games and everything, just not as fast as they could.

## What can we do about it?

Luckily, the kernel has some options to control mtrr setup:

```
fk ~ $ gunzip < /proc/config.gz  | grep -i MTRR_SANITIZER
CONFIG_MTRR_SANITIZER=y
CONFIG_MTRR_SANITIZER_ENABLE_DEFAULT=0
CONFIG_MTRR_SANITIZER_SPARE_REG_NR_DEFAULT=1
```

Entry MTRR_SANITIZER! The first option switches it on or off, if its off in your kernel, you need to rebuild it with sanitizer support enabled. The following lines configure the sanitizer itself, the first one switches it on or off by default, the next one tells the sanitizer, whether he shall left a number of registers empty on boot for later use. Bingo! We need one register left untouched for drm to grab it. One can enable the sanitizer by default, set the spare reg to 1 and reboot, but as its unusual to build kernel for yourself these times, there are 2 kernel parameter which overload the kernel config. just add

```
enable_mtrr_cleanup mtrr_spare_reg_nr=1
```


to your grub, syslinux, lilo, whatever kernel line and reboot to sanitize your mtrrs. 

## That's it?

Propably yes. Just check the message buffer whether the "mtrr  allocation failed" message is gone and check your /proc/mtrr whether there is a write-combined area defined for the graphics. In my case, it still was not. Instead, I found these new lines:

mtrr_cleanup: can not find optimal value
please specify mtrr_gran_size/mtrr_chunk_size


The sanitizer was not able to find an optimal mtrr setup, but he kindly printed a long list with possible setups for me into the buffer, here is just the relevant snippet, as that list is really long:

```
gran_size: 16M chunk_size: 128M num_reg: 7 lose cover RAM: 14M
gran_size: 16M chunk_size: 256M num_reg: 7 lose cover RAM: 14M
gran_size: 16M chunk_size: 512M num_reg: 7 lose cover RAM: 14M
gran_size: 16M chunk_size: 1G   num_reg: 7 lose cover RAM: 14M
gran_size: 16M chunk_size: 2G   num_reg: 7 lose cover RAM: 14M
gran_size: 32M chunk_size: 32M  num_reg: 7 lose cover RAM: 478M
gran_size: 32M chunk_size: 64M  num_reg: 7 lose cover RAM: 478M
gran_size: 32M chunk_size: 128M num_reg: 6 lose cover RAM: 30M
gran_size: 32M chunk_size: 256M num_reg: 6 lose cover RAM: 30M
gran_size: 32M chunk_size: 512M num_reg: 6 lose cover RAM: 30M
```

The problem is, poor sanitizer found no configuration which meets my requirements of using 6 mtrrs only, and still cover all memory. So he decided to let me choose a tradeoff config for myself. Look at the table, gran_size and chunk_size are the config parameters, num_reg shows how many mtrrs are needed for that config and 'lose cover ram' tells us how many ram we lose when this config is chosen. For my setup, num_reg has to be 6 or less, and I want to lose as little ram as possible, so I chose the first line which loses 30M only. 

Finally, appending  

```
enable_mtrr_cleanup mtrr_spare_reg_nr=1 mtrr_gran_size=32M mtrr_chunk_size=128M
```


to the boot parameters made my box happy again :-)
Now 24MiB of ram are gone, but my 3d workloads, which are basically webgl content and the hidden flight simulator inside google earth, run both at significant better framerates.

## Commentary

Q: In the examples given in this article, you state that 6 MTRRs are required. How do you calculate this?
A: In this example, the `num_reg` line logged by Linux provides the MTRRs required: `gran_size: 32M chunk_size: 128M num_reg: 6 lose cover RAM: 30M`

Q: But why does it have to be exactly 6 MTRRs or less, how did you choose this very line from the many possible combinations Linux suggests there? Here is what it looks like for me:
```
$ cat /proc/mtrr
reg00: base=0x23c000000 ( 9152MB), size= 64MB, count=1: uncachable
reg01: base=0x0be000000 ( 3040MB), size= 32MB, count=1: uncachable
reg02: base=0x000000000 ( 0MB), size= 2048MB, count=1: write-back
reg03: base=0x080000000 ( 2048MB), size= 1024MB, count=1: write-back
reg04: base=0x100000000 ( 4096MB), size= 4096MB, count=1: write-back
reg05: base=0x200000000 ( 8192MB), size= 1024MB, count=1: write-back
reg06: base=0x0bde00000 ( 3038MB), size= 2MB, count=1: uncachable
```
A: In my experience, on computers with an Intel CPU with integrated graphics, when you examine how many MTRRs to set, one register needs to be left available for the Intel DRM (direct rendering module) driver (i915/i965). It will use it as write-back cache for onboard graphics memory. 
Your CPU, just like the one in the article, supports 7 MTRRS (reg00 to reg06). So if you leave one for the integrated GPU, you can go with a 6 MTRR configuration, and `mtrr_spare_reg_nr=1`. While running with these settings, check the kernel log (dmesg / journalctl -k) to confirm whether the Intel DRM uses the leftover MTRR.
