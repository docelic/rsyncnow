# rsyncnow

This tool is for you if you have a need to rsync data sets so large
that just building the index of files to sync takes days or weeks.

## Is it any good?

YES.

## The rsync problem

Rsync's delta algorithm doesn't need any changes - once a file to sync
is identified, the algorithm does its job well (or the whole file is
copied with rsync option `-W`).

But when rsync is told to sync two directories, before it begins with
the actual syncing, it builds an index of all the files that need to be
synced.

On large data sets, this index building can take hours, days, or weeks.

In addition to just taking time, it is making it harder to schedule data
migrations at the end/beginning of months when the extra traffic will not
influence the month's 95th percentile and is essentially free.

Also, it is making it harder to fully sync the source and destination if
the source is still being modified or uploaded to, because by the time
rsync completes, the destination is already out of date and requires
another sync.

## The gist of rsyncnow operation

So how does `rsyncnow` help?

The above-described behavior of rsync in which it first builds an index
and then starts syncing cannot be changed.

However, by using the appropriate command line options, rsync does support
a mode where it will print the files that need syncing to STDOUT without
delay (it will print them immediately as it finds/identifies them, during
index building).

This enables `rsyncnow` to introduce a huge increase in efficiency as follows:

1. It runs a set of `rsync` processes (by default 1) that will be finding
files to sync (in dry run mode) and printing them to STDOUT as a stream.
We call these processes `finders`.

1. As the finders keep printing files to sync, `rsyncnow` keeps reading
their STDOUT in real-time and pushing the files to be synced to a small
internal queue.

1. As it populates the queue and finds enough paths to form a "block"
(or every X seconds if a block has not been filled up yet), it runs
separate rsync processes (called "syncers") which are given those
specific files to sync. Syncers start syncing immediately since they
are given specific paths, there are no indexes to build.

1. Additionally, if the files to sync are being found faster than they
are synced, and the bandwidth/resource limits allow it, one can run
`rsyncnow` with multiple `syncer` processes to achieve faster/concurrent
syncing of multiple files.

## Usage instructions

You need Ruby installed to run the script. This is hopefully a trivial requirement.


```
Usage: rsyncnow [OPTIONS...] SRC... DST -- [FIND OPTIONS...] -- [SYNC OPTIONS...]

OPTIONS:
  -f, --finders 1    - Number of rsync find processes. Currently always gets
                       reset to the number of specified SRC paths
  -s, --syncers 1    - Nr. of respawning rsync sync/copy processes, per finder
  -b, --blocksize 5  - Number of files to queue before starting a sync
  -q, --queuesize 50 - Max number of paths to queue for sync. If not specified,
                       defaults to blocksize * 10. Rsync find processes get
                       automatically paused when their queue goes above this
                       limit and are resumed when queue falls below threshold
  -t, --timeout 5.0  - After timeout seconds, run rsync sync/copy process even
                       if block wasn't full

  -r, --rsync rsync  - Name (and/or path) of rsync binary

  -v, --verbose      - Enable rsyncnow and rsync verbose mode
  -h, --help         - Show help and exit
  -e, --examples     - Show examples and exit

SRC, DST:
  Rsync source and destination as usual (including the "/" magic)

FIND OPTIONS:
  If specified, overrides all default cmdline options for rsync find processes.
  If you use this, options `-niR` must always be present/included.
  Default value: -aniRe=ssh

SYNC OPTIONS:
  If specified, overrides all default cmdline options for rsync sync processes.
  If you use this, options `-0 --files-from=-` must always remain present.
  Default value: -lptgoD0e=ssh --files-from=-
    NOTE: options -lptgoD are used explicitly instead just specifying -a
    because -a also includes option -r which should not be present. Recursion
    is controlled via FIND OPTIONS (where it is enabled/implied by -a)

EXAMPLES:

# Most basic example:
# (implies finding files to sync with rsync options -aniRe=ssh,
# and syncing the actual files with rsync options -lptgoD0e=ssh --files-from=-)
rsyncnow -v /source/dir /target/dir

# Finding files with size differences only, without full checksum (--size-only), and
# syncing them by copying, without using rsync's delta algorithm (-W):
rsyncnow -v /source/dir /target/dir -- -aniRe=ssh --size-only -- -lptgoD0e=ssh --files-from=- -W
```

## Extra notes

Currently there is always 1 rsync finder process that is started for each
source directory, concurrently. If you don't want multiple finders running
at the same time (for example if all source directories to sync are on the
same partition), you should call `rsyncnow` multiple times with 1 source path
in every invocation instead of once with multiple source paths.

Rsyncnow doesn't put any restrictions on the rsync options that one can use in
either find or sync phase (options related to comparing/finding files,
what to copy/sync, max bandwidth to use etc.).
See a myriad of options available in the [rsync man page](https://download.samba.org/pub/rsync/rsync.1).

## Feedback

Please report any comments or suggestions!
