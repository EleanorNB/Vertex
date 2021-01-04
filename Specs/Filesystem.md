Vertex uses a modular file attribute system with built-in types and verification. Properties are expressed with a series of small headers. A process has multiple views on the grand tree through **schemes**, or mount points; if a process requests a resource from URI `"scheme:/path/to/file"`, and `scheme:` is mounted at `/path/from/root` on the external CD drive, the kernel will request URI `"/path/from/root/path/to/file"` from the CD driver. Each storage driver permits a method of finding free space of a specified minimum and maximum contiguous size (making no guarantees of bias), and of determining whether a given range is entirely free or at least partially occupied; this information must have the same lifetime as the stored data, and does not distinguish between headers and file data. All file elements are `usize`-aligned.

Assumptions:
- The primary storage medium permits unambiguous access to any location with no more than a single `usize`

Interface:
- `obtain(Uri: [*:0]const u8, require_mask: u8, deny_mask: u8) {denied, wrong_width, unknown_type, file_driver, unsupported, overwrite}!Handle`
Requests a resource from `Uri`, which must have `require_mask` and must not have `deny_mask` permissions, and returns the ID of the resource. C, X, R, W permissions are not granted if not requested. Will fail if C, X, R, or W permissions cannot be granted (`denied`), if the driver's platform verification fails (`wrong_width`), if the process does not have an opener for the file type (`unknown_type`), if the file driver has an internal error (`file_driver`), if the storage class does not support opening (`unsupported`), or if the resource table is full (`overwrite`).
- `drop(r: Handle) void`
Removes a handle from a process' resource table. If the D permission is held, destroys the resource.
- `acquire(r: Handle, timeout: ?usize) {timeout}!void`
Waits at most `timeout` ticks to acquire the resource. Without this, every resource operation performs an implicit acquire-release.
- `release(r: Handle) void`
Release the resource, allowing other processes to acquire it.
- `readAtMost(r: Handle, ptr: ?usize, buf: ?[*]u8, len: usize, timeout: ?usize) {denied, timeout, end}!usize`
Reads however many bytes are available from `r`, starting at address `ptr` orelse current read head, into `buf` orelse discard, up to a maximum of `len`, and returns the number of bytes read. Will fail if required permissions are not held or resource is unavailable within `timeout` ticks. Requires the R permission. Provided `ptr` requires the B permission; no provided `ptr` requires the S permission.
- `readExact(r: Handle, ptr: ?usize, buf: ?[*]u8, len: usize, timeout: ?usize) {denied, timeout, end}!void`
Reads exactly `len` bytes, waiting up to `timeout` ticks if `len` bytes are not available. Does not advance read head on fail.
- `writeAtMost`
Analogous to `readAtMost`. Requires the W permission.
- `writeExact`
Analogous to `readExact`.
- `map(r: Handle, ptr: [*]u8) {denied, wrong_width, overwrite}!void`
Adds the resource to the process' virtual memory space at `ptr`, with applicable permissions. Will fail if `ptr` has insufficient alignment, or if it is already mapped. Requires the M permission.
- `unmap(r: Handle) void`
Removes the resource from the process' virtual memory space.

In addition, every one of the above syscalls, whether marked as erroring or not, will return -1 when given invalid input, for instance an unused or dangling handle or attempting to deny some function which was never granted.

Errors:
- `wrong_width`
- `unknown_type`
- `unsupported`
- `overwrite`
- `file_driver`
- `denied`
- `timeout`
- `end`

# Resources
URIs are opened to **resources**: kernel-mapped objects representing I/O spaces/streams. A process accesses a resource via a **handle** in its resource table. Handles have associated permissions: **C**lone, e**X**ecute, **R**ead, **W**rite, seek (**B**lock), **S**tream, **M**ap, and **D**estroy. These determine the operations allowed on them. Additionally, only one process may use a resource's R or W permissions at a time, and they must first **acquire** the resource from the handle to do so; if a resource is not acquired, any read/write operation on it also performs an implicit acquire-release. A resource with both B and S permissions includes a **read-write head**, which determines the starting position of reads and writes when none is provided.

# Types
Every file has an associated **type**. A process requesting a URI requires an **opener** for its type; attempting to open a file of an unsupported type results in an error (`unknown_type`), and the type in question.

# Capabilities
A process has associated **capabilities**: alphanumeric strings, which are required to obtain some permissions on resources. A process that requests a resource with a C, X, R, or W permission must have the appropriate capabilities for that permission; otherwise, the request will fail (`denied`) with a mask of which permissions could not be granted.

# Headers
Rather than a single monolithic header that encodes everything about the file, a Vertex file header has multiple layers, to facilitate separation and reinterpretation as well as wrapping of arbitrary data. The headers are as follows:

## Object Header
The **object header** lists basic information concerning how to read the file. Its layout is as follows:

### `usize` Width
Width of a `usize` in bits, as a `usize` in platform endianness order. This allows the storage driver to verify its assumptions of the file layout; if these are incorrect, the request will fail (`wrong_width`), prompting the requesting process to either give up or use a compatibility driver.

### Storage Class
Null-terminated byte sequence identifying the **storage class** of the file. This indicates to the driver how to interpret the header contents as a block; see below.

### Layout
The layout of the file data. Interpretation is subject to the storage class: it may be a simple length and inline block, it may be a list of inodes, it may be a path to another file (symlink), it could be a directory etc.

## Block Header
With the object header, the driver can reconstruct a view of the file as a contiguous block. This block is begun with a **block header**. An address within this header is given as a `usize`, and interpreted as a block offset. The contents of this header are as follows:

### File Start
The address of the first byte of the file payload. Must be `usize`-aligned.

### File Stop
The address of the last byte of the file payload.

### Type
The file's type (above), as a null-terminated byte string.

## Access Control Header
The **access control header** begins at the first aligned address after the file type, and runs until file start (exclusive). It lists the capabilities (above) required to access the file, for each of C, X, R, and W permissions, as well as a mode byte determining B, S, M, and D permissions. Its contents are as follows:

### C/X/R/W Stop
The addresses of the null byte of the C, X, R, and W lists, respectively. C is taken to start beyond the mode byte (below), and each subsequent list to start beyond the previous list's null.

### Mode Byte
Four two-bit fields, corresponding to the B, S, M, and D permissions. The first bit sets the default value of the permission, while the second determines whether the process may override it. Bits are ordered according to platform endianness.

### C/X/R/W List
A null-terminated newline-separated list of space-separated lists of capabilities, for each of C, X, R, and W. If, for any element of the outer list, the requesting process has every capability of the inner list, it may request that permission.

The remainder of the block, until file stop, contains the file payload.

# Notes
- This structure allows any file of any format under any other system to be embedded as a block with appropriate header, and have access to all features of Vertex files.
- I considered a more fine-grained approach to capability addressing, but decided against it as there would then be far too many layers of indirection.
