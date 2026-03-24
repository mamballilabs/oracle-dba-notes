Its always the instance that goes through the cycle never the database.
The database has no states, it just exists on disk.
The instance moves through states - Nomount - Mount - Open
The database files are passive. They sit on disk whether the instance is running or not. It is always the instance that transitions, that reads files, that gains awareness, that opens access.
```
You start an instance, The instance then finds, mounts, and opens the database.
```

Instance = Memory(SGA) + Processes - it lives in RAM, it's alive or dead 
Database = Files on disk (datafiles, redo logs, control files) - they always exist on disk

```
Becuase they are two separate things, they can be in different states independently
```

The complete cycle - Instance always

```
SHUTDOWN  →  NOMOUNT  →  MOUNT  →  OPEN
```

Shutdown - state where nothing exists in memory.
no SGA, no Background processes
