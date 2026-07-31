# Boot-Stall-Diagnosis

This write up will explain the process and procedures of diagnosing a 45-second boot stall on Fedora 44 after an SSD swap.

# Summary

The machine that this was performed on is a Lenovo 80QF with an Intel i5-6200U CPU.

- After replacing the HDD with a Team Group T-Force 2.5 inch SATA SSD 1TB (Vulcan Z) and doing a clean install of Fedora 44, boot time was sitting around 114 seconds, way longer than expected for a new 1TB SSD. The process of elimination traced the delay to a single failure: 45 seconds for a TPM device (/dev/tpm0) that never responded, because Intel PTT (Platform Trust Technology) was misbehaving at the firmware level. Disabling PTT in BIOS and cleaning up the related systemd units dropped the total boot time to about 30 seconds.

# Symptoms

- Fresh SSD, fresh Fedora 44 install, boots to GRUB and the kernel loads fine
- Long pause after the Fedora splash before reaching the desktop
- No visible errors, no crash, and no obvious failure

# Step 1: Establish a baseline

- First and foremost, it was necessary to actually measure the boot time.
  - Bash: systemd-analyze
  - Code: Startup finished in 7.441s (firmware) + 6.174s (loader) + 1.373s (kernel) + 45.570s (initrd) + 53.390s (userspace) = 1min 53.950s
- This baseline isolated exactly where the stall time was at: 45.570 seconds sitting in initrd (the stage before the real root filesystem is even mounted). Everything after that (firmware, loader, kernel) was normal.

# Step 2: Verify that initrd is the cause

- _systemd-analyze blame_ and _systemd-analyze critical-chain_ showed every real service completing in milliseconds, nothing in userspace was slow and thus verifying that _initrd_ was the isolated cause. Therefore it was necessary to check the most common causes of a silent _initrd_ stall.
- Check the storage/SATA layer:
  - Bash: sudo dmesg | frep -i -E "ata|sata|ahci"
  - Result: SATA link negotiated at the full 6.0 Gbps, the drive was detected and mounted within 1.2 seconds
  - Ruled Out: SSD and controller
- Check disk encryption (LUKS):
  - Bash: lsblk -f
  - Result: Plain ext4/btrfs/vfat and no crypto_LUKS anywhere
  - Ruled Out: disk encryption
- Check for stale fstab/UUID references:
  - Bash: cat /etc/fstab
  - Result: All UUIDs matched _lsblk_ output cleanly, no stale entries
  - Ruled Out: stale fstab/UUID
- Check Entropy/RNG stall:
  - Bash: sudo dmesg | grep -i "crng\|random"
  - Result: _crng init_ done at 0.072s
  - Ruled Out: entropy/RNG
- Check Hibernation resume at the kernel command line
  - Bash: cat /proc/cmdline
  - Result: no resume=parameter
  - Ruled Out: Hibernation resume at the kernel command line
 
# Step 3: Watch the raw boot text directly

  1. At GRUB, press "e" to edit the boot entry
  2. Remove _rhgb quiet_ from the kernel line
  3. Boot with the edit (one-time, not persisted)

     - [***] (1 of 2) Job dev-tpm0.device/start running (22s / 45s)
       - This indicates that the system was waiting on a TPM device unit with a 45 second timeout (matching the stall time almost exactly)

# Step 4: Confirm and fix

- Confirm the issue:
- Bash: systemctl status dev-tpm0.device
- Result: Loaded: masked (Reason: Unit dev-tpm0.device is masked.)
          Active: inactive (dead)
- This confirmed that the device unit was present but _inactive (dead)_
  - systemd expected a TPM and never got a response from one
 
- First fix:
- Bash: sudo systemctl mask dev-tpm0.device
- Bash: sudo dracut -f --regenerate-all
- Result: the boot time was reduced from ~114 seconds to ~ 66 seconds. The journal showed that the device was still timing out despite being masked in the main system, because _dracut's initrd_ carries its own embedded copy of systemd unit state, separate from the one on the real root filesystem. A mask applied without a subsequent _dracut -f_ rebuild (or applied to the wrong kernel's _initramfs_ won't take effect during the _initrd_ phase.

  - Further investigation:
  - Bash: systemctl list-units --all | grep -i tpm
  - Result: systemd-tpm2-setup.service, systemd-tpm2-setup-early.service, systemd-pcrphase-initrd.service, etc.
    - Fedora's measured-boot chain, which attempts to record PCR (Platform Configuration Register) measurements into the TPM at each boot stage. Several of these were still active and independently capable of waiting on TPM hardware, regardless of the device mask.
   
- Actual fix:
  - The actual fix was at the firmware level: disabling Intel PTT (Intel's firmware-based TPM implementation) in BIOS. This stopped the kernel from ever presenting a TPM device to the system at all, therefore nothing downstream had anything to wait on.
  - Bash: sudo systemctl mask systemd-pcrphase-initrd.service
  - Bash: sudo dracut -f --regenerate-all
  - Bash: systemctl reboot
  - Result: Startup finished in 6.813s (firmware) + 3.573s (loader) + 1.376s (kernel) + 3.068s (initrd) + 11.406s (userspace) = 26.240s
    - _initrd_ dropped from 45.7s to 3.1s
    - total boot time dropped from ~114s to ~26s

# Root Cause

- The laptop's Intel PTT (firmware TPM) wasn't responding correctly to the kernel, but Fedora's measured-boot systemd units were configured to wait for it anyway, with a hard 45 second timeout before giving up. Since no disk encryption or TPM-backed key storage was in use on this system, there was no functional loss in disabling PTT. Disabling PTT eliminated an unnecessary and non-functional wait.

# Tools used

- _systemd-analyze_
- _systemd-analyze blame_
- _systemd-analyze critical-chain_
- _dmesg_
- _jouralctl_
- _lsblk_
- _dracut_
- _systemctl mask_
- GRUB boot editing
