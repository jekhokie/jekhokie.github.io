---
layout: post
title:  "Disk Recovery - Photos"
date:   2020-07-24 22:00:00 -0400
categories: linux python forensics
logo: disk-recovery-photos.jpg
---

Panic usually ensues when an external disk drive used for photo (or other file backups) fails or is leading towards failure and files are
seemingly lost. The good news is, if you're close enough to when the drive is about to fail, you can usually recover the files. However, if that
drive seems toast and you had been editing/storing the files on your local hard drive for any amount of time, you can usually recover many of
the previous photos assuming there hasn't been much activity on the drive or that positions on the drive have not been overwritten with new data.
This tutorial walks through how to go about recovering deleted files from a disk drive that have not yet been completely overwritten by new data
on the disk.

### Warning/Note

It is best, if you detect that your backup drive is going downhill, to have multiple copy solutions available. If you don't, make sure that
whatever device you had been using to interact with the external drive is immediately frozen. The faster and sooner you stop using the device,
the better chance you have at recovering data due to the existing bits on the disk not being overwritten by new data.

In addition, when performing a recovery, ensure you're doing so using an external drive that works. If you attempt to perform the following
recovery methods on the current drive that you wish to recover the data from, you risk that the data being recovered is going to overwrite
additional data that has not yet been recovered (because the operating system sees those positions as "free" space on the drive). In short,
plug an external drive into your laptop that can be used as a target for placing the recovered and filtered images into so you don't
inadvertently overwrite existing data on the drive being recovered.

### Photorec Software

There is a piece of free software known as [Photorec](https://www.wondershare.net/ad/photo-recovery/new.html) that can be used for recovering
data from disk drives that had previously been deleted. However, often times, this results in MANY duplicate images/files, and in addition,
a lot of garbage icons and low resolution images. While this is a great start (and we will use it to start), there is often a lot of pruning that
needs to occur so you don't need to parse potentially millions of photos.

This tutorial assumes you're performing this work on a Mac device, but may actually work for other operating systems such as Linux, etc.

#### Running Photorec

First, download the software from [this link](https://www.wondershare.net/ad/photo-recovery/new.html). Run the binary, and walk through the various
pages to configure the recovery operation you'd like to attempt. Make sure that you filter down the types of files that you wish to collect
or you will end up with a **lot** of information that is likely just garbage that you'll have to go back and delete. Once you run the operation,
let it run - it will likely take many hours to complete, and this is fine given the value of the images you're attempting to recover. As it
progresses, you should start to see folders created in the directory where you specified recovery with a naming convention following
`recup_dir.<INCR>/`, where `<INCR>` is an incremental number over time.

### Simple Filtering - Garbage and Lowres Photos

In this particular case, there were a lot of garbage photos that weren't needed/valuable because the file format we selected wasn't useful. To
start, we moved these files out of the way because duplicate detection using an MD5 algorithm can be expensive, and when you're dealing with
many photos (we had 250,000+) it can be helpful to reduce the total number. To do this, first create a list of directories for the files you are
most certain will not be valuable (as a note, you could also simply filter these out as part of the Photorec software configuration so they aren't
even attempted to be recovered):

```bash
$ mkdir -p garbage/gif
$ mkdir garbage/bmp
$ mkdir garbage/tif
$ mkdir garbage/png
```

Next, go ahead and move these files out of the way - as a note, we're storing these in a temporary directory structure so we can go through them
quickly just to be absolutely certain we don't need any of these:

```bash
# move bmp
$ find recup_dir* -type f -name “*.bmp”-exec mv {} garbage/bmp/ \;

# move tif
$ find recup_dir* -type f -name “*.tif”-exec mv {} garbage/tif/ \;

# move gif
$ find recup_dir* -type f -name “*.gif”-exec mv {} garbage/gif/ \;

# move png
$ find recup_dir* -type f -name “*.png”-exec mv {} garbage/png/ \;
```

Now that the garbage types are out of the way, let's do filtering by size. Here, we'll make an assumption that anything under 500KB is likely
a low resolution photo, possibly a thumbnail or icon that was recovered, and that we don't need these either. Create a directory to store these
and move these photos into the temporary directory:

```bash
# make the directory
$ mkdir low-res

# move low-res photos into the temp directory
$ find recup_dir* -type f -size -500k -exec mv {} lowres/ \;
```

### MD5 Duplicate Detection

We're now ready to do a crude duplicate detection and removal. This method will generate an MD5 of the image contents and store it in memory.
If a future image is detected with the same exact MD5 hash, it is assumed to be a duplicate (very safe) and will be removed. Create a script
named `remove_dupes_md5.py` and insert the following Python contents:

```python
#!/usr/bin/env python

import hashlib
import glob, os
import shutil

shas = []

fqdn_dupes = "/dupes"

for filename in glob.iglob('./recup_dir*/*'):
  with open(filename, 'rb') as file_read:
    base_file = os.path.basename(filename)
    md5 = hashlib.md5(file_read.read()).hexdigest()

    if md5 in shas:
      print("DUPLICATE: {}: {}".format(filename, md5))
      shutil.move(filename, "{}/{}-{}".format(fqdn_dupes, md5, base_file))
    else:
      print(“ORIGINAL: {}: {}".format(filename, md5)
      shas.append(md5)          
```

We'll need to create the directory where the duplicates will be stored (again, just being overly cautious):

```bash
$ mkdir dupes
```

Next, run the script so that it can filter through and move duplicates into the newly-created directory:

```bash
$ python3 remove_dupes_md5.py
```

This process will likely take some time given that it's needing to MD5 hash the contents of each image. This can be a timely operation
depending on how large your images are (for raw images, this takes a decent amount of time).

### Sorting Resulting Images

Now that we've removed exact duplicates, let's go ahead and sort the image types into a `GOOD` directory for easier exploration (by
file type) - obviously adapt to the file types you've selected:

```bash
# make directories for each type
$ mkdir GOOD
$ mkdir GOOD/jpg
$ mkdir GOOD/raw
$ mkdir GOOD/movies
$ mkdir GOOD/audio
$ mkdir GOOD/heic
$ mkdir GOOD/psd

# move the file types into their respective directories
$ find recup_dir* -type f -name “*.jpg” -exec mv {} GOOD/jpg/ \;
$ find recup_dir* -type f -name “*.cr2” -exec mv {} GOOD/raw/ \;
$ find recup_dir* -type f -name “*.heic” -exec mv {} GOOD/heic/ \;
$ find recup_dir* -type f -name “*.mov” -o -name “*.mp4" -exec mv {} GOOD/movies/ \;
$ find recup_dir* -type f -name “*.m4p" -exec mv {} GOOD/audio/ \;
$ find recup_dir* -type f -name “*.psd” -exec mv {} GOOD/psd/ \;
```

### Similar Images

Finally, there is one last comparison that we can do on "like" images. Often, when performing a recovery, you will get multiple image files that
are the same image but drastically different file size/resolution. These are, in essence, the same image. If you want to remove these duplicates
programmatically and only keep the resulting largest image size (assuming it's the best resolution), the following script can be run on the
JPG image types to perform this operation, where it will recurse all of the filtered files and delete all but the largest file it finds.

This comparison is not perfect - where file sizes are drastically different (significantly different resolutions), this script will not accurately
detect that the image is the same. However, this script, in the resulting set of 70,000 images after the above filtering, was able to remove
an additional ~20,000 duplicate lower resolution files. Create a Python file `remove_dupes_size.py` with the following contents (adapted from the
referenced "Credit" heading below):

```python
import glob
import imagehash
import shutil
import os
from PIL import Image

DATADIR = "GOOD/jpg"
DUPEDIR = "dupes_round_2/"

file_meta = []
cur_file = 0

if __name__ == '__main__':
  for image_path in glob.glob(DATADIR + "/*.jpg"):
    # help keep track
    cur_file += 1

    # get some image properties
    image = Image.open(image_path)
    file_hash = str(imagehash.dhash(image))

    # get some params
    file_name = image_path[image_path.rfind("/") + 1:]
    file_size = os.stat(image_path).st_size

    # see if we already passed an item with the same hash
    found_item = next((x for x in file_meta if x['hash'] == file_hash), None)
    if found_item:
      print("{} | Found DUP [{}] - analyzing size...".format(cur_file, file_hash))

      # only start messing with things if the current file
      # is larger (assuming this means better resolution)
      if file_size > found_item['file_size']:
        # move and replace the existing file because it's lower resolution
        shutil.move(image_path, "{}{}".format(DUPEDIR, found_item['file_name']))
        file_meta[:] = [x for x in file_meta if x['hash'] != file_hash]
        file_meta.append({
          "hash": file_hash,
          "file_name": file_name,
          "file_size": file_size
        })
      else:
        # move the current file because it's lower resolution
        shutil.move(image_path, "{}{}".format(DUPEDIR, file_name))
    else:
      # technically a brand new image - store it
      print("{} | New image: {} | {} | {}".format(cur_file, file_hash, file_name, file_size))

      file_meta.append({
        "hash": file_hash,
        "file_name": file_name,
        "file_size": file_size
      })
```

Run the above file:

```bash
$ python3 remove_dupes_size.py
```

Again, this script will take some time to run as it's doing an image compare using image libraries in Python.

### Completion

At the end of all of this, you should have a `GOOD` directory with resulting image file types that are (mostly) filtered for duplicates
and lower resolution images. In addition, you'll have all of the duplicates and lower resolution images in various sub-directories created
so you can inspect if you're suspicious that you may have removed something you needed.

There are likely more complex/sophisticated ways to handle this type of activity, but the above worked well for recovering approximately
~250,000+ images as old as 8 years back, and reducing them to ~50,000 quality unique images that could be filtered down further.

### Credit

The above tutorial was pieced together with some information from the following sites/resources, among others that were likely missed in this list:

* [Finterprinting Images for Near-Duplicate Detection](https://realpython.com/fingerprinting-images-for-near-duplicate-detection/)
* [Image Fingerprinting](https://github.com/realpython/image-fingerprinting)
