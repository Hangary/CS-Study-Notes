# File System

***What is a file system?***
- A filesystem is how information is organized on a storage medium, such as disk, including *HDD* or *SSD*. The storage medium is also called ***backing***. A filesystem is implemented on top of this backing. 
- A filesystem is one layer of abstraction, which provides some "API"s (*callbacks* functions) for the operating systems to control files over a backing.
- Some common filesystems: `EXT`, `NTFS`, `FAT32`.

What is a ***file-like object***?
- An old UNIX adage: "everything is a file".
- File operations provide an interface to abstract many different operations: Network sockets, hardware devices, and data on the disk are all represented by file-like objects. 
- A file-like object must follow the following conventions:
  - It must present itself to the filesystem.
  - It must support common filesystem operations, such as `open`, `read`, `write`. At a minimum, it needs to be opened and closed.

## Storing data on disk

Here is a sample image of a filesystem structure:
![20201107211440-687474703a2f2f74696e66322e7675622e61632e62652f7e647665726d6569722f6d616e75616c2f75696e74726f2f6469736b2e676966](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201107211440-687474703a2f2f74696e66322e7675622e61632e62652f7e647665726d6569722f6d616e75616c2f75696e74726f2f6469736b2e676966.gif)

It can be broken into:
- **Superblock**: store metadata about the filesystem, how large, last modified time, a journal, number of inodes and the first inode start, number of data block and the first data block start.
  - It might also store journal, which logs changes to the filesystem.
  - Superblock is always the first block to read in a disk when booting.
- **Inode (Index Node)**: store metadata of a file and the pointers to the disk blocks
  - This is the key abstraction. ==An inode is a file.==
  - An inode represents a filesystem object. A filesystem object can be one of various things including a file or a directory.
  - Each inode stores the meta-information of the filesystem object's data, including filesystem object attributes and disk block location(s).
    - Filesystem object attributes may include manipulation metadata (e.g. change, access, modify time), as well as owner and permission data (e.g. group-id, user-id, permissions).
- ==One thing to note: **filename != file**. An inode (file) does not contain filenames. Filenames are stored in other files and used as an indirection to the inode.==
- **Disk Blocks**: store the contents of a file or a directory
  - The actual contents of the file.
  - To support virtual memory, ==disk blocks usually have same size as memory pages==.
  - ***Direct Blocks*** and ***Indirect Blocks***:
    - Direct Blocks are normal disk blocks that store the contents of a file or directory.
    - Indirect Blocks are special blocks that store pointers to other blocks.
    - ==Indirect Blocks are used to solve the case where a large file is bigger than the maximum space addressable by the direct blocks of an inode.==

## Directory File: a mapping from names to inode numbers

A ***directory*** is a special inode, which contains a mapping of names to inode numbers. 

Theoretically, directories are just like actual files. The disk blocks will contain directory entries or dirents. What that means is that our disk block can look like this:
| inode_num  | name  |
|---|---|
| 2043567  | hi.txt  |

Each directory entry could either be a fixed size, or a variable c-string. It depends on how the particular filesystem implements it at the lower level.

POSIX provides a small set of functions to read the filename and inode number for each entry (see below).

### System call: `stat`

We can use the `stat` system calls to find out the meta-information about a file, which is stored in the inode. 

```C
// pre-defined stat struct
struct stat {
    dev_t     st_dev;     /* ID of device containing file */
    ino_t     st_ino;     /* inode number */
    mode_t    st_mode;    /* protection */
    nlink_t   st_nlink;   /* number of hard links */
    uid_t     st_uid;     /* user ID of owner */
    gid_t     st_gid;     /* group ID of owner */
    dev_t     st_rdev;    /* device ID (if special file) */
    off_t     st_size;    /* total size, in bytes */
    blksize_t st_blksize; /* blocksize for file system I/O */
    blkcnt_t  st_blocks;  /* number of 512B blocks allocated */
    time_t    st_atime;   /* time of last access */
    time_t    st_mtime;   /* time of last modification */
    time_t    st_ctime;   /* time of last status change */
};

// use stat system call
struct stat s;
stat("notes.txt", &s);
printf("Last accessed %s", ctime(&s.st_atime));
```

Three versions of `stat`:
```C
int stat(const char *path, struct stat *buf);
int fstat(int fd, struct stat *buf);  // with file handler
int lstat(const char *path, struct stat *buf);  // read link info
```

### System call: `opendir`, `readdir`, `closedir`

A simple implementation of `ls`:
```C
#include <stdio.h>
#include <dirent.h>
#include <stdlib.h>

int main(int argc, char **argv) {
    if(argc ==1) {
        printf("Usage: %s [directory]\n", *argv);
        exit(0);
    }
    struct dirent *dp;
    DIR *dirp = opendir(argv[1]);
    while ((dp = readdir(dirp)) != NULL) {
        printf("%s %lu\n", dp-> d_name, (unsigned long)dp-> d_ino );
    }

    closedir(dirp);
    return 0;
}
```

Note: 
- The `readdir` function returns "." (current directory) and ".." (parent directory). If you are looking for sub-directories, you need to explicitly exclude these directories.
- The `readdir` function is not thread-safe! For multi-threaded searches use `readdir_r` which requires the caller to pass in the address of an existing dirent struct.

### File Remove

When you remove a file (using `rm` or `unlink`), you are removing an inode reference from a directory. However the inode may still be referenced from other directories. In order to determine if the contents of the file are still required, each inode keeps a ***reference count*** that is updated whenever a new link is created or destroyed.

## File Permissions and Bits

### File Permission Bits

Permissions are a key part of the way UNIX systems provide security in a filesystem. 

File permission bits include three roles: **user**, **group**, **world**.

Each role has three bits for accordingly **Read (R)**, **Write (W)**, and **Execute (X)**.

How to determine roles:
- **User**: Each file has an owner. If your process has the same user id as the owner (or root) then the permissions in the first triad apply to you. 
- **Group**: If you are in the same group as the file (all files are also owned by a group) then the next set of permission bits applies to you. 
- **World**: If none of the above apply, the last triad applies to you.

We can use command `chown username filename` to change the ownership of a file.

Some common file permissions include in octal format (base 8):
- 755: `rwx r-x r-x`
  - user: rwx, group: r-x, others: r-x
  - User can read, write and execute. Group and others can only read and execute.
- 644: `rw- r-- r--`
  - user: rw-, group: r--, others: r--
  - User can read and write. Group and others can only read.
- Remember the 3 bits:
  - `7 = 4 + 2 + 1` : full permission
  - `6 = 4 + 2 + 0` : read and write permission
  - `5 = 4 + 0 + 1` : read and execute permission
  - `4 = 4 + 0 + 0` : only read permission 

An example mapping table:

| Octal Code | User | Group	| Others |
|------------|------|-------|--------|
| 755	| rwx	| r-x	| r-x |
| 644	| rw- |	r- |	r- |



One thing to note: `rwx` bits have a slightly different meaning for directories.
- *Write* access to a directory that will allow a program to create or delete new files or directories inside. 
  - You can think about this as having write access to the directory entry (dirent) mappings. 
- *Read* access to a directory will allow a program to list a directory's contents. 
  - This is read access to the directory entry (dirent) mapping. 
- *Execute* access will allow a program to enter the directory using `cd`. 
  - Without the execute bit, it any attempt create or remove files or directories will fail since you cannot access them. You can, however, list the contents of the directory.

#### User ID / Group ID

Every user in a UNIX system has a user ID. This is a unique number that can identify a user. 

Similarly, users can be added to collections called groups, and every group also has a unique identifying number. 

Every file, upon creation, has an owner, who is the creator of the file. This owner's user ID (`uid`) can be found inside the `st_mode` file of a struct `stat` with a call to `stat`. Similarly, the group ID (`gid`) is set as well.

Every process has a `uid` and `gid`, which can be determined with `getuid` and `getgid`. They would be used to check whether the process has the permission to operate over files.

#### `chmod`

We can use `chmod` (change the file mode bits) to change the file permissions.
It is a command as well as a system call.
```
$ chmod 700 file1
$ chmod o-rx file2
```

```C
int chmod(const char *path, mode_t mode);
```

### Sticky Bit

> Sticky bits as we use them today serve a different purpose from initial introduction. 
> Sticky bits were once a bit that could be set on an executable file that would allow a program's text segment to remain in swap even after the end of the program's execution. This made subsequent executions of the same program faster. 
> Today, this behavior is no longer supported and the sticky bit only holds meaning when set on a directory,

When a directory's sticky bit is set, only the file's owner, the directory's owner, and the root user can *rename* or *delete* the file.

This is useful when multiple users have write access to a common directory.

To set the sticky bit, use `chmod +t <dir_name>`.

### `umask`

The `umask` subtracts permission bits from `777` and is used when new files and new directories are created by `open`, `mkdir` etc. 
- For example, `022` (octal) means that group and other privileges will not include the writable bit. 
- Each process (including the shell) has a current umask value. When forking, the child inherits the parent's umask value.

For example, by setting the umask to 077 in the shell, ensures that future file and directory creation will only be accessible to the current user:
```
$ umask 077
$ mkdir secretdir
```




### Temporary Permission

#### `setuid`

The ***set-user-ID-on-execution*** bit changes the user associated with the process when the file is run. This is typically used for commands that need to run as root but are executed by non-root users. 

The most common usecase is so that the user can have root(admin) access for the duration of the program. 

An example of this is `sudo`:
```
$ ls -l /usr/bin/sudo
-r-s--x--x  1 root  wheel  327920 Oct 24 09:04 /usr/bin/sudo
```
The `s` bit means execute and set-uid; the effective userid of the process will be different from the parent process. In this example it will be root.

#### `geteuid`

How do I ensure only privileged users can run my code?
Check the effective permissions of the user by calling `geteuid()`. A return value of zero means the program is running effectively as root.

What's the difference between `getuid()` and `geteuid()`?
- `getuid` returns the real user id (zero if logged in as root)
- `geteuid` returns the effective userid (zero if acting as root, e.g. due to the setuid flag set on a program)


## `ls`

We can check the permission by using `ls -l`. Note that the permissions will output in the format `drwxrwxrwx`. 
- The first character (`d`) indicates the type of file type. Possible values for the first character:
  - (-) regular file
  - (d) directory
  - (c) character device file
  - (l) symbolic link
  - (p) pipe
  - (b) block device
  - (s) socket
- The middle nine characters represent the permission modes.

To list all files including the normally hidden entries use ls with `-a` option.

## `mkdir`

Specify the permission modes when creating a directory:
```
$ mkdir -m 700 /tmp/mystuff
```

Automatically create parent directories:
```
$ mkdir -p d1/d2/d3
# if d1 or d2 does not exist, they can be created automatically
```

## File Links

Links are what force us to model a filesystem as a tree rather than a graph.

While modeling the filesystem as a tree would imply that every inode has a unique parent directory, links allow inodes to present themselves as files in multiple places, potentially with different names, thus leading to an inode having multiple parent directories. 

There are two kinds of links:
1. **Hard Links**: A hard link is simply an entry in a directory assigning some name to an inode number that already has a different name and mapping in either the same directory or a different one. 
    - If we already have a file on a file system we can create another link to the same inode using the `ln` command.
2. **Soft Links**: The second kind of link is called a soft link, symbolic link, or symlink. A symbolic link is different because it is a file with a special bit set and <u>stores a path to another file</u>. 
    - Quite simply, without the special bit, it is nothing more than a text file with a file path inside. 
    - To create a symbolic link in the shell, use `ln -s`.
3. When people generally talk about a link without specifying hard or soft, they are referring to a *hard link*.

### Hard Links

Since filename does not relate to the file inode, we can create a new link to the same file inode for an existing file:
```bash
$ ln file1.txt blip.txt
```
This is like duplicate directory entries pointing to the same inode.

We can use the `link` system call to create a link in `C`:
```C
link(const char *path1, const char *path2);
// example
link("file1.txt", "blip.txt");
```

This kind of links are called **hard links**.

Note: ==**Hard links are not recommended or even banned for *directories* in most filesystems.**== The reason is to prevent cyclic file structures, which is against the integrity and cause undefined behaviors.


### Symbolic Links

We can create a symbolic link by using `ln -s` command or `symlink` system call.
```C
symlink(const char *target, const char *symlink);
```

To read the contents of the link as just a file use `readlink`:
```bash
$ readlink myfile.txt
../../dir1/notes.txt
```

To read the meta-(stat) information of a symbolic link use `lstat` not `stat`:
```C
struct stat s1, s2;
stat("myfile.txt", &s1); // stat info about  the notes.txt file
lstat("myfile.txt", &s2); // stat info about the symbolic link
```

Advantages of symbolic links:
- Can refer to files that don't exist yet
- Unlike hard links, can refer to directories as well as regular files
- Can refer to files (and directories) that exist outside of the current file system.

Main disadvantage: 
- **Slower** than regular files and directories. When the links contents are read, they must be interpreted as a new path to the target file.


### Remove an inode reference

When you remove a file using `rm` or `unlink`, you are removing an inode reference from a directory. 

Difference between `remove` and `unlink`: `remove` is based on `unlink`. If path specifies a directory, `remove(path)` is the equivalent of `rmdir(path)`. Otherwise, it is the equivalent of `unlink(path)`.

However, after removing, the inode may still be referenced from other directories. 

To determine if the contents of the file are still required, each inode keeps a ***reference count*** that is updated whenever a new link is created or destroyed. This count only tracks hard links, symlinks are allowed to refer to a non-existent file and thus, do not matter.


## Hide Files or Directories

If files or directories that start with a ".", then (by default) they are not displayed by standard tools and utilities. This is often used to hide configuration files inside the user's home directory. 

Another trick for hidding directories to ***turn off the execute bit on directories***.
The execute bit for a directory is used to control whether the directory contents is listable.
```bash
$ chmod ugo-x dir1 
$ ls dir1
ls: dir1: Permission denied

$ mkdir /tmp/mystuff
$ chmod 700 /tmp/mystuff
$ ls /tmp/mystuff
ls: /tmp/mystuff: Permission denied
```
In other words, the directory itself is discoverable but its contents cannot be listed.

A better and more secure way:
```
$ mkdir -m 700 /tmp/mystuff
```



## File Globbing

Before executing the program the shell expands parameters into matching filenames.

```
$ echo my*
```
Expands to
```
$ echo my1.txt mytext.txt myomy
```

This is known as file globbing and is processed before the command is executed. ie the command's parameters are identical to manually typing every matching filename.

## `touch`

The touch executable:
1. creates file if it does not exist and 
2. updates the file's last modified time to be the current time. 

An example use of touch is to force make to recompile a file that is unchanged after modifying the compiler options inside the makefile:
```
$ touch myprogram.c   # force my source file to be recompiled
$ make
```

## Virtual File Systems

POSIX systems, such as Linux and Mac OSX (which is based on BSD) include several virtual filesystems that are mounted (available) as part of the file-system. 
- Files inside these virtual filesystems do not exist on the disk; 
- they are generated dynamically by the kernel when a process requests a directory listing.

Linux provides 3 main virtual filesystems:
```
/dev  - A list of physical and virtual devices (for example network card, cdrom, random number generator)
/proc - A list of resources used by each process and (by tradition) set of system information
/sys - An organized list of internal kernel entities
```

- `/proc`:
  - `/proc/meminfo`
  - `/proc/cpuinfo`
- `/dev`: virtul files
  - `/dev/null`: Bytes sent to `/dev/null/` are never stored - they are simply discarded. A common use of `/dev/null` is to discard standard output.
  - `/dev/zero`: provides an infinite stream of zero bytes
  - `/dev/urandom`: provides an infinite stream of random bytes.
    - Difference between `random` and `urandom`:
      - `/dev/random` is a file which contains number generator where the entropy is determined from environmental noise. Random will block/wait until enough entropy is collected from the environment.
      - `/dev/urandom` is like random, but differs in the fact that it allows for repetition (lower entropy threshold), thus wont block.
- `/tmp`: sticky bit is set, therefore normal users cannot delete or rename files in `tmp`

## `stat`

`stat` command displays the detailed status of a particular file or a file system.
- Display detailed status of a file: `stat <filename>`
- Display the status of a file system: `stat -f <filesystem>`


## `mount`

All files accessible in Linux, are arranged in one big tree: the file hierarchy, rooted at `/`. These files can be spread out over several devices. The `mount` command mounts a storage device or filesystem to the file tree, making it accessible and attaching it to an existing directory structure Conversely, the umount command will detach it again.

Using `mount` without any options generates a list (one filesystem per line) of mounted filesystems including networked, virtual and local (spinning disk / SSD-based) filesystems.

```
$ mount
/dev/mapper/cs241--server_sys-root on / type ext4 (rw)
proc on /proc type proc (rw)
sysfs on /sys type sysfs (rw)
devpts on /dev/pts type devpts (rw,gid=5,mode=620)
tmpfs on /dev/shm type tmpfs (rw,rootcontext="system_u:object_r:tmpfs_t:s0")
/dev/sda1 on /boot type ext3 (rw)
/dev/mapper/cs241--server_sys-srv on /srv type ext4 (rw)
/dev/mapper/cs241--server_sys-tmp on /tmp type ext4 (rw)
/dev/mapper/cs241--server_sys-var on /var type ext4 (rw)rw,bind)
/srv/software/Mathematica-8.0 on /software/Mathematica-8.0 type none (rw,bind)
engr-ews-homes.engr.illinois.edu:/fs1-homes/angrave/linux on /home/angrave type nfs (rw,soft,intr,tcp,noacl,acregmin=30,vers=3,sec=sys,sloppy,addr=128.174.252.102)
```

Mount a filesystem:
```
$ sudo mount /dev/cdrom /media/cdrom  # mount cdrom to media
```

Mount a disk image:
```
$ mkdir arch
$ sudo mount -o loop archlinux-2015.04.01-dual.iso ./arch # mount to arch
```

### File System: Memory mapped files and Shared memory

How does the operating system load my process and libraries into memory?
- It maps the executable and library file contents into the address space of the process. If many programs only need read-access to the same file (e.g. /bin/bash, the C library) then the same physical memory can be shared between multiple processes.
- The same mechanism can be used by programs to directly map files into memory.