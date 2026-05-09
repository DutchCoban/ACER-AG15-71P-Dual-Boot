```markdown
# Fix: Geen geluid op Acer AG15-71P (Kali Linux / Raptor Lake Audio)

**Probleem:** De interne speakers geven geen geluid in Linux vanwege een conflict met de moderne SOF-driver en de digitale microfoon op de Intel Raptor Lake-P/U/H cAVS audiochip.  
**Oplossing:** Forceer de oudere, stabielere HDA-driver en schakel de detectie van de digitale microfoon uit via de kernel parameters.

## Stappenplan

### 1. Pas het GRUB-configuratiebestand aan
Open de terminal en bewerk het GRUB-bestand met root-rechten:
```bash
sudo nano /etc/default/grub
```

### 2. Voeg de kernel parameters toe
Zoek de regel die begint met `GRUB_CMDLINE_LINUX_DEFAULT`. Voeg de volgende parameters toe aan het einde (binnen de aanhalingstekens):
`snd-intel-dspcfg.dsp_driver=1 snd_hda_intel.dmic_detect=0`

De regel ziet er dan ongeveer zo uit:
```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash snd-intel-dspcfg.dsp_driver=1 snd_hda_intel.dmic_detect=0"
```
*Sla het bestand op (`Ctrl + O`, `Enter`) en sluit af (`Ctrl + X`).*

### 3. Update GRUB en herstart
Voer de wijzigingen door in de bootloader en herstart de laptop:
```bash
sudo update-grub
sudo reboot
```

### 4. Controleer het volume (Alsamixer)
Na de herstart kan het geluid door de nieuwe driver standaard gedempt (muted) zijn.
1. Open de terminal en start Alsamixer:
   ```bash
   alsamixer
   ```
2. Druk op **F6** en selecteer de geluidskaart (bijv. *HDA Intel PCH*).
3. Controleer of de kanalen (zoals *Master* en *Speaker*) onderaan op **[MM]** (gedempt) staan.
4. Gebruik de pijltjestoetsen (links/rechts) om te navigeren, druk op **M** om te unmuten (verandert naar **[00]**) en zet het volume op 100% met pijltje omhoog.
5. Druk op **Esc** om af te sluiten.
```

Je kunt deze tekst direct kopiëren en opslaan. Mocht je Kali Linux in de toekomst opnieuw installeren, dan heb je de oplossing direct bij de hand!
