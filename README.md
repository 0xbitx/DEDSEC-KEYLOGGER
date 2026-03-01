

<p align="center">
<img src="https://media0.giphy.com/media/v1.Y2lkPTc5MGI3NjExajJ2bDNnOXNvZHRhNGtudmdyeXU0OTVmc2Y4ZDQ3cHluanAyamRpbSZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/6Rp9ZYEy13cKhHZH5Z/giphy.gif", width="400", height="400">
</p>

<h1 align="center">D A K</h1>
</p><h4 align="center">A sophisticated Linux-based polymorphic keylogger builder with Google Calendar C2 infrastructure</h4><p align="center">
  
## DESCRIPTION

DAK (DEDSEC ADVANCED KEYLOGGER) is a professional-grade keylogging framework operating at ring 0 (kernel level) through a Loadable Kernel Module (LKM). It captures keystrokes directly from the keyboard interrupt handler using the Linux input subsystem, bypassing userspace detection mechanisms entirely. The module writes captured data to memory for stealthy storage. The tool leverages Google Calendar as a dead-drop resolver for C2 communications with configurable beacon intervals to evade packet analysis and network-based detection.

## KEY FEATURES

  * Polymorphic Builder	Generates unique payloads with each build - no two binaries are identical
  * Google Calendar C2	Uses Google Calendar API as dead-drop resolver for command & control
  * Stealth Operation	No suspicious network traffic - blends with Google API calls
  * Persistence Mechanism	Maintains operation across system reboots
  * Kernel-Mode Keylogging	LKM operates in ring 0, capturing keystrokes via register_keyboard_notifier()
  * Junk Code Injection: Random decoy classes and functions are inserted to confuse analysis
  * Input Subsystem Hooking	Registers a keyboard notifier chain to capture all keyboard events
  * Shift-Aware Capture	Handles shifted keys (uppercase, symbols) with proper mapping
  * Stealth Storage	Writes to memory (RAM-based filesystem, no disk writes)
  * Dual-Buffering System	Double buffering prevents data loss during write operations
  * Workqueue Architecture	Asynchronous write operations don't block keyboard interrupts
  * Code Injection	Injects kernel module loader into legitimate Python applications, maintaining original functionality while adding stealthy keylogging capabilities

## Command & Control
  * Multi-Machine Management: Control multiple infected hosts simultaneously
  * Real-time Shell Access: Interactive shell on target machines
  * Keylog Dump: Retrieve captured keystrokes on demand
  * Start/Stop Control: Remotely control keylogger status
  * Machine Fingerprinting: Detailed system information collection (hostname, IP, MAC, hardware specs, OS, users, processes)

### C2 Communication Flow
```text
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────────────────┐
│   DAK Builder   │────▶│  Google Calendar │────▶│       Target Machine        │
│  (Attacker VM)  │     │   (C2 Channel)   │     │         (Victim)            │
└─────────────────┘     └──────────────────┘     └─────────────────────────────┘
         │                        │                            │
  Builds kernel module     Events store:              ┌─────────────────┐
   with beacon config:      - Machine identity        │  Userspace      │
  - Normal (instant)        - Encrypted commands      │  Beacon Daemon  │
  - Beacon (10-600s)        - Keylog output           │  (Timer-based)  │
                                                      └────────┬────────┘
                                                               │
                                                      ┌────────▼────────┐
                                                      │  Kernel Module  │
                                                      │    (ring 0)     │
                                                      └────────┬────────┘
                                                               │
                                                      ┌────────▼────────┐
                                                      │  Keyboard       │
                                                      │  Notifier Chain │
                                                      └─────────────────┘
                                                               │
                                                      ┌────────▼────────┐
                                                      │      Memory     │
                                                      └─────────────────┘
```

### Kernel Module Workflow
```text
┌────────────────────────┐
│ keyboard_notifier_call │
│ (ring 0 - IRQ context) │
└────────────────────────┘
        │
        ├─► Check key_param->down (only capture key presses)
        ├─► Map keycode to string via input_code_to_string()
        ├────► Uses shift state (key_param->shift) for uppercase/symbols
        ├─► Write to input_buffer (side_buffer)
        ├─► If buffer nearly full, call flush_buffer()
        └─► Return NOTIFY_OK (allow normal keyboard processing)
        │
        ▼
┌───────────────────────┐
│    flush_buffer()     │
└───────────────────────┘
        │
        ├─► Swap buffers: input_buffer ↔ write_buffer
        ├─► schedule_work(&handler->writer_task) → defer to workqueue
        └─► Return immediately (interrupt handler can continue)
        │
        ▼
┌───────────────────────┐
│   write_debug_task    │
│   (workqueue context) │
└───────────────────────┘
        │
        ├─► kernel_write() to memory
        ├─► Update file offset (handler->file_off)
        ├─► Clear write_buffer
        └─► Return (I/O complete, not blocking interrupts)
        │
        ▼
┌───────────────────────┐
│   Userspace Beacon    │
│   Daemon (optional)   │
└───────────────────────┘
        │
        ├─► Reads memory
        ├─► If beacon mode: sleep(interval + jitter)
        ├─► Check Google Calendar for commands
        └─► Execute commands if present
```        

## SETUP
Setup (Google Calendar C2)

Follow the steps below to configure your Google Cloud project and enable Google Calendar API for C2 communication.

1. Create a Google Cloud Project and Service Account

    Go to: https://console.cloud.google.com

    From the sidebar, go to IAM & Admin → Service Accounts

    Click + Create Project
     * Project name: (e.g.,) rat-calendar-c2
     * Click Create

    Click + Create Service Account
     * Service account name: e.g. test-c2
     * Description: (optional)
     * Click Create and Continue

    Grant this service account access:
      * Role: Owner
      * Click Continue → Done

    Click your newly created service account
      * Go to the Keys tab
      * Click Add Key → Create New Key
      * Select JSON, then click Create
      * Save the file as: creds.json in your directory

3. Share Google Calendar with Service Account

    Visit: https://calendar.google.com

    On the left, click the 3-dot menu beside your calendar → Settings and sharing

    Scroll to Share with specific people
     * Click + Add people and groups
     * Paste your Service Account email
     * Example: test-c2@your-project-id.iam.gserviceaccount.com
     * Set permission to: Make changes and manage sharing
     * Click Send

5. Enable Google Calendar API

     * Go to: https://console.cloud.google.com/apis/library
     * Search for: Google Calendar API
     * Click it → Click Enable


### INSTALLATION
    * git clone https://github.com/0xbitx/DEDSEC_KEYLOGGER.git
    * cd DEDSEC_KEYLOGGER
    * sudo pip3 install tabulate --break-system-packages
    * chmod +x dedsec_keylogger
    * sudo ./dedsec_keylogger

### TESTED ON FOLLOWING
* Kali Linux 
* Parrot OS 
* Ubuntu

## Support

If you find my work helpful and want to support me, consider making a donation. Your contribution will help me continue working on open-source projects.

**Bitcoin Address: `36ALguYpTgFF3RztL4h2uFb3cRMzQALAcm`**
   
<h1 align="center"> DISCLAIMER </h1>

<h4 align="center">I'm not responsible for anything you do with this program, so please only use it for good and educational purposes. </h4>

