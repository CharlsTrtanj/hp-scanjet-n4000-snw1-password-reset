# HP ScanJet Pro N4000 snw1: Recover the Admin Password via UART

This guide explains how I regained access to the **HP Embedded Web Server (EWS)** on my ScanJet Pro N4000 snw1 after a password change went wrong. It can also help with a forgotten EWS admin password, provided the device behaves like the unit tested here.

The procedure: open the scanner, connect to its **3.3 V UART console at 115200 baud**, read the password actually stored on the device, and use it to set a new password in the web interface. On the tested unit, this required neither a factory reset nor a firmware flash, and the existing settings could be preserved.

> **Scope:** A firsthand account based on a single device running **main-board firmware v0.95**. Other firmware versions and board revisions may behave differently. This is not an official HP service procedure. Use it only on your own device or with the owner's permission.

## My case: The correct password suddenly became “incorrect”

I had set the following admin password on the scanner running firmware v0.95:

| Value | Password | Length |
|---|---|---:|
| What I entered | `Lw-7QgpNAx47rovftTttGffEhIOhs3d` | 31 characters |
| What the device stored | `7QgpNAx47rovftTt` | 16 characters |

The stored password was neither the original value nor its first 16 characters. It matched **characters 4 through 19** of the value I entered. The `Lw-` prefix and the remaining suffix were missing. Login with the original password was rejected.

I contacted HP. Support opened a ticket and worked through several procedures with me. In the end, they suspected that I had “bricked” the device. Those attempts did not produce a working way to restore access.

After opening the scanner, I found its UART connection. Connecting to it gave me a Linux root shell directly, without having to log in. From there, I could read the stored EWS password and see how it differed from the value I had entered. The scanner still booted; in my case, the problem was the loss of admin access.

**This single observation does not establish a general truncation algorithm.** In particular, it does not prove that every long password is truncated at the same position under v0.95, or that the special character near the beginning caused the problem.

The two password values shown above are intentionally included in this public case report. Do not use them as a new password.

## Before opening the scanner: A simple thing to try

If you still know the long password you originally set, you can first try its **first 16 characters** in EWS. A user restored access this way under firmware **0.51**, as described in this [firsthand report in the HP Support Community](https://h30434.www3.hp.com/t5/Scanning-Faxing-Copying/ScanJet-Pro-N4000-snw1-password-reset/td-p/8030034).

This is not a guaranteed solution: my device running v0.95 had **not** stored the first 16 characters. If login still fails, or the original password is unknown, continue with the UART procedure.

## What you need

- An HP ScanJet Pro **N4000 snw1**; my unit has product number **6FW08A**.
- A USB-UART adapter with **3.3 V signal levels**. I used a **DSD TECH SH-U09B with a CP2102**.
- A computer with a serial terminal; the examples below use macOS and `screen`.
- A multimeter, suitable wires, and tools to open the device. The wires in the photos are soldered to the board; clean soldering or another secure connection is required.
- Recommended: a **1 kΩ resistor in series between adapter TXD and scanner RX**, to limit current if a pin is misidentified. This additional resistor is not shown in the photos.

**Electrical requirements:** Use 3.3 V TTL UART, not conventional RS-232 voltage levels or a 5 V TX signal. **Leave the adapter's 3V3, 5V0, and any other power pins disconnected.** The scanner uses its own power supply. A pin labelled 3V3 on the adapter does not, by itself, prove the voltage of its TXD signal; measure idle TXD against adapter GND before connecting it to the scanner.

Open the device and solder only with both the scanner power supply and USB adapter disconnected. Use ESD protection and avoid loose wires, solder bridges, or contact with the metal chassis. During the later measurements, the board will be exposed and powered; touch only the intended low-voltage connections. Incorrect wiring can damage the scanner or adapter.

## 1. Find the UART connection on the main board

The overview photo shows the **solder side of the main board**. The connection used here is circled in red. The designation **J8** was identified from photos of the opposite, component side; the label is not legible in the two photos included here.

![Overview of the solder side: the four-pad UART connection used here is circled in red.](assets/frontpcb.jpg)

In the **close-up below, use the lower row of four pads**, where the three wires are attached. The row of three pads above it is not part of this connection. The close-up is rotated by approximately 180° relative to the overview photo.

![Close-up of the four UART pads: unused pad on the left, then scanner RX in blue, scanner TX in white, and GND in black.](assets/pins.jpg)

**RX and TX are labelled from the scanner's perspective.** The colours refer only to my soldered wires, not to any standard USB-adapter wire colours.

| Connection | Function on the scanner | Wire in the photo | USB-UART adapter connection |
|---|---|---|---|
| A | GND | Black | GND |
| B | TX: scanner transmits | White | RXD: adapter receives |
| C | RX: scanner receives | Blue | TXD: adapter transmits, preferably through 1 kΩ |
| D | Unknown | Not wired | Do not connect |

The viewing direction matters:

- **Overview, `frontpcb.jpg`, left to right:** A/GND, B/TX, C/RX, D/unknown.
- **Close-up, `pins.jpg`, lower row, left to right:** D/unknown, C/RX, B/TX, A/GND.

GND was identified electrically; TX and RX were confirmed by readable boot messages and a working shell connection. A measured voltage of approximately 3.3 V alone does not distinguish an RX input from a power pin or another output. Leave D disconnected.

## 2. Connect the adapter and open a terminal

Prepare the wiring with the scanner unpowered and the adapter disconnected from USB. Then power the scanner before connecting the adapter to the computer. This prevents adapter TXD from driving a signal into an unpowered scanner.

| Setting | Value |
|---|---|
| Signal level | 3.3 V |
| Baud rate | 115200 |
| Data bits | 8 |
| Parity | None |
| Stop bits | 1 |
| Flow control | None: neither RTS/CTS nor XON/XOFF |

On macOS, list the available ports:

```sh
ls /dev/cu.*
```

My adapter appeared as `/dev/cu.usbserial-0001`. Replace this name with your adapter's port:

```sh
screen -fn /dev/cu.usbserial-0001 115200
```

`-fn` disables flow control in `screen`. This guide does not enable automatic terminal logging because passwords will become visible during the later inspection.

On Linux, the port might be named `/dev/ttyUSB0`. If you use a different terminal program, apply the same settings from the table. Open the port in only one program at a time.

After the scanner has booted, press **Enter**. The tested firmware provided this prompt directly:

```text
/ #
```

Check your privileges:

```sh
id
```

My device returned:

```text
uid=0(root) gid=0(root)
```

If you can only read output, check the adapter-TXD → scanner-RX connection in particular. If the characters are unreadable, check signal levels, ground, and terminal settings. An actual login prompt instead of a root shell would differ from the tested unit; no default password for that situation has been established here.

To exit `screen`: press **Ctrl+A**, release the keys, then press **K** and confirm with **Y**.

To capture boot messages, you can initially connect the adapter using only GND and its RXD pin, and start receiving before switching the scanner on. Connect the TXD wire only once the scanner is powered. **Disconnect TXD before switching the scanner off** if the adapter remains powered by USB; otherwise, the signal can back-power the scanner.

## 3. Read the EWS password actually stored on the device

On my unit, the password was stored **in plaintext** in:

```text
/datafs/H1_nvram.bin
```

My investigation notes identify it as the NVRAM field **`sNetSetting.szEWSPass`**. This field name is a technical identification from the investigation, not a file path or a write command documented in this guide.

First check that the file exists and the data directory is mounted:

```sh
ls -l /datafs/H1_nvram.bin
mount | grep /datafs
```

Read the printable strings in the file:

```sh
strings -a -t x /datafs/H1_nvram.bin
```

`-t x` also displays each string's position as a hexadecimal file offset. If the version of `strings` on your device does not support that option, use:

```sh
strings /datafs/H1_nvram.bin
```

**Inspect the output locally.** It may contain other confidential configuration values as well as the EWS password. Do not upload the complete output, the NVRAM file, terminal captures, or unredacted screenshots to a public issue.

In the investigated case, the password string appeared near the model and host name strings. If you still know the password you originally set, you can compare it with a stored fragment. If the password is completely unknown, other printable values are not sufficient proof: identify a candidate from the file context and test it in EWS.

My notes place the relevant value at offset **`0x158`**. This is an observation from this device, **not a universal offset**. The commands above only read the file; they do not modify NVRAM data.

## 4. Log in to the web interface and set a new password

1. Find the current IP address on the control panel or in your router's device list, then open EWS in a browser, for example at `https://<scanner-IP>`.
2. Choose the administrator login and enter **exactly the password found on the device**. In my case, this was `7QgpNAx47rovftTt`.
3. Open the administrator-password setting in the EWS security section. The exact menu label can vary by language and firmware; do not confuse the admin password with scan PINs, SMTP, Wi-Fi, or Linux root passwords.
4. As a workaround, set a different random password with **no more than 16 ASCII characters**. A random value containing letters and digits avoids additional uncertainty about special characters or multibyte characters in this firmware version. This is a cautious workaround, not proof that all other characters are processed incorrectly.
5. Log out, then log in with the new password in a fresh browser session. An existing authenticated session does not test the new password.
6. Also check login after a normal restart when convenient. Only then have you confirmed that the change is stored persistently on your device.

An eight-character password set during my investigation worked. This is not a recommendation to use short passwords: choose the longest random value you can within the limit used here, and restrict EWS access to your trusted network.

A factory reset is not required for this procedure. It would be an additional action and may remove network and scan settings. **The Linux command `passwd root` is not the method used in this case to change the EWS admin password.**

## Technical observations and limitations

| Observation | Evidence and scope |
|---|---|
| Direct UART root shell without login | Confirmed on the described unit with Main Board v0.95 / Panel v0.71 |
| UART console | `console=ttyS0,115200n8` in the kernel command line |
| Operating system | Linux 4.4.76, kernel built on April 16, 2025, `armv5tel` |
| Root filesystem | `/dev/mmcblk0p2`, SquashFS; mounted read-only in the boot log |
| EWS password | Found as readable plaintext in the investigated `/datafs/H1_nvram.bin` |
| Password change | Performed through EWS after logging in with the stored value |

UART access and plaintext storage mean that, on the tested firmware, the EWS password does not provide a security boundary against someone with physical access to the exposed board. Changing the web password does not automatically protect this UART shell.

During recovery, I **did not overwrite password fields with `dd`, run unknown NVRAM tools with write options, or flash modified firmware**. Those operations are not covered here: an incorrect offset or a damaged configuration structure could cause additional damage to a working scanner.

The behaviour of all other firmware versions, the cause of the lost prefix, the function of pin D, and a general repair procedure for scanners that genuinely no longer boot have not been established.

## Other user reports and sources

My observations under v0.95 sit alongside earlier user reports of EWS password problems. The following reports concern different or unspecified firmware versions:

- **EWS password, firmware 0.51, April 7, 2021:** [FFWFAF: ScanJet Pro N4000 snw1 password reset](https://h30434.www3.hp.com/t5/Scanning-Faxing-Copying/ScanJet-Pro-N4000-snw1-password-reset/td-p/8030034). The affected user reports that a long password was truncated to its first 16 characters, and login with that version succeeded. They recall receiving no warning. This does not establish the same truncation pattern as my case under v0.95.
- **EWS lockout, firmware not specified, October 16, 2023:** [Cannot login to EWS scanner, only user guest](https://h30434.www3.hp.com/t5/Scanning-Faxing-Copying/Reset-password-Cannot-login-to-ews-scanner-only-user-guest/td-p/8890757). Another N4000 user reports losing admin access after a password change. The post does not establish a specific truncation pattern or a confirmed cause.

These posts are firsthand accounts from their respective users, not confirmation of the issue by HP. The UART and NVRAM findings in this guide come from my own investigation and its accompanying notes.

Additional technical sources:

- [HP user guide for the N4000 snw1 and related models](https://h10032.www1.hp.com/ctg/Manual/c06556933.pdf): EWS access and scanner settings; it does not document the UART procedure described here.
- [DSD TECH SH-U09B](https://www.deshide.com/product-details_SH-U09B.html): The CP2102 adapter used here and its pin labels.
- [Silicon Labs CP2102/9 datasheet](https://www.silabs.com/documents/public/data-sheets/CP2102-9.pdf): UART signal pins and electrical characteristics.

## Quick guide

This repository documents recovery of the **EWS/web-admin password** on one HP ScanJet Pro N4000 snw1 running **main-board firmware v0.95**. A 31-character password was stored as a 16-character substring, losing both its prefix and suffix. This is an observed case, not a proven general truncation rule.

With physical access and permission to service the device, connect a **3.3 V USB-UART adapter at 115200, 8N1, no flow control**. Prepare the wiring with both the scanner power supply and USB adapter disconnected. Power the scanner before connecting the adapter to USB. Scanner GND → adapter GND; scanner TX → adapter RXD; scanner RX → adapter TXD, preferably through a 1 kΩ series resistor. Leave adapter power pins and the unidentified fourth pad disconnected. The two photos have opposite orientations; follow their captions.

The tested scanner provided a root shell without login. Inspect `strings -a -t x /datafs/H1_nvram.bin` locally to find the stored plaintext EWS password, then log into EWS with that exact value and set a different random password of at most 16 ASCII characters as a workaround. Verify login in a fresh browser session and after a restart. Do not publish the configuration dump or overwrite NVRAM offsets. Disconnect adapter TXD before powering off the scanner.
