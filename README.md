# (WIP) GnuPG Smartcard (WIP)

## Credits

Thanks [Fan Jiang](https://blog.amayume.net/diy-smartcard-authentication-with-smartpgp/)

## Prepare the smartcard

In the following, I am using the [SystemRescueCD](https://www.system-rescue.org/) as well as the smartcard "J3H145" (recommended [here](https://github.com/ANSSI-FR/SmartPGP/search?q=J3H145&type=issues)). You also need a supported smartcard reader of whom I prefer those including a pinpad. Further info:

- [CCID Driver](https://github.com/LudovicRousseau/CCID#ccid-and-iccd-readers)
- [Arch Linux Wiki](https://wiki.archlinux.org/title/Electronic_identification)
- [Gentoo Linux Wiki](https://wiki.gentoo.org/wiki/PCSC-Lite#Additional_software)

After booting from SystemRescueCD:

```bash
# Change the keyboard layout
loadkeys de-latin1-nodeadkeys

# If you don't want to work on tty0 set password for ssh login, alternatively set /root/.ssh/authorized_keys
passwd root

# Insert iptables rules at correct place for SystemRescueCD to accept SSH clients.
# Verify with "iptables -L -v -n"
iptables -I INPUT 4 -p tcp --dport 22 -j ACCEPT -m conntrack --ctstate NEW
```

`ssh` into the SystemRescueCD and prepare the system:

```bash
useradd -m -s /bin/bash aur && \
useradd -m -s /bin/bash -G tty gpg && \
useradd -m -s /bin/bash tools && \
chmod g=rw "$(tty)" && \

# select jdk8-openjdk; 
pacman -Sy ant git maven pcsc-tools && \
rm -fv /var/cache/pacman/pkg/* && \
pacman -Sy binutils fakeroot gcc pcsclite python2 python2-pyasn1 python2-setuptools swig && \
rm -fv /var/cache/pacman/pkg/* && \

su - aur -c "mkdir /home/aur/pyscard && curl --tlsv1.3 --proto '=https' -o /home/aur/pyscard/PKGBUILD https://aur.archlinux.org/cgit/aur.git/plain/PKGBUILD?h=python2-pyscard"
```

Start service:

```bash
systemctl start pcscd
```

Compile the required tools. This creates `GlobalPlatformPro/gp.jar` and `SmartPGP/SmartPGPApplet.cap`. Make sure to use JavaCard 3.0.4 which is the latest version supported by smartcard "J3H145".

```bash
su - tools -c "
git clone https://github.com/ANSSI-FR/SmartPGP.git && \
pushd SmartPGP && \
git checkout v1.20-3.0.4 && \
git submodule add https://github.com/martinpaljak/oracle_javacard_sdks sdks && \
sed -i 's#<javacard>#<javacard jckit=\"./sdks/jc304_kit\">#' build.xml && \
ant && \
popd && \

git clone https://github.com/martinpaljak/GlobalPlatformPro.git && \
pushd GlobalPlatformPro && \
git checkout v20.01.23 && \
sed -i 's#http://central.maven.org#https://repo1.maven.org#' build.xml && \
mvn package && \
ant && \
popd; echo $?"
```
We need to patch python scripts:

```bash
sed -i 's|#!/usr/bin/env python$|#!/usr/bin/env python2|' /home/tools/SmartPGP/bin/smartpgp/commands.py /home/tools/SmartPGP/bin/smartpgp-cli /home/tools/SmartPGP/bin/example-set-mixed-crypto.py
```

Flash, create key and lock (copy, paste and execute one after another):

```bash
# Switch to "tools" user
su - tools

# Flash the smartcard
java -jar GlobalPlatformPro/gp.jar -install SmartPGP/SmartPGPApplet.cap -default; echo $?

# Don't forget to backup this key at a secure location
( umask 0077 && openssl rand --hex 16 | awk '{print toupper($0)}' > /home/tools/.card_secret ); echo $?

# Lock the smartcard (important!!!)
cat /home/tools/.card_secret | xargs java -jar GlobalPlatformPro/gp.jar -lock; echo $?

# Switch back to root
exit
```

## Create GnuPG keypair

I prefer Curve25519 and Curve448 which are recommended by [Daniel J. Bernstein and Tanja Lange](https://safecurves.cr.yp.to/). Support for both has been added with JavaCard 3.1, but compatible smartcards are missing. Furthermore, Curve448 is only supported on [GnuPG >=2.3.0](https://dev.gnupg.org/source/gnupg/browse/tag%253A%2520gnupg-2.3.0/NEWS;c922a798a341261f1aafaf7c1c0217e4ce3e3acf$32). The next best algorithm (IMHO) is `nistp521` which I use for the subkeys. The primary key, however, is created using `ed25519` as it's not going to be copied to the smartcard.

I create my keypairs for the smartcard the following way in `bash` as user `gpg` (`su - gpg`). 

```bash
echo "" && \
read -r -s -p 'Passphrase to set for private key: ' PASSPHRASE && \
echo "" && \
read -r -s -p 'Please, repeat the passphrase: ' PASSPHRASE_REPEAT && \
[ "${PASSPHRASE}" != "${PASSPHRASE_REPEAT}" ] && \
echo -e "\nPassphrases don't match! Aborting...\n" || (
    echo -e "\n" && \
    read -r -p 'Name and e-mail (e.g. "Max Mustermann <max@mustermann.de>"): ' CONTACT && \
    echo "" && \
    read -r -p 'How many years do you want the subkeys to be valid?
You can always extend the validity or create new subkeys later on! ' YEARS && \
    GPG_HOMEDIR="$( umask 0077 && mktemp -d )" && \
    echo "${PASSPHRASE}" | gpg --homedir "${GPG_HOMEDIR}" --batch --pinentry-mode loopback --quiet --passphrase-fd 0 \
        --quick-generate-key "${CONTACT}" ed25519 cert 0 && \
    FINGERPRINT=$(gpg --homedir "${GPG_HOMEDIR}" --list-options show-only-fpr-mbox --list-secret-keys 2>/dev/null | awk '{print $1}') && \
    echo "${PASSPHRASE}" | gpg --homedir "${GPG_HOMEDIR}" --batch --pinentry-mode loopback --quiet --passphrase-fd 0 \
        --quick-add-key "${FINGERPRINT}" nistp521/ecdsa sign "${YEARS}y" && \
    echo "${PASSPHRASE}" | gpg --homedir "${GPG_HOMEDIR}" --batch --pinentry-mode loopback --quiet --passphrase-fd 0 \
        --quick-add-key "${FINGERPRINT}" nistp521 encrypt "${YEARS}y" && \
    echo "${PASSPHRASE}" | gpg --homedir "${GPG_HOMEDIR}" --batch --pinentry-mode loopback --quiet --passphrase-fd 0 \
        --quick-add-key "${FINGERPRINT}" nistp521/ecdsa auth "${YEARS}y" && \
    echo -e '\nSuccess! You can find the GnuPG homedir containing your keypair at \e[0;1;97;104m'"${GPG_HOMEDIR}"'\e[0m\nPlease, copy that directory somewhere safe!\n'
)
unset PASSPHRASE PASSPHRASE_REPEAT
```

Sample run:

```
Passphrase to set for private key: 
Please, repeat the passphrase: 

Name and e-mail (e.g. "Max Mustermann <max@mustermann.de>"): Maria Musterfrau <maria@musterfrau.de>

How many years do you want the subkeys to be valid?
You can always extend the validity or create new subkeys later on! 1

Success! You can find the GnuPG homedir containing your keypair at /tmp/tmp.TlHwQO6Vw3
Please, copy that directory somewhere safe!

bash-5.1$ gpg --homedir /tmp/tmp.TlHwQO6Vw3 --list-secret-keys
/tmp/tmp.TlHwQO6Vw3/pubring.kbx
-------------------------------
sec   ed25519 2021-09-24 [C]
      AE30BDCCBC2E8C82EC9423910212A7DB46B2F258
uid           [ultimate] Maria Musterfrau <maria@musterfrau.de>
ssb   nistp521 2021-09-24 [S] [expires: 2022-09-24]
ssb   nistp521 2021-09-24 [E] [expires: 2022-09-24]
ssb   nistp521 2021-09-24 [A] [expires: 2022-09-24]
```

## Copy GnuPG subkeys to smartcard

⚠ First of all, make a backup of the GnuPG homedir! If you save after a `keytocard` command, the key - copied to the smartcard - will be removed by GnuPG locally and exists only on the smartcard! You won't be able to copy said key to another smartcard! ⚠


## Troubleshooting

⚠ First of all, make sure that the smartcard is unlocked! Otherwise, you may brick your smartcard! ⚠

```bash
cat .card_secret | xargs java -jar GlobalPlatformPro/gp.jar -unlock -key
```

### Show info

- Show info on **unlocked** smartcard:

```bash
java -jar GlobalPlatformPro/gp.jar -info
```

- List applets on **unlocked** smartcard:

```bash
java -jar GlobalPlatformPro/gp.jar -list
```

### Delete/uninstall applet

- Delete **default** applet on **unlocked** smartcard:

```bash
java -jar GlobalPlatformPro/gp.jar -delete -default
```

- Delete applet on **unlocked** smartcard:

  1. First copy `From:` value of `APP` in applet list output
  2. Execute while providing said value:

```bash
java -jar GlobalPlatformPro/gp.jar -delete ASDFASDFASDF
```

To uninstall use flag `-uninstall` instead of `-delete`.
