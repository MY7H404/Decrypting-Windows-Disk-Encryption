# BitLocker Hacked: Decrypting Windows Disk Encryption

![](https://github.com/MY7H404/Decrypting-Windows-Disk-Encryption/blob/main/images/1.webp)

My first question is how much protection does BitLocker provide, is BitLocker that much secure?!, let’s see…

### What is BitLocker

BitLocker, a Windows feature, offers volume encryption to prevent unauthorized access to data on lost or stolen devices. When used with a Trusted Platform Module (TPM), it enhances security by binding encryption keys to specific hardware configurations, ensuring protection against tampering or unauthorized access.

![](https://github.com/MY7H404/Decrypting-Windows-Disk-Encryption/blob/main/images/2.webp)

According to Microsoft’s documentation, BitLocker encrypts entire volumes to mitigate the risks associated with data theft or exposure from compromised devices. The integration of TPM further fortifies this encryption by securely storing and managing cryptographic keys within the hardware, thus preventing unauthorized access even if the physical storage device is compromised.

It means that if someone steals hard drive and plugin to another computer they should not be able to read the data because the whole drive is encrypted.

### Let’s try to read data from another computer

I plugin BitLocker enabled hard drive on another computer and when I tried to mount that drive I got an error

![](https://github.com/MY7H404/Decrypting-Windows-Disk-Encryption/blob/main/images/3.webp)

The entire Windows partition is encrypted, and we can’t access any data on it. We can’t read or write anything on a BitLocker-enabled drive from another computer. But wait a minute — when I plug it back into the main computer, Windows boots up fine without any errors, and I don’t need to enter a password during boot. But how is this possible when the entire partition is encrypted? How can Windows boot up fine without any errors or the need to enter a password? Isn’t the partition encrypted? How exactly does BitLocker work?

### Working of TPM

Trusted Platform Module (TPM) serves as a vital component in enhancing the security of BitLocker, a feature in Windows designed to encrypt the entire hard drive. TPM functions as a secure storage mechanism for cryptographic keys, ensuring they are protected from unauthorized access. When setting up BitLocker, TPM generates a unique cryptographic key called the Storage Root Key (SRK). This key is securely stored within the TPM chip, inaccessible to outside parties. BitLocker, in turn, generates another key known as the Full Volume Encryption Key (FVEK), responsible for encrypting the entire contents of the hard drive. To add an extra layer of security, BitLocker encrypts the FVEK using the SRK stored in the TPM. This means that even if someone gains access to the hard drive, they cannot decrypt its contents without the SRK securely held within TPM. Additionally, TPM plays a crucial role in ensuring the integrity of the system’s boot process. It verifies that essential system components haven’t been tampered with before releasing the SRK to decrypt the FVEK during boot-up. This ensures that only trusted devices with a secure boot process can access the encrypted data, bolstering the overall security of the system.

### Attacking the TPM

Attacking the TPM directly is very unlikely to bear fruit within the timeframe of testing. As a result, we must look at the trust relationships around the TPM and what it relies on. It is a distinct and separate chip from other components on the motherboard and may be susceptible to a variety of attacks. Our particular TPM in question is shown here:

![](https://github.com/MY7H404/Decrypting-Windows-Disk-Encryption/blob/main/images/4.webp)

Researching [that specific TPM chip](https://www.st.com/en/secure-mcus/st33tphf20spi.html) revealed it communicates to the CPU using the Serial Peripheral Interface (SPI) protocol:

![](https://github.com/MY7H404/Decrypting-Windows-Disk-Encryption/blob/main/images/5.webp)

This was further supported when we found the TPM mentioned in the laptop’s schematics:

![](https://github.com/MY7H404/Decrypting-Windows-Disk-Encryption/blob/main/images/6.webp)

SPI, a widely used communication protocol in embedded systems, lacks built-in encryption features due to its simplicity. This means that any encryption needs to be managed by the individual devices communicating via SPI. Currently, BitLocker does not employ the encrypted communication capabilities of the TPM 2.0 standard. As a result, data transmitted from the TPM is sent in plaintext, including the decryption key for Windows. Exploiting this vulnerability could potentially grant access to the encrypted drive.

Navigating around the TPM in this way is like bypassing a heavily guarded fortress and focusing on a less fortified transport vehicle. However, intercepting data over the SPI bus poses its own challenges. The TPM, housed in a [VQFN32](https://en.wikipedia.org/wiki/Flat_no-leads_package) footprint, features exceptionally tiny pins — measuring only 0.25mm wide and spaced 0.5mm apart. These pins, flat against the chip’s wall, make traditional attachment methods virtually impossible. While soldering “fly leads” to the solder pads is an option, it poses significant physical instability.

Before delving into the intricate process of attaching leads to the TPM, we explored alternative avenues. It’s common for SPI chips to share the same “bus” with other devices, simplifying connections and reducing costs. After meticulous probing and scrutinizing the schematics, we discovered that the TPM shares an SPI bus with a solitary companion — the CMOS chip. Unlike the TPM, the CMOS chip boasts larger pins, specifically a SOP-8 (SOIC-8), offering a more accessible target for interception.

![](https://github.com/MY7H404/Decrypting-Windows-Disk-Encryption/blob/main/images/7.webp)

This was ideal. We proceeded to hook up our [Saleae logic analyzer](https://www.saleae.com/) to the pins according to the [CMOS’s datasheet](https://www.macrogroup.ru/sites/default/files/uploads/mx25l12873f_3v_128mb_v1.1.pdf):

![](https://github.com/MY7H404/Decrypting-Windows-Disk-Encryption/blob/main/images/8.webp)

![](https://github.com/MY7H404/Decrypting-Windows-Disk-Encryption/blob/main/images/9.webp)

In contrast, a sophisticated attacker, as mentioned earlier, would opt for a [SOIC-8 clip](https://www.digikey.com/en/products/detail/pomona-electronics/5250/745102), streamlining the connection process and expediting a real-world attack by saving precious minutes. With the probes securely attached, we initiated the laptop’s boot sequence and meticulously recorded every byte of SPI data traversing the traces. Within the deluge of millions of data points lay the elusive BitLocker decryption key. The challenge now lay in pinpointing its exact location amidst the sea of information. Employing Henri Nurmi’s [BitLocker-spi-toolkit](https://github.com/WithSecureLabs/bitlocker-spi-toolkit) in an attempt to automatically extract the key proved unsuccessful with our capture. The High-Level Analyzer (HLA) depicted in the screenshot below illustrates the toolkit’s operation — some transactions were parsed accurately, while others remained elusive. Our capture exhibited peculiarities that confounded the HLA’s parsing capabilities, indicating a unique aspect of our data that eluded automated extraction.

![](https://github.com/MY7H404/Decrypting-Windows-Disk-Encryption/blob/main/images/10.webp)

After days of troubleshooting, comparing captures, and pulling hair, we finally figured out it was a combination of different bit masks for the TPM command packets as well as a different regex for finding the key. The BitLocker-spi-toolkit can parse these types of requests as well. Once we had that, lo and behold, the key popped out.

![](https://github.com/MY7H404/Decrypting-Windows-Disk-Encryption/blob/main/images/11.webp)

Now armed with the decryption key in hand, it was time to unlock the mysteries concealed within the SSD. With careful precision, we extracted the SSD from its housing, mounted it onto an adapter, and seamlessly plugged it into our system:

![](https://github.com/MY7H404/Decrypting-Windows-Disk-Encryption/blob/main/images/12.webp)

We made a disk image of the drive which we operated on moving forward. Interestingly, in the entire process of the attack chain, the part that takes the longest is simply copying the 256GB of files. Once we had the image locally, we could use the [Dislocker](https://github.com/Aorimn/dislocker) toolset to decrypt the drive:

We took a strategic step forward by creating a disk image of the drive, a pivotal move that shaped our subsequent actions. Surprisingly, amidst the entire attack chain, the most time-consuming aspect was merely copying the vast expanse of data spanning 256GB. With the image securely stored locally, we seamlessly transitioned to leveraging the Dislocker toolset to initiate the decryption process:
```
$ echo daa0ccb7312<REDACTED> | xxd -r -p > ~/vmk  
$ mkdir ~/ssd ~/mounted  
$ sudo losetup  -P /dev/loop6 /mnt/hgfs/ExternalSSD/ssd-dd.img   
$ sudo fdisk -l /dev/loop6  
    Disk /dev/loop6: 238.47 GiB, 256060514304 bytes, 500118192 sectors  
    Units: sectors of 1 * 512 = 512 bytes  
    Sector size (logical/physical): 512 bytes / 512 bytes  
    I/O size (minimum/optimal): 512 bytes / 512 bytes  
    Disklabel type: gpt  
    Disk identifier: BD45F9A-F26D-41C9-8F1F-0F1EE74233  
    Device         Start       End   Sectors   Size Type  
    /dev/loop6p1    2048   1026047   1024000   500M Windows recovery environment  
    /dev/loop6p2 1026048   2050047   1024000   500M EFI System  
    /dev/loop6p3 2050048   2312191    262144   128M Microsoft reserved  
    /dev/loop6p4 2312192 500117503 497805312 237.4G Microsoft basic data <- bitlocker drive  
$ sudo dislocker-fuse -K ~/vmk /dev/loop6p4 -- ~/ssd  
$ sudo ntfs-3g ~/ssd/dislocker-file ~/mounted  
$ ls -al ~/mounted  
    total 19156929  
    drwxrwxrwx  1 root root        8192 May  5 19:00  .  
    drwxrwxrwt 17 root root        4096 Jun 15 09:43  ..  
    drwxrwxrwx  1 root root           0 May  6 14:29 '$Recycle.Bin'  
    drwxrwxrwx  1 root root           0 May  4 10:55 '$WinREAgent'  
    -rwxrwxrwx  1 root root      413738 Dec  7  2019  bootmgr  
    -rwxrwxrwx  1 root root           1 Dec  7  2019  BOOTNXT  
    lrwxrwxrwx  2 root root          15 May  4 11:18 'Documents and Settings' -> ~/mounted/Users
```
---

In conclusion, while this journey into breaking TPM security and gaining access to the BitLocker key may have been filled with technical complexities, it’s also been a fascinating exploration into the inner workings of these systems. Remember, curiosity and persistence can often lead to unexpected discoveries in the world of technology. So, keep tinkering, keep learning, and who knows what other secrets you might uncover in the future!
