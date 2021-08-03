# PINE64 Battery Management

This branch includes several commits :
- battery management without NTC (thermal sensor in the battery) (driver and dts patch)
- power on button activation reduced to 128ms (driver patch)
- power on button completely disabled in the kernel (driver patch, irq handling commented)
- RS845 emulation on UART4 (dts patch) (see [branch RS485](https://github.com/pi-r-p/linux/tree/RS485Pine64))
- RTC always busy fix. Because Allwinner RTC status bits can freeze to one. reboots and shutdown won't clear the problem, RTC is always ON.
