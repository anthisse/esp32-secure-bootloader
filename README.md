# ESP32 Secure Bootloader and Games

## Notice: This project is in its early days and should be treated as such

## Abstract

The goal of this project is to demonstrate and experiment with the BootROM capabilities of the ESP32, and have some fun along the way by making some games for the SOC. This README.md exists to document my plans, progress, and thoughts on the project.

## Roadmap

1. Configure the toolchain
    - Install ESP-IDF
    - This will be considered completed when an [example program from the ESP IDF repository](https://github.com/espressif/esp-idf/tree/master/examples/get-started) is able to run successfully.

2. Partitioning
    - ⭐ We are here!
    - Determine how the USB port is wired to get into download mode rather than flashing the entire thing from a computer
    - Determine the partition layout
    - OTA data
    - OTA slot 0 & 1
    - Metadata partition
    - Storage partition
    - Flash a different program to each partition (LED blinker, UART dump, LCD colors)
    - Use `esptool` or `parttool`? Need to do more research
        - [ESP32 docs on partitions](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/partition-tables.html)
    - Optionally, a shared `include/` or `share/` directory between slots might be nice to have
    - This will be considered completed when both programs are able to be booted manually, logs appear over UART, and `idf.py partition-table` shows the correct layout

3. Slot selection and custom bootloader draft
    - Use some functions from within an application to watch the `otadata` change
        - [ESP32 docs on these functions](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/ota.html)
    - Override `bootloader_components`
         - [ESP32 docs on the bootloader](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/bootloader.html)
    - This will be considered completed when a slot switch from an application persists between resets.

4. Interim: Write the steering logic for the display, if not done already
    - A nice break from building the scaffolding for the setup.
    - Should be able to stop relying on UART for all of the textual output
    - Write a graphics driver for the display if required
        - Surely somebody has already done this...
    - This will be considered done when a simple [TV test pattern](https://en.wikipedia.org/wiki/Test_card) is able to be displayed on the LCD.
    - Optionally, start writing a game for this system. Some [source code for Asteroids on an RTOS](https://github.com/anthisse/G8RTOS) might already exist...

5. Signed image formatting
    - Need to choose a cryptographic algorithm. ED25519 or SHA-512?
        - [There might be a SHA hardware accelerator on this chip?](https://www.reddit.com/r/NerdMiner/comments/1iu7g8p/builtin_sha256_accelerator_on_the_esp32/)
    - Determine the format for the crytography header
        - TODO: I need to do some research on what would be good here
    - Images should be signed on a host machine; this will need a simple Python
        - *Stretch*: This would be a nice excuse to learn Rust, since the software should be relatively simple
    - Images that are detected as tampered as a result of the above should not run, and instead should print a UART error or an error to the display
    - This is considered done when:
        - Valid images boot and a corruption in any single cryptographic header field (i.e. bad key, bad signature, etc) causes the image to be rejected.
        - Regression tests for each of the above cases have been written.

6. Metadata & Logging
    - Create a `bootmeta` partition with an append-only log of records.
        - Slot states, attempt count, minimum security counter, etc
        - This log should be kept hardware independent, since we don't want to *actually* burn e-fuses yet; we do not want to lock ourselves out of the device!
    - TODO: I need to do some research on exactly what I would want in the logs
    - Make sure that the flashing process is robust against power failures or other interruptions
    - This is considered done when:
       - Software is written on a host machine that emulates the flash process
       - The ESP-32 does not accept data that is partially flashed
       - Regression tests for each of the above cases have been written

7. Flashing a new image
    - Write a new image while the system is online via USB/UART and verify that the image flashed successfully
        - [ESP docs on OTA flashing](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/ota.html)
        - TODO Flashing the SRAM might cause a performance degradation for the actively running software. While not catastrophic, it would be nice to fix this potential problem.
    - Measure the total update time and try to optimize it.
    - This is considered done when:
       - An update successfully completes
       - Regression tests for each of the above cases have been written

8. Rollback
    - A new image should have to prove itself before it is trusted. If it can't, the device must return to the last known-good slot on its own.
    - Enable ESP-IDF's app rollback support (`CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE`) and use its image states (new, pending verify, valid, invalid, aborted)
        - [ESP32 docs on OTA rollback](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/ota.html)
        - Need to confirm the exact function names and state transitions for the pinned ESP-IDF version
    - Define the lifecycle of a freshly flashed image:
        1. Update verified, slot marked pending
        2. Bootloader boots it and counts the attempt in `bootmeta`
        3. App runs its self-check, then marks itself valid and appends `CONFIRMED` to `bootmeta`
        4. If the app resets before confirming, the bootloader counts another attempt. After the attempt limit, the slot is marked invalid and the previous slot boots
    - Needs some critera for exactly what a self-check/POST actually means
       - App has been running health for some $x$ number of seconds?
    - Keep the `bootmeta` write ordering from the metadata milestone (record intent in `bootmeta` first, then `otadata`) so that a power cut between the two is recoverable
    - Enable the task watchdog with panic & reset behavior
    - Don't start a new update while the running image is still unconfirmed
    - Decide what happens when there is no valid fallback (first install, or both slots invalid).
    - Recovery is `esptool` in ROM download mode if all else fails
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

9. Anti-Rollback
    - Use `bootmeta` information to determine if a flash is attempting to write an older version of the software
    - Implement some flags to allow for a downgrade in certain conditions
        - Might also be nice to create a dev build with an obvious debug banner
    - We still aren't going to flash actual e-fuses sicne we don't want to brick the system
        - TODO allowing for downgrades is incongruent with physical e-fuse behavior.
        - Obviously, this doesn't *actually* make the board at all secure. If an attacker writes the flash directly with `esptool` or similar (like how we're doing in our development process), the security is completely bypassed. This is intractable with a software counter strategy, but is a worthy tradeoff for this project for now, since each board isn't cheap (~$40 at the time of writing this), and burning all the e-fuses effectively bricks the board.
    - This is considered done when
        - A lower version number image is refused and a signed override is accepted, resetting the floor
        - A dev banner has been implmented when downgrade protections have been disabled
        - Regression tests for each of the above cases have been written

10. Fault injections
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

11. Software
     - The fun part! Wire up some games with the touchscreen, buttons, and more to show off how the system works in a real demo.
         - External devices like a keyboard, generic game controller, tactile switches, and more could be attached and used to make the games more fun!
         - Online firmware updates might cause the CPU to get overwhelmed, so we need to make sure that this doesn't lag out the system
         - This is considered done when at least two games have been written and can be played without or crashing, noticable lag, etc during an update to the other slot

12. CI/CD, documentation, further testing, and chores
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
