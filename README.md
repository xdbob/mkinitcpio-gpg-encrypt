# GnuPG hook for Arch Linux initcpio

This mkinitcpio hook allows you to use a PGP-compatible smart card during early boot to decrypt your full disk encrypted system.

# Why?

This is a great solution if you want the best possible encryption strength, but without having to type a huge passphrase in every time you boot. Using a hardware-backed RSA key on a PGP smart card, with the card performing decryption of the FDE key on-chip without releasing the private key to the OS, gives you extremely strong protection from attacks, even sophisticated/well-equipped adversaries.

As is the case with all LUKS systems, anyone who gets root on your box after it's booted can run `dmsetup table --showkeys` to dump the master key, the changing of which requires a complete wipe of your disk, in contrast to LUKS user key changing which requires only wiping and replacing the key slot. So don't let an adversary get root - watch your setuid programs, sudo rights, running services, etc. carefully.

PGP smart cards have varying options for PIN limits and reset or self-destruct functionality - choose one that fits your needs.

This hook has only been tested with the Yubikey NEO.

## Disclaimer

Use this hook at your own risk. It's highly recommended to have a backup key somewhere, in case you lose or destroy your primary key.

# Configuration process

1. Install Arch onto a LUKS encrypted system and get it booting using the stock `encrypt` hook and keyfile.
1. Encrypt your keyfile with GnuPG: `gpg -r YOURKEYID -o keyfile.gpg --encrypt keyfile`.
1. Delete/Backup/Watever your original keyfile.
1. Update accordingly your kerner args (ie: append .gpg to the keyfile name)
1. Export yourt private/shadow Key to /etc/private.gpg: `gpg --export-secret-keys YOURKEYID > /tmp/private.gpg && mv /tmp/private.gpg /etc`
1. Edit `/etc/mkinitcpio.conf` and replace the `encrypt` hook with `gpg-encrypt`. Do not leave both `encrypt` and `scencrypt` enabled.
1. Run `mkinitcpio -p linux`. If there are no errors, reboot with your smart card plugged in to find out if it works.

# Technical details

The hook works by copying your encrypted key file to the initramfs, decrypting it in memory, passing it to LUKS to unseal the disk, and then using `shred` to overwrite it in memory.

Behind the scenes, `gpg` starts `scdaemon`, which talks to `pcscd` and `pinentry-tty` to get your PIN and pass it to the card along with the payload for decryption. The private key itself is held securely on the smartcard - it cannot be released even with the PIN on hand. But the decryption is quick because the payload is small. Once the disk is mounted, the smartcard can safely be removed from the system - the result of the decryption is merely a "user key" that LUKS uses to decrypt the volume's master key. There is an excellent [white paper](http://clemens.endorphin.org/nmihde/nmihde-A4-ds.pdf) written by one of the original LUKS authors detailing LUKS's extensive anti-forensic hardening.

# Thanks
This hook is largely inspired from [this project](https://github.com/fuhry/initramfs-scencrypt).
