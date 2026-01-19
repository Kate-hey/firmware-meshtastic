# T-Lora Pager Firmware Release

Custom Meshtastic firmware build for the **LilyGo T-Lora Pager** with significant UI and usability improvements.

**Version:** 2.7.18-tlora-pager-fixes
**Based on:** Meshtastic firmware v2.7.18
**Release:** https://github.com/Kate-hey/firmware-meshtastic/releases/tag/v2.7.18-tlora-pager-fixes

---

## Summary of Changes

This release includes fixes and enhancements in two repositories:
- **Firmware** (`Kate-hey/firmware-meshtastic`, branch: `tlora-pager-device-ui-fixes`)
- **Device-UI** (`Kate-hey/device-ui`, branch: `feature/tlora-pager-scrollwheel`)

---

## Firmware Changes

### 1. Display Wake from Power Save Fix

**Problem:** The T-Lora Pager screen would not wake up when receiving messages or pressing keyboard keys while in power save mode.

**Solution:**
- Added a TFT screen wake callback mechanism to PowerFSM for device-ui integration
- Registered callbacks in `tftSetup.cpp` to trigger `DisplayDriver::requestWake()` and `lv_display_trigger_activity()` on keyboard input or message received
- Added T-Lora Pager exception to always wake on message (it's a pager!)
- Fixed `tftSleepObserver` initialization order (create after `deviceScreen` is valid)
- Added TCA8418 keyboard controller reset sequence in `main.cpp`

**Files changed:**
- `src/PowerFSM.cpp` / `src/PowerFSM.h` - Added `setTFTScreenWakeCallback()` and `triggerScreenWake()`
- `src/graphics/Screen.cpp` - T-Lora Pager always wakes on message
- `src/graphics/tftSetup.cpp` - Register wake callbacks, fix observer initialization
- `src/main.cpp` - TCA8418 keyboard reset sequence

### 2. Muted Channel Detection Fix

**Problem:** The previous muted channel detection incorrectly used `mp.channel > 0` to detect DMs, but channel 0 is the primary channel, not a DM indicator. This caused:
- Primary channel (channel 0) messages to always wake the screen even when muted
- Incorrect mute behavior for channel-based messages

**Solution:** Now correctly detects DMs by checking the `to` field:
- A DM is when `to` is a specific node (not 0 and not `NODENUM_BROADCAST`)
- Muted channels properly suppress screen wake
- DMs always wake the screen regardless of channel mute settings

**Files changed:**
- `src/graphics/Screen.cpp` - Check `packet->to` for DM detection
- `src/graphics/draw/MessageRenderer.cpp` - Respect `isChannelMuted` for screen wake
- `src/modules/TextMessageModule.cpp` - Check `mp.to` for DM detection

### 3. Build Configuration Updates

- Added `MESHTASTIC_EXCLUDE_INPUTBROKER=1` flag - disables firmware keyboard handling so device-ui handles it
- Updated display resolution flags for native 480x222 UI
- Local device-ui symlink for development

**Files changed:**
- `platformio.ini` - Local device-ui symlink
- `variants/esp32s3/tlora-pager/platformio.ini` - Build flags

---

## Device-UI Changes

### 1. Display Wake Mechanism

**Problem:** Device-ui had no way to wake the display from power save when keyboard input occurred or messages arrived.

**Solution:**
- Added static `requestWake()`/`isWakeRequested()`/`clearWakeRequest()` to `DisplayDriver` for thread-safe wake requests
- Check wake request flag in `LGFXDriver` power save loop alongside GPIO/activity
- Added `InputEventCallback` to `I2CKeyboardInputDriver` to notify firmware on keypress
- Call `lv_display_trigger_activity()` on keyboard input to reset LVGL inactivity

**Files changed:**
- `include/graphics/driver/DisplayDriver.h` - Wake request API
- `include/graphics/driver/LGFXDriver.h` - Check wake flag in task handler
- `include/input/I2CKeyboardInputDriver.h` - Input event callback
- `source/graphics/driver/DisplayDriver.cpp` - Static member init
- `source/input/I2CKeyboardInputDriver.cpp` - Trigger activity on keypress

### 2. Muted Channel Detection Fix

**Problem:** Same as firmware - incorrect DM detection using channel number.

**Solution:** Check `to` field to detect DMs. Muted channels now properly:
- Skip unread counter increment
- Skip message popup
- DMs always notify

**Files changed:**
- `source/graphics/TFT/TFTView_480x222.cpp` - `newMessage()` function

### 3. Rotary Encoder Improvements

- **Reversed flex flow** on chats panel so encoder clockwise moves UP (to newer chats) - more intuitive
- Encoder navigation throughout the UI

**Files changed:**
- `source/graphics/TFT/TFTView_480x222.cpp`
- `source/graphics/TFT/TFTView_320x240.cpp`

### 4. Unread Message Badge

- Added **red message count badge** on Messages button showing unread count
- Grey rounded background for readability
- Uses bolder font (montserrat_20)

**Files changed:**
- `source/graphics/TFT/TFTView_480x222.cpp`
- `source/graphics/TFT/TFTView_320x240.cpp`

### 5. Sym/Alt Modifier Indicator

- Display **orange "S"** at bottom-left when Sym key is pressed
- Visual feedback for keyboard modifier state

**Files changed:**
- `source/graphics/TFT/TFTView_480x222.cpp`
- `source/graphics/TFT/TFTView_320x240.cpp`

### 6. Hold-to-Scroll Behavior

**Problem:** Using Sym key for scrolling while typing was awkward with toggle behavior.

**Solution:** Sym key now has **hold-to-scroll** behavior:
- Hold Sym + rotate encoder = scroll
- Tap Sym = toggle symbol entry mode (one-shot)

**Files changed:**
- `source/input/I2CKeyboardInputDriver.cpp`

### 7. Message Popup Keyboard Navigation

- **Enter** = navigate to the chat
- **Backspace/ESC** = dismiss popup
- Better keyboard-driven workflow

**Files changed:**
- `source/graphics/TFT/TFTView_480x222.cpp`

### 8. Alert Auto-Dismiss

- Native 480x222 UI optimizations
- Alert popup improvements

---

## Installation

### First-Time Flash (Factory Image)
Use `firmware-tlora-pager-tft-2.7.18.factory.bin` - includes bootloader and partition table.

```bash
esptool.py --chip esp32s3 --port /dev/ttyACM0 write_flash 0x0 firmware-tlora-pager-tft-2.7.18.factory.bin
```

### OTA Update
Use `firmware-tlora-pager-tft-2.7.18.bin` - firmware only.

Upload via the Meshtastic web interface or app.

---

## Links

- **Firmware branch:** [tlora-pager-device-ui-fixes](https://github.com/Kate-hey/firmware-meshtastic/tree/tlora-pager-device-ui-fixes)
- **Device-UI branch:** [feature/tlora-pager-scrollwheel](https://github.com/Kate-hey/device-ui/tree/feature/tlora-pager-scrollwheel)
- **Release:** [v2.7.18-tlora-pager-fixes](https://github.com/Kate-hey/firmware-meshtastic/releases/tag/v2.7.18-tlora-pager-fixes)

---

## Testing Checklist

- [ ] Screen wakes on keyboard press from power save
- [ ] Screen wakes on message received
- [ ] Muted channel messages don't wake screen
- [ ] DMs always wake screen (even if channel is muted)
- [ ] Unread badge shows on Messages button
- [ ] Encoder scrolls correct direction in chat list
- [ ] Sym indicator shows when Sym is active
- [ ] Hold Sym + encoder scrolls
- [ ] Message popup responds to Enter/Backspace keys
