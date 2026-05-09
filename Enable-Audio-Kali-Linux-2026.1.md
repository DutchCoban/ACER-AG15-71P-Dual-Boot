```markdown
# Fix: No Sound on Acer AG15-71P (Kali Linux / Raptor Lake Audio)

**Problem:** The internal speakers produce no sound in Linux due to a conflict between the modern SOF driver and the digital microphone on the Intel Raptor Lake-P/U/H cAVS audio chip.  
**Solution:** Force the older, more stable HDA driver and disable the digital microphone detection via kernel parameters.

## Step-by-Step Guide

### 1. Edit the GRUB configuration file
Open the terminal and edit the GRUB file with root privileges:
```bash
sudo nano /etc/default/grub
```

### 1. Add the kernel parameters
Find the line starting with `GRUB_CMDLINE_LINUX_DEFAULT`. Add the following parameters to the end of the line (inside the quotation marks):
`snd-intel-dspcfg.dsp_driver=1 snd_hda_intel.dmic_detect=0`

The line should look something like this:
```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash snd-intel-dspcfg.dsp_driver=1 snd_hda_intel.dmic_detect=0"
```
*Save the file (`Ctrl + O`, `Enter`) and exit nano (`Ctrl + X`).*

### 2. Update GRUB and reboot
Apply the changes to the bootloader and restart the laptop:
```bash
sudo update-grub
sudo reboot
```

### 3. Check the volume (Alsamixer)
After rebooting, the sound might be muted by default due to the driver change.
1. Open the terminal and start Alsamixer:
   ```bash
   alsamixer
   ```
2. Press **F6** and select your sound card (e.g., *HDA Intel PCH*).
3. Check if the channels (such as *Master* and *Speaker*) show **[MM]** (muted) at the bottom.
4. Use the arrow keys (left/right) to navigate, press **M** to unmute (it will change to **[00]**), and set the volume to 100% using the up arrow key.
5. Press **Esc** to exit.
```

You can copy this directly into a `.md` file in your GitHub repository or personal notes!
