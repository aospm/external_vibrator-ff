# vibrator-ff, a generic vibrator HAL for force feedback haptics.

Most downstream haptics / vibrator drivers are implemented as LED class devices, this kinda makes sense, but.. they aren't LEDs.

As haptics drivers appear in mainline, they seem to be implemented as force-feedback input devices. Using the force-feedback API originally designed for game controllers.

I'm certainly not about to defend this API, it raises the complexity required to support haptics substantially. But it's a good start.

This HAL is designed to be completely generic, and should work with any force-feedback device.
On start-up it scans all input devices and picks the first device which supports the force-feedback API.  It then uploads effects to the kernel for every effect and strength combination and plays them on demand.

If there are no FF input devices available, it will stub the behaviour rather than fail as failing will cause Android to get stuck and refuse to boot.

## Integrating this HAL

The HAL is quite straightforwar to integrate into your device. Ensure the HAL is cloned in your manifests:

```xml
<remote name="aospm" fetch="https://github.com/aospm" />

<project path="external/vibrator-ff" name="external_vibrator-ff" revision="main" remote="aospm" groups="default" />
```

Add
```m
PRODUCT_PACKAGES += \
    android.hardware.vibrator@1.1-service.ff
```

In your devices `manifest.xml` add
```xml
<hal format="hidl">
    <name>android.hardware.vibrator</name>
    <transport>hwbinder</transport>
    <version>1.1</version>
    <interface>
        <name>IVibrator</name>
        <instance>default</instance>
    </interface>
</hal>
```

Ensure your `sepolicy/file_contexts` contains the following:
```sh
/dev/input/.*		u:object_r:input_device:s0
```

Then create a new file `sepolicy/hal_vibrator.te` with the following contents:
```
# Vibrator HAL scans input devices to find the haptics device
# it then calls ioctls on it.
init_daemon_domain(hal_vibrator_default);

allow hal_vibrator_default input_device:chr_file { ioctl open read write };
# EVIOCGBIT + EV_FF
allowxperm hal_vibrator_default input_device:chr_file ioctl 0x4535;
# EVIOCSFF
allowxperm hal_vibrator_default input_device:chr_file ioctl 0x4580;
# EVIOCRMFF
allowxperm hal_vibrator_default input_device:chr_file ioctl 0x4581;
allow hal_vibrator_default input_device:dir { open read search };
```

This will give the HAL permissions to scan input devices and perform only the IOCTLs it needs. This doesn't enable reading arbitrary events from input devices.