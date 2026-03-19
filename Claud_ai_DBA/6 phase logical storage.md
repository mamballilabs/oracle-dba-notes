```
TABLESPACE    →   the whole building
SEGMENT       →   one floor
EXTENT        →   one room on that floor
BLOCK         →   one desk in that room
```

Block:
smallest unit = 8kb
everything that oracle reads and writes moves in blocks
one block holds rows from exactly one table segment , oracle never mixes data from different objects fit in your buffer cache

Extent:
blocks are allocated in groups called extents 
```
Extent = a contiguous group of blocks
         all in the same datafile
         all belonging to the same segment
```
eg:
```
First extent allocated    →   8 blocks  (64 KB)
Table grows, needs more   →   8 blocks  (64 KB) more
Grows again               →  16 blocks (128 KB) more
```

segment:
every object that stores data gets its own segment 
```
One table        =   one table segment
One index        =   one index segment
Undo for ROOT    =   one undo segment
Temp sort space  =   one temporary segment
```
A segment is made up of one or more extents. As the object grows — more extents are added to its segment.
eg:
```
EMPLOYEES table segment
├── Extent 1   blocks 1-8     (first allocation)
├── Extent 2   blocks 9-16    (table grew)
├── Extent 3   blocks 33-48   (table grew again)
└── Extent 4   blocks 97-128  (table grew again)
```
Notice extents do not have to be contiguous — they just have to be in the same datafile or tablespace.

tablespace
this is the logical container that groups related segments together.
it maps directly to one or more datafiles
```
TABLESPACE          DATAFILE(S)
──────────────────  ──────────────────────────────────
SYSTEM              system01.dbf
SYSAUX              sysaux01.dbf
UNDOTBS1            undotbs01.dbf
USERS               users01.dbf
```
One tablespace can span multiple datafiles. One datafile belongs to exactly one tablespace.

```
USERS tablespace  (7 MB)
└── users01.dbf   (physical file on disk)
    └── Segments  (tables, indexes created in USERS)
        └── Extents (groups of blocks allocated per segment)
            └── Blocks (8 KB each, holds actual row data)
```
