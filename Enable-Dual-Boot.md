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
```

**2. Fix the GRUB Menu Paths**
Because the bootloader is from Debian, it looks for a configuration file in a `debian` folder. We must create this folder and set up an automated pointer so you don't get stuck at a `grub>` prompt.
```bash
# Create the Debian EFI directory
mkdir -p /boot/efi/EFI/debian

# Create an automated pointer script in the Kali folder
cat > /boot/efi/EFI/kali/grub.cfg << 'EOF'
search.file /boot/grub/grub.cfg root
set prefix=($root)/boot/grub
configfile ($root)/boot/grub/grub.cfg
EOF

# Copy the pointer to the Debian folder
cp /boot/efi/EFI/kali/grub.cfg /boot/efi/EFI/debian/grub.cfg
```

---

## Phase 2: Secure the Kali Kernels (The MOK Method)
The Debian bootloader will refuse to load Kali's unsigned kernels. We must act as our own Certificate Authority, create a Machine Owner Key (MOK), and sign the kernels ourselves.

**1. Generate the Custom Key**
```bash
mkdir -p /root/secureboot-keys
cd /root/secureboot-keys

# Generate a 10-year key
openssl req -new -x509 -newkey rsa:2048 -keyout MOK.priv -out MOK.pem -nodes -days 3650 -subj "/CN=Kali Custom MOK/"

# Convert it to DER format for the motherboard
openssl x509 -in MOK.pem -out MOK.der -outform DER
```

**2. Sign the Kernels**
```bash
# Install the signing tool
apt install -y sbsigntool

# Sign all existing kernels in the /boot directory
for kernel in /boot/vmlinuz-*; do sbsign --key MOK.priv --cert MOK.pem "$kernel" --output "$kernel"; done
```

**3. Stage the Key for Motherboard Enrollment**
```bash
mokutil --import MOK.der
```
> **Note:** You will be prompted to create a simple, one-time password. Remember this for the next reboot.

---

## Phase 3: Enable Secure Boot
**1. Configure the BIOS**
* Restart the laptop and enter the BIOS (usually **F2** or **Del**).
* *(Acer Specific)*: Go to the **Security** tab and set a **Supervisor Password** to unlock Secure Boot options.
* Go to the **Boot** tab and change **Secure Boot** to **Enabled**.
* Press **F10** to Save and Exit.

**2. Enroll the Key in MokManager**
* Upon reboot, a blue screen (**MOK management**) will appear.
* Press any key to enter the menu.
* Select **Enroll MOK** -> **Continue** -> **Yes**.
* Enter the one-time password you created in Phase 2.
* Select **Reboot**. The system will now boot securely into Kali Linux.

---

## Phase 4: Enable Windows BitLocker
BitLocker will fail to encrypt if it detects GRUB in the boot path during its initial hardware test. We must temporarily give Windows direct control of the boot process.

**1. Give Windows Direct Boot Priority**
* Restart and enter the BIOS (**F2**).
* In the **Boot** tab, move **Windows Boot Manager** to the very top of the priority list (Position 1).
* Press **F10** to Save and Exit. The laptop will boot directly into Windows, bypassing GRUB.

**2. Turn on BitLocker**
* Log into Windows 11.
* Open **Manage BitLocker** and click **Turn on BitLocker**.
* ⚠️ **CRITICAL:** Save your Recovery Key to a USB drive or your Microsoft Account. You *will* need this.
* Check the box to **Run BitLocker system check** and restart. 
* BitLocker will pass its hardware test and begin encrypting the drive in the background.

**3. Restore GRUB and Update the TPM**
* Restart and enter the BIOS (**F2**).
* Move **KaliLinux** back to the top of the priority list (Position 1). Press **F10**.
* The GRUB menu will appear. Select **Windows Boot Manager**.
* Windows will detect that the bootloader has changed and will ask for your **BitLocker Recovery Key**. 
* Enter the long numeric key. This explicitly tells the TPM chip to trust GRUB. 

**Result:** You can now dual-boot seamlessly with Secure Boot and BitLocker fully active!

---

## Phase 5: GRUB Polish (Optional)
To ensure Windows is detected and set as the default OS with a 5-second timer:

```bash
# 1. Enable OS Prober to find Windows
sed -i 's/^#GRUB_DISABLE_OS_PROBER=false/GRUB_DISABLE_OS_PROBER=false/' /etc/default/grub
echo "GRUB_DISABLE_OS_PROBER=false" >> /etc/default/grub

# 2. Set Windows as default and timeout to 5 seconds
sed -i 's/^GRUB_DEFAULT=.*/GRUB_DEFAULT="Windows Boot Manager (on \/dev\/nvme0n1p1)"/' /etc/default/grub
sed -i 's/^GRUB_TIMEOUT=.*/GRUB_TIMEOUT=5/' /etc/default/grub

# 3. Update GRUB
update-grub
```

---

## Phase 6: Maintenance & Major Updates (CRITICAL)
Because Kali Linux ships its kernels unsigned, **any time you run a major update (`apt upgrade` or `apt dist-upgrade`) and a new Linux kernel is installed, it will fail to boot** (throwing a *"you need to load the kernel first"* error). Additionally, GRUB updates might overwrite our Debian-signed bootloader.

To fix this, you must re-sign the new kernels and restore the Debian bootloader after major updates. 

### The Post-Update Script
To automate this, create a bash script on your system:

**1. Create the script:**
```bash
cat > /root/update-secureboot.sh << 'EOF'
#!/bin/bash
echo "Re-signing Kali kernels..."
for kernel in /boot/vmlinuz-*; do
    sbsign --key /root/secureboot-keys/MOK.priv --cert /root/secureboot-keys/MOK.pem "$kernel" --output "$kernel"
done

echo "Restoring Debian signed GRUB..."
cp /root/grub_extracted/usr/lib/grub/x86_64-efi-signed/grubx64.efi.signed /boot/efi/EFI/kali/grubx64.efi

echo "Secure Boot maintenance complete!"
EOF
```

**2. Make it executable:**
```bash
chmod +x /root/update-secureboot.sh
```

**When to use it:**
Whenever you run `sudo apt update && sudo apt upgrade` (or `dist-upgrade`) and notice that a new `linux-image` or `grub-efi` package was installed, simply run:
```bash
sudo /root/update-secureboot.sh
```
Do this *before* you reboot, and your system will continue to boot flawlessly under Secure Boot.
```
