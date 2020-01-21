# UnifyFS Client API

This document describes the background and motivation for the development of a new client application programming interface (API) for [_UnifyFS_][1], including key design choices that were made. The current API, along with brief documentation on intended
usage, is also presented. Finally, potential useful additions to the API are proposed.

## Background

_UnifyFS_ was originally designed to enable direct integration within user applications based on POSIX I/O or MPI-IO. Interposition on the application's existing I/O operations was chosen as the most seamless method, as it requires the least amount of changes to application source code. The only source modifications necessary were to insert calls to `unifyfs_mount()` and `unifyfs_unmount()` at the beginning and end (respectively) of your application. With these two simple additions, applications could just prepend `/unifyfs` to their existing file paths to make use of the unified storage space.

## Motivation

Although the original design accomplished its goal of easy application integration, the resulting client library code was not developed in a modular fashion that exposed clean internal APIs for core functionality such as file I/O. The lack of modularity has limited the ability of _UnifyFS_ to explore additional integrations that would increase its usability.

For example, existing commonly used distributed I/O storage libraries such as HDF5 and ADIOS have modular designs that permit implementation of new storage backend technologies, but _UnifyFS_ provided no API that could be leveraged to support such developments.

Further, users had no way of exploring the _UnifyFS_ namespace from outside of their applications, since common system tools (e.g., Unix shells and file system commands) could not be used without explicitly modifying their source code. The _UnifyFS_ team has explored various options to provide system tools (e.g., FUSE and custom command-line tools), but initial development on these approaches stalled due to the lack of appropriate APIs.

Finally, using interposition as the only means of affecting application I/O behavior meant that important _UnifyFS_ semantics, particularly [file lamination][2], required hijacking existing system calls like `fsync()` and `chmod()` to impart new meaning to their use in applications. For many applications, these system calls were not already used, and thus had to be added to the source code to effect the desired behavior.

Together, these limitations motivated our desire to define a new client API that provided the required interfaces for these use cases, as well as potential future uses.

## Design Choices

The primary goal in providing a new client API is to expose all core functionality of _UnifyFS_. This functionality includes methods for file access, file I/O, file lamination, and file transfers. Additionally, _UnifyFS_ provides some user control over its behavior. Some of this behavior is already exposed via existing configuration settings that can be controlled by clients using environment variables. Where appropriate, it is useful to provide explicit control mechanisms in the API as well.

Although POSIX I/O and MPI-IO are the most common use cases for _UnifyFS_, it is important that the client API does not contain semantics specific to those storage strategies, as that could impose unnecessary limitations for other use cases. In particular, this means that implicit state such as that found in POSIX file descriptors (e.g., file position) or MPI-IO (e.g., collective file handles and communicators) should be avoided.

Finally, _UnifyFS_ utilizes a multi-threaded runtime system that operates asynchronously from client actions. Exposing this asynchrony at the API level is beneficial for applications that can overlap I/O with computation. For this reason, we have chosen to use asynchronous requests as the basis for both file I/O and transfers.

## Current API

### API Initialization

All interactions with the API require a file system handle that is obtained by calling `unifyfs_initialize()`. The handle type `unifyfs_handle` is a pointer to an opaque struct for each client. Upon successful return from `unifyfs_initialize()`, the `fshdl` handle parameter is set.

```C
/* UnifyFS file system handle (opaque struct pointer) */
typedef struct unifyfs_client_handle* unifyfs_handle;

/* Initialize client's use of UnifyFS */
unifyfs_rc unifyfs_initialize(const unifyfs_options* opts,
                              unifyfs_handle* fshdl);
```

The `unifyfs_initialize()` method takes an optional parameter of type `struct unifyfs_options` (shown below) that can be used to provide user-specified configuration and behavior choices for _UnifyFS_. The current options include:
<ul>
 <li>the desired UnifyFS namespace prefix
 <li>the path on a shared file system to use for persisting files
 <li>whether to automatically laminate files that have been written when they are closed
 <li>whether to automatically transfer laminated files when finalizing UnifyFS
 <li>an application provided rank value used in debugging messages
</ul>

```C
/* UnifyFS client options */
typedef struct unifyfs_options {
    /* namespace prefix (default: '/unifyfs') */
    char* fs_prefix;

    /* file system persist path (default: NULL) */
    char* persist_path;

    /* auto-laminate files opened for writing at close? (default: 0) */
    int laminate_at_close;

    /* auto-transfer laminated files to persist path upon fini (default: 0) */
    int transfer_at_finalize;

    /* application debug rank (default: 0) */
    int debug_rank;
} unifyfs_options;
```

To support multiple concurrently active namespaces, a client can manage multiple file system handles. The client shoud call `unifyfs_initialize()` for each namespace, using the `opts` parameter to specify the desired namespace.

### File Access

All file accesses in _UnifyFS_ require passing the client handle obtained from `unifyfs_initialize()`. Only files that exist in the namespace associated with the handle (i.e., residing in the filesystem subtree rooted at `fs_prefix`) can be accessed. Files are added to the namespace in one of two ways, by explicit creation using `unifyfs_create()`, or by transferring existing files into _UnifyFS_ (see "File Transfers" section below). Once a file has been created by one client, other clients may access the file using `unifyfs_open()`. Both `unifyfs_create()` and `unifyfs_open()` initialize a global file identifier of type `unifyfs_gfid`, which is a globally consistent integer generated by hashing the full file path. The `unifyfs_close()` method invalidates the calling client's access to the file corresponding to the given `unifyfs_gfid`. Finally, the `unifyfs_remove()` method provides a means to remove files from the namespace.

```C
/* global file id type */
typedef int unifyfs_gfid;

/* Create and open a new file in UnifyFS */
unifyfs_rc unifyfs_create(unifyfs_handle fshdl,
                          const int flags,
                          const char* filepath,
                          unifyfs_gfid* gfid);

/* Open an existing file in UnifyFS */
unifyfs_rc unifyfs_open(unifyfs_handle fshdl,
                        const int flags,
                        const char* filepath,
                        unifyfs_gfid* gfid);

/* Close an open file in UnifyFS */
unifyfs_rc unifyfs_close(unifyfs_handle fshdl,
                         const unifyfs_gfid gfid);

/* Remove an existing file from UnifyFS */
unifyfs_rc unifyfs_remove(unifyfs_handle fshdl,
                          const char* filepath);
```

### File Status

The API provides methods for querying the status of a _UnifyFS_ file. The `unifyfs_stat()` method collects all available information and stores it within the provided `struct unifyfs_status` parameter. Methods also exist to query the individual pieces of information, such as `unifyfs_stat_laminated()` that provides the file's lamination status.

```C
/* File status struct */
typedef struct unifyfs_status {
    int laminated;
    off_t local_file_size;
    off_t global_file_size;
    size_t local_write_nbytes;
} unifyfs_status;

/* Get all available file status */
unifyfs_rc unifyfs_stat(unifyfs_handle fshdl,
                        const unifyfs_gfid gfid,
                        unifyfs_status* status);

/* Get file lamination status */
unifyfs_rc unifyfs_stat_laminated(unifyfs_handle fshdl,
                                  const unifyfs_gfid gfid,
                                  int* is_laminated);
```

The `unifyfs_stat_global_size()` method obtains the current maximum valid offset across all servers. If the file has been laminated, this is the final file size.

The `unifyfs_stat_local_size()` method obtains the maximum valid offset known by the client's local server. If the file has been laminated, this is the final file size.

```C
/* Get current global size for file */
unifyfs_rc unifyfs_stat_global_size(unifyfs_handle fshdl,
                                    const unifyfs_gfid gfid,
                                    off_t* global_sz);

/* Get current local size for file */
unifyfs_rc unifyfs_stat_local_size(unifyfs_handle fshdl,
                                   const unifyfs_gfid gfid,
                                   off_t* local_sz);
```

A client can also query the number of bytes it has written using the `unifyfs_stat_local_write()` method. This is the sum of the lengths of all extents written since opening the file.

```C
/* Get current local bytes written for file */
unifyfs_rc unifyfs_stat_local_write(unifyfs_handle fshdl,
                                    const unifyfs_gfid gfid,
                                    size_t* local_write_bytes);

```

### File I/O

TODO

```C
/* enumeration of supported I/O request operations */
typedef enum unifyfs_ioreq_op {
    UNIFYFS_IOREQ_NOP = 0,
    UNIFYFS_IOREQ_OP_READ,
    UNIFYFS_IOREQ_OP_WRITE,
    UNIFYFS_IOREQ_OP_TRUNC,
    UNIFYFS_IOREQ_OP_ZERO,
} unifyfs_ioreq_op;

/* enumeration of I/O request states */
typedef enum unifyfs_ioreq_state {
    UNIFYFS_IOREQ_STATE_INVALID = 0,
    UNIFYFS_IOREQ_STATE_IN_PROGRESS,
    UNIFYFS_IOREQ_STATE_CANCELED,
    UNIFYFS_IOREQ_STATE_COMPLETED
} unifyfs_ioreq_state;

/* structure to hold I/O request result values */
typedef struct unifyfs_ioreq_result {
    int error;
    int rc;
    size_t count;
} unifyfs_ioreq_result;

/* I/O request structure */
typedef struct unifyfs_io_request {
    /* user-specified fields */
    void* user_buf;
    size_t nbytes;
    off_t offset;
    unifyfs_gfid gfid;
    unifyfs_ioreq_op op;

    /* async callbacks (not yet supported)
     *
     * unifyfs_req_notify_fn fn;
     * void* notify_user_data;
     */

    /* status/result fields */
    unifyfs_ioreq_state state;
    unifyfs_ioreq_result result;

    /* internal fields */
    int _reqid;
} unifyfs_io_request;

/* Dispatch an array of I/O requests */
unifyfs_rc unifyfs_dispatch_io(unifyfs_handle fshdl,
                               const size_t nreqs,
                               unifyfs_io_request* reqs);

/* Cancel an array of I/O requests */
unifyfs_rc unifyfs_cancel_io(unifyfs_handle fshdl,
                             const size_t nreqs,
                             unifyfs_io_request* reqs);

/* Wait for an array of I/O requests to be completed/canceled */
unifyfs_rc unifyfs_wait_io(unifyfs_handle fshdl,
                           const size_t nreqs,
                           unifyfs_io_request* reqs,
                           const int waitall);
```

### File Lamination

File lamination designates that no further writes are allowed. The API provides two methods related to lamination.

The first method, `unifyfs_laminate()`, indicates that the file at the given path is to be laminated globally. This method should be used by a single client once all clients have completed their writes. After this method returns successfully, no clients will be permitted to write to the file, and the file status will reflect its laminated state.

The second method, `unifyfs_laminate_local()`, informs the local server that the calling client will no longer write to the file. At this point, the server is free to propagate information about the client's writes to other servers. Note that calling this method is entirely optional, and a client's writes are not guaranteed to be visible to other clients until the file is globally laminated.

```C
/* Global lamination - no further writes to file are permitted */
unifyfs_rc unifyfs_laminate(unifyfs_handle fshdl,
                            const char* filepath);

/* Local lamination - all client writes have been completed */
unifyfs_rc unifyfs_laminate_local(unifyfs_handle fshdl,
                                  const unifyfs_gfid gfid);
```

### File Transfers

TODO

```C
/* enumeration of supported I/O request operations */
typedef enum unifyfs_transfer_mode {
    UNIFYFS_TRANSFER_MODE_INVALID = 0,
    UNIFYFS_TRANSFER_MODE_COPY, // simple copy to destination
    UNIFYFS_TRANSFER_MODE_MOVE  // copy, then remove source
} unifyfs_transfer_mode;

/* File transfer request structure */
typedef struct unifyfs_transfer_request {
    /* user-specified fields */
    const char* src_path;
    const char* dst_path;
    unifyfs_transfer_mode mode;

    /* async callbacks (not yet supported)
     *
     * unifyfs_req_notify_fn fn;
     * void* notify_user_data;
     */

    /* status/result fields */
    unifyfs_ioreq_state state;
    unifyfs_ioreq_result result;

    /* internal fields */
    int _reqid;
} unifyfs_transfer_request;

/* Dispatch an array of transfer requests */
unifyfs_rc unifyfs_dispatch_transfer(unifyfs_handle fshdl,
                                     const size_t nreqs,
                                     unifyfs_transfer_request* reqs);

/* Cancel an array of transfer requests */
unifyfs_rc unifyfs_cancel_transfer(unifyfs_handle fshdl,
                                   const size_t nreqs,
                                   unifyfs_transfer_request* reqs);

/* Wait for an array of transfer requests to be completed/canceled */
unifyfs_rc unifyfs_wait_transfer(unifyfs_handle fshdl,
                                 const size_t nreqs,
                                 unifyfs_transfer_request* reqs,
                                 const int waitall);
```

### API Finalization

When a client has completed all interactions with _UnifyFS_ files, they should call `unifyfs_finalize()` to release resources associated with their client handle.

```C
/* Finalize client's use of UnifyFS */
unifyfs_rc unifyfs_finalize(unifyfs_handle fshdl);
```

## Future Functionality

In this section, we describe potential additions to the client API. This functionality may be included in future versions of the API based upon feedback on user requirements.

### Directory Operations

TODO

### Memory Mapped Files

Mapping file data into memory (e.g., using `mmap()`) is a popular approach for maximizing read and write performance. _UnifyFS_ could support this behavior for non-shared files fairly easily. For shared files, it is likely that only non-overlapping mappings could be supported. The proposed APIs for memory mapping _UnifyFS_ files are shown below.

```C
/* Map global file contents into memory starting at given offset */
unifyfs_rc unifyfs_map(unifyfs_handle fshdl,
                       const unifyfs_gfid gfid,
                       const off_t offset,
                       const size_t length,
                       void** addr);

/* Remove memory mapping for global file */
unifyfs_rc unifyfs_unmap(unifyfs_handle fshdl,
                         const unifyfs_gfid gfid,
                         const off_t offset,
                         const void* addr);

/* Sync contents of memory mapping for global file */
unifyfs_rc unifyfs_map_sync(unifyfs_handle fshdl,
                            const unifyfs_gfid gfid,
                            const off_t offset,
                            const void* addr);
```

[1]: https://unifyfs.readthedocs.io/en/v0.9.0/overview.html
[2]: https://unifyfs.readthedocs.io/en/v0.9.0/assumptions.html#consistency-model
