# FlatMap - a binary file format for OpenStreetMap data

<br/>
<br/>

![OpenStreeMap Logo](/images/osm-logo.png)

<br/>
<br/>

## Purpose

### Efficient data access
is the primary goal of this data format. Operations that shall be fast are
- getnode (id)
- getway (id)
- getrelation (id)
- iterate all nodes
- iterate all ways
- iterate all relations
- getnodes (wayid)

getnode/way/relation are log(n)<br/>
iterators locate the first item in log(n) and are then linear to the number of iterated elements<br/>
getnodes (wayid) accesses the way in log(n), iterating the nodes is linear to the number of nodes<br/>

### Compactness
supports efficiency, because smaller data often leads to faster data access. Begin compact makes flatmap a usful format for the transfer of osm datasets, making it an alternative to the prevailing pbf file format. [See also](https://github.com/snuup/flatmap/wiki/Compactness)



## File Format

A flat-map-file holds these elements:

| name                      | mandatory  |
|:-|:-|
| file-header               | yes        |
| nodes-blocks              | no         |
| ways-blocks               | no         |
| relations-blocks          | no         |
| nodes-blocktable          | no         |
| ways-blocktable           | no         |
| relations-blocktable      | no         |
| string-table              | no         |

The dependencies considering the links inside these elements and their number of occuriencies are:

```
file-header (1)
-> nodes-blocktable (0|1)
    -> nodes-blocks (0..*)
-> ways-blocktable (0|1)
    -> ways-blocks (0..*)
-> relations-blocktable (0|1)
    -> relations-blocks (0..*)
-> string-table (0|1)
```

There is no neccessity to store all nodes/ways/relations-blocks as a sequence, the only requirement is that all links are valid. While flatmap is designed as a simple readonly file format, its utility as an updateable file format is also envisaged and this flexibility of allocation becomes relevant than.


## Sorting

The blocktables and the blocks are considered a BTree data structure, a B+Tree where all data is in the leaves. The blocks are the leafs, the 3 blocktables the 3 roots. Accordingly, BlockTableEntries must be ordered by their first-id values and the elements inside the blocks must be ordered as well.


## File-Header

| name                     | size | type | comment       |
|:-------------------------|------|--------------------|-|
| magic file identifier    | 4    | uint32 | 0xF1AD8ABB  | "flat map"
| file-format-version      | 4    | uint32 | 0x00000001  |
| nodes-blocks-count       | 8    | uint64 | number of nodes blocks
| nodes-blocks-start       | 8    | uint64 | link to first DataBlockEntry
| ways-blocks-count        | 8    | uint64 | number of ways blocks
| ways-blocks-start        | 8    | uint64 | link to first DataBlockEntry
| relations-blocks-count   | 8    | uint64 | number of relations blocks
| relations-blocks-start   | 8    | uint64 | link to first DataBlockEntry
| string-table-header      | 32   |        | as defined below

The mandatory file-header points to all other elements. **Links** are expressed as absolute file positions, a value of 0 means that no such element exists.

## DataBlockEntry
| name | size | type | comment
|:-|:-|:-|:-|
| first-id | 8 | uint64 | first node/way/relation id
| data-block | 8 | uint64 | link to node/way/relation block

## Stringtable-Header

| name | size | type | comment |
|-|-|-|-|
| count          | 8 | uint64 | number of strings
| strings        | 8 | uint64 | link to stream of Strings
| alpha-sorted   | 8 | uint64 | link to aplhanumerically sorted StringIndexEntry[]
| sid-sorted     | 8 | uint64 | link to numerically sorted StringIndexEntry[]

## String

| name | type | comment |
|:-|:-|:-|
| length | int32 | varint encoded number of utf8 bytes
| text | utf8[] |

**maximum value needed**: Currently, the largest strings have a length of 765 utf8 bytes, one of them is the description here: https://www.openstreetmap.org/relation/12372925 .

**encoding**: 99,67% of the strings have a length <= 127 such that varint encoding uses 1 byte for the length field.


## StringIndexEntry
| name | type |
|:-|:-|
| sid | int32  | string-id
| String | uint64 | link to string

This structure is used by both string indexes: by string-id and by string (alphanumeric).

## Nodes-Block

|name | size | type | comment |
|:-|:-|:-|:-|
|**header**
| nodecount-minus-1 | 1 | byte | 255 means a count of 256 |
| id-size | 1 | byte | datatype of the local-ids array: 1 &vert; 2 &vert; 4 &vert; 8 |
| tag-size | 1  | byte | datatype of the tagsizes array: 1 &vert; 2 &vert; 4 &vert; 8 |
|**arrays**
| local-ids | | int[] | 1, 2, 4 or 8 byte sized integer
| lon-lat |  | LonLat[] | lon-lat of the node
| tag-sizes | | int[] | 1, 2, 4 or 8 byte sized integer
|**streams**
| tag-stream | | | stream of varint encoded key-value sequence for all nodes

Adaptive data structures are applied at the block level: In the node-block this applies to local-ids and tag-sizes: Both are stored in arrays whose integer type is minimized such that it can hold the largest value.

**local-ids** are the offset of the node-id from the first node in the block. the value for the first node is not stored in the block but in the data-block-entry for the block.

**tag-sizes** is an array that holds the length of the tag-stream for each node. to find the start of the tag-stream for the i-th node, the sum of the
tag-stream-lengths of the preceeding nodes has to be computed.

## LonLat
| name | size | type | comment |
|:-|:-|:-|:-|
| lon | 4 | int | longitude in 100 nanodegrees
| lat | 4 | int | latitude in 100 nanodegrees


## Ways-Block

| name | size | type | comment |
|:-|:-|:-|:-|
|**header**
| waycount-minus-1 | 1 | byte | 255 means a count of 256 |
| id-size | 1 | byte | datatype of the local-ids array: 1 &vert; 2 &vert; 4 &vert; 8 |
| tag-size | 1  | byte | datatype of the tagsizes array: 1 &vert; 2 &vert; 4 &vert; 8 |
| nodes-size | 1  | byte | datatype of the nodesizes array: 1 &vert; 2 &vert; 4 &vert; 8 |
| tagstream-size | 4 | uint32 | size of the whole tag-stream, so can jump to nodestream
|**arrays**
| local-ids | | int[] | 1, 2, 4 or 8 byte sized integer
| tag-sizes | | int[] | 1, 2, 4 or 8 byte sized integer
| nodes-sizes | | int[] | 1, 2, 4 or 8 byte sized integer
|**streams**
| tag-stream | | | stream of varint encoded key-value sequence for all nodes
| node-stream | | | stream of nodes, see below

The node-stream stores the way-nodes of all ways. This includes the node-ids as well as the lon/lats of those nodes. This implements the locations-on-ways design in the variant where node-ids are kept. See also https://github.com/osmlab/osm-data-model for a discussion of this design.

Tagless nodes that are embedded in ways that way do not need to be stored as nodes by themselves, although the file format does not mandate that.

The first node and the follower nodes are encoded as follows:

## First-Node

| name | size | type | comment |
|:-|:-|:-|:-|
| id | 5 | uint64 | only lower 5 bytes are stored
| lon | 4 | int32 | in 100 nanodegrees
| lat | 4 | int32 | in 100 nanodegrees


There are currently around 842 mio ways in the database (https://taginfo.openstreetmap.org/reports/database_statistics). The size of the first node affects the file size by several percent of the file size. Encoding it as varint with 7bit encoding currently leads to an average of 4.98byte per id which is around 40bit. This is larger than
the size needed to for the maximum node-id, which is 34bit. In the long run we expect this id to be uniformly distributed across the node-id domain. Making the id type a
5 byte sized integer value should be safe for many years and avoids surprises of varint encoding. Further research could measure skewness, skewness
as it evolves over time and investigate other variable integer encodings, like 15bit or 31bit codings.

## Follower-Node
| name | type | comment |
|:-----|:-----|:--------|
| id   | uint64 | varint delta coded node-id
| lon  | int32 | varint zigzag delta coded lon
| lat  | int32 | varint zigzag delta coded lat



## Relations-Block

| name | size | type | comment |
|:-|:-|:-|:-|
|**header**
| relationcount-minus-1 | 1 | byte | 255 means a count of 256 |
| id-size | 1 | byte | datatype of the local-ids array: 1 &vert; 2 &vert; 4 &vert; 8 |
| tag-size | 1  | byte | datatype of the tagsizes array: 1 &vert; 2 &vert; 4 &vert; 8 |
| members-size | 1  | byte | datatype of the membersizes array: 1 &vert; 2 &vert; 4 &vert; 8 |
| tagstream-size | 4 | uint32 | size of the whole tag-stream, so can jump to nodestream
|**arrays**
| local-ids | | int[] | 1, 2, 4 or 8 byte sized integer
| tag-sizes | | int[] | 1, 2, 4 or 8 byte sized integer
| members-sizes | | int[] | 1, 2, 4 or 8 byte sized integer
|**streams**
| tag-stream | | Tag[] | stream of varint encoded key-value sequence for all nodes
| members-stream | | Member[] | stream of members, see below


## Member

| name | type | comment |
|:-|:-|:-|
| member-id | uint64 varint | node/way/rel id
| role | uint32 varint | string-id
| member-type | MemberType | see below

## MemberType
1 = Node
2 = Way
3 = Relation


## Encodings

### varint encoding
means 7-bit encoding

### zigzag encoding
means 7-bit zigzag encoding
