# The Zinc Memory Allocator ZALLOC

The zmalloc is simple, and very structurated. It divides the memory as in teh following:

```
Address|Name
-------|-----
AB:0x00|AllocCountHigh
AB:0x01|AllocCountLow
AB:0x02|AllocHeadHigh
AB:0x03|AllocHeadLow
```

Then all allocation are just a simple metadata of:

```
Address|Name
-------|-----
RE:0x00|TaskID
RE:0x01|AllocStartHigh
RE:0x02|AllocStartLow
RE:0x03|AllocEndHigh
RE:0x04|AllocEndLow
```

Where the allocator creates new tasks allocation by adding one entry to AC (AllocCount) * sizeof(metadata) = AC*6.

When TaskId is 0x00, the allocation is taken for freed, When is 0xFF is taken as metadata ready to be erased as soon as possible.

## ZALLOC best fit

Zalloc uses a basic averaging best fit system that averages the size of all the free blocks, then takes the wanted size to be found, and teh average. and averages all freed regions within that space, and this keeps repiting until the system cannot find any better fit tahn the last one it found that fitted whitin the space given to it.
