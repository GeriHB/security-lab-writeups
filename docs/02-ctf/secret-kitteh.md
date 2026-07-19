# Secret Kitteh - TUCTF Forensics Challenge

## Overview

Secret Kitteh appeared to be an ordinary JPEN image of a cat.

![alt text](../../assets/cat.png)

The useful evidence was not in the visible image or its metadata. A hexadecimal inspection showed that the JPEG ended before the physical end of the file, revealing an appended, password-protected 7z archive.

The investigation followed this path:

```text
JPEG image
    ↓
Metadata provides no useful lead
    ↓
Bytes remain after the JPEG end marker
    ↓
Trailing data begins with a 7z signature
    ↓
Archive is carved from the image
    ↓
Archive password is recovered offline
    ↓
Challenge content is extracted
```

## Initial inspection

I first confirmed the file type and reviewed the available metadata.

```bash
file secret_kitteh.jpg
exiftool secret_kitteh.jpg
```

The file was recognised as a JPEG, but the metadata did not expose an obvious clue or embedded message.

This ruled out hte easiest explanation and shifted the investigation toward the file structure itself.

## Inspecting the JPEG structure

I opened the file in `GHex` and inspected its hexadecimal representation.

A JPEG normally begins with the start of image marker:

```text
FF D8
```

and ends with the end of image marker:

```text
FF D9
```

This was present, but it was not located at the end of the file. Additional bytes followed it.

Trailing bytes are not automatically malicious, btu in forensicc challenge they are a strong indication that another object may have been appended to the alid image.

## Identifying ht eappended data.

The trailing data began with:

```text
37 7A BC AF 27 1C
```

This is the signature associated with a 7z arhcive.

The file therefore consisted of two logical parts.

```text
Valid JPEG data
    +
Appended 7z archive
```

The image viewer displayed the JPEG normally because its stopped interpreting the file at the JPEG end marker, while the archive data remained available after it.

## Carving the archive

I originally used `GHex` to identify the start of the archive and extracted the trailing bytes

A repeatable command-line equivalent is:

```bash
dd if=secret_kitteh.jpg of=embedded.7z bs=1 skip=<7Z_START_OFFSET> status=progress
```

The carved output can then be checked with:

```bash
file embedded.7z
7z 1 embedded.7z
```

The archive was valid but password protected.

## Recovering the archvie password

To test the archive password offline, I first converted the 7z authentication data into a format accepted by John the Ripper:

```bash
7z2john embedded.7z > embedded.hash
```

I then tested canddate passwords from a wordlist:

```bash
john --wordlist=<wordlist> embedded.hash
```
The result was reviewed with:

```bash
john --show embedded.hash
```

The password was `password`, and it allowed the arhcive to be extracted, which contained hte challenge flag.

## Result

The challenge was solved by recognizing that the visible JPEG and the complete file were not the same thing.

The important forensic steps were:

1. Confirm the apparent file type
2. Check metadata without assuming the answer would be there
3. Inspect structural markers
4. Identify data beyond the logical end of the image
5. Recognize the embedded archive signature
6. Carve and validate the archive
7. Recover the weak archive password offline

## What I learned

The challenge reinforced the value of moving through forensic layers methodically.

Metadata inspection was useful even though it returned nothing. That negative reuslt narrowed the search. The decisive step was understanding the file format well enough to notice that valid image data had ended while the physical file continued.

It also showed why file signatures are useful evidence but should be validated. Recognizing the 7z signature produced a hypothesis; carving the bytes and successfully listing the archive confirmed it.

## References

- [7-Zip: 7z format](https://www.7-zip.org/7z.html)
- [7-Zip archive structure](https://www.7-zip.org/recover.html)
- [John the Ripper](https://www.openwall.com/john/)
- [John the Ripper command-line options](https://www.openwall.com/john/doc/OPTIONS.shtml)

---

[Back to repository overview](../../README.md)
