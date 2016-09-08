# _Split dm-crypt_ for Qubes R3.2-rc3 and later

**Isolates device-mapper based secondary storage encryption (i.e. not
the root filesystem) and LUKS header processing to DisposableVMs.**

Instead of directly attaching an encrypted LUKS partition from a source
VM such as sys-usb to a destination VM and decrypting it there, it works
like this:

1. The encrypted partition is attached from the source VM to a
   (long-lived) offline _device DisposableVM_ configured not to parse
   its content in any way: The kernel partition scanners, udev probes,
   and UDisks handling are disabled.

2. From there, the LUKS header is sent to a (short-lived) offline
   _header DisposableVM_ prompting for the password, and the encryption
   key is sent back to the device DisposableVM, which validates that it
   received an AES-XTS key and creates the dm-crypt mapping.

3. Finally, the decrypted partition is attached from the device
   DisposableVM to the destination VM.

**If the destination VM is compromised, it does not know the password or
encryption key. It also cannot easily exfiltrate decrypted data to the
disk in a form that would allow an attacker who seizes the disk contents
later to read it.** (But see below for caveats.)


## Usage

The `qvm-block-split` attach/detach commands accept a subset of the
familiar `qvm-block` syntax, and some other commands are included:

- Fully overwrite a device with random data

- Overwrite just the LUKS header with random data

- Format a new LUKS device with modern crypto parameters: AES-XTS with
  256+256 (instead of 128+128) bit keys, SHA512 (instead of SHA1) PBKDF2
  key derivation with 5 (instead of 0.1) seconds iteration time

When attaching, the destination VM argument can be omitted, in which
case the decrypted disk will be attached to yet another offline
DisposableVM.

```
qvm-block-split --attach|-a [--ro] [<dst-vm>] <src-vm>:<device>
                --detach|-d                   <src-vm>:<device>

                --overwrite-everything=random <src-vm>:<device>
                --overwrite-header=random     <src-vm>:<device>
                --overwrite-header=format     <src-vm>:<device>
                --overwrite-header=shell      <src-vm>:<device>
                --modify-header=shell         <src-vm>:<device>
```


## Remaining attacks

- After detaching, the password and/or key will linger in more RAM
  locations than without _Split dm-crypt_. Until there is a way to wipe
  the DisposableVMs' memory, and `qvm-block-split` is modified not to
  pass the key through dom0's memory, **power off your computer when
  memory forensics is a concern.**

- If both the destination VM and the source VM/disk are compromised,
  they could establish a covert channel using e.g. read and write access
  patterns, slowly saving some amount of decrypted data to the disk.

- If the source VM/disk is compromised and successfully exploits the
  header DisposableVM using a malicious LUKS header, a known AES-XTS key
  could be sent to the device DisposableVM and used to present malicious
  device content to the destination VM to potentially exploit it as
  well. **Be suspicious if you do not see the expected filesystem data
  in the destination VM. Or simply use a DisposableVM as the destination
  VM.**

- **Don't forget to overwrite your disk with random data before creating
  a LUKS volume on it.** Otherwise, a compromised destination VM could
  trivially save decrypted data to the disk in its free space, by
  encoding each bit as an unmodified (still empty or in some other way
  nonrandom-looking) or modified (random-looking) 128 bit AES block.


## Installation

1. Copy `vm/` to the DisposableVM template, inspect the code, and `sudo
   make install` there; also install the `pv` (Pipe Viewer) package to
   be able to run the `--overwrite-everything=random` command. Shut down
   the template when finished.

2. Copy `dom0/bin/qvm-block-split` to dom0, e.g. into `~/bin/`, inspect
   the code extra carefully, and `chmod +x` the script.