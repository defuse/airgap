WARNING
========

**I'm still writing this guide. There are probably *serious* mistakes. Do not
implement it until it has had some more review.**

Airgap
=======

This document presents the design of, and procedures surrounding, a practical
and affordable air-gap system. We aim to provide a minimal set of useful
functionality, with minimal security assumptions.

The target audience is technically proficient people who could benefit from an
air-gap system, but currently are not doing so because they either don't know
how or because they think it's too expensive. Examples of such people are
software developers (who want to sign their code), security consultants (who
want to sign the copy of the software they audited), and journalists who want to
accept material from sources securely. People with extremely high risk, such as
those being targeted by government agencies, must realize that, for them, there
is no "one size fits all" defense, and that while they may use this document as
a starting point, they must proceed with much more caution.

**Important Note:** We are *not* trying to design something that's NSA-proof.
Rather, the goal is to be *secure enough that a targeted physical attack is
necessary*.  We should strive to make the NSA's job as hard as possible
(expensive with lots of detection risk), but we should not overcomplicate things
trying to protect ourselves from an adversary as powerful as the NSA.

Features
---------

We want an air-gapped system that can perform the following tasks securely:

- Let the operator examine a file provided from the outside, and after
  reviewing it, sign it with a PGP private key.

- Let the operator decrypt text messages that were encrypted to their public
  key.

- Let the operator encrypt and sign text messages (typed in to the air-gap
  system) to a public key that was provided from the outside, possibly in
  response to a message they decrypted.

- Backup the air-gap system locally and to an off-site secure location.

More functionality can be added at the operator's discretion. However, anything
more than this is out of scope of this document and may require considerable
design and policy changes.

Security Assumptions
---------------------

This section should be an *exhaustive* list of all the things that need to be
assumed for this air-gap design to be secure. The assumptions should be
*reasonable*, i.e. not assuming unrealistic things. Unreasonable or conflicting
assumptions are considered vulnerabilities in the design.

We should, as much as possible, avoid assuming code is secure.

If you are implementing the proposed design, it is *your responsibility* to
ensure that these assumptions or satisfied against the class of adversaries that
you want to defend against. This document *does not* tell you what to do to
satisfy these assumptions. It *assumes* they are satisfied, and tells you how to
be secure under these assumptions.

If these assumptions are satisfied, the proposed design should be secure. The
converse -- that if the assumptions are not satisfied, then the proposed design
is not secure -- is not necessarily true. Given what is assumed here, it is
possible to have security without needing an air-gap at all. The air-gap
provides an additional safety margin, so that the adversary (sometimes) has to
contradict more than one of the assumptions in order to compromise the system.

### Kerckhoffs's Principle

- We assume the adversary knows everything about how the air-gap works (i.e.
  they have this document).

### Physical Access

- The hardware used for the air-gap system is (at least initially) trustworthy.
  The adversary has not modified it to insert a backdoor, or even to make side
  channels easier to exploit.

- The adversary does not have physical access to the air-gapped system while it
  is powered on.

- If the adversary gains physical access to the air-gapped system while it is
  powered off, their access will be detected and can be acted upon.

- There are at least two locations where these assumptions hold. In other words,
  the air-gap system is at one location, and there is a distinct "Location X"
  that also satisfies these properties, with the additional assumption that
  transportation to and from this location is secure.

### Side Channels

- The adversary cannot measure acoustic emanations, electromagnetic radiation,
  or the power fluctuations coming from the air-gapped system, or its
  surrounding environment, with enough accuracy to perform side-channel attacks.

- Things that move into and out of the side-channel-protected area are not
  themselves side channels. For example, we assume that the person using the
  AirGap is not wearing a recording device that records the sound of themselves
  typing, to be retrieved by the adversary after they leave the
  side-channel-protected zone. 

- TODO: GapProxy has its own set of assumptions. Add those here.

### Electronic Compromise (Malware Capabilities)

- The adversary cannot compromise a Tails boot. That is, if we give the
  adversary root access (but *not* physical access) to a computer running any
  operating system, then completely power off the system, then boot a Tails Live
  CD from a DVD-R [1], the adversary cannot compromise the Tails environment.

- Suppose the adversary can write arbitrary data to a flash drive (we suppose
  they *can not* physically modify the drive), then plug it in to a system
  running Tails. This should not give them access to the Tails environment.

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

- TODO: Carefully check these assumptions against what happens in the primitives
  and features.

System Design (Physical)
-------------------------

The proposed design actually needs two computer systems. First, the system that
will actually be air-gapped. Second, an internet connected system that can boot
a Tails Live CD. The purpose of the second system is, when booted into Tails, to
provide an "adversary-free" environment for preparing the data to be sent to the
air-gapped system, and for receiving data from the air-gapped system.

We need to give these systems names. The air-gap system will be called AirGap,
and the internet-connected Tails-booting system will be called GapProxy. We will
also need to give a name to a special flash drive called TransferMedia.

We will also suppose there is only one human operator of the airgap, which we
will refer to by the name HumanOperator.

### AirGap

AirGap consists of the following components:

- A desktop PC that that has no WiFi or Bluetooth capability, no speakers,
  and no microphone [2], with the hard drive removed, and the BIOS set to only
  boot from a DVD-R.

- A keyboard dedicated to this system.

- A mouse dedicated to this system.

- A clean copy of Tails burned to a DVD-R (*not* DVD-RW) disc. **TODO:**
  Find out if DVD-R is really unwritable, maybe you can "OR" to it with a DVD-RW
  drive, in which case you could still modify code.... is that possible?

- Four 16GB USB flash drives. We give these names: Store1, Backup1, Offsite1,
  Offsite2.

- Hardware Cryptographic (True) Random Number Generator [3]. Later, we'll refer
  to this device (or collection of devices) by the name HWRNG.

### GapProxy

- A desktop PC or laptop that can boot from DVD-R. This does not need to be
  a dedicated system. It could be the user's regular PC, for example, as long as
  all of the assumptions above (especially the Physical Access ones) hold for it
  as well. **TODO:** Instead of making this assumption here, it should be
  written explicitly in the Security Assumptions section.

- A clean copy of Tails burned to a DVD-R disc.

- Any other peripherals (mouse keyboard, external drives, speakers, etc.) that
  happen to be plugged into it, as long as they are trustworthy and don't
  violate any of the security assumptions. If feasible, unplug everything.

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
    // TODO: encrypted drives should be urandom'd, not zeroed.
    Set up full disk encryption on Store1, Backup1, Offsite1, and Offsite2 using
        a 128-bit equivalent passphrase that only HumanOperator knows.
    // TODO: The private key must be protected with a passphrase
    Use GnuPG to generate a key pair and store the private key on Store1.
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
    EndIf
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

    **TODO** Backups are write-frequently, read-almost-never. So full-disk
    encryption is not the best way to encrypt them. There should at least be
    some integrity checking... maybe encrypt them with GPG?

### Performing Features

Here we explain how each Goal can be met by using the primitives.

In this section, we explain how each feature listed in the Features section can
be performed using the primitives we defined in the last section.

Note: If the steps are followed to the letter, there is a lot of unnecessary
rebooting involved. If the operator is careful, they can eliminate these
unnecessary reboots (use common sense). **TODO:** It might actually be useful to
keep these reboots in, e.g. so that the private key is never decrypted while the
TransferMedia is plugged in... think about this more.

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

### Passwords

- TODO: Don't reuse passwords, either memorize or (worse) write them down (or
  a bit of both). Figure out where to keep the passwords (e.g. the offsite
  passwords can probably be kept physically with the airgap? -- actually, no,
  because then it'd still be easier to decrypt them than the main one)

#### No Re-Use of Hardware

Every piece of hardware that touches AirGap should never be used with any other
system again. For example, AirGap's keyboard should never be used with another
computer. The *only* exception to this rule is the TransferMedia flash drive.

#### Keep AirGap Powered Off

AirGap should never be powered on for a period longer than one hour, and it must
be attended by the operator for the entire time it is powered on.

#### Preventing Accidental Misuse

**TODO:** These points might be useful to *some* implementations, but could be
harmful to others. Either remove these points or designate them as "only things
you should do if your situation is similar to mine".

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
  a secondary precaution, all flash drives should be removed before booting
  AirGap.

#### No Execution from Store1, Backup1, Offsite1, or Offsite2

At no point should any data on Store1, Backup1, Offsite1, or Offsite2 be
executed. The term "executed" is broad, so it is up to the operator to choose
a sufficiently conservative definition.

The following things should be explicitly disallowed:

- Executing binary code.
- Executing shell scripts or scripting language scripts.
- Viewing documents whose viewers execute macros or are likely to contain
  exploitable vulnerabilities (e.g. PDF viewers).
- Copying and pasting commands that are stored in a text file.
- Manually following *any kind* of instructions stored in a text file.

#### Decommissioning

When the system is decommissioned, all components should be physically
destroyed. This is not only to destroy traces of secret data, it also ensures
that once the system is decommissioned, it is never re-commissioned into another
air-gap system, after potentially being compromised while decommissioned.

Restoring Backups
------------------

So far, we have described how to *create* backups of the air-gap system. We have
yet to describe how to *restore* these backups. The restore process is as
follows:

    ---------------------------------------------------------------------------
    Preconditions:
        - AirGap is powered off and no flash drives are plugged in.
            (Note, if one of the offsite backups is being used, it might be the
             case that all of AirGap's hardware is new. That's ok, as long as
             SystemSetup() has been performed on the new hardware.).
        - Store1 is broken.
        - One of Backup1, Offsite1, and Offsite2 contain the most recent backup.
    Postconditions:
        - Store1 is replaced by a new drive.
        - The old Store1 is decommissioned.
        - The new Store1 contains the restored backup.
    ---------------------------------------------------------------------------

    Boot AirGap to Tails.
    Insert and mount Store1 (new) and the backup flash drive.
    Copy the backup onto Store1 (new).
    Remove all flash drives from AirGap.
    Power off AirGap.

    Decommission the old Store1.

Even though backups are created frequently, data loss is still possible.
Sometimes, losing data can create other security issues, or can create
situations that require a special response. It is up to the operator to identify
these cases and respond appropriately.

Detecting and Responding to Attacks
------------------------------------

This document describes a system that should be 100% secure as long as the
security assumptions are met. If it is possible to compromise the system without
violating one of the security assumptions, then this document is flawed.

In practice, however, the security assumptions *will* be violated, and the
system *will* be compromised. It is therefore important to try to detect these
violations, and then respond in a way that protects the integrity of the system.
It is not always possible to detect a violation, and sometimes there is no good
response to a violation. However, a "best effort" at detection and response
should be made.

The details of detection and response are out of scope of this document.

Conclusion
-----------

Audit Status
-------------

This document has not yet received any third-party review.

Notes
------

[1]: We should also assume that the Tails CD was in the drive when the adversary
     had root access, and that that drive is capable of burning DVD+/-RW discs.

[2]: We've already assumed the adversary can't receive these signals even if
     AirGap did have these things, but having this requirement makes that
     assumption much easier to satisfy in practice.

[3]: Note that this can, and should, be more than one device. There should
     always be at least two independent sources of entropy. Not only because it
     prevents one's backdoor from being exploitable (if only one of the two are
     backdoored), but it reduces the chance of accidental failure. A webcam can
     be used as a secondary randomness source. Just pipe the video stream into
     /dev/random. A cautious operator should add entropy manually (dice rolls,
     coin flips, etc.).

TODOs
-------

- In practice I screwed up by copying the plaintext (reply.txt) instead of the
  ciphertext (reply.txt.asc), which leaked the contents of my message. There
  needs to be some kind of policy, or technical measure, or something, to make
  this less likely.
