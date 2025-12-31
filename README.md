# ESP32SerialSSHProxy-VSC-build

This project provides remote access to a serial (UART) console connected to the `ESP32`, using an `SSH` network connection.

## Setup

1. Wire up your `ESP32` to the serial console of the target:
    TARGET | ESP32 - LOLIN D32  (`UART2`)
    -|- 
    `RX` | `TX2` (GPIO `17`)
    `TX` | `RX2` (GPIO `16`)
    `+5V` | `VBUS` (optional supply for D32)
    `GND` | `GND`
    `/INT` | `Blue LED` (optional GPIO `5`)

    Remember that the `ESP32` is using `3.3V` logic levels. Connecting it to `+5V` logic level serial consoles will
    damage it, so use an SN74LVC1T45DBVR or similar for 3.3V <> 5V level conversion.\
    You can also connect the `ESP32` `VBUS` pin to +5V to power the `ESP32` from the target.

1. Create the file `src/credentials.hpp`:
    ```cpp
    #define WIFISSID "WIFI SSID"
    #define WIFIPASS "WIFI Password"
    #define SSHUSER "SSH username"
    #define SSHPASS "SSH password"
    ```

2. Create a folder `data` in the same location as `src`, then generate `RSA` hostkeys for the `SSH` server as follows:
    ```
    mkdir data
    ssh-keygen -t rsa -f data/hostkey_rsa
    ```

3. From VSC open the project, then execute PlatformIO command `Platform>Upload Filesystem Image` to load the encryption keys to the `ESP32`

4. Compile and upload the project to the `ESP32` using VSC PlatformIO commands `General>Build` and `General>upload`.  

5. Access the device (from Windows Administrator Command Prompt) using:
    ```sh
    ssh -t username@esp32sshserial
    ```
    or
    ```sh
    ssh -t username@192.168.x.y
    ```
    and log in using the credentials created in `credentials.hpp`. If `esp32sshserial` doesn't work, you can try appending your network suffix, or use the IP address provided in the VSC Serial Monitor during `ESP32` boot.

6.  Your `SSH` session is now directly connected
    with the `Serial2` port of the `ESP32`.

7.  Use `Ctrl-G` (Bell) to terminate the `SSH` connection. \
    *Context*: Since `Ctrl-C` is directly forwarded
    to the serial console,
    and there is no equivalent for it on serial consoles,
    the connection effectively cannot terminate ever.
    `Ctrl-G` is captured by the `ESP32` before relaying to Serial
    and will do nothing else than terminate the connection.
    You can then anytime reconnect and continue where you left of. \
    **Remember**: Disconnecting from the `SSH` session will not log
    you out of the connected serial console.

## Terminal configuration

If you have a working connection with a linux machine,
you can do these additional steps
to enhance the serial experience
almost to a native `SSH`-like level.

### Terminal Size

Import the `resizeserial` function:
```bash
resizeserial() {
  local old=$(stty -g)
  stty raw -echo min 0 time 5
  printf '\033[18t' > /dev/tty
  local rows
  local cols
  IFS=';t' read -r _ rows cols _ < /dev/tty
  stty "$old"
  stty cols "$cols" rows "$rows"
}
```

Then run `resizeserial` anytime to resize the terminal.

[Source](https://unix.stackexchange.com/a/283206)

### Colors

Run
```bash
export TERM=xterm-256color
```
to enable colored output.

### Automatic configuration in `.bashrc`

Both above mentioned mechanisms can be automated very easily.

Create a file named `.serialrc`:
```bash
#!/bin/bash

# if not accessing from serial console, stop futher execution
[ "$TERM" != "vt220" ] && return 0

resizeserial() {
  local old=$(stty -g)
  stty raw -echo min 0 time 5
  printf '\033[18t' > /dev/tty
  local rows
  local cols
  IFS=';t' read -r _ rows cols _ < /dev/tty
  stty "$old"
  stty cols "$cols" rows "$rows"
}

# force enable color
export TERM=xterm-256color

# run resizeserial everytime before command execution
#trap resizeserial DEBUG
PROMPT_COMMAND='_trace=yes'
trap '[[ "$_trace" == "yes" ]] && resizeserial; _trace=no' DEBUG
```

Then in `.bashrc` directly after the interactive terminal check insert
```bash
[ -f .serialrc ] && source .serialrc
```
It needs to be in the beginning, for the color settings to apply properly.

## Credits

All credit to 
[`ESP32SerialSSHProxy`](https://github.com/programminghoch10/ESP32SerialSSHProxy)
and
[`LibSSH-ESP32`](https://github.com/ewpa/LibSSH-ESP32)
for making this possible.
