# RS485 on PINE64

This branch starts from megous work on the kernel for armbian, (branch 5.10)[https://github.com/megous/linux/tree/orange-pi-5.10].

Megous kernel can be compiled and can replace existing armbian kernel.

allwinner 8250 uart are similar to designware ones. The driver is 8250_dw.c.

The hardware for RS485 needs three signals:
- RX (same as any UART)
- TX (same as any UART)
- TXENABLE : This is a half duplex bus, TXENABLE must be high during the transmission. 
The most obvious is to use the RTS control line output to drive create TXENABLE.

Current em485 emulation in the mainline is not precise enough for fast peripherals. Let me detail that.

## Current em485 implementation details

The only interrupt available is "tx FIFO empty" (THRE flag, ETBEI to enable the interrupt). In em485:
- the app fill the tx circular buffer in the kernel
- as soon as this buffer has one byte, em485 turns on TXENABLE, and starts a xx milliseconds timer (first value of rs485-rts-delay table in the device tree).
- the end of this "pre transmission timer" triggers the first real transmission (from the kernel circular buffer to the hardwar FIFO)
- as soon as the circular buffer is empty, THRE is disabled, and em485 starts a xx milliseconds timer (second value of rs485-rts-delay table in the device tree).
- the end of the "post transmission timer" triggers the turn off of TXENABLE

The current em485 emulation can work only if:
- the second timer setting is greater than the time to transmit the hardware FIFO size (16 bytes)
- this time depends on the baudrate
- this time depends on the number of "bits ber byte" (number of data bits, stop bits, parity bits presence)

For example, in a 9600N8 setting, there is 10 bits per byte. The allwinner 8250 has a 16 bytes FIFO. The post transmission timer must be greater than 16*10/9600 = 17ms.

If you send 17 bytes:
- the driver will fill the 16 bytes FIFO (1 byte remain in the circular buffer)
- the THRE interrupt fires. the driver fill 1 bytes in the FIFO and triggers post transmission timer

It means you take the bus control during 16ms after the last byte is sent. *But most RS485 peripherals are microconcrollers that answer in far less than 5 milliseconds*. That's why I created precise_em485. 


## precise em485 implementation details

There is no interrupt to tell us "the last stop bit of the last byte is effectively out".
The current THRE interrupt tell us "the hardware FIFO is empty". It tells us that the last bytes was moved from FIFO to the shift register to be serialized.
It means that anyway, the interrupt will fire one byte before the physical end of transmission.

The first step is to compute the number of nanoseconds to effectively transmit a byte. It depends on baudrate and serial configuration. It is `byte_duration_ns` in my code. Then there is the propagation time margin `time_after_lastbyte_ns` (you need to keep the driver transmitting to be sure than all nodes get the last bit). I hardcoded `time_after_lastbyte_ns = byte_duration_ns / 5`. These values are computed with termios setting call.

The "pre transmission timer" feature is removed. Physically, there is no need to waste time before transmission.

The "RX while TX" feature is removed. RS485 is half duplex anyway, you just need to connect together tx enable and rx disable together on the hardware to avoid echo.

In precise em485:
- the app fill the tx circular buffer in the kernel 
- as soon as this buffer has one byte, em485 turns on TXENABLE (and cancel the stop_tx_timer timer), and starts filling the buffer.
- as soon as the circular buffer is empty, do not mask the THRE interrupt.
- as soon as the FIFO empty interrupt fires, em485 starts stop_tx_timer timer with a `time_after_lastbyte_ns + time_after_lastbyte_ns` delay.
- the end of the stop_tx_timer triggers the turn off of TXENABLE

Configuration use the same elements of the device tree and userland functions:
- device tree `linux,rs485-enabled-at-boot-time` or `SER_RS485_ENABLED` userland flag: enables the RS485 feature at boot.
- device tree `rs485-rts-active-low` or `SER_RS485_RTS_ON_SEND / SER_RS485_RTS_AFTER_SEND` userland flags: when present, RTS=1 while transmitting. 
- device tree `rs485-rts-delay`: no effect.
- device tree `rs485-rx-during-tx` or `SER_RS485_RX_DURING_TX` userland flag: no effect.



## How to

Boot armbian on a PINE64.
Extract kernel config from /proc/config.gz, then copy it in the root directory, and use it as a start:

 cp configArmbianPine64_kernel5.10.conf .config

customize .config to enable auto reboot on oops, because nothing as stupide as an iot device stuck in a oops.
```
CONFIG_PANIC_ON_OOPS=y
CONFIG_PANIC_ON_OOPS_VALUE=1
CONFIG_PANIC_TIMEOUT=30
```


./MakeAndCopyToTarget.sh

Enable the uart4, add to  /boot/armbianEnv.txt
overlays=uart4



decompile, change and recompile uart4 overlay to activate RS485 at boot and enable PD4 and P5 the simplest way.
Note rs485-rts-active-low means turn high the RTS pin during tx (most of the RS485 transceiver expects that).

```
dtc -I dtb -O dts sun50i-a64-uart4.dtbo -o sun50i-a64-uart4.dts
```
edit /boot/dtb/allwinner/overlay/sun50i-a64-uart4.dts


```
/dts-v1/;

/ {
        compatible = "allwinner,sun50i-a64";

        fragment@0 {
                target-path = "/aliases";

                __overlay__ {
                        serial4 = "/soc/serial@1c29000";
                };
        };

        fragment@1 {
                target = <0xffffffff>;

                __overlay__ {

                        uart4-pins {
                                pins = "PD2", "PD3", "PD4", "PD5";
                                function = "uart4";
                                phandle = <0x1>;
                        };

                        uart4-rts-cts-pins {
                                pins = "PD4", "PD5";
                                function = "uart4";
                                phandle = <0x2>;
                        };
                };
        };

        fragment@2 {
                target = <0xffffffff>;

                __overlay__ {
                        pinctrl-names = "default";
                        pinctrl-0 = <0x1>;
                        status = "okay";
                        linux,rs485-enabled-at-boot-time;
                        rs485-rts-active-low;
                        rs485-rts-delay = <0 0>;
                };
        };

        __symbols__ {
                uart4_pins = "/fragment@1/__overlay__/uart4-pins";
                uart4_rts_cts_pins = "/fragment@1/__overlay__/uart4-rts-cts-pins";
        };

        __fixups__ {
                pio = "/fragment@1:target:0";
                uart4 = "/fragment@2:target:0";
        };

        __local_fixups__ {

                fragment@2 {

                        __overlay__ {
                                pinctrl-0 = <0x0>;
                        };
                };
        };
};

```

```
dtc -I dts -O dtb sun50i-a64-uart4.dts -o sun50i-a64-uart4.dtbo

```


