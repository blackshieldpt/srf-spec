# Simple Record File

SRF is a binary container file format for arbitrary data; Each item of data is stored as a **Record** - a bundle of the
data itself, optional metadata and a simple header. A SRF file contains a sequence of Records, and the number of
Records per file is only limited by the underlying filesystem.

Typically, Records are appended to the file, and read operations are sequential. There is no index or table of contents
to access specific records. A SRF file has no file header - instead, it is just a sequence of [Records](#record).

The primary use case is ad-hoc import and export of data between systems, e.g. capturing arbitrary messages from a
messaging
system and re-queuing them on a local system for development purposes; it can, however, be used to store any kind of
application-defined data resource that is typically loaded sequentially.

> Note on on-disk byte order: All values are stored in little-endian format.

**What SRF is**
- a simple - but flexible - container format to exchange collections of data records in a single file;
- a cross-language, lowest denominator file-based data interchange format;
- a file format to be both written and consumed sequentially;

**What SRF isn't**
- a long-term storage/backup format; 
- a database format;
- a file compression format;
- a file storage system;
- an indexed data file;

## Record

A **Record** can either have a base record type or an application-defined record type, and can optionally be stored in
compressed form. A Record can have upto (2^64)-1 bytes of size; However, keep in mind both read and write operations
may require a memory buffer **at least** the size of the record; buffering of operations is implementation-specific,
and the file format does not enforce it.

Each record can optionally have application-defined metadata. The metadata must be encoded in JSON format, and it is
always stored in compressed form using Zstandard with crc suffix.

> Note: the record type is merely informative; no validation is performed on the specifics of the data type. Client
> applications must implement type validation, if required.

### Record structure

A Record is composed of a header

| Scope    | Offset          | Size in bytes | Description                                             |
|----------|-----------------|---------------|---------------------------------------------------------|
| header   | 0               | 4             | Magic signature: 'SRF0'                                 |
| header   | 4               | 4             | [Record flags and type](#record-flags-and-type)         |
| header   | 8               | 4             | [Metadata size](#metadata-size), can be zero, see below |
| header   | 12              | 8             | [Record size](#record-size)                             |
| metadata | 20              | 0...n         | Compressed metadata, can be empty if Metadata size is 0 |
| data     | 20+metadata...n | 0...n         | Record data                                             |

### Record Flags and Type

Record flags and type is a 32 bit value, typically corresponding to an unsigned integer. The bit layout is as follows:

| bit 31           | bits 30-16         | bits 15-0   |
|------------------|--------------------|-------------|
| Compression Flag | Reserved;must be 0 | Record Type |

**Compression Flag**

The most-significant bit is the compression flag; if the value is 1, the record data is compressed using Zstandard with
crc suffix.

**Reserved bits**

Reserved bits are reserved for future usage, and must have the value 0. Record readers **must** validate if these bits
are actually 0, and fail if they are not.

**Record Type**

Record type is a 16 bit unsigned value indicating the content type of the Record; the first 1024 values (least
significant 10 bit, from 0-2023) are reserved for system types; the remainder values (1024-65,535) can be used by client
applications.

| Value | Type   | Description                   |
|-------|--------|-------------------------------|
| 0     | -      | Invalid type                  |
| 1     | binary | Record is a sequence of bytes |
| 2     | text   | Record is a utf8 string       |
| 3     | JSON   | Record is a JSON string       |

### Metadata Size

Unsigned 32 bit value indicating metadata size. The value reflects the size of the compressed stream on disk, not the
uncompressed metadata. Metadata can have a compressed size upto 4GB. If Record has no metadata, this value must be 0.

### Record Size

Unsigned 64 bit value indicating the size of the Record data; this value is the size of the actual bytes on disk.

### Metadata

A Record can optionally contain metadata; The metadata is **always** a JSON string, and is stored in compressed format using
ZStandard with crc suffix. Metadata contents are client-defined, and can be either parsed or ignored by applications.

### Record data

The actual Record data; it can be stored as-is or using ZStandard compression with crc suffix, depending on the value of
the *Compression Flag* in [Record Flags and Type](#record-flags-and-type). 

