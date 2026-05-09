# 🛡️ Securing a Dual-Boot Windows 11 & Kali Linux System (Secure Boot + BitLocker)

**Tested Hardware:** Acer Aspire AG15-71P (BIOS V1.14)  
**Objective:** Enable Secure Boot and Windows BitLocker on an existing GRUB dual-boot setup without triggering `(0x1A) Security Violation` or SBAT revocation errors.

## ⚠️ The Problem Addressed
Modern Secure Boot implementations (specifically Shim 15.8+) strictly enforce "SBAT" (Secure Boot Advanced Targeting) metadata and will aggressively block unsigned or custom-signed bootloaders. Furthermore, Kali Linux repositories often lack pre-compiled, officially signed GRUB packages. 

This guide uses a hybrid approach: 
1. Using the official **Debian-signed bootloader** to satisfy the motherboard's strict Shim/SBAT checks.
2. Using a **custom Machine Owner Key (MOK)** to satisfy the bootloader's kernel checks, since Kali ships its kernels unsigned.

---

## 📋 Prerequisites
* A working dual-boot system with Windows 11 and Kali Linux (using GRUB).
* Secure Boot currently **Disabled** in the BIOS.
* Root access in Kali Linux (`sudo su`).

---

## Phase 1: Secure the Bootloader (The Debian Method)
We will extract Debian's officially signed bootloader, which is natively trusted by Microsoft's Secure Boot keys and contains the required SBAT metadata.

**1. Download and Extract the Signed Bootloader**
Open your Kali terminal and run:
```bash
# Download the official signed GRUB package from Debian
wget http://ftp.us.debian.org/debian/pool/main/g/grub-efi-amd64-signed/grub-efi-amd64-signed_1+2.14+2_amd64.deb

# Extract the contents without installing (to avoid package conflicts)
dpkg-deb -x grub-efi-amd64-signed_1+2.14+2_amd64.deb /root/grub_extracted

# Copy the signed bootloader to your active EFI partition
cp /root/grub_extracted/usr/lib/grub/x86_64-efi-signed/grubx64.efi.signed /boot/efi/EFI/kali/grubx64.efi
