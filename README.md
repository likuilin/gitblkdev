# gitblkdev

This project consists of several bash scripts that allow a user to meaningfully use version control on an entire block device.

TODO: The entire project is unimplemented. This README is currently just a plan, and all details of it may change as I find free time to actually make this thing.

TODO: Example use will go here.

## What? [Overview]

* `gitblkdev-pack.sh [FILE] [TARGET]` will read FILE as a block device and create/update TARGET, the target repo (a directory (likely a git repo)), into a state that can reversibly be mapped back to the original FILE. Any unsaved changes in the target repo will be deleted. I alias this to `gbd`.
* `gitblkdev-unpack.sh [TARGET] [FILE]` will do the reverse and write to FILE based on TARGET. It reads chunks on FILE first, to not re-write a chunk that doesn't need to be written. For safety, it also writes the current FILE back to TARGET (so, the two are exchanged). I alias this to `gbd -r`.

The contents of the target repo that are managed by gitblkdev are:
* `./config.txt` - Configuration parameters, including a listing of "special" sections. The format is explained in the file itself. Arbitrary (unaligned) sections can be defined here, as well as ignored sections.
* `./checksum.sha256` - A checksum of the entire block device, portably equivalent to running `cat [FILE] | sha256sum` directly.
* `./checksum-sections.sha256` - A checksum of all of the sections. Not portable, but, unlike the other checksum, it is stable when ignored sections change.
* `./data/section-[NAME].txt` - Files containing hashes of chunks with sections. Default sections are named according to their offset, whereas manual sections are named as configured.
* `./data/chunks/[HASH].[SIZE].bin.gz` - Chunks of actual data, compressed and stored by their hash and uncompressed size.

Extra files and directories may be stored in the target repo, and will not be read or written to by gitblkdev. This enables this directory itself to be a git repo or worktree.

I recommend checking out this repo (with the gitblkdev scripts) into the target repo as a submodule, in order to pin what version generated the repo, in case of future changes or bugs. However, that is not required, and gitblkdev itself makes no version checks. The block device doesn't have to be a block device - any file that can be seeked will work just fine. The scripts are dependency-lite and try to make as few assumptions as possible.

## Why?? [Motivation]

I like version control. I don't trust filesystem implementations to not leave metadata lying around if, for instance, I perform a sequence of operations that I thought wouldn't change anything on the disk (for instance, adding and then immediately removing a key from LUKS (did you know it has an epoch?), or mounting a filesystem read-only (ntfs touching atime is the classic example, but did you know that some btrfs operations can modify an ro-mounted drive, and some combinations of overlays can cause modifications _on mount_? Read-only is a statement about the filesystem side of an FS impl, not the block-device side!)). I also don't trust my future self to not make mistakes.

Like most nerds in our field, I spent a good chunk of my youth messing around with weird boot configurations and partition layouts. While I was doing that, because of what I outlined above, I wished I had a way to easily record or revert changes. Sure, if I thought that I might regret an action I was going to take, then I could `cat /dev/nvme0n1p19 | gzip > backup.bin.gz` and then restore it in the future - but, that can potentially be slow and is incredibly unsatisfying. If I only changed a few bytes, but I just didn't know where those bytes were changed, why do I need to wait while my computer writes and compresses data it already previously saved?

This is clearly a job for version control! But, I can't just `git add /dev/nvme0n1` and call it a day, because git's design relies on the assumption that one file is one human-comprehensible unit, and with disk layouts, it is all one file. More specifically, the human-comprehensible unit layout that we'd want is dependent on the implementation details of _multiple_ disparate layers of abstraction (for example, a GPT-labelled disk with a partition that's an LVM pv (internally dm), which contains a LUKS volume (internally dm-crypt), which contains a btrfs filesystem, which references another device as its seed device that's outside the lv...) in a way that can't really cohere together without more mental overhead than I want to take. Manually calculating dm offsets is not a vibe, especially when it's done to be proactively protective.

Hence, this project. I wish it existed back when I was experimenting. But, I'm glad it exists now, even if I'm the only one who uses it.

## How? [Implementation]

This section describes how `gitblkdev-pack.sh` works - unpack can be extrapolated.

First, we delete `./data` - it should be version-controlled anyways, so discarding unsaved changes is OK.

Then, the configuration is parsed to determine the chunk size. This chunk size cannot be changed without essentially recreating all the files, which the target repo's version control wouldn't like. A chunk should be small enough so that copying one chunk is relatively fast, but large enough so that a list of chunks up to the size of any (default or manually-defined) section can be comfortably stored and interpreted as a flat text file.

Then, the sections are parsed. Manual section definitions are completely optional - they just exist to make the internal data structure better represent the actual layout of a disk. For instance, if a partition boundary is inside a chunk, then that chunk will contain both of the partitions. In this case, the chunk is modified if the start of the second partition is modified, and it's also modified if the end of the first partition is modified. So, the target repo's version control will emit unexpected merge conflicts if, say, two changes need to be merged that both touch this chunk, but that the user does not expect to logically conflict. A manually-defined section will split the data into two separate text files, causing them to not conflict.

Sections are not allowed to overlap. I guess this could be designed in such a way where manually-defined sections can overlap, but that would push the complexity past the point where this cannot be a bash script, in my opinion - I'd rather write that in C, but then what about portability, and... eh, meh.

Then, the default sections are determined. For every maximal continguous sequence of bytes on the disk that is not covered by a section, that sequence is split into sections that are no larger than the default section size.

Finally, the disk is read contiguously, and the data is copied into both the actual section data files and piped into `checksum.sha256`. For efficiency, only one read pass is needed.

## Who/Where? [License]

(Okay, the subsection title is a bit of a stretch, lol)

This project is currently explicitly not licensed, except via this explicitly-vague statement wherein I declare that I don't expect wanting or needing to pursue any legal action in the future against anyone for using, modifying, reverse-engineering (lol), and whatevering this code, unless this project becomes popular enough where I start caring about my ability to take rent from for-profit institutions, which aren't completely aligned with my interests, that also want to use this code.

I think this idea is very novel, because I looked around for a suitable solution before undertaking the task of rolling my own, and couldn't find any. Unfortunately, I don't have much of a moat - most of the effort of this idea is in the ideation and the design, and, given the broad design, the rest can pretty easily be black-box-reimplemented by anyone over a few weekends. More crucially, anyone who knew enough about how computers work to _care enough to want_ this thing to exist also has enough skill to _make it_ in the first place.

So, in the context of the only goal that really matters with regards to license choice (see the first line and, more broadly, capitalism), no specific license choice made in advance would actually help much. If I'm wrong about any of this, please do let me know, maybe I should file a patent or whatever.
