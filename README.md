# ESP32 Secure Bootloader and Games

## Notice: This project is in its early days and should be treated as such

## Abstract

The goal of this project is to demonstrate and experiment with the BootROM capabilities of the ESP32, and have some fun along the way by making some games for the SOC. This README.md exists to document my plans, progress, and thoughts on the project.

## Roadmap

1. Configure the toolchain
    - Install ESP-IDF
    - This will be considered completed when an [example program from the ESP IDF repository](https://github.com/espressif/esp-idf/tree/master/examples/get-started) is able to run successfully.

2. Partitioning
    - Determine how the USB port is wired to get into download mode rather than flashing the entire thing from a computer
    - Write a custom `partitions.csv` with:
        - `otadata`
        - `ota_0` and `ota_1`
            - These will be the application slots
        - `bootmeta` data partition for the custom metadata log
        - Storage partition
    - Flash a different program to each slot (UART dump, LCD colors)
        - Use `esptool` or `parttool`? Need to do more research
        - [ESP32 docs on partitions](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/partition-tables.html)
        - TODO: Research offsets & alignment rules for the partition layouts
            - 64 KiB alignment?
    - Optionally, a shared `include/` or `share/` directory between slots might be nice to have
    - This will be considered completed when both programs are able to be booted manually, logs appear over UART, and `idf.py partition-table` shows the correct layout

3. Stock OTA and rollback baseline
    - ⭐ We are here!
    - Before writing any custom bootloader code, get ESP-IDF's built-in A/B update and rollback working.
        - We'll compare against this for any new bootloader code later
    - Use some functions from within an application to switch slots and watch the `otadata` change
        - `esp_ota_get_next_update_partition`, `esp_ota_begin`, `esp_ota_write`, `esp_ota_end`, `esp_ota_set_boot_partition`
        - [ESP32 docs on these functions](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/system/ota.html)
    - Enable stock app rollback (`CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE`) and have the app determine validity after a self-check
    - Enable signed-app verification without hardware Secure Boot (`CONFIG_SECURE_SIGNED_APPS_NO_SECURE_BOOT`) to avoid burning efuses
    - Write a few deliberately bad apps (hang, panic, no confirm, etc) and record what stock behavior does with each
        - Record baseline numbers: boot-to-app time, bootloader size, update throughput, and total update time
    - This will be considered completed when:
        - A slot switch from an application persists between resets
        - An update via the stock path completes and the device boots the new slot
        - Each bad app is rolled back by stock behavior, or the exceptions are documented
        - Baseline numbers are recorded in a README.md in a child directory

4. Interim: Write the steering logic for the display, if not done already
    - A nice break from building the scaffolding for the setup.
    - Should be able to stop relying on UART for all of the textual output
    - This will be considered done when a simple [TV test pattern](https://en.wikipedia.org/wiki/Test_card) is able to be displayed on the LCD.
    - Optionally, start writing a game for this system. Some [source code for Asteroids on an RTOS](https://github.com/anthisse/G8RTOS) might already exist...

5. Custom bootloader component
    - Override the second-stage bootloader with a `bootloader_components` directory
        - [ESP32 docs on the bootloader](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/bootloader.html)
    - Print a banner from it while checking
    - The bootloader stays immutable for this project
    - This will be considered completed when the custom banner appears on every boot and the slot-switching behavior from the baseline still works.

6. Signed image formatting
    - Algorithm decisions:
        - We'll use ED25519, using Monocypher. ESP-IDF's hardware Secure Boot v2 on the S3 uses RSA-3072, not Ed25519, so do note that this is a software layer on top and not the hardware RoT.
        - Hashing will use SHA-256, since there apparently is a hardware accelerator for this algorithm.
            - TODO: confirm hardware SHA is usable inside the bootloader. (Ed25519 uses SHA-512 internally)
    - Header format: a signed 64-byte header placed after the ESP app image, since ESP-IDF needs its own header at the start of the image. Fields are little-endian and serialized byte by byte, not copied from a struct.
        - Though `alignas` or `#pragma pack` could probably make it so that byte-by-byte serialization is not necessary

        | Field | Notes |
        | --- | --- |
        | `magic`, `header_version`, `header_len` | Reject garbage and unknown formats early |
        | `product_id` | Stops an image built for another project from booting |
        | `image_size` | Exact length of the esptool-produced image |
        | `version` | Version number |
        | `security_counter` | Anti-rollback |
        | `flags` | Includes `ALLOW_DOWNGRADE`; reserved bits must be zero |
        | `image_hash` | SHA-256 of the image |
        | `signature` | Ed25519 over everything before it |

    - Verification order: sanity checks, then the signature over the header, then use the (now trusted) size and hash fields
    - TODO: work out how the trailer coexists with ESP-IDF's own image checksum and SHA-256, and with the secure padding and signature block that hardware Secure Boot would add later
    - Images should be signed on a host machine; this will need a simple Python tool
        - *Stretch*: This would be a nice excuse to learn Rust, since the software should be relatively simple
    - Compile the same verification code on the PC and test it against Python-produced signatures before putting it on the board
    - Images that are detected as tampered as a result of the above should not run, and instead should print a UART error or an error to the display
    - Measure verify time with a GPIO toggle and compare it against the built-in path
    - This is considered done when:
        - Valid images boot and a corruption in any single header field or in the image (bad key, bad signature, flipped byte, etc) causes the image to be rejected.
        - Regression tests for each of the above cases have been written.

7. Metadata & Logging
    - Create a `bootmeta` partition with an append-only log of fixed-size records.
        - Sequence number, slot states, attempt count, minimum security counter, flags, etc
    - Keep the log code hardware independent behind a small flash interface. This lets it be tested against a flash simulator on a host machine, and it means no eFuses are needed; we do not want to lock ourselves out of the device!
    - Design for two sources of truth: `otadata` (which slot ESP-IDF will boot) and `bootmeta` (policy). Write intent to `bootmeta` first, then update `otadata`, and reconcile at boot if they disagree.
    - A corrupted record must be counted as used space and never overwritten. On rollover, always erase the other sector before using it.
    - Make sure that the flashing process is robust against power failures or other interruptions
    - This is considered done when:
       - Software is written on a host machine that emulates the flash process
       - Crashes between the `bootmeta` and `otadata` writes are covered, and the reconcile step repairs them
       - The ESP-32 does not accept data that is partially flashed
       - Regression tests for each of the above cases have been written

8. Flashing a new image
    - Write a new image while the system is online via USB/UART and verify that the image flashed successfully
        - [ESP docs on OTA flashing](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/ota.html)
        - Flashing the SRAM might cause a performance degradation for the actively running software. While not catastrophic, it would be nice to fix this potential problem.
    - Measure the total update time and try to optimize it.
    - This is considered done when:
       - An update successfully completes
       - Throughput and total update time are recorded and compared with the stock baseline
       - Regression tests for each of the above cases have been written

9. Rollback
    - A new image should have to prove itself before it is trusted. If it can't, the device must return to the last known-good slot on its own.
    - Build on ESP-IDF's image states (new, pending verify, valid, invalid, aborted) and add the policy stock does not have: stock gives a new app one boot attempt, so add an attempt limit in `bootmeta`
        - [ESP32 docs on OTA rollback](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/system/ota.html)
        - Need to confirm the exact function names and state transitions for the pinned ESP-IDF version
    - Define the lifecycle of a freshly flashed image:
        1. Update verified, slot marked pending
        2. Bootloader boots it and counts the attempt in `bootmeta`
        3. App runs its self-check, then marks itself valid and appends `CONFIRMED` to `bootmeta`
        4. If the app resets before confirming, the bootloader counts another attempt. After the attempt limit, the slot is marked invalid and the previous slot boots
    - Needs some criteria for exactly what a self-check/POST actually means.
       - Display and touch initialized?
       - The other slot's header is readable?
       - App has been running healthily for some $x$ number of seconds?
    - Keep the `bootmeta` write ordering from the metadata milestone (record intent in `bootmeta` first, then `otadata`) so that a power cut between the two is recoverable
    - Enable the task watchdog with panic & reset behavior, and confirm that a panic actually reboots
    - Don't start a new update while the running image is still unconfirmed
    - Decide what happens when there is no valid fallback (first install, or both slots invalid).
    - Recovery is `esptool` in ROM download mode if all else fails. This only applies while the eFuse security features are off.
    - Test apps for failures:
        - Hangs after boot
        - Panics after boot
        - Boots fine but never confirms
        - Fails self-check and requests rollback
        - Crashes only on the second boot (checks that the attempt count persists)
    - This is considered done when:
        - Each bad app triggers a reset and the bootloader falls back to the last confirmed slot within the attempt limit
        - A good app confirms, and its state survives a power cycle
        - `bootmeta` and `otadata` agree after every rollback
        - The attempt limit and self-check definition are documented
        - Regression tests for each of the above cases have been written

10. Anti-Rollback
    - Keep a minimum `security_counter` in `bootmeta` and use it to determine if a flash is attempting to write an older version of the software
    - Raise the minimum only when an image reaches confirmed, so if a new image fails it can never lock out the old known-good one
    - Implement flags to allow a downgrade in certain conditions
        1. A signed `ALLOW_DOWNGRADE` flag in the header (only the key holder can produce it). Accepting it resets the stored minimum.
        2. A dev build flag (`DEV_IGNORE_ANTIROLLBACK`) that prints a debug banner
        3. Optionally, a physical-presence command using the BOOT button, with the tradeoff documented
    - We still aren't going to burn actual eFuses since we don't want to brick the system
        - ESP-IDF's eFuse anti-rollback is one-way and limited to 16 increments, so allowing downgrades is incongruent with physical eFuse behavior
        - Obviously, this doesn't *actually* make the board at all secure. If an attacker writes the flash directly with `esptool` or similar (like how we're doing in our development process), the security is completely bypassed. This is intractable with a software counter strategy, but is a worthy tradeoff for this project for now, since each board isn't cheap (~$40 at the time of writing this), and burning all the eFuses effectively bricks the board.
    - This is considered done when
        - A lower version number image is refused and a signed override is accepted, resetting the floor
        - A dev banner has been implmented when downgrade protections have been disabled
        - Regression tests for each of the above cases have been written

11. Fault injections
     - The system needs to survive and continue working, even in the case of a power loss
         - Power loss during sector erase
         - Power loss during programming
         - Power loss during `bootmeta` log append
         - Power loss during `otadata` write
     - Apparently the ESP32 has a brownout detector? Could be useful.
     - There are a lot of test cases to consider here, and each one needs to be individually solved to ensure the system is at least reasonably robust
         - This seems to call for some automation
         - Probably a good idea to pull a pin high before a firmware operation so we can cut power from a power supply when an oscilloscope detects when the pin goes high
     - This is considered done when
         - A reasonably large number of trials (a few hundred?) are all passing
         - Regression tests for each of the above cases have been written
             - Since these tests require more equipment, namely a power supply and an oscilloscope, it might be a good idea to mark these tests optional after the initial testing set.

12. Software
     - The fun part! Wire up some games with the touchscreen, buttons, and more to show off how the system works in a real demo.
         - External devices like a keyboard, generic game controller, tactile switches, and more could be attached and used to make the games more fun!
         - Online firmware updates might cause the CPU to get overwhelmed, so we need to make sure that this doesn't lag out the system
         - This is considered done when at least two games have been written and can be played without or crashing, noticable lag, etc during an update to the other slot

13. CI/CD, documentation, further testing, and chores
     - Once the above milestones have been reached, the project itself is in a finished state, however, good documentation and tests are nice to have for anybody else interested in the project.
     - This is considered done when
         - GitHub actions (or similar) are able to build the applications, the bootloader, and run the previously written tests.
         - `clang-tidy` and `clang-format` are able to be run on the repository to standardize the code style
         - A nice README.md with block diagrams, flash layout, etc has been created
         - A writeup of a project postmortem has been written
             - What went well, what was difficult, areas where collaboration would have been helpful, and potential attacks against the security features of the project
         - A demo video showing off the software (games) and the features of each previous milestone

### Stretch Goals

- Create a Rust program for firmware signing
- Pico 8 or similar images being able to be run on the system
- Utilization of the MicroSD card slot to flash games without being tethered to a computer
  - This means that one of the firmware slots needs to hold a filesystem manager so that an end user can navigate the SD card
- Physical e-fuse burning
  - Scary, but is a logical next step for the project
- Ability to run a shell language like `zsh` or `bash`
- WiFi updates for remote flashing

## Required Materials

Check the [Amazon list.](https://www.amazon.com/hz/wishlist/ls/JVE4DMFIOHV?ref_=wl_share)

## Setup Guide

1. Install ESP IDF.

    - Follow Espressif's [Get Started Guide.](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/get-started/index.html#)
    - If you want to build from source on Arch Linux, you may find [this guide](https://wiki.archlinux.org/title/ESP32) useful. Note that the AUR package as of 2026-10-17 installs ESP IDF 6.0.2 instead of the latest 6.1 release. Alternatively, follow the [Arch Linux installation guide](https://docs.espressif.com/projects/idf-im-ui/en/latest/#arch-linux-installation-via-pacman-repository)
    - Note that I will be using [CLion's integration with ESP IDF and PlatformIO](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/third-party-tools/clion.html) to develop for this project.
    - See the [installation process README.md](milestone-1-esp-idf-example-project/README.md)

2. Gather all the parts for the project:

| Item                                                                                                                                                                     | Brand     | Price  | Qty |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------|--------|-----|
| [USB A to USB C Cables](https://www.amazon.com/s?k=TAKAGI+USB+C+Cable+3ft+2-Pack+USB+A+to+USB+C+nylon+braided)                                                           | TAKAGI    | $5.99  | 1   |
| [USB C to USB 3.0 Adapter](https://www.amazon.com/s?k=JSAUX+USB+C+to+USB+3.0+Adapter+2+Pack+OTG)                                                                         | JSAUX     | $6.99  | 1   |
| [Tactile Push Button Switches](https://www.amazon.com/s?k=uxcell+12x12x12mm+4+Pin+Tactile+Push+Button+Switch+DIP+10PCS)                                                  | uxcell    | $6.69  | 1   |
| [32GB Ultra microSDHC](https://www.amazon.com/s?k=SanDisk+32GB+Ultra+microSDHC+120MB%2Fs+A1+Class+10+UHS-I)                                                              | SanDisk   | $22.79 | 1   |
| [Breadboard Set with Jumper Wires](https://www.amazon.com/s?k=BOJACK+3+Values+130+Pcs+Solderless+Breadboard+830+400+Tie+Points+Jumper+Wires)                             | BOJACK    | $9.99  | 1   |
| [ESP32-S3 4.3inch Capacitive Touch LCD Development Board, 800×480](https://www.amazon.com/s?k=Waveshare+ESP32-S3+4.3inch+Capacitive+Touch+LCD+Development+Board+800x480) | Waveshare | $40.31 | 1   |

Total: $92.76 as of 2026-10-1
