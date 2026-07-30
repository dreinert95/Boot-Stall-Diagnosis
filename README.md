# Boot-Stall-Diagnosis

This write up will explain the process and procedures of diagnosing a 45-second boot stall on Fedora 44 after an SSD swap.

# Summary

The machine that this was performed on is a Lenovo 80QF with an Intel i5-6200U CPU.

- After replacing the HDD with a Team Group T-Force 2.5 inch SATA SSD 1TB (Vulcan Z) and doing a clean install of Fedora 44, boot time was sitting around 114 seconds, way longer than expected for a new 1TB SSD. The process of elimination traced the delay to a single failure: 45 seconds for a TPM device (/dev/tpm0) that never responded, because Intel PTT (Platform Trust Technology) was misbehaving at the firmware level. Disabling PTT in BIOS and cleaning up the related systemd units dropped the total boot time to about 30 seconds.

# Symptom

- Fresh SSD, fresh Fedora 44 install, boots to GRUB and the kernel loads fine
- Long pause after the Fedora splash before reaching the desktop
- No visible errors, no crash, and no obvious failure

# Step 1: Establish a baseline

- First and foremost, it was necessary to actually measure the boot time.
  - Bash: systemd-analyze
  - Code: Startup finished in 7.441s (firmware) + 6.174s (loader) + 1.373s (kernel) + 45.570s (initrd) + 53.390s (userspace) = 1min 53.950s
- This baseline isolated exactly where the stall time was at: 45.570 seconds sitting in initrd (the stage before the real root filesystem is even mounted). Everything after that (firmware, loader, kernel) was normal.

# Step 2: Verify that initrd is the cause

- 
