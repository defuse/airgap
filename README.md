Airgap
=======

This document presents the design of, and procedures surrounding, a practical
and affordable air-gap system. We aim to provide a minimal set of useful
functionality, with minimal security assumptions.

**Important Note:** We are *not* trying to design something that's NSA-proof.
Rather, the goal is to be *secure enough that a targeted physical attack is
necessary*.  We should strive to make the NSA's job as hard as possible
(expensive with lots of detection risk), but we should not overcomplicate things
trying to protect ourselves from an adversary as powerful as the NSA.

Features
---------

We want an air-gapped system that can perform the following tasks securely:

- Let the operator examine [1] a file provided from the outside, and after
  reviewing it, sign it with a PGP private key.

- Let the operator decrypt text messages that were encrypted to their public
  key.

- Let the operator encrypt and sign text messages (typed in to the air-gap
  system) to a public key that was provided from the outside, possibly in
  response to a message they decrypted.

- Backup the air-gap system locally and to an off-site secure location.

We plan to add more functionality later. However, every added feature increases
the attack surface. The operator should understand this, and should use as few
features as possible.

Security Assumptions
---------------------

This section should be an *exhaustive* list of all the things that need to be
assumed for this air-gap design to be secure. The assumptions should be
*reasonable*, i.e. not assuming unrealistic things. Unreasonable/impossible
assumptions are considered vulnerabilities in the design.

We should, as much as possible, avoid assuming code is secure.

### Kerckhoffs's Principle

- We assume the adversary knows everything about how the air-gap works (i.e.
  they have this document).

### Physical Access

- The hardware used for the air-gap system is (initially) trustworthy. The
  adversary has not modified it to insert a backdoor, or even to make side
  channels easier to exploit.

- The adversary does not have physical access to the air-gapped system while it
  is powered on.

- If the adversary gains physical access to the air-gapped system while it is
  powered off, their access will be detected and can be acted upon.

- There are at least two such locations where these assumptions hold. In other
  words, the air-gap system is at one location, and there is a distinct
  "Location X" that also satisfies these properties, with the additional
  assumption that transport to and from this location satisfies these
  properties.

### Side Channels

- The adversary does not have detailed enough access to the air-gapped system's
  power usage to perform side channel analysis. (Question: Can we eliminate this
  assumption by having the air-gap system run off a UPS that isn't plugged in
  whenever it's powered on?).

- The adversary cannot measure electromagnetic radiation coming from the air-gap
  system with enough detail to perform side channels.

- The adversary cannot measure acoustic emanations from the air-gap system with
  enough detail to perform side channels.

- The above side channel points also hold for the air-gap system's
  "surrounding environment", e.g. the adversary cannot hear the user typing.

- Things that move into and out of the side-channel-protected area are not
  themselves side channels. For example, we assume that the person using the
  AirGap is not wearing a recording device that records the sound of them
  typing, to be retrieved by the adversary after they leave the
  side-channel-protected zone. Another example, we assume the adversary does not
  have a microscopic flying robot that hides in your lungs, flys out, records
  your typing, then flys back into your lungs.

### Electronic Compromise (Malware Capabilities)

- The adversary cannot maintain a compromise across a Tails reboot. That is, if
  we boot Tails from a DVD-R, give the adversary root access for a while, then
  power the system off (completely), then boot back into Tails, the adversary
  will no longer have useful access to the system (they should not be able to
  see or modify what we're doing in Tails). This excludes things like BIOS
  viruses, or viruses that hide in other hardware components.

- The adversary cannot compromise a Tails boot. That is, if we give the
  adversary root access (but *not* physical access) to a computer running any
  operating system, then completely power off the system, then boot a Tails Live
  CD from a DVD-R [2], the adversary cannot compromise the Tails environment.

- Suppose the adversary can write arbitrary data to a flash drive (we suppose
  they *can not* physically modify the drive), then plug it in to a system
  running Tails. This should not give the adversary access to the Tails
  environment.

- A physical True Random Number Generator (TRNG) is available, working, and is
  not compromised by the adversary.

- The adversary cannot exploit Tails even if they can:

    - Give arbitrary input files to the `dd` program.
    - Have an arbitrary file downloaded from the Internet using the operator's
      choice of download tool (e.g. wget).
    - Subject an arbitrary file to the operator's evaluation when considering
      signing the file. For example, if the operator needs to unzip a file to
      evaluate it for signing, the adversary should not be able to exploit
      a vulnerability in the `unzip` program.
    - Have the operator open arbitrary files in their text editor of choice.
    - Pass arbitrary files to be decrypted and encrypted to the `gpg` program.

- TODO: Add assumptions about code being secure from the primitives.

System Design (Physical)
-------------------------

The proposed design actually needs two computer systems. First, the system that
will actually be air-gapped. Second, an internet connected system that can boot
a Tails Live CD. The purpose of the second system is, when booted into Tails, to
provide an "adversary-free" environment for preparing and storing the data to be
sent and received from the air-gap system.

We need to give these systems names. The air-gap system will be called AirGap,
and the internet-connected Tails-booting system will be called GapProxy. We will
also need to give a name to a special flash drive called TransferMedia.

We will also suppose there is only one human operator of the airgap, which we
will refer to by the name HumanOperator.

### AirGap

AirGap consists of the following components:

- A desktop PC that that has no WiFi or Bluetooth capability, no speakers,
  and no microphone [3], with the hard drive removed, and the BIOS set to only
  boot from a CD.

- A keyboard dedicated to this system.

- A mouse dedicated to this system.

- A clean copy of Tails burned to a DVD-R disc.

- Four 16GB USB flash drives. We give these names: Store1, Backup1, Offsite1,
  Offsite2.

- Hardware Cryptographic (True) Random Number Generator [4]. Later, we'll refer
  to this device (or collection of devices) by the name HWRNG.

### GapProxy

- A desktop PC or laptop that can boot from CD. This does not need to be
  a dedicated system. It could be the user's regular PC, for example, as long as
  all of the assumptions above (especially the Physical Access ones) hold for it
  as well.

- A clean copy of Tails burned to a DVD-R disc.

- Any other peripherals (mouse keyboard, external drives, speakers, etc.) that
  happen to be plugged into it, as long as they are trustworthy and don't
  violate any of the security assumptions.

### TransferMedia

- A 16GB flash drive.

System Design (Policy)
-----------------------

The previous section describes the hardware we need. This section explains how
to use it. To do this, we define "primitives" that are used to construct the
desired functionality (listed in the Features section).

The instructions are written in a code-like format for HumanOperator to follow.

### Primitives

#### Primitive "SystemSetup()": Set up the system so other things can be done.

    ---------------------------------------------------------------------------
    Preconditions: 
        - The hardware listed in the System Design (Physical) section is
          available.
    Postconditions:
        - The air-gap system is configured so that the primitives and actions in
          the following sections can be performed.
    ---------------------------------------------------------------------------

    Place AirGap in a secure location.
    Boot AirGap to Tails.
    Permanently plug HWRNG into AirGap and ensure that it is working.
    Insert Store1, Backup1, Offsite1, Offsite2, and TransferMedia.
    Using `dd`, zero Store1, Backup1, Offsite1, Offsite2, and TransferMedia.
    Set up full disk encryption on Store1, Backup1, Offsite1, and Offsite2 using
        a password that only HumanOperator knows.
    Use GnuPG to generate a key pair and store the private key it on Store1.
    Write down the public key fingerprint on a physical piece of paper.
    Clone Store1's filesystem to Backup1's, Offsite1's, and Offsite2's.
    Remove all flash drives from AirGap.
    Power off AirGap.

    Send Offsite1 to Location X.

#### Primitive "RECV(url)": Transfer File from Internet to AirGap.

    ---------------------------------------------------------------------------
    Preconditions:
        - The SystemSetup() has been performed.
        - The file to be transferred is available from the public Internet, at
          the URL 'url'.
        - GapProxy is powered off.
        - AirGap is powered off and no flash drives are plugged in.
    Postconditions:
        - The file at 'url' has been written to the filesystem on Store1.
    ---------------------------------------------------------------------------
    
    Boot GapProxy to Tails.
    Retrieve the file from 'url'.
    Use `dd` to write the file directly to TransferMedia.
    Write down the exact file size on a physical piece of paper.
    Remove TransferMedia from GapProxy.
    Power off GapProxy.
    
    Boot AirGap to Tails.
    Insert and mount Store1 and Backup1.
    LocalBackup()
    Insert TransferMedia into AirGap.
    Use `dd` and the written-down size to transfer the file to Store1.
    Remove TransferMedia from AirGap.
    Remove all flash drives from AirGap.
    Power off AirGap.

#### Primitive "SEND(file)": Transfer File from AirGap to Internet.

    ---------------------------------------------------------------------------
    Preconditions:
        - The SystemSetup() has been performed.
        - The file to be uploaded is available as 'file' on Store1.
        - GapProxy is powered off.
        - AirGap is powered off and no flash drives are plugged in.
    Postconditions:
        - The file is available for download from some URL on the Internet.
    ---------------------------------------------------------------------------
    
    Boot AirGap to Tails.
    Insert and mount Store1 and Backup1.
    LocalBackup()
    Insert TransferMedia into AirGap.
    Use `dd` to write the file to TransferMedia.
    Write down the exact file size on a piece of paper.
    Remove TransferMedia from AirGap.
    Remove all flash drives from AirGap.
    Power off AirGap.

    Boot GapProxy to Tails.
    Use `dd` and the written-down size to obtain the file from TransferMedia.
    Upload the file to the Internet and obtain the URL.
    Write down the URL where the file can be downloaded.
    Remove TransferMedia from GapProxy.
    Power off GapProxy.

#### Primitive "OffsiteBackup()": Backup AirGap to Location X.

    ---------------------------------------------------------------------------
    Preconditions:
        - The SystemSetup() has been performed.
        - AirGap is powered off and no flash drives are plugged in.
    Postconditions:
        - The current contents of Store1 are replicated to Location X.
    ---------------------------------------------------------------------------
    
    Boot AirGap to Tails.
    Insert and mount Store1 and Backup1.
    If Offsite1 is at Location X
        Insert and mount Offsite2.
        Copy files on Store1 to Offsite2 so that they have identical contents.
        // The order of these steps is important so that there's no SPOF.
        Send Offsite2 to Location X.
        Retrieve Offsite1 from Location X.
    Else
        Do the same as above, but with Offsite2 instead of Offsite1.
    End
    Remove all flash drives from AirGap.
    Power off AirGap.

#### Primitive "LocalBackup()": Backup AirGap locally.

    ---------------------------------------------------------------------------
    Preconditions:
        - The SystemSetup() has been performed.
        - AirGap is powered on and Store1 and Backup1 are mounted.
    Postconditions:
        - Store1 and Backup1 have identical contents.
    ---------------------------------------------------------------------------

    Copy files on Store1 to Backup1 so that they have identical contents.

### Performing Features

Here we explain how each Goal can be met by using the primitives.

In this section, we explain how each feature listed in the Features section can
be performed using the primitives we defined in the last section.

Note: If the steps are followed to the letter, there is a lot of unnecessary
rebooting involved. If the operator is careful, they can eliminate these
unnecessary reboots (use common sense).

#### Feature ExamineAndSignFile(url): Examine a file on the Internet then sign it.

    ---------------------------------------------------------------------------
    Preconditions:
        - The SystemSetup() has been performed.
        - AirGap is powered off and no flash drives are plugged in.
        - RECV()'s preconditions are met.
    Postconditions:
        - The operator has reviewed the file at 'url' and, if they chose to, has
          uploaded a signature of that file to the Internet, available through
          some URL.
    ---------------------------------------------------------------------------

    RECV(url) to a 'file' on Store1.

    Boot AirGap to Tails.
    Insert and mount Store1 and Backup1.
    LocalBackup()
    // Make sure to examine 'file' safely!
    HumanOperator should now examine 'file' and decide if they want to sign it.
    If HumanOperator wants to sign 'file'
        Sign 'file' with GnuPG.
        Remove all flash drives from AirGap.
        Power off AirGap.
        SEND(the signed file)
    Else
        Remove all flash drives from AirGap.
        Power off AirGap.
    EndIf

#### Feature DecryptPGPMessage(url): Decrypt an PGP message from the Internet.

    ---------------------------------------------------------------------------
    Preconditions:
        - The SystemSetup() has been performed.
        - AirGap is powered off and no flash drives are plugged in.
        - RECV()'s preconditions are met.
    Postconditions:
        - The the file at 'url' has been decrypted and stored on Store1.
        - The operator is able to view the decrypted file's contents.
    ---------------------------------------------------------------------------

    // NOTE: The public key of the file sender should be first obtained with
    // RECV, and the fingerprint should be verified.

    RECV(url) to 'file' on Store1.

    Boot AirGap to Tails.
    Insert and mount Store1 and Backup1.
    LocalBackup()
    Decrypt and verify the signature of 'file' with GnuPG.
    Optionally, view the file with a text editor.
    Remove all flash drives from AirGap.
    Power off AirGap.

#### Feature EncryptPGPMessage(): Let the operator encrypt and sign a message.

    ---------------------------------------------------------------------------
    Preconditions:
        - The SystemSetup() has been performed.
        - AirGap is powered off and no flash drives are plugged in.
    Postconditions:
        - The operator has composed their message and saved it on Store1.
        - The message ciphertext, encrypted to its recipient, is available for
          download from some Internet URL.
    ---------------------------------------------------------------------------

    // NOTE: The public key of the message recipient should first be obtained
    // with RECV, and the fingerprint should be verified.

    Boot AirGap to Tails.
    Insert and mount Store1 and Backup1.
    LocalBackup()
    Type the message and save it to a file on Store1.
    Encrypt and sign the file with GnuPG.
    Remove all flash drives from AirGap.
    Power off AirGap.

    SEND(the encrypted file).

#### Feature: Back up the system.

- LocalBackup() is run every time after AirGap boots, so that the files on
  Backup1 are synced to the files on Store1, being at most one use behind.

- OffsiteBackup() should be performed at least once every 30 days, so that the
  flash drive stored in Location X (either Offsite1 or Offsite2) is synced with
  Store1, at most 30 days behind.

### Declarative Policies

This section contains extra bits of policy that are declarative, and apply for
all time, unlike the procedural policy in the previous section.

#### No Re-Use of Hardware

Every piece of hardware that touches AirGap should never be used with any other
system again. For example, AirGap's keyboard should never be used with another
computer. The *only* exception to this rule is the TransferMedia flash drive.

#### Preventing Accidental Misuse

Precautions should be in place to prevent accidental misuse. Note that these
precautions are not intended to be robust, they are only intended to be
informative. They are to make it very obvious when the operator, or someone else
granted access to the air-gap area, is doing something wrong. They are not
intended to stop attacks.

- Keep AirGap and all peripherals stored in a locked container when powered off.
  This prevents accidental misuse/repurposing of the hardware.

- When transporting Offsite1 and Offsite2 to and from Location X, as well as
  while they are stored at Location X, they should be in a locked container
  whose only keys are kept with the AirGap hardware.

- Place warning signs explaining what the AirGap hardware is and why none of its
  components should be re-purposed.

- Place instruction signs, or an instruction manual, for the operator to follow
  in case they forget any of the procedures defined above.

- AirGap's BIOS should be configured to never boot from a flash drive. As
  a secondary precaution, all flash drives should be removed when booting
  AirGap.

#### No Execution from Store1, Backup1, Offsite1, or Offsite2

At no point should any data on Store1, Backup1, Offsite1, or Offsite2 be
executed. The term "executed" is broad, so it is up to the operator to choose
a conservative definition. The following things should be explicitly *not*
allowed:

- Executing binary code.
- Executing shell scripts or scripting language scripts.
- Viewing documents whose viewers execute macros.
- Copying and pasting commands that are stored in a text file.
- Manually following *any kind* of instructions stored in a text file.

#### Decommissioning

When the system is decommissioned, all components should be physically
destroyed. This is not only to destroy traces of secret data, it also ensures
that once the system is decommissioned, it is never re-commissioned into another
air-gap system, after potentially being compromised while decommissioned.

Detecting Attacks
------------------

Responding to Attacks
----------------------

Restoring Backups
------------------

Conclusion
-----------

Notes
------

[1]: Where the process of examination is assumed to maintain system integrety
     even though the file being examined may be malicious.

[2]: We should also assume that the Tails CD was in the drive when the adversary
     had root access, and that that drive is capable of burning DVD+/-RW discs.

[3]: We've already assumed the adversary can't receive these signals even if
     AirGap did have these things, but having this requirement makes that
     assumption much easier to satisfy in practice.

[4]: Note that this can, and should, be more than one device. There should
     always be at least two independent sources of entropy. Not only because it
     prevents one's backdoor from being exploitable (if only one of the two are
     backdoored), but it reduces the chance of accidental failure. A webcam can
     be used as a secondary randomness source. Just pipe the video stream into
     /dev/random.
