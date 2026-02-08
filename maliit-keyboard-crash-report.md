# Maliit Keyboard Crash Report - Fedora 43

## Summary
maliit-keyboard crashes repeatedly with SIGSEGV when handling text input in Wayland sessions, making the on-screen keyboard unusable.

## Crash Details

- **First occurrence:** November 9, 2025
- **Most recent:** February 8, 2026
- **Frequency:** Multiple crashes daily (dozens recorded since Nov 2025)
- **Signal:** 11 (SIGSEGV - Segmentation Fault)
- **Severity:** High - Recurring crashes affecting input method functionality

## Technical Details

### Crash Location
- **Function:** `QUtf8::convertToUnicode(QChar*, char const*, int)`
- **Library:** libQt5Core.so.5 (offset +0x278670)
- **Trigger:** `Maliit::Wayland::InputMethodContext::zwp_input_method_context_v1_surrounding_text()`

### Stack Trace (February 6, 2026 @ 21:24:49)
```
#0  0x00007f30c9278670 _ZN5QUtf816convertToUnicodeEP5QCharPKci (libQt5Core.so.5 + 0x278670)
#1  [Qt string conversion]
#2  [Maliit Wayland context handling]
#3  0x00007efcdc61162b _ZN6Maliit7Wayland18InputMethodContext44zwp_input_method_context_v1_surrounding_textERK7QStringjj (libmaliit-plugins.so.2 + 0x4f62b)
```

### Journal Log Entry (Feb 6, 2026)
```
Feb 06 21:24:49 fedora kernel: maliit-keyboard[1846]: segfault at 55920581c000 ip 00007f30c9278670 sp 00007ffe4b27f9b0 error 4 in libQt5Core.so.5.15.18[278670,7f30c9000000+2f2000] likely on CPU 0 (core 0, socket 0)
Feb 06 21:24:51 fedora kwin_wayland[1743]: Input Method crashed "maliit-keyboard" QList() 11 QProcess::CrashExit
```

## Environment

### System Information
- **Distribution:** Fedora Linux 43
- **Desktop Environment:** KDE Plasma
- **Display Server:** Wayland
- **User:** francisco (UID 1000)

### Package Versions
- **maliit-keyboard:** 2.3.1-11.fc43.x86_64
- **maliit-framework:** 2.3.0-10.fc43.x86_64
- **qt5-qtbase:** 5.15.18-1.fc43.x86_64

### Related Modules
```
/usr/bin/maliit-keyboard (maliit-keyboard-2.3.1-11.fc43.x86_64)
libinputpanel-shell.so (maliit-framework-2.3.0-10.fc43.x86_64)
libQt5Core.so.5 (qt5-qtbase-5.15.18-1.fc43.x86_64)
libmaliit-plugins.so.2 (maliit-framework-2.3.0-10.fc43.x86_64)
```

## Reproduction

The crash occurs automatically during normal text input operations in a Wayland session. No specific user action required - the crash appears to be triggered by Wayland's input method protocol sending surrounding text context to maliit-keyboard.

## Impact

When the crash occurs:
1. KWin Wayland logs: "Input Method crashed 'maliit-keyboard'"
2. On-screen keyboard becomes unavailable
3. systemd-coredump captures the crash
4. DrKonqi crash handler attempts processing but often fails with "Nothing handled the dump :O"
5. Multiple coredump processing attempts may follow

The crash severely impacts users who rely on the on-screen keyboard for input.

## Core Dumps

Core dumps are stored in: `/var/lib/systemd/coredump/`

Example dumps (with present status):
- Sun 2026-01-25 14:49:06: 4.4M
- Sun 2026-01-25 14:49:08: 4.1M
- Multiple additional dumps ranging from 3.6M to 6M

Many earlier dumps show "missing" status, indicating they may have been cleaned up by systemd-coredump retention policies.

## Analysis

The crash appears to be a bug in the UTF-8 text conversion handling within the Maliit Wayland input method context. When Wayland sends surrounding text information via `zwp_input_method_context_v1_surrounding_text`, the Qt5 UTF-8 conversion function crashes, likely due to:

1. Invalid UTF-8 sequence in the input
2. Buffer overflow or memory corruption
3. Null pointer dereference
4. Race condition in the Wayland protocol handling

The recurring nature (2+ months, dozens of crashes) suggests this is a systematic bug rather than a random memory corruption issue.

## Workarounds

None identified. The crash appears to be deterministic based on certain input patterns.

Temporary mitigation:
```bash
# Restart the input method service
systemctl --user restart plasma-maliit.service
```

Or:
```bash
# Reinstall packages (may not fix the bug)
sudo dnf reinstall maliit-keyboard maliit-framework
```

## Upstream Links

- **Fedora Bugzilla:** https://bugzilla.redhat.com/enter_bug.cgi?product=Fedora&component=maliit-keyboard
- **Maliit Project:** https://github.com/maliit/keyboard
- **Maliit Framework:** https://github.com/maliit/framework

## Additional Information

To view crash details:
```bash
# List all maliit-keyboard crashes
coredumpctl list maliit-keyboard

# View details of most recent crash
coredumpctl info maliit-keyboard

# Extract full backtrace
coredumpctl gdb maliit-keyboard
```

To monitor for new crashes:
```bash
# Watch journal for maliit crashes
journalctl -f | grep maliit

# Show only error-level messages
journalctl -p err -f
```

---

**Report generated:** February 8, 2026  
**Reported by:** francisco  
**Co-Authored-By:** Warp <agent@warp.dev>
