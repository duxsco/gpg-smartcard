# (WIP) GnuPG Smartcard (WIP)

## Credits

Thanks [Fan Jiang](https://blog.amayume.net/diy-smartcard-authentication-with-smartpgp/)

## Prepare the system

In the following, I am using the [Ubuntu Live Server ISO](https://releases.ubuntu.com/focal/) as well as the smartcard "J3H145" (recommended [here](https://github.com/ANSSI-FR/SmartPGP/search?q=J3H145&type=issues)). You also need a supported smartcard reader of whom I prefer those including a pinpad. Further info:

- [CCID Driver](https://github.com/LudovicRousseau/CCID#ccid-and-iccd-readers)
- [Arch Linux Wiki](https://wiki.archlinux.org/title/Electronic_identification)
- [Debian Wiki](https://wiki.debian.org/Smartcards)
- [Gentoo Linux Wiki](https://wiki.gentoo.org/wiki/PCSC-Lite#Additional_software)

After booting from the Ubuntu Live Server ISO, setup keyboard layout and network. `ssh` into the system:

![ssh config](assets/ssh0.png)

![ssh config](assets/ssh1.png)

In the SSH session, open a shell:

![ssh config](assets/ssh2.png)

Install required packages and start the `pcscd` service:

```bash
useradd -m -s /bin/bash gpg && \
useradd -m -s /bin/bash tools && \

apt-get update && \
apt-get install 2to3 ant git maven openjdk-8-jdk pcscd python3-pyscard scdaemon && \
rm -rf /var/lib/apt/lists/* && \
systemctl start pcscd; echo $?
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
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64 && \
mvn package && \
ant && \
popd; echo $?"
```

We need to patch python scripts:

```bash
sed -i 's|#!/usr/bin/env python$|#!/usr/bin/env python3|' /home/tools/SmartPGP/bin/smartpgp/commands.py /home/tools/SmartPGP/bin/smartpgp-cli /home/tools/SmartPGP/bin/example-set-mixed-crypto.py && \
find /home/tools/SmartPGP/bin/ -type f -exec 2to3 -w {} \; && \
sed -i 's#is not#!=#' /home/tools/SmartPGP/bin/smartpgp/commands.py; echo $?
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
```

Unset passphrases:

```bash
unset PASSPHRASE PASSPHRASE_REPEAT
```

Sample run:

```
Passphrase to set for private key:
Please, repeat the passphrase:

Name and e-mail (e.g. "Max Mustermann <max@mustermann.de>"): Maria Musterfrau <maria@musterfrau.de>

How many years do you want the subkeys to be valid?
You can always extend the validity or create new subkeys later on! 1

Success! You can find the GnuPG homedir containing your keypair at /tmp/tmp.Ovi2fCyOGG
Please, copy that directory somewhere safe!

gpg@ubuntu-server:~$ gpg --homedir /tmp/tmp.Ovi2fCyOGG --list-secret-keys
/tmp/tmp.Ovi2fCyOGG/pubring.kbx
-------------------------------
sec   ed25519 2021-09-25 [C]
      FEFC68B98B189A04B21ABE42733CE6C77184CEDD
uid           [ultimate] Maria Musterfrau <maria@musterfrau.de>
ssb   nistp521 2021-09-25 [S] [expires: 2022-09-25]
ssb   nistp521 2021-09-25 [E] [expires: 2022-09-25]
ssb   nistp521 2021-09-25 [A] [expires: 2022-09-25]
```

## Copy GnuPG subkeys to smartcard

⚠ First of all, make a backup of the GnuPG homedir! If you save after a `keytocard` command, the subkey - copied to the smartcard - will be removed by GnuPG locally and exists only on the smartcard! You won't be able to copy said subkey to another smartcard! ⚠

### Switch to nistp521 subkeys

By default, `rsa2048` is used for all subkeys:

```bash
root@ubuntu-server:/# su - gpg -c "gpg --card-status"
Reader ...........: KOBIL KAAN Advanced (E_094100658) 00 00
Application ID ...: D276000124010304AFAF000000000000
Application type .: OpenPGP
Version ..........: 3.4
Manufacturer .....: unknown
Serial number ....: 00000000
Name of cardholder: [not set]
Language prefs ...: en
Salutation .......:
URL of public key : [not set]
Login data .......: [not set]
Signature PIN ....: forced
Key attributes ...: rsa2048 rsa2048 rsa2048
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 0 3
Signature counter : 0
KDF setting ......: off
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]
```

To switch over to `nistp521`, GnuPG cannot be used. You have to use SmartPGP:

```bash
root@ubuntu-server:/# su - tools -c "/home/tools/SmartPGP/bin/smartpgp-cli switch-p521"
Select OpenPGP Applet
90 00
Verify Admin PIN
90 00
Switch to P-521 (sig)
90 00
Switch to P-521 (dec)
90 00
Switch to P-521 (auth)
90 00
```

Thereafter, you have:

```bash
root@ubuntu-server:/# su - gpg -c "gpg --card-status"
Reader ...........: KOBIL KAAN Advanced (E_094100658) 00 00
Application ID ...: D276000124010304AFAF000000000000
Application type .: OpenPGP
Version ..........: 3.4
Manufacturer .....: unknown
Serial number ....: 00000000
Name of cardholder: [not set]
Language prefs ...: en
Salutation .......:
URL of public key : [not set]
Login data .......: [not set]
Signature PIN ....: forced
Key attributes ...: nistp521 nistp521 nistp521
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 0 3
Signature counter : 0
KDF setting ......: off
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]
```

### Set smartcard pin and admin pin

```bash
# Switch to user "gpg"
su - gpg

gpg --card-edit

# Enable "admin" commands
gpg/card> admin

# Switch to "passwd" mode
gpg/card> passwd

# Select "3 - change Admin PIN" and set a new 8-digit admin pin.
# The initial admin pin is "12345678".
# First, you need to type in the initial admin pin and press "Enter".
# Then, you have to type in your new admin pin and press "Enter".
# Type in your new admin pin again and confirm again.

# Select "1 - change PIN" and set a new 6-digit pin.
# The procedure is the same as for the admin pin,
# but the initial pin is "123456".

# Quit and exit
```

### Set name (optional)

```bash
# Switch to user "gpg"
su - gpg

gpg --card-edit

# Enable "admin" commands
gpg/card> admin

# Execute "name" command and type in your name thereafter
gpg/card> name
```

### Set keyserver (optional)

```bash
# Switch to user "gpg"
su - gpg

gpg --card-edit

# Enable "admin" commands
gpg/card> admin

# Execute "url" command and type in your keyserver, hkps://keys.openpgp.org recommended
gpg/card> url
```

### Copy subkeys to smartcard

⚠ First of all, make a backup of the GnuPG homedir! If you save after a `keytocard` command, the subkey - copied to the smartcard - will be removed by GnuPG locally and exists only on the smartcard! You won't be able to copy said subkey to another smartcard! ⚠

```bash
# Switch to user "gpg"
su - gpg

gpg@ubuntu-server:~$ gpg --list-keys
/tmp/tmp.Ovi2fCyOGG/pubring.kbx
-------------------------------
pub   ed25519 2021-09-25 [C]
      FEFC68B98B189A04B21ABE42733CE6C77184CEDD
uid           [ultimate] Maria Musterfrau <maria@musterfrau.de>
sub   nistp521 2021-09-25 [S] [expires: 2022-09-25]
sub   nistp521 2021-09-25 [E] [expires: 2022-09-25]
sub   nistp521 2021-09-25 [A] [expires: 2022-09-25]

gpg@ubuntu-server:~$ gpg --edit-key FEFC68B98B189A04B21ABE42733CE6C77184CEDD
gpg (GnuPG) 2.2.19; Copyright (C) 2019 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

sec  ed25519/733CE6C77184CEDD
     created: 2021-09-25  expires: never       usage: C # certify
     trust: ultimate      validity: ultimate
ssb  nistp521/37EBFD26F43F59E2
     created: 2021-09-25  expires: 2022-09-25  usage: S # sign
ssb  nistp521/8684AA0423C56A75
     created: 2021-09-25  expires: 2022-09-25  usage: E # encrypt
ssb  nistp521/D427AB87DFA10C89
     created: 2021-09-25  expires: 2022-09-25  usage: A # authenticate
[ultimate] (1). Maria Musterfrau <maria@musterfrau.de>

gpg> key 1

sec  ed25519/733CE6C77184CEDD
     created: 2021-09-25  expires: never       usage: C
     trust: ultimate      validity: ultimate
ssb* nistp521/37EBFD26F43F59E2                           # pay attention to the star!
     created: 2021-09-25  expires: 2022-09-25  usage: S
ssb  nistp521/8684AA0423C56A75
     created: 2021-09-25  expires: 2022-09-25  usage: E
ssb  nistp521/D427AB87DFA10C89
     created: 2021-09-25  expires: 2022-09-25  usage: A
[ultimate] (1). Maria Musterfrau <maria@musterfrau.de>

gpg> keytocard
Please select where to store the key:
   (1) Signature key
   (3) Authentication key
Your selection? 1

sec  ed25519/733CE6C77184CEDD
     created: 2021-09-25  expires: never       usage: C
     trust: ultimate      validity: ultimate
ssb* nistp521/37EBFD26F43F59E2
     created: 2021-09-25  expires: 2022-09-25  usage: S
ssb  nistp521/8684AA0423C56A75
     created: 2021-09-25  expires: 2022-09-25  usage: E
ssb  nistp521/D427AB87DFA10C89
     created: 2021-09-25  expires: 2022-09-25  usage: A
[ultimate] (1). Maria Musterfrau <maria@musterfrau.de>

gpg> key 1                                               # deselect key 1

sec  ed25519/733CE6C77184CEDD
     created: 2021-09-25  expires: never       usage: C
     trust: ultimate      validity: ultimate
ssb  nistp521/37EBFD26F43F59E2
     created: 2021-09-25  expires: 2022-09-25  usage: S
ssb  nistp521/8684AA0423C56A75
     created: 2021-09-25  expires: 2022-09-25  usage: E
ssb  nistp521/D427AB87DFA10C89
     created: 2021-09-25  expires: 2022-09-25  usage: A
[ultimate] (1). Maria Musterfrau <maria@musterfrau.de>

gpg> key 2

sec  ed25519/733CE6C77184CEDD
     created: 2021-09-25  expires: never       usage: C
     trust: ultimate      validity: ultimate
ssb  nistp521/37EBFD26F43F59E2
     created: 2021-09-25  expires: 2022-09-25  usage: S
ssb* nistp521/8684AA0423C56A75                           # subkey 2 selected
     created: 2021-09-25  expires: 2022-09-25  usage: E
ssb  nistp521/D427AB87DFA10C89
     created: 2021-09-25  expires: 2022-09-25  usage: A
[ultimate] (1). Maria Musterfrau <maria@musterfrau.de>

gpg> keytocard
Please select where to store the key:
   (2) Encryption key
Your selection? 2

sec  ed25519/733CE6C77184CEDD
     created: 2021-09-25  expires: never       usage: C
     trust: ultimate      validity: ultimate
ssb  nistp521/37EBFD26F43F59E2
     created: 2021-09-25  expires: 2022-09-25  usage: S
ssb* nistp521/8684AA0423C56A75
     created: 2021-09-25  expires: 2022-09-25  usage: E
ssb  nistp521/D427AB87DFA10C89
     created: 2021-09-25  expires: 2022-09-25  usage: A
[ultimate] (1). Maria Musterfrau <maria@musterfrau.de>

gpg> key 2                                               # deselect subkey 2

sec  ed25519/733CE6C77184CEDD
     created: 2021-09-25  expires: never       usage: C
     trust: ultimate      validity: ultimate
ssb  nistp521/37EBFD26F43F59E2
     created: 2021-09-25  expires: 2022-09-25  usage: S
ssb  nistp521/8684AA0423C56A75
     created: 2021-09-25  expires: 2022-09-25  usage: E
ssb  nistp521/D427AB87DFA10C89
     created: 2021-09-25  expires: 2022-09-25  usage: A
[ultimate] (1). Maria Musterfrau <maria@musterfrau.de>

gpg> key 3                                               # select subkey 3

sec  ed25519/733CE6C77184CEDD
     created: 2021-09-25  expires: never       usage: C
     trust: ultimate      validity: ultimate
ssb  nistp521/37EBFD26F43F59E2
     created: 2021-09-25  expires: 2022-09-25  usage: S
ssb  nistp521/8684AA0423C56A75
     created: 2021-09-25  expires: 2022-09-25  usage: E
ssb* nistp521/D427AB87DFA10C89
     created: 2021-09-25  expires: 2022-09-25  usage: A
[ultimate] (1). Maria Musterfrau <maria@musterfrau.de>

gpg> keytocard
Please select where to store the key:
   (3) Authentication key
Your selection? 3

sec  ed25519/733CE6C77184CEDD
     created: 2021-09-25  expires: never       usage: C
     trust: ultimate      validity: ultimate
ssb  nistp521/37EBFD26F43F59E2
     created: 2021-09-25  expires: 2022-09-25  usage: S
ssb  nistp521/8684AA0423C56A75
     created: 2021-09-25  expires: 2022-09-25  usage: E
ssb* nistp521/D427AB87DFA10C89
     created: 2021-09-25  expires: 2022-09-25  usage: A
[ultimate] (1). Maria Musterfrau <maria@musterfrau.de>

gpg> quit
Save changes? (y/N) n                                    # saving leads to deletion of subkeys on the local filesystem
Quit without saving? (y/N) y
```

### Copy required info from air gapped machine to working machine

In order to be able to fully use your smartcard, you need to copy the public key as well as ownertrust to all machines you want to use your smartcard with.

Create backup of public key and ownertrust:

```bash
su - gpg
gpg --export --armor > pubkey.asc
gpg --export-ownertrust > ownertrust.txt
```

On your working machine, import:

```bash
gpg --import pubkey.asc
gpg --import-ownertrust ownertrust.txt
```

Execute on your working machine `gpg --card-status` to make it aware of the private keys on the smartcard. If everything went well you should see that the subkeys are located on the card. A `#` after the initial tags sec or ssb means that the primary key or subkey is currently not usable (offline) which is what we wish for. The primary key is only used for the creation of subkeys and are only needed on your air gapped machine.

![secret keys](assets/gnupg_secret_keys.png)

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
