# Libsquash

Portable, user-land SquashFS that can be easily linked and embedded within your application; a [squashfuse](https://github.com/vasi/squashfuse) fork.

[![Build status: Linux and Darwin](https://travis-ci.org/pmq20/libsquash.svg?branch=master)](https://travis-ci.org/pmq20/libsquash)
[![Build status: Windows](https://ci.appveyor.com/api/projects/status/f4htq948gag3l2k8/branch/master?svg=true)](https://ci.appveyor.com/project/pmq20/libsquash/branch/master)

## About the fork

This project was forked from https://github.com/vasi/squashfuse with the following modifications.

1. __NO FUSE REQUIRED__, so that it can be so portable that even systems without fuse could use it.
1. Read data from the memory instead of from a file by passing a pointer to an array of bytes,
which could be generated by
an independently installed [mksquashfs](http://squashfs.sourceforge.net/) tool
and loaded into memory in advance.
1. Introduced vfd(virtual file descriptor) as a handle for follow-up.
libsquash operations. The vfd is a non-negative integer that lives together with
other ordinary file descriptors of the process.
1. Added new API's that mirror the calling style of common system calls;
also added are a sample on how to use libsquash in an unobtrusive way by utilizing those API's.
1. Made it compile on 3 platforms simultanesly: Windows, Mac OS X and Linux. Added CMake so that Xcode and Vistual Studio Projects could be easily generated.
1. Added tests for both the old and new API's of the library.

## Building

On most systems you could build the library using the following commands,

    mkdir build
    cd build
    cmake ..
    cmake --build .

Use `cmake -DBUILD_TESTS=ON ..` to build the tests in addition and use `ctest --verbose` to run them.

## New API's

### `squash_stat(error, fs, path, buf)`

Obtains information about the file pointed to by `path` of a SquashFS `fs`.
The `buf` argument is a pointer to a stat structure as defined by
`<sys/stat.h>` and into which information is placed concerning the file.
Upon successful completion a value of `0` is returned.
Otherwise, a value of `-1` is returned and `error` is set to the reason of the error.

### `squash_lstat(error, fs, path, buf)`

Acts like `squash_stat()` except in the case where the named file is a symbolic link;
`squash_lstat()` returns information about the link,
while `squash_stat()` returns information about the file the link references.

### `squash_fstat(error, fs, vfd, buf)`

Obtains the same information as `squash_stat()`
about an open file known by the virtual file descriptor `vfd`.

### `squash_open(error, fs, path)`

Opens the file name specified by `path` of a SquashFS `fs` for reading.
If successful, `squash_open()` returns a non-negative integer, termed a vfd(virtual file descriptor).
It returns `-1` on failure and sets `error` to the reason of the error.
The file pointer (used to mark the current position within the file) is set to the beginning of the file.
The returned vfd should later be closed by `squash_close()`.

### `squash_close(error, vfd)`

Deletes a `vfd`(virtual file descriptor) from the per-process object reference table.
Upon successful completion, a value of `0` is returned.
Otherwise, a value of `-1` is returned and `error` is set to the reason of the error.

### `squash_read(error, vfd, buf, nbyte)`

Attempts to read `nbyte` bytes of data from the object
referenced by `vfs` into the buffer pointed to by `buf`,
starting at a position given by the pointer associated with `vfd` (see `squash_lseek`),
which is then incremented by the number of bytes actually read upon return.
When successful it returns the number of bytes actually read and placed in the buffer;
upon reading end-of-file, zero is returned;
Otherwise, a value of `-1` is returned and `error` is set to the reason of the error.

### `squash_lseek(error, vfd, offset, whence)`

Repositions the offset of `vfs` to the argument `offset`, according to the directive `whence`.
If `whence` is `SQUASH_SEEK_SET` then the offset is set to `offset` bytes;
if `whence` is `SQUASH_SEEK_CUR`, the `offset` is set to its current location plus `offset` bytes;
if `whence` is `SQUASH_SEEK_END`, the `offset` is set to the size of the file
and subsequent reads of the data return bytes of zeros.
The argument `fildes` must be an open virtual file descriptor.
Upon successful completion,
it returns the resulting offset location as measured in bytes from the beginning of the file.
Otherwise, a value of `-1` is returned and `error` is set to the reason of the error.

### `squash_readlink(error, fs, path, buf, bufsize)`

Places the contents of the symbolic link `path` of a SquashFS `fs`
in the buffer `buf`, which has size `bufsize`.
It does not append a NUL character to `buf`.
If it succeeds the call returns the count of characters placed in the buffer;
otherwise `-1` is returned and `error` is set to the reason of the error.

### `squash_opendir(error, fs, filename)`

Opens the directory named by `filename` of a SquashFS `fs`,
associates a directory stream with it and returns a pointer
to be used to identify the directory stream in subsequent operations.
The pointer `NULL` is returned if `filename` cannot be accessed,
or if it cannot allocate enough memory to hold the whole thing,
and sets `error` to the reason of the error.
The returned resource should later be closed by `squash_closedir()`.

### `squash_closedir(error, dirp)`

Closes the named directory stream and
frees the structure associated with the `dirp` pointer,
returning `0` on success.
On failure, `-1` is returned and `error` is set to the reason of the error.

### `squash_readdir(error, dirp)`

Returns a pointer to the next directory entry.
It returns `NULL` upon reaching the end of the directory or on error. 
In the event of an error, `error` is set to the reason of the error.

### `squash_telldir(dirp)`

Returns the current location associated with the named directory stream.

### `squash_seekdir(dirp, loc)`

Sets the position of the next `squash_readdir()` operation on the directory stream.
The new position reverts to the one associated with the directory stream
when the `squash_telldir()` operation was performed.

### `squash_rewinddir(dirp)`

Resets the position of the named directory stream to the beginning of the directory.

### `squash_dirfd(error, dirp)`

Returns the integer Libsquash file descriptor associated with the named directory stream.
On failure, `-1` is returned and `error` is set to the reason of the error.

### `squash_scandir(error, fs, dirname, namelist, select, compar)`

Reads the directory `dirname` of a SquashFS `fs` and
builds an array of pointers to directory entries using `malloc`.
If successful it returns the number of entries in the array; 
otherwise `-1` is returned and `error` is set to the reason of the error.
A pointer to the array of directory entries is stored
in the location referenced by `namelist` (even if the number of entries is `0`),
which should later be freed via `free()` by freeing each pointer
in the array and then the array itself.
The `select` argument is a pointer to a user supplied subroutine which is
called by `scandir` to select which entries are to be included in the array.
The `select` routine is passed a pointer to a directory entry
and should return a non-zero value if the directory entry
is to be included in the array.
If `select` is `NULL`, then all the directory entries will be included.
The `compar` argument is a pointer to a user supplied subroutine
which is passed to `qsort` to sort the completed array.
If this pointer is `NULL`, then the array is not sorted.

## Acknowledgment

Thank you [Dave Vasilevsky](https://github.com/vasi) for the excellent work of squashfuse!

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/pmq20/libsquash.

## License

Copyright (c) 2012 **Dave Vasilevsky** &lt;dave@vasilevsky.ca&gt;, **Phillip Lougher** &lt;phillip@squashfs.org.uk&gt;.

Copyright (c) 2016-2017 **Minqi Pan** &lt;pmq2001@gmail.com&gt;, **Shengyuan Liu** &lt;sounder.liu@gmail.com&gt;.
