Hier ist das vollständige, aktualisierte Dokument. Es enthält nun die gesamte Architektur, inklusive der neuen Matrix für Short-, Double- und Long-Press, gebündelt in einem sauberen, englischsprachigen Guide, den du direkt für dein Repository oder deine Notizen verwenden kannst.

---

# Headless PC Multi-OS Hardware Boot Selector

This document outlines the architecture, hardware wiring, and software implementation of a completely headless, bidirectional hardware boot selector. It allows switching between multiple operating systems using a physical rotary switch, initiating graceful or forced reboots from within the running OS via a multi-action push-button, and communicating with the GRUB bootloader via hardware LED signals.

---

## 1. Concept & Existing Alternatives

The goal is to control the boot process of a headless PC reliably without guessing timings. Before arriving at this architecture, existing solutions in the maker community were evaluated:

* **The "Fake USB Mass Storage" Approach:** Uses an ATmega32u4 to emulate a USB drive containing a fake file indicating the switch position. GRUB reads this file.
* *Drawbacks:* Requires complex filesystem emulation firmware, provides no active feedback loop, and lacks the ability to trigger OS reboots from a running state.


* **The Physical VCC Cutoff:** Involves splicing a USB thumb drive's 5V line through a physical toggle switch. GRUB checks for the presence of the drive's UUID.
* *Drawbacks:* Limited mostly to binary (A/B) choices, requires manual OS shutdown, and offers no bidirectional communication.



**Our Solution:** A stateless, bidirectional handshake leveraging the legacy PS/2 keyboard controller (`outb 0x60`) within GRUB to signal the microcontroller via the Caps Lock LED. This is combined with a unified C-daemon for OS-level reboots using a multi-action input matrix.

---

## 2. The Input Matrix: Short, Double, and Long Press

By wiring the rotary encoder's push-button in parallel to the motherboard's `PWR_SW` pins and the microcontroller, we achieve a powerful control matrix:

* **Short Press (< 1.5s):** The microcontroller sends a command for a **Graceful Reboot**. If the OS blocks it (e.g., pending updates), the C-daemon flashes the Caps Lock LED to warn the user.
* **Double Press:** The microcontroller sends a command for a **Force Reboot**. The OS immediately restarts, ignoring blocking applications.
* **Long Press (1.5s - 3s):** The microcontroller sends a command for a **Graceful Shutdown**. The PC powers off safely.
* **ATX Hardware Override (> 4s):** The motherboard's hardware timer takes over and cuts power instantly (Hard Power Off) in case of a total system freeze.

*(Note: Ensure the OS power settings are set to "Do Nothing" on physical power button presses to avoid conflicts with the software listener.)*

---

## 3. Hardware Architecture & Wiring

### Components Required

* **Microcontroller:** Digispark ATtiny85 (or equivalent clone with a native USB plug).
* **Input:** Multi-position rotary switch with an integrated push-button (Rotary Encoder style).
* **Feedback:** The built-in LED on the Digispark (usually Pin 1) or an external LED.

### Pin Wiring

* **Pin 0 (P0):** Rotary Switch Position 1 (e.g., Windows). Connects to GND when active.
* **Pin 2 (P2):** Rotary Switch Position 2 (e.g., Ubuntu). Connects to GND when active.
* **Pin 5 (P5):** Push Button. Connects to GND when pressed. This pin is also wired in parallel to the motherboard's `PWR_SW` header.
* *Note: Pin 3 and Pin 4 are reserved for USB D- and D+ communications on the Digispark.*

---

## 4. The Bidirectional Handshake (The PS/2 Hack)

Because the microcontroller remains powered continuously via USB standby power, relying on timing or software resets creates race conditions. Instead, the system is stateless:

1. **GRUB initiates:** When the bootloader starts, GRUB writes raw bytes directly to the legacy PS/2 keyboard controller port (`0x60`), turning on the Caps Lock LED.
2. **BIOS Emulation:** The motherboard's UEFI USB Legacy Support translates this PS/2 command into a USB HID packet and sends it to the Digispark.
3. **The Handshake:** The Digispark waits in a loop for the Caps Lock signal. Once detected, it reads the physical rotary switch and types the selection (`opt1`, `opt2`) into the GRUB prompt.

---

## 5. Microcontroller Firmware (Digispark C++)

Upload this firmware to the Digispark via the Arduino IDE. It handles the GRUB handshake and the complex click-timing logic.

```cpp
#include "DigiKeyboard.h"
#include <avr/wdt.h>

const int pinOpt1 = 0;
const int pinOpt2 = 2;
const int pinButton = 5;

// Timing variables for button press detection
unsigned long buttonTimer = 0;
bool buttonActive = false;
bool longPressActive = false;
int pressCount = 0;

void setup() {
  wdt_disable();
  pinMode(pinOpt1, INPUT_PULLUP);
  pinMode(pinOpt2, INPUT_PULLUP);
  pinMode(pinButton, INPUT_PULLUP);

  // Wait for GRUB's Caps Lock Handshake (Bit 2 = Value 2)
  unsigned long startTime = millis();
  while (!(DigiKeyboard.getLEDs() & 2)) {
    DigiKeyboard.update();
    DigiKeyboard.delay(50);
    // Fallback: Prevent infinite lockup if GRUB fails to signal
    if (millis() - startTime > 15000) break; 
  }

  // Signal received. Send correct GRUB option
  if (digitalRead(pinOpt1) == LOW) DigiKeyboard.print("opt1");
  else if (digitalRead(pinOpt2) == LOW) DigiKeyboard.print("opt2");
  else DigiKeyboard.print("opt0");
  
  DigiKeyboard.sendKeyStroke(KEY_ENTER);
}

void loop() {
  DigiKeyboard.update(); // Keep USB connection alive

  bool isPressed = (digitalRead(pinButton) == LOW);

  // Button was just pressed
  if (isPressed && !buttonActive) {
    buttonActive = true;
    buttonTimer = millis();
  }

  // Button is being held down -> Check for Long-Press (1500 ms)
  if (isPressed && buttonActive && !longPressActive) {
    if (millis() - buttonTimer > 1500) {
      longPressActive = true;
      sendHotkey('S'); // S = Graceful Shutdown
      resetDigispark();
    }
  }

  // Button was released
  if (!isPressed && buttonActive) {
    buttonActive = false;
    
    // If it was not a long press, count the short clicks
    if (!longPressActive) {
      pressCount++;
      buttonTimer = millis(); // Restart timer for double-click window
    }
    longPressActive = false;
  }

  // Evaluate clicks if 400ms have passed since the last release
  if (pressCount > 0 && !isPressed && (millis() - buttonTimer > 400)) {
    if (pressCount == 1) {
      // Single Click -> Graceful Reboot based on target OS
      if (digitalRead(pinOpt1) == LOW) sendHotkey('W');
      else if (digitalRead(pinOpt2) == LOW) sendHotkey('U');
    } 
    else if (pressCount >= 2) {
      // Double Click -> Force Reboot (regardless of target OS)
      sendHotkey('F'); 
    }
    
    resetDigispark();
  }
}

void sendHotkey(char key) {
  // Send Hyper-Key combo
  uint8_t modifiers = MOD_CONTROL_LEFT | MOD_SHIFT_LEFT | MOD_ALT_LEFT | MOD_GUI_LEFT;
  DigiKeyboard.sendKeyStroke(key, modifiers);
  DigiKeyboard.delay(500);
}

void resetDigispark() {
  // Trigger software reset to prepare for the next GRUB handshake
  wdt_enable(WDTO_15MS);
  while(1);
}

```

---

## 6. GRUB Configuration (`/etc/grub.d/40_custom`)

Append this script to your custom GRUB config. It handles the `outb` LED switching and the string mapping. Run `sudo update-grub` after editing.

```sh
#!/bin/sh
exec tail -n +3 $0

set timeout=0

# 1. Signal the Digispark (Caps Lock ON)
outb 0x60 0xED
outb 0x60 0x04

echo "Hardware Handshake initialized. Waiting for selector..."
set switch_input=""

# Wait up to 5 seconds for the Digispark to type the string
read --timeout=5 switch_input

# 2. Turn LED OFF
outb 0x60 0xED
outb 0x60 0x00

# Mapping variables (Replace strings with your exact `menuentry` titles)
set os_opt1="Windows Boot Manager (on /dev/nvme0n1p1)"
set os_opt2="Ubuntu"
set os_default="Ubuntu"

if [ "${switch_input}" = "opt1" ]; then
    set default="${os_opt1}"
elif [ "${switch_input}" = "opt2" ]; then
    set default="${os_opt2}"
else
    set default="${os_default}"
fi

```

---

## 7. Unified OS Listener (C Daemon)

This unified C daemon intercepts the Digispark's commands across operating systems. It executes graceful reboots, forced reboots, or shutdowns. If a graceful reboot is blocked by the OS, it flashes the Caps Lock LED.

`switch_listener.c`:

```c
#include <stdio.h>
#include <stdlib.h>

#ifdef _WIN32
    #include <windows.h>
    #define SLEEP(ms) Sleep(ms)
#else
    #include <unistd.h>
    #include <X11/Xlib.h>
    #include <X11/extensions/XTest.h>
    #define SLEEP(ms) usleep(ms * 1000)
#endif

void log_action(const char* action) {
    // Basic logging identifiable for debugging purposes
    printf("[switch_listener] Executing: %s\n", action);
}

void toggle_caps_lock() {
#ifdef _WIN32
    keybd_event(VK_CAPITAL, 0x3A, 0, 0);
    keybd_event(VK_CAPITAL, 0x3A, KEYEVENTF_KEYUP, 0);
#else
    Display *display = XOpenDisplay(NULL);
    if (display) {
        XTestFakeKeyEvent(display, XKeysymToKeycode(display, XK_Caps_Lock), True, CurrentTime);
        XTestFakeKeyEvent(display, XKeysymToKeycode(display, XK_Caps_Lock), False, CurrentTime);
        XFlush(display);
        XCloseDisplay(display);
    }
#endif
}

void trigger_error_blink() {
    log_action("Reboot blocked. Flashing Caps Lock.");
    for (int i = 0; i < 10; i++) {
        toggle_caps_lock();
        SLEEP(300);
    }
}

void execute_action(char action_type) {
    log_action("Initiating power state change...");
#ifdef _WIN32
    HANDLE hToken;
    TOKEN_PRIVILEGES tkp;
    OpenProcessToken(GetCurrentProcess(), TOKEN_ADJUST_PRIVILEGES | TOKEN_QUERY, &hToken);
    LookupPrivilegeValue(NULL, SE_SHUTDOWN_NAME, &tkp.Privileges[0].Luid);
    tkp.PrivilegeCount = 1;
    tkp.Privileges[0].Attributes = SE_PRIVILEGE_ENABLED;
    AdjustTokenPrivileges(hToken, FALSE, &tkp, 0, (PTOKEN_PRIVILEGES)NULL, 0);

    if (action_type == 'G') {
        ExitWindowsEx(EWX_REBOOT, SHTDN_REASON_MAJOR_HARDWARE);
    } else if (action_type == 'F') {
        ExitWindowsEx(EWX_REBOOT | EWX_FORCE, SHTDN_REASON_MAJOR_HARDWARE);
    } else if (action_type == 'S') {
        ExitWindowsEx(EWX_POWEROFF, SHTDN_REASON_MAJOR_HARDWARE);
    }
#else
    if (action_type == 'G') {
        system("systemctl reboot");
    } else if (action_type == 'F') {
        system("systemctl --force reboot");
    } else if (action_type == 'S') {
        system("systemctl poweroff");
    }
#endif
}

int main() {
    log_action("Listener daemon started.");
#ifdef _WIN32
    // Windows: Register Hotkeys (HyperKey + U/W/F/S)
    RegisterHotKey(NULL, 1, 0x0001 | 0x0002 | 0x0004 | 0x0008, 'U'); // Graceful Ubuntu
    RegisterHotKey(NULL, 2, 0x0001 | 0x0002 | 0x0004 | 0x0008, 'W'); // Graceful Windows
    RegisterHotKey(NULL, 3, 0x0001 | 0x0002 | 0x0004 | 0x0008, 'F'); // Force Reboot
    RegisterHotKey(NULL, 4, 0x0001 | 0x0002 | 0x0004 | 0x0008, 'S'); // Shutdown

    MSG msg = {0};
    while (GetMessage(&msg, NULL, 0, 0) != 0) {
        if (msg.message == WM_HOTKEY) {
            if (msg.wParam == 1 || msg.wParam == 2) {
                execute_action('G');
                SLEEP(3000); 
                trigger_error_blink(); 
            } else if (msg.wParam == 3) {
                execute_action('F');
            } else if (msg.wParam == 4) {
                execute_action('S');
            }
        }
    }
#else
    // Linux: X11 Global Grabber
    Display* display = XOpenDisplay(NULL);
    if (!display) return 1;

    Window root = DefaultRootWindow(display);
    unsigned int modifiers = ControlMask | Mod1Mask | ShiftMask | Mod4Mask;
    
    KeyCode key_u = XKeysymToKeycode(display, XK_U);
    KeyCode key_w = XKeysymToKeycode(display, XK_W);
    KeyCode key_f = XKeysymToKeycode(display, XK_F);
    KeyCode key_s = XKeysymToKeycode(display, XK_S);

    XGrabKey(display, key_u, modifiers, root, False, GrabModeAsync, GrabModeAsync);
    XGrabKey(display, key_w, modifiers, root, False, GrabModeAsync, GrabModeAsync);
    XGrabKey(display, key_f, modifiers, root, False, GrabModeAsync, GrabModeAsync);
    XGrabKey(display, key_s, modifiers, root, False, GrabModeAsync, GrabModeAsync);
    XSelectInput(display, root, KeyPressMask);

    XEvent ev;
    while (1) {
        XNextEvent(display, &ev);
        if (ev.type == KeyPress) {
            if (ev.xkey.keycode == key_u || ev.xkey.keycode == key_w) {
                execute_action('G');
                SLEEP(3000);
                trigger_error_blink();
            } else if (ev.xkey.keycode == key_f) {
                execute_action('F');
            } else if (ev.xkey.keycode == key_s) {
                execute_action('S');
            }
        }
    }
    XCloseDisplay(display);
#endif
    return 0;
}

```

### Installation

* **Windows:** Compile via MinGW: `gcc switch_listener.c -o switch_listener.exe -mwindows`. Place a shortcut to the executable in your startup folder (`shell:startup`).
* **Linux:** Compile via GCC: `gcc switch_listener.c -o switch_listener -lX11 -lXtst`. Add the executable to your Desktop Environment's startup applications or run it as a `systemd` user daemon.

---

## 8. Testing Procedures

The entire software stack can be tested without the Digispark hardware connected:

1. **Test the OS Listener:** Run the compiled C daemon. On a standard keyboard, press `Ctrl+Alt+Shift+Win + U`. The system should gracefully reboot.
2. **Test the Error Feedback:** Open an unsaved document to forcefully block the shutdown. Press the hotkey combo again. Within 3 seconds, your physical keyboard's Caps Lock LED should begin flashing rapidly.
3. **Test Force Reboot:** Press `Ctrl+Alt+Shift+Win + F`. The system should restart immediately, ignoring the unsaved document.
4. **Test the GRUB Emulation:** Boot into the GRUB menu. Press `c` to open the command line. Type `outb 0x60 0xED` followed by `outb 0x60 0x04`. If your standard USB keyboard's Caps Lock light turns on, the BIOS PS/2 emulation is functioning perfectly, confirming compatibility with the Digispark handshake.


