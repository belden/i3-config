# Screen Lock Uptime Tracking Discussion

## Initial Script Review

The `i3-lock-screen` script creates a blurred/negated screenshot, locks the screen with i3lock, and includes fingerprint unlock functionality.

### Main Flow:
1. **Screenshot & blur** (line 3): Takes screenshot, negates colors, and blurs it
2. **Caffeine mode** (line 4): Calls `i3-caffeine tired` to manage screen power
3. **Force screen off** (line 5): Immediately turns off display with `xset dpms force off`
4. **Lock screen** (lines 57 or 65): Uses i3lock with the blurred screenshot
5. **Fingerprint unlock** (lines 17-46): Continuously polls for fingerprint match to unlock

### Key Features:
- **Fingerprint unlock loop** (`wait_fingerprint:17-46`): Tries fingerprint verification 5 times in sequence, with 5-second timeouts. If matched, kills i3lock. Forces display off between attempts.
- **XSS-lock integration** (lines 49-62): Handles proper sleep lock file descriptor management when called from xss-lock
- **Commented out code** (lines 7-14, 70-71): Previously handled DND mode, pausing media, and muting audio

## Bug Fixes Applied

### Line 57 Bug
**Issue**: Missing `$` before `XSS_SLEEP_LOCK_FD`
**Fixed**: Changed `{XSS_SLEEP_LOCK_FD}<&-` to `${XSS_SLEEP_LOCK_FD}<&-`

### Line 60 Bug
**Issue**: Same variable expansion issue
**Fixed**: Changed `exec {XSS_SLEEP_LOCK_FD}<&-` to `exec ${XSS_SLEEP_LOCK_FD}<&-`

Without the `$`, bash would try to close a file descriptor literally named `XSS_SLEEP_LOCK_FD` instead of the numeric value stored in that variable.

## Feature Request: Login/Logout Accounting

### Proposal
Register login records when unlocking the screen and logout records when locking it, so that `last` shows accounting records for usage.

### Solution: Using `sessreg`

The `sessreg` command (part of X11 utilities) can register login/logout records that will show up in `last`.

**For locking (logout record):**
```bash
sessreg -d -l "$DISPLAY" "$USER"
```

**For unlocking (login record):**
```bash
sessreg -a -l "$DISPLAY" "$USER"
```

The `-a` flag adds a login record, `-d` deletes (logout), and `-l` specifies the line (display name).

### Integration Points

1. Add the logout record right before locking (around line 57/65)
2. Add the login record after `wait_fingerprint` returns (after line 62/66)

### Caveats

`sessreg` typically requires root privileges. Options:
- Use `sudo` with NOPASSWD configuration for `sessreg`
- Make `sessreg` setuid root (security consideration)
- Use an alternative like writing to a custom log file that mimics the `last` format
