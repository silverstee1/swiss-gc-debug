# Debug Swiss build — gcldr "Failed to Write" instrumentation

Goal: your RAM dumps show every DI transaction succeeding, yet Swiss reports
"Failed to Write!" / failed delete. The error code is therefore produced
host-side, somewhere between FatFs and the DI layer. This build prints the
exact failing call and its error code to your USB Gecko, alongside the
print_debug output you already capture.

Base commits (patches were generated against these):
  swiss-gc: f79afa3f5fbb9215111544a340f15b89be758d4b
  libogc2:  efce083023784df25724b8e45801e587acae2a1b

## Setup (same pattern as the cubiboot harness)

1. Fork https://github.com/emukidid/swiss-gc on GitHub.
2. Clone your fork, then:

       cd swiss-gc
       git checkout f79afa3f5fbb9215111544a340f15b89be758d4b -b gcldr-debug
       git apply /path/to/swiss-debug.patch
       cp /path/to/libogc2-debug.patch .                # workflow applies this to libogc2
       cp /path/to/continuous-integration-workflow.yml .github/workflows/
       git add -A && git commit -m "gcldr write debug instrumentation"
       git push -u origin gcldr-debug

3. GitHub → your fork → Actions → "Swiss debug build" run → download the
   `swiss-gc-debug` artifact. Use the DOL/ISO from it exactly as you run
   Swiss today.
4. Reproduce: boot debug Swiss with USB Gecko attached, copy the ISO from
   sda: to gcldr:, let it fail, then delete it. Save the full Gecko log.
   Grab a picoloader RAM dump afterwards too — pairing the two timelines is
   the point.

## What the prints mean

Swiss side (print_debug, same channel as your existing log):
  DBG writeFile: f_open/... res=N     FatFs FRESULT of the failing call
  DBG writeFile: f_write len/bw/res   f_write failed or wrote short
  DBG closeFile / deleteFile res=N    f_close / f_unlink FRESULT
  DBG disk_read/disk_write ... FAILED the dvm-cache-level sector op failed
  DBG disk_write: WRPRT! features=X   write REJECTED host-side: the disc
                                      lost FEATURE_MEDIUM_CANWRITE. No bus
                                      traffic happens for these -> matches
                                      your "clean dump, failed UI" signature
  DBG disk_ioctl: CTRL_SYNC FAILED    the f_sync/f_close cache flush failed

libogc2 side (SYS_Report, same Gecko):
  DVDDBG gcode Startup: inq/rel_date/pad1/pad3
      printed on every (re)mount. pad1 must be 'w' (0x77) and rel_date
      0x20196c64 or the drive is treated read-only from then on.
  DVDDBG WriteSectors: <reason>       which precondition rejected the write
                                      before anything touched the bus
  DVDDBG GcodeWrite ... ret=N         the DI write state machine failed
  DVDDBG write commit IMMBUF=X        drive returned nonzero immediate
  DVDDBG TIMEOUT currcmd=X            libogc's 10 s alarm fired (currcmd
                                      0x19 = gcode write, 0x1A = erase,
                                      0x11/0x12 = gcode read)
  DVDDBG GcodeRead ... FAILED         a gcode read failed (RMW path)

FatFs FRESULT values: 1=FR_DISK_ERR 2=FR_INT_ERR 3=FR_NOT_READY 4=FR_NO_FILE
5=FR_NO_PATH 7=FR_DENIED 9=FR_INVALID_OBJECT 10=FR_WRITE_PROTECTED (full
list in ff.h).

## Reading the result

- "WRPRT! features=..." or "WriteSectors: no CANWRITE" -> a remount's
  inquiry came back wrong and the disc went read-only. Compare against the
  "gcode Startup" lines to see which mount cleared it; then the fix is in
  picoloader's inquiry path under load.
- "DVDDBG TIMEOUT currcmd=19" -> a real 10 s stall on a write; pair with
  the picoloader dump's WR-START/WR-DONE gap to see if the card stalled.
- "f_write res=1 (FR_DISK_ERR)" right after a "disk_read ... FAILED" ->
  the failure is a read (FAT/cluster chain) inside the write path, not a
  write at all.
- "CTRL_SYNC flush FAILED" with all disk_writes OK -> the failure is inside
  the libdvm cache flush chain.
