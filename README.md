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
  reviewing it, sign it with a private key.

- Let the operator decrypt ASCII text messages that were encrypted to their
  public key.

- Let the operator encrypt and sign ASCII text messages (typed in to the air-gap
  system) to a public key that was provided from the outside, possibly in
  response to a message they decrypted.

- Backup the air-gap system locally to an off-site secure location.

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

**TODO: We haven't said anything about encryption, or what to do when a physical
compromise is detected!!**

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
    Write 1KB from /dev/urandom to a Randomness file on Store1.
    Clone Store1's filesystem to Backup1's, Offsite1's, and Offsite2's.
    Unplug all flash drives.
    Power off AirGap.

    Send Offsite1 to Location X.

    TODO: add the explicit RNG read steps to the other shit

#### Primitive "RECV(url)": Transfer File from Internet to AirGap.

    ---------------------------------------------------------------------------
    Input: A URL with the file to transfer to AirGap.
    Result: AirGap is running and the file from the url is available.
    ---------------------------------------------------------------------------
    
    // TODO: there are assumptions in here like following this process doesn't lead
    to compromise, i.e. you can download files safely
    
    Boot GapProxy to Tails.
    Retrieve the file from 'url'.
    // Don't bother with filesystems, just write it to the beginning of the
    // device.
    Use `dd` to write the file to TransferMedia.
    Write down the exact file size on a physical piece of paper.
    Remove TransferMedia from GapProxy.
    Power off GapProxy.
    
    Boot AirGap to Tails.
    Insert and mount Store1 and Backup1.
    LocalBackup()
    Insert TransferMedia into AirGap.
    Mount the storage USB drive. // TODO: clarification
    // Use the written-down file size for the following step:
    Use `dd` to transfer the file onto the storage USB drive.
    Remove TransferMedia from AirGap.

#### Primitive "SEND(file)": Transfer File from AirGap to Internet.

    ---------------------------------------------------------------------------
    Input: AirGap is running and a file is available.
    Result: A URL on the public Internet where that file can be downloaded.
    ---------------------------------------------------------------------------
    
    Insert TransferMedia into AirGap.
    Use `dd` to write the file to TransferMedia.
    Write down the exact file size on a peice of paper.
    Remove TransferMedia from AirGap.
    Remove all flash drives.
    Power off AirGap.

    Boot GapProxy to Tails.
    Use `dd` and the written-down size to get the file.
    Upload the file to any website.
    Write down the URL where the file can be downloaded.
    Remove TransferMedia from AirGap.
    Power off GapProxy.

#### Primitive "OffsiteBackup()": Backup AirGap to Location X.

    TODO: pre/post
    Boot AirGap to Tails.
    Insert and mount Store1 and Backup1.
    If Offsite1 is at Location X
        Insert and mount Offsite2.
        Clone the files on Store1 to Offsite2.
        Send Offsite2 to Location X.
        Retrieve Offsite1 from Location X.
    Else
        Do the same as above, but swap Offsite1 and Offsite2.
    End
    Remove all flash drives from AirGap.
    Power off AirGap.

#### Primitive "LocalBackup()": Backup AirGap locally.

    TODO: pre/post
    Clone the files on Store1 to Backup1.

### Performing Features

Here we explain how each Goal can be met by using the primitives.

#### Goal ExamineAndSignFile(url): Examine a file on the Internet then sign it.

    TODO: add pre/post

    RECV(url)
    Examine the file.
    If you want to sign the file:
        Sign the file.
        SEND(signature file)
    Else
        Remove all flash drives.
        Power off AirGap.
    EndIf

#### Goal DecryptPGPMessage(url): Decrypt an ASCII PGP message from the Internet.

    TODO: add pre/post

    RECV(url)
    Decrypt the file. // TODO: how to do this safely, ascii-only?
    View the file.
    Remove all flash drives.
    Power off AirGap.

    // If they want to EncryptPGPMessage() next, do they really have to power
    // off here? I don't think it can be useful, but maybe...

#### Goal EncryptPGPMessage(): Let the operator encrypt and sign a message.

    TODO: add pre/post

    // Before doing this, if you need to, use RECV() to get the public key, then
    // verify the fingerprints.
    Boot AirGap to Tails.
    Insert and mount Store1 and Backup1.
    LocalBackup()
    Type the message and save it to a file (keep it in non-volatile media).
    Encrypt and sign the file with GPG. // TODO: details?
    SEND(the encrypted file).

#### Goal Backup(): Backup the system.

- LocalBackup() is run every time after AirGap boots, so that the files on
  Backup1 are synced to the files on Store1, being at most one use behind.

- OffsiteBackup() should be performed at least once every 30 days, so that the
  flash drive stored in Location X (either Offsite1 or Offsite2) is synced with
  Store1, at most 30 days behind.

### Extra Conditions

TODO: put things that should hold for all time here (i.e. not procedure), like
not plugging the USB into anythign else.

#### No Executable Code on Store1, Backup1, Offsite1, or Offsite2

#### Preventing Accidental Misuse

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
