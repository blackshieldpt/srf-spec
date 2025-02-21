# Simple Record File

SRF is a binary container file format for arbitrary data; Each item of data is stored as a **Record** - a bundle of the data itself, optional metadata and a simple header. A SRF file contains a sequence of Records, and the number of Records per file is only limited by the underlying filesystem. 
Typically, Records are appended to the file, and read operations are sequential. There is no index or table of contents to access specific records.

The primary use case is the import and export of data between systems, e.g. capturing arbitrary messages from a messaging
system and exporting them to a local system for development purposes; it can, however, be used to store any kind of application-defined data resource that is typically loaded sequentially.


A **Record** can either have a base data type or an application-defined datatype, and can optionally be stored in compressed form. A Record can have upto (2^64)-1 bytes of size; However, keep in mind both read and write operations may require a memory buffer **at least** the
size of the record; buffering operations is implementation-specific, and the file format does not enforce it.

Each record can optionally have application-defined metadata. The metadata is encoded in JSON format, and it is stored in compressed form using ZStandard.


## Record Header

|Offset|Size|Description|
|---|---|---|
|0|4|Magic signature: 'SRF0'|
|4|4|Record Type, see below|
|8|4|Metadata size, can be zero, see below|
|12|8|Record size|
|20|0...n|Compressed metadata, can be empty if Metadata size is 0|
|20...n|Record data|

## Record Type

Record type is a 32 bit value, typically corresponding to an unsigned integer. The bit layout is as follows:

|31|30-16|15-0|
|---|---|---|
|Compression Flag|Reserved;must be 0|Record Type|


### Compression Flag

If compression flag is 1, record data is compressed using ZStandard; if 0, it is an uncompressedf stream of byes;

### Reserved bits

Reserved bits are for future usage and *must* be 0; 

### Record Type

Record type is a 16 bit unsigned value indicating the content type of the Record; the first 1023 values (least significant 10 bit, from 1-2023) are reserved for system types; the remainder values upto the 65,535 16 bit limit are application-defined.

|Value|Type|Description|
|---|---|---|
|1|binary|Record is a sequence of bytes|
|2|text|Record is a utf8 string|
|3|JSON|Record is a JSON string|
|16|zlib|Record is a Zlib compressed stream|
|17|zstd|Record is a ZStd compressed stream|


## Metadata Size

Unsigned 32 bit value indicating metadata size, if metadata is present. The value reflects the size of the compressed stream, not the uncompressed metadata. Metadata can have a compressed size upto 4GB. 
If no metadata present, this value must be 0.

## Record Size

Unsigned 64 bit value indicating the size of the Record data; this value is the size of the compressed stream on disk, if *Compression Flag* is active.

## Metadata

A Record can optionally contain metadata; The metadata is **always** a JSON string, stored in compressed format using ZStandard. Metadata contents are arbitrary, and can be either parsed or ignored by applications.

## Record data

The actual Record data; it can be stored as-is or using ZStandard compression, depending on the *Compression Flag* in Record Type. 

