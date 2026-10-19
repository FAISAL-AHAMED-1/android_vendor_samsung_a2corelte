H-VENDOR-A2CORELTE v1.6
Target: Samsung SM-A260G / a2corelte / Exynos 7870
Target userspace: Android 10 ARM32, A-only GSI

This package is built from the A260G stock vendor baseline used by v0.2 and
uses an A260G kernel built from Samsung's open-source 3.18.91 source.

Important kernel compatibility fix:
  CONFIG_ANDROID_BINDER_IPC_32BIT=y
The A260G uses a 64-bit kernel with 32-bit Android userspace. The Binder UAPI
uses protocol 7 when BINDER_IPC_32BIT is enabled and protocol 8 otherwise.

v1.1 - black screen fix (design):
  v1.0 booted (adb worked) but stayed on a black screen. Root cause: the
  vendor's Mali GLES/EGL blob only ships as one monolithic file,
  lib/egl/libGLES_mali.so, with nothing declaring the "mali" driver tag
  or providing the libEGL_mali.so / libGLESv1_CM_mali.so / libGLESv2_mali.so
  names the AOSP EGL loader looks for. Fixed via ro.hardware.egl=mali,
  vendor/etc/egl.cfg, and the 3 driver-name symlinks.

v1.2 - packaging fix:
  The v1.1 fix wasn't reaching the device because a separately-built raw
  vendor.img was out of sync with this zip. The 3 symlinks are now stored
  as real symlinks inside the zip (`zip -y`) instead of created at install
  time via the updater-script's "symlink()" command, removing a dependency
  on the custom update-binary supporting that opcode.

v1.3 - still black screen after v1.2, SELinux enforcing, no adb root:
  With the EGL wiring now confirmed correct, a still-black screen with
  SELinux enforcing points at a second, independent problem: this vendor's
  sepolicy (nonplat_sepolicy.cil / nonplat_file_contexts) was written
  against an older platform policy and may not fully cover what this GSI's
  newer platform sepolicy expects from the graphics HAL chain (SurfaceFlinger
  / hwcomposer / gralloc opening the Mali device, ion, etc). An enforcing
  denial here produces the exact same symptom as v1.0: everything else
  boots, adb stays alive, the screen never lights up.
  Added vendor/etc/init/00-permissive.rc, which forces SELinux to
  permissive at early-init:
      on early-init
          write /sys/fs/selinux/enforce 0
  This is a DIAGNOSTIC WORKAROUND, not a real fix: it tells you whether
  sepolicy is the blocker, but it disables enforcement device-wide. If the
  screen comes up after flashing v1.3, the next step is to capture the
  actual avc denials (now-permissive means they get logged as warnings
  instead of blocking) via `adb shell logcat -d -b all | grep -i avc` or
  `adb bugreport` (bugreport does not require adb root on most builds),
  and write narrow allow rules for those specific denials instead of
  staying permissive long-term.
  If the screen still does NOT come up after v1.3, SELinux was not the
  cause - grab the same logs (logcat + bugreport) and share them; without
  an actual crash/denial trace at this point, further changes here would
  just be guessing.

v1.4 - black screen persists after v1.3, root cause found (HAL interface
gap), 00-permissive.rc left in place:
  With SELinux permissive (v1.3), the screen still didn't come up. Logs
  showed hwservicemanager repeatedly trying to lazy-start
  android.hardware.configstore@1.0::ISurfaceFlingerConfigs for
  SurfaceFlinger and failing. Root cause: this vendor baseline is an
  Android 8.1-era package, and none of its HAL *.rc files under
  vendor/etc/init declare "interface <fqname> <instance>" lines inside
  the service{} block. Android 8.1's init started every "class hal"
  service unconditionally at boot, so the interface line (which only
  matters for hwservicemanager's on-demand ctl.interface_start lookup,
  introduced for lazy HAL support) was never required. Android 10 GSIs
  rely on that mapping to find and start a HAL on demand; without it,
  hwservicemanager has no service to start and configstore (and
  potentially other passthrough-turned-binderized HALs) never comes up,
  which stalls SurfaceFlinger before it can post the first frame -
  the black screen.
  Added an "interface" line (or lines, where a binary registers more
  than one HIDL interface/instance) to every HAL service.rc under
  vendor/etc/init: audio, bluetooth, camera.provider, configstore, drm
  (base + widevine), gatekeeper, graphics.allocator, graphics.composer,
  health, keymaster, light, media.omx, memtrack, power, sensors,
  thermal, vibrator, wifi, and the vendor.samsung*/vendor.samsung_slsi*
  HALs (gnss, miscpower, ExynosHWCServiceTW). Each interface/instance
  pair was taken from the actual registerAsService() calls compiled
  into that binary (verified via its symbol table), not assumed from
  stock AOSP - this device's keymaster and gnss/miscpower/HWC services
  register vendor-custom HIDL interfaces (e.g.
  vendor.samsung.security.skeymaster@3.0::ISKeymasterDevice), and the
  camera provider and base drm HAL register non-default instance names
  ("legacy/0", "clearkey") that had to match exactly.
  00-permissive.rc has NOT been removed in this release. This fix
  addresses a separate, independent failure path from the sepolicy gap
  v1.3 was diagnosing; leaving SELinux permissive keeps that variable
  out of the picture while this fix is verified on real hardware. Once
  a v1.4 boot is confirmed past the black screen, re-enable enforcing
  mode and capture avc denials as v1.3 described, then remove
  00-permissive.rc once nonplat sepolicy covers them.
  NOTE: this fix only touches the flashable zip (device/a2corelte/vendor
  as extracted by the updater-script). If you are also flashing a raw
  vendor.img directly (fastboot/dd) rather than this zip, that image
  needs the same rc changes applied separately with its SELinux file
  contexts preserved - ask if you'd like that done too.

v1.5 - black screen persists after v1.4, second HAL-manifest bug found via
  static binary analysis (no device logs available yet - this is the
  leading hypothesis, not a confirmed fix):
  Checked registerAsService() symbols compiled into every vendor/bin/hw/*
  binary against both the init.rc "interface" lines and etc/vintf/manifest.xml.
  All graphics-path binaries (composer@2.1, allocator@2.0, configstore@1.0)
  correctly register exactly what v1.4 declared - that fix stands.
  Two HALs do not: android.hardware.keymaster@3.0-service only calls
  vendor::samsung::security::skeymaster::V3_0::ISKeymasterDevice::
  registerAsService() - it never registers the standard
  android.hardware.keymaster@3.0::IKeymasterDevice that Keystore on a
  generic GSI actually asks for. vendor.samsung.hardware.gnss@1.0-service
  has the identical problem for android.hardware.gnss@1.0::IGnss. The
  manifest declared both the (unreachable) standard hwbinder HAL and the
  correct vendor-specific one side by side, which doesn't help - nothing
  ever answers the standard name, so any client requesting it blocks/fails.
  This is not a typo: it's architectural. The real Samsung framework had a
  proprietary keystore/keymaster shim in system.img that knew to call
  ISKeymasterDevice directly; a generic AOSP GSI's Keystore does not know
  that name and will only ever ask for the standard IKeymasterDevice HAL.
  A missing hardware-backed Keymaster HAL is a well-known cause of Android
  10 boot failure (keystore/system_server crash-looping), which can present
  visually indistinguishable from a graphics-stack black screen.
  The fix does NOT require building a shim: this vendor already ships
  vendor/lib/hw/android.hardware.keymaster@3.0-impl.so and
  vendor/lib/hw/android.hardware.gnss@1.0-impl.so - real, standard,
  passthrough-mode implementations (HIDL_FETCH_IKeymasterDevice /
  HIDL_FETCH_IGnss are both present; the keymaster one wraps a legacy
  keymaster0/1 module via AOSP's own SoftKeymasterDevice). These were
  simply never wired up. etc/vintf/manifest.xml now declares both HALs as
  `passthrough` (arch 32) instead of hwbinder, so Keystore/the GNSS HAL
  client will dlopen these .so files directly instead of waiting on a
  hwbinder service that was never going to answer.
  No init.rc changes were needed for this fix (passthrough HALs are
  dlopen'd by the calling process, not run as standalone services); the
  existing keymaster-3-0 / sec_gnss_service binderized services were left
  running as-is since they're otherwise harmless.
  00-permissive.rc is still in place from v1.3/v1.4 and has not been
  touched by this change.
  STATUS: unverified on hardware. If the screen still doesn't come up
  after this, capture `adb shell logcat -d -b all` and `adb bugreport`
  as v1.3 described - at that point static analysis has diminishing
  returns and a real boot trace is needed to keep narrowing this down.

This package does not repartition SYSTEM and does not include a PIT.
It is experimental and hardware testing is still required.


v1.6 - black screen root cause found from dmesg_su9/logcat_su9 (v1.5 log):
  * surfaceflinger aborted every ~5 s: "failed to get hwcomposer service" /
    "Could not load android.hardware.graphics.composer@2.1-impl.so from sphal
    namespace: library libfmq.so not found".
    Cause: etc/vintf/manifest.xml declared graphics.composer as
    <transport arch="32">passthrough</transport>, so SurfaceFlinger ignored the
    running binderized service (hwcomposer-2-1) and tried to dlopen the -impl
    inside its own process, where libfmq.so (VNDK-core, not VNDK-SP) is not
    visible. Fixed: composer is now hwbinder. (vendor/manifest.xml already said
    hwbinder but Android 10 reads etc/vintf first; both are now identical.)
  * Added missing manifest entries for memtrack@1.0 and thermal@1.0
    (hwservicemanager: "Cannot find entry ... IMemtrack/default").
  * 00-permissive.rc used "on early-init", which vendor rc files can never hook
    (parsed after mount_all); SELinux was still enforcing in the log. Now
    "on post-fs" + "on post-fs-data".
  * restorecon /dev/event-log-tags (zygote/system_server read denials).

v1.7 - vndbinder fix (from v1.6 logs):
  * Stock /vendor/bin/vndservicemanager speaks the 64-bit binder ABI (protocol 8,
    ioctl 0xc0306201). The a2corelte kernel is CONFIG_ANDROID_BINDER_IPC_32BIT=y
    (protocol 7), so it logged "failed to open binder driver /dev/vndbinder" and slept
    forever. Nobody owned /dev/vndbinder, so the composer and ExynosHWCServiceTW looped
    on dead PING transactions and IComposer/default never registered.
  * Replaced by a small static ARM32 vndservicemanager (32-bit binder ABI), built by
    GitHub Actions from vndsm.c and self-tested under qemu. Same path, same SELinux label.
  * Removed 00-permissive.rc: init is not allowed to setenforce, it never had any effect.

v1.8 - stuck on boot animation (from v1.7 logs):
  * v1.7 fixed vndbinder: composer registers, boot animation shows.
  * dmesg: "Failed to mount an un-encryptable or wiped partition on .../CACHE ...
    Invalid ext4 magic", then "fs_mgr_mount_all returned an error" / "unexpected error 255".
    mount_all aborts, ro.crypto.state is never set, the "zygote-start" trigger never
    fires, so zygote/system_server never start.  /data (f2fs) itself mounted fine.
  * Fix: boot.img ramdisk fstab.samsungexynos7870: /cache entry gets ",nofail" (same flag
    /cpefs already had). Only that one file changed; kernel and dtb untouched.
  * Alternative that needs no reflash: TWRP -> Wipe -> Advanced -> Cache -> format ext4.

v1.9 - zygote loops, system_server dies (from v1.8 logs):
  * zygote now starts (cache/nofail fix worked) but system_server aborts in
    TrustManagerService: IKeystoreService is null. keystore aborts every 5 s:
    'Failed to get service for android.hardware.keymaster@3.0::IKeymasterDevice/default'
    then 'Could not load ...keymaster@3.0-impl.so from sphal namespace: libsoftkeymasterdevice.so
    not found'.
  * Same bug as the composer in v1.6: manifest declared keymaster@3.0 as
    <transport arch="32">passthrough</transport>, so keystore tried to dlopen the -impl in its own
    process instead of talking to the running binderized keymaster-3-0 service. Now hwbinder.
  * Only remaining passthrough HALs: graphics.mapper (correct, in-process) and android.hardware.gnss.
  * Known, not boot-blocking: android.hardware.media.omx@1.0-service fails to link
    (MediaCodecsXmlParser::defaultSearchDirs missing in the Q system) -> media codecs unavailable.

v2.1 - built on H-VENDOR-A2CORELTE-v1_9-mediafix (Q libstagefright_xmlparser that exports
MediaCodecsXmlParser::defaultSearchDirs, so android.hardware.media.omx@1.0-service links and
audio/ringtones work). omx is left enabled and declared.
  * WiFi: wifi HAL 'Failed to open socket to set up/down state (Permission denied)'.
    Kernel CONFIG_ANDROID_PARANOID_NETWORK=y -> HAL gets groups inet net_raw net_admin and file
    capabilities NET_ADMIN|NET_RAW.
  * Bluetooth: /dev/ttySAC0 (HCI UART) was root-only; chown bluetooth + 0660 so libbt-vendor can
    open it (com.android.bluetooth aborted in startup_timer_expired).
  * Brightness: lights HAL 'write_int failed to open -1'; stock init.rc only chowns
    /sys/class/leds/lcd-backlight, this panel is /sys/class/backlight/panel -> chown system.

v2.3 - built on H-VENDOR-A2CORELTE-v2_1-brightnessfix (sepolicy: hal_light_default may use
sysfs_lcd_writable; precompiled_sepolicy removed so the cil change is actually used) + Bluetooth fix:
  * Bluetooth HAL: 'bt_upio ... write(/sys/class/rfkill/rfkill0/state) failed: Operation not permitted'
    and 'unable to set SCHED_FIFO'. Needs CAP_NET_ADMIN + CAP_SYS_NICE; ramdisk init cannot grant rc
    'capabilities' (3.18 kernel, no ambient caps) -> set as file capabilities 0x801000 on
    android.hardware.bluetooth@1.0-service (same mechanism that fixed the wifi HAL).
  * MTP (file transfer) still needs a kernel change; the one early-boot netd abort in the logs was
    transient (netd is running).

v2.4 - Bluetooth still failing after v2.3 (logcat_bt2/dmesg_bt2):
  * Same two errors with the v2.3 file capabilities in place:
    'bt_upio: write(/sys/class/rfkill/rfkill0/state) failed: Operation not permitted' and
    'unable to set SCHED_FIFO ... Operation not permitted'. No SELinux capability denial is logged and
    vendor policy already allows hal_bluetooth capability {net_admin sys_nice} + sysfs_bluetooth_writable,
    so the process simply never receives the capabilities (init here is the old ramdisk init, kernel 3.18).
  * Fix: bluetooth-1-0 service now 'user root' (full capability set), group bluetooth net_bt_admin net_bt inet.
  * Next if it still times out: look at bt_hwcfg lines after 'HCI reset' (firmware/baud), and the missing
    /etc/bluetooth/bt_vendor.conf.

v2.5 - Bluetooth: root cause found in the v2.4 boot (bt_hal_log / logcat_all):
  * v2.4 worked: HAL runs as root, rfkill power-on OK, chip answers HCI reset, "Chipset BCM43430B0".
  * Then 'vendor lib preload failed to open [/vendor/firmware/bcm43436B0_V0096.0154.hcd]' ->
    chip runs without its patchram -> the Q stack aborts in hci_packet_parser.cc(335)
    (short response parameter) -> "Bluetooth keeps stopping".
  * The .hcd open was an SELinux denial: avc denied { read } hal_bluetooth_default -> tcontext=unlabeled.
    updater-script labelled /vendor/firmware as u:object_r:vendor_firmware_file:s0, a type that does not
    exist in this vendor policy (nor the 27.0 mapping), so every firmware file was 'unlabeled'. Same denial
    hit mfc_fw.bin (ueventd, mediacodec) and fimc_is_lib.bin (camera HAL).
  * Fix: /vendor/firmware labelled vendor_file (exists, readable by the vendor domains) + explicit cil allows
    for hal_bluetooth_default, hal_camera_default, mediacodec, ueventd (read vendor_file files/dirs).

v2.7 - MTP file transfer (kernel change), built on H-VENDOR-A2CORELTE-v2_5-videofix2-mtpfix:
  * boot.img kernel rebuilt with CONFIG_USB_ANDROID_SAMSUNG_MTP disabled. Samsung's f_mtp_samsung.c does
    not implement MTP_SEND_FILE / MTP_RECEIVE_FILE / MTP_SEND_FILE_WITH_HEADER, so Android 10 MtpServer got
    'Inappropriate ioctl for device' (response 0x2002) on every copy. android.c now includes the standard
    f_mtp.c, which has those ioctls and registers /dev/mtp_usb itself (ueventd.rc already sets 0660 root mtp).
  * Only config difference vs the previous kernel: CONFIG_USB_ANDROID_SAMSUNG_MTP (verified from IKCONFIG).
    Ramdisk, dtb and header are byte-identical to the previous boot.img; only kernel + sha id changed.
  * Removed 00-mtp-fix.rc (the /dev/usb_mtp_gadget -> /dev/mtp_usb symlink is obsolete).

v2.8 - cleanup (no functional change intended), built on v2.7. Removed only files proven unused:
  * 150 toybox applet copies in /vendor/bin (byte-identical to toybox_vendor, never referenced by any rc or
    binary; init's PATH is /sbin:/system/sbin:/system/bin:... so the GSI's own toybox is used anyway)
    plus bin/grep, bin/egrep, bin/fgrep (same small binary, unreferenced) and bin/vndservice (debug CLI).
    toybox_vendor and vendor sh are kept.
  * lib/libarcsoft_beautyshot_preview.so and lib/libmpbase_preview.so: byte-identical to the non-preview
    libraries with the same SONAME; consumers link libarcsoft_beautyshot.so / libmpbase.so (kept).
  * lib/libreference-ril.so (AOSP reference RIL; build.prop uses libsec-ril).
  * etc/carrier/*: ten copies of a CDMA permission xml for US carriers (Samsung CSC layout, not read by the GSI).
  * partition/README-3.5GB-SYSTEM.md (documentation only).
  Kept on purpose although nothing in the vendor names them: libGLES_mali.so (EGL loader), libstagefrighthw.so
  (OMX plugin loaded by the system), mediadrm/mediacas plugins, audio effect libs, clearkey, local_time HAL,
  NOTICE.xml.gz (licence text), firmware, tee, wifi firmware variants.
  * updater-script: removed two dead set_metadata lines pointing at /vendor/lib/vndk-sp-29 (folder never existed in the package).

v2.9 - Telegram "unable to play this video" fix, built on v2.8:
  * Cause (from dmesg during a failed playback): zswap cannot allocate its per-CPU compressor transform, so
    bringing CPU4 online fails (_cpu_up: attempt to bring up CPU 4 failed). Samsung's cpuhotplug governor
    (CONFIG_EXYNOS_HOTPLUG_GOVERNOR) retries every ~100 ms; the retry storm starves the codec buffer queue,
    CCodec reports "previous call to queue exceeded timeout" and playback dies. Not a codec/vendor-config bug.
  * Disabling the governor (cpuhotplug/enable = 0) is impossible: it first tries to online all cores and fails
    at CPU4 for the same reason. Instead 01-hvendor-fixes.rc writes max_online_cpu = 4 at post-fs and boot,
    so the governor never tries CPU4-7. Verified live: failures stopped after the cap, including after a
    screen blank/unblank. Side effect: the device runs on 4 cores.

v3.0 - all 8 cores online, built on v2.9. Kernel rebuilt from FAISAL-AHAMED-1/android_kernel_samsung_a2corelte
  (stock 3.18.91) with CONFIG_ZSWAP off.
  * Cause: zswap's CPU_UP_PREPARE handler (zswap_cpu_notifier) failed for CPU4 ("zswap: can't allocate compressor
    transform"), which aborted bringing CPU4-7 online; the Samsung cpuhotplug governor then retried every ~100 ms
    and starved the media codec queue (Telegram "unable to play this video"). v2.9 capped the cores at 4 to stop that.
  * v3.0 removes zswap, so the cap is raised to 8 (max_online_cpu = 8 at post-fs and boot in 01-hvendor-fixes.rc).
    Verified: under load online goes 0-3 -> 0-7 with 0 "Failed to hotplug" messages; MTP copy still works.
  * Kernel diff vs v2.8/v2.9 (from IKCONFIG): ZSWAP and the Samsung memory options that depend on it (VNSWAP etc.)
    are off, CRYPTO_FIPS is off, KernelSU (CONFIG_KSU) is ON. CONFIG_USB_ANDROID_SAMSUNG_MTP stays off (MTP fix kept).
  * Only the kernel, header size and header sha id changed in boot.img; ramdisk and dtb are byte-identical.
  * Side effect: no compressed memory any more (ZRAM is not enabled), so expect somewhat more background app kills
    on this 885 MB device; all 8 cores can also mean more heat/battery use under load.

v3.1 - touch driver fix (display hang), built on v3.0. Do not use v3.0: its kernel can hang the display.
  * Cause: synaptics_rmi4_irq_enable() (synaptics_td4x00/synaptics_i2c_rmi.c) locked rmi4_irq_enable_mutex but
    three early exits used "return" instead of "goto exit", leaving the mutex locked forever. With double-tap-to-wake
    the touch IC is left running while the display (in-cell TD4101) power-cycles, so an I2C read in irq_enable(true)
    can fail; the leak then blocks the next screen-off inside the fb notifier: screen stays black while Android
    reports Awake/ON, until reboot. Fix: the three returns now jump to exit so the mutex is always released.
  * Same kernel config as v3.0 (zswap off, KernelSU on, Samsung MTP off); boot.img differs only in the kernel.
  * Double-tap-to-wake is experimental software detection and is ENABLED by default
    (/sys/module/synaptics_i2c_rmi/parameters/dt2w, 0=off). dt2w_key: 143=KEY_WAKEUP (default), 116=KEY_POWER.
    Whether the touch IC keeps scanning while the in-cell panel sleeps is not yet confirmed.

v3.2-test - dt2w experiment: keep the TD4101 in sleep-out while double-tap-to-wake is active (dt2w is ON by default).
  * td4101_exit() (td4101_a2corelte_lcd_ctrl.c) skips the final SLEEP_IN when the touch driver reports dt2w active
    (synaptics_dt2w_keep_awake, weak extern); display-off and the backlight-IC suspend still run, so the screen stays
    black. td4101_init() on wake is unchanged. Hypothesis: SLEEP_IN is what silences the in-cell touch half
    (v3.1: "I2C read over retry limit" every ~15 s while the screen was off).
  * Success looks like: dmesg shows "dt2w active, staying in sleep-out", no I2C retry/incell_reset_work loop while
    the screen is off, and "double tap detected" when double-tapping the black screen.
  * Standby battery use is higher while dt2w is on: measure before keeping it. Off switch:
    echo 0 > /sys/module/synaptics_i2c_rmi/parameters/dt2w
  * Kernel config unchanged from v3.1; the 8-core, zswap-off and mutex fixes are included.

v3.3-test - dt2w experiment 2: keep the panel power rails on while double-tap-to-wake is active (dt2w is ON by default).
  * v3.2 log: td4101_exit skipped SLEEP_IN ("staying in sleep-out") but the touch IC still stopped answering I2C ~5 s after
    screen-off. The device tree shows why: the touch node has no regulators of its own; it runs off the display supply.
    dsim_disable() runs dsim_set_panel_power_disable, which drops gpio_blic_on, gpio_panel_enn/enp, gpio_lcd_rst (reset low)
    and disables the LCD_1P8 regulator.
  * dsim_drv.c: dsim_set_panel_power(0) returns early while synaptics_dt2w_keep_awake is set (weak extern, same flag as the
    panel driver); on wake dsim_set_panel_power_early() skips the LCD_1P8 enable once so the regulator count stays balanced.
  * Success looks like: dmesg "dt2w active, panel power kept on", no I2C retry loop while the screen is off, and
    "double tap detected" on a double tap.
  * Standby battery use is the highest of the variants: panel bias, backlight IC and reset stay powered while the screen
    is black. This is a feasibility test; if it works a lighter variant (keep only LCD_1P8 + reset) is the next step.
    Off switch: echo 0 > /sys/module/synaptics_i2c_rmi/parameters/dt2w
  * Kernel config unchanged from v3.2; boot.img differs only in the kernel (header sha id updated).

v3.4 - double-tap-to-wake removed. Built on v3.3-test; use this instead of v3.1 to v3.3-test.
  * Why: with the display off the TD4101 touch half did not raise a single interrupt, even with the panel power rails kept
    on (v3.3-test: no I2C errors, 0 touch IRQs over ~10 taps). The touch driver has no gesture / low-power wake mode and the
    touch node has no supply of its own (it runs off LCD_1P8 and the panel reset line), so dt2w is not possible on this phone.
  * Removed: all dt2w code from synaptics_i2c_rmi.c/.h (detection, wake IRQ handling, KEY_WAKEUP/KEY_POWER capabilities,
    input_open/close hooks), the SLEEP_IN skip in td4101_a2corelte_lcd_ctrl.c, the panel power hold in dsim_drv.c, and
    dt2w_td4101.patch. The screen-off path is the stock one again (touch IC powered down, panel power cut).
  * Kept: KernelSU, zswap off (8 cores online), CRYPTO_FIPS off, Samsung MTP off, binder 32-bit, and the irq_enable mutex
    leak fix from v3.1.
  * Kernel config is identical to v3.3-test (IKCONFIG compared); boot.img differs only in the kernel (header sha id updated);
    ramdisk, dtb and vendor files unchanged.

v3.5 - ZRAM compressed swap (multitasking). Built on v3.4; use this instead of v3.4.
  * Why: since v3.0 (zswap off, so the 8 cores work) the phone had no swap at all: SwapTotal 0, ~37 MB free and ~27 MB of
    cached apps, Android reporting memory status "critical". Samsung's sswap helper only knows vnswap0, which depends on ZSWAP
    and is gone, so it silently did nothing.
  * Kernel: CONFIG_ZRAM=y and CONFIG_ZRAM_LZ4_COMPRESS=y (defconfig). ZRAM's lz4 path calls the lz4 library directly and does
    not use the crypto framework that zswap used. Samsung's zsmalloc has an extra ops argument on zs_create_pool(), so
    zram_drv.c passes NULL (ZRAM never uses the evict handler). Nothing else in the kernel config changed from v3.4
    (IKCONFIG compared): KernelSU on, NR_CPUS=8, CRYPTO_FIPS off, Samsung MTP off.
  * Ramdisk: the sswap "swapon" service and "swappiness 160" are removed (the kernel caps swappiness at 100 without the
    ZSWAP-only INCREASE_MAXIMUM_SWAPPINESS option). On boot init writes lz4 to /sys/block/zram0/comp_algorithm, sets swappiness 100
    and page-cluster 0, then runs swapon_all, which reads a new fstab line: /dev/block/zram0, zramsize=463470592 (442 MB, about
    half of the 885 MB RAM), max_comp_streams=4. The sswap binary stays in /sbin, unused.
  * Not done: Samsung's other memory tuning options (INCREASE_MAXIMUM_SWAPPINESS, FIX_INACTIVE_RATIO, TIGHT_PGDAT_BALANCE,
    DIRECT_RECLAIM_FILE_PAGES_ONLY) all depend on ZSWAP and remain off. Measure first, then decide.
  * Check after flashing: cat /sys/devices/system/cpu/online -> 0-7; cat /proc/swaps -> /dev/block/zram0 ~452600 KB;
    cat /sys/block/zram0/comp_algorithm -> [lz4]; cat /proc/sys/vm/swappiness -> 100.

v3.6 - ZRAM size 442 MB -> 640 MB (ramdisk fstab only; kernel identical to v3.5).
  * Measured on v3.5 after normal use: 340 MB stored in ZRAM compressed to 119 MB (ratio 2.85:1), costing 155 MB of RAM with
    allocator overhead; swap was 78% full (SwapFree 100 MB of 452 MB), so apps were close to being killed again.
  * zramsize=671088640. If full it holds about 640 MB of pages for roughly 290 MB of RAM. If RAM feels tight or the phone
    stutters under load, v3.5 (442 MB) is the safer size.

v3.7 - SD card shows 0 B after copying (sdcardfs stale size). Ramdisk identical to v3.6 (640 MB ZRAM).
  * Symptom: a file copied to the SD card showed 0 B in every app while the real size on the card was correct. Renaming the file,
    ejecting the card or dropping the dentry/inode caches made the real size appear.
  * Cause: in the Oreo sdcardfs variant this kernel builds (fs/sdcardfs/o), sdcardfs_getattr() refreshed mode, owner and times
    from the lower FAT inode but reported the size from sdcardfs's own cached inode, which was only updated by writes through
    the same mount view (default/read/write/full each have their own). A file written through another view kept i_size 0.
  * Fix (kernel only): fsstack_copy_inode_size() in sdcardfs_getattr() (inode.c) and in sdcardfs_open() (file.c).
    Kernel config is identical to v3.6.

v1.2 (release name; internal build = v3.8 lineage) - kernel named H-KERNEL V-1, new updater-script look.
  * Release naming: this package is published as H-VENDOR-A2CORELTE v1.2. The earlier "v1.2 - packaging fix" entry above is
    from the original v1.x lineage and is a different release.
  * Kernel: CONFIG_LOCALVERSION="-H-KERNEL V-1" (the space is a non-breaking space so Android's Settings > Kernel version
    still parses it), CONFIG_LOCALVERSION_AUTO off. Shows as 3.18.91-H-KERNEL V-1. Everything else in the kernel config
    is identical to v3.7 (IKCONFIG compared): ZRAM 640 MB, KernelSU, 8 cores, sdcardfs stale-size fix.
  * updater-script: only the ui_print text changed (new banner, step counter, finish message). Every command, asserts,
    set_progress and set_metadata line is byte-identical to the previous package.
  * Ramdisk, dtb and vendor files unchanged from v3.7.
