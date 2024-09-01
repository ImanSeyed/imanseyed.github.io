---
title: "Do You REALLY Know inotify?"
categories: [linux-kernel]
tags: [inotify, fs]
---

## General Overview
Whenever you want to monitor a file or directory with inotify, you use `inotify_init()` to get a file descriptor known as *"inotify instance"*, and then you create a watcher to watch your path and assign that path to the inotify instance using the `inotify_add_watch()` syscall. By performing these tasks, you are telling the `fsnotify` subsystem which events you want to monitor over the specified path. Then, you can `poll` or just `read` from the *inotify instance* and parse the buffer that you read from it as an `inotify_event` structure or series of this structure. Finally, you can use `inotify_rm_watch()` to remove the specified watcher from the watchers list.

## `inotify` Instance Initialization
### Definition of `inotify_init`

The definition of the `inotify_init()` system call is provided below:

```c
SYSCALL_DEFINE0(inotify_init)
{
	return do_inotify_init(0);
}
```
Seems simple enough. Let us check what happens inside of [`do_inotify_init()`](https://elixir.bootlin.com/linux/v6.10.6/source/fs/notify/inotify/inotify_user.c#L694). I'll provide only the key lines here:
```c
/* inotify syscalls */
static int do_inotify_init(int flags)
{
	struct fsnotify_group *group;
	int ret;

	group = inotify_new_group(inotify_max_queued_events);
	ret = anon_inode_getfd("inotify", &inotify_fops, group,
				  O_RDONLY | flags);

	return ret;
}
```

A `group` is a "thing" that intends to get notification of filesystem events. Each group must implement its `fsnotify_ops`. All of `fsnotify_ops` can be found in [inotify_fsnotify.c](https://elixir.bootlin.com/linux/v6.10.6/source/fs/notify/inotify/inotify_fsnotify.c), and the operation that we care the most about is `handle_inode_event`, which points to [`inotify_handle_inode_event()`](https://elixir.bootlin.com/linux/v6.10.6/source/fs/notify/inotify/inotify_fsnotify.c#L59). I will come back to this function shortly, but for now, let's move on.

`inotify_new_group()` allocates a `group` and assigns its `fsnotify_ops` argument to `inotify_fsnotify_ops`. It then allocates an area of kernel memory for [`inotify_event_info`](https://elixir.bootlin.com/linux/v6.11-rc5/source/fs/notify/inotify/inotify.h#L6) structure. It is quite similar to the `inotify_event` structure that we usually use in userspace, but it includes an additional member named `fse` which is a `fsnotify_event`. This is a `list_head` that allows us to iterate over all of the information about the original object we want to send to a group.

The `anon_inode_getfd` function brings up a mysterious topic, which I'll discuss in details. I created a simple program that uses inotify, and placed a breakpoint at `inotify_init()`. When the syscall got called, I listed all of its file descriptors and used `lsof` to figure out what was going on, and the following output is the result:

```
(gdb) frame
#0  main () at notify.c:10
10              inotify_fd = inotify_init();
(gdb) n
(gdb) info inferiors 
  Num  Description       Connection           Executable        
* 1    process 12865     1 (native)           /tmp/notify 
(gdb) !ls -l /proc/12865/fd
total 0
lrwx------ 1 iman iman 64 Aug 28 12:59 0 -> /dev/pts/8
lrwx------ 1 iman iman 64 Aug 28 12:59 1 -> /dev/pts/8
lrwx------ 1 iman iman 64 Aug 28 12:59 2 -> /dev/pts/8
lr-x------ 1 iman iman 64 Aug 28 12:59 3 -> anon_inode:inotify
(gdb) !lsof -p 12865 | grep a_inode
notify  12865 iman   3r  a_inode   0,14        0     1048 inotify
```
It appears that `fd` number 3 is linked to a file called `anon_inode:inotify`. The following quote is from the [proc_pid_fd(5)](https://man7.org/linux/man-pages/man5/proc_pid_fd.5.html) manual page:

> For file descriptors that have no corresponding inode (e.g., file descriptors produced by bpf(2), epoll_create(2), eventfd(2), inotify_init(2), perf_event_open(2), signalfd(2), timerfd_create(2), and userfaultfd(2)), the entry will be a symbolic link with contents of the form
>
> ```
> anon_inode:<file-type>
> ```

And the `lsof` output indicates that our inotify instance creates a device somewhere, and it has a major number of zero. The file [devices.txt](https://www.kernel.org/doc/Documentation/admin-guide/devices.txt) shows that major number 0 is for unnamed devices, but what is this `anon_inode` exactly?

### Demystifying [`anon_inode`](https://elixir.bootlin.com/linux/v6.10.6/source/fs/anon_inodes.c)
There is a pseudo-filesystem called `anon_inodefs` that runs on a virtual block device and handles all operations in memory. Like all filesystems, it has a superblock that defines the filesystem. As expected, inotify defines a `file_operations` for `anon_inodefs` and passes it to `anon_inode_getfd`. The `file_system_type` of `anon_inodefs` contains the implementations of `init_fs_context` and `kill_sb`, which allow us to create or free the context and the virtual block device of `anon_inodefs` as needed.

```c
static struct file_system_type anon_inode_fs_type = {
	.name		= "anon_inodefs",
	.init_fs_context = anon_inodefs_init_fs_context,
	.kill_sb	= kill_anon_super,
};
```

There is also a `dentry_operations`. Do you remember the command `ls -l /proc/<pid>/fd`? The code below is called in the kernel and is rather straight forward. In this scenario, `dentry->d_name.name` equals to *"inotify"*:

```c
/*
 * anon_inodefs_dname() is called from d_path().
 */
static char *anon_inodefs_dname(struct dentry *dentry, char *buffer, int buflen)
{
	return dynamic_dname(buffer, buflen, "anon_inode:%s",
				dentry->d_name.name);
}

static const struct dentry_operations anon_inodefs_dentry_operations = {
	.d_dname	= anon_inodefs_dname,
};
```

Paths in this filesystem are like this, however in other filesystems, such as ext4, they appear like '/path/to/file'. However, they are just strings, with no magic included. This `dentry_operations` gets assigned to the `anon_inodefs` context in the `anon_inodefs_init_fs_context()` function.

```c
static int anon_inodefs_init_fs_context(struct fs_context *fc)
{
    // [...]
	struct pseudo_fs_context *ctx = init_pseudo(fc, ANON_INODE_FS_MAGIC);
	ctx->dops = &anon_inodefs_dentry_operations;
    // [...]
}
```

The `init_pseudo()` function is a typical aid for pseudo-filesystems. This function is used by unmountable file systems such as sockfs and pipefs to allocate filesystem context. It just allocates an area for [`pseudo_fs_context`](https://elixir.bootlin.com/linux/v6.10.6/source/fs/libfs.c#L704) and fills the context for us. If you follow the `pseudo_fs_get_tree` in the `get_tree` operation of `pseudo_fs_context_ops`, you will find the function that actually builds the block device for our inotify by using [`get_anon_bdev(dev_t *p)`](https://elixir.bootlin.com/linux/v6.10.7/source/fs/super.c#L1202). 

```c
int get_anon_bdev(dev_t *p)
{
	int dev;

	dev = ida_alloc_range(&unnamed_dev_ida, 1, (1 << MINORBITS) - 1,
			GFP_ATOMIC);
    // [...]
	*p = MKDEV(0, dev);
	return 0;
}
```

It allocates a unique id from the `unnamed_dev_ida` [XArray](https://docs.kernel.org/core-api/xarray.html) for our virtual block device by using `ida_alloc_range()`, sets the major number to zero, and dereferences the `p` to fill it with `MKDEV(0, minor)`.  As you may already know, `dev_t` is simply a `typedef` of `u32` that stores the block device's major and minor numbers. The `super_block` structure will store this `dev_t` for us, and `lsof` can find out about these numbers by requesting it.

### `anon_inode_getfd` Function

Now that we know what `anon_inodefs` actually is, we can finally talk about `anon_inode_getfd()`. It returns a file descriptor with `file_operations` of `inotify_file_operations` in the `anon_inodefs` filesystem.

## `inotify_add_watch()` System Call

Again, I'll share only the important lines (with some comments):

```c
SYSCALL_DEFINE3(inotify_add_watch, int fd, const char __user * pathname,
		u32 mask)
{
	struct fsnotify_group *group;
	struct inode *inode;
	struct path path;
	struct fd f;
	int ret;
	unsigned flags = 0;

	f = fdget(fd);

	if (!(mask & IN_DONT_FOLLOW))
		flags |= LOOKUP_FOLLOW; // Dereference pathname if it is a symbolic link.
	if (mask & IN_ONLYDIR)
		flags |= LOOKUP_DIRECTORY; // Find inode of *pathname* only if it is a directory
	ret = inotify_find_inode(pathname, &path, flags, (mask & IN_ALL_EVENTS));
	/* inode held in place by reference to path; group by fget on fd */
	inode = path.dentry->d_inode;
	group = f.file->private_data;
	/* create/update an inode mark */
	ret = inotify_update_watch(group, inode, mask);
	return ret;
}
```

The `fdget()` converts a `fd` number to a `fd` structure which allows us to access the `file` structure. This helps `inotify_add_watch()` to read the `priv` of the `file`, which in our case, is the `group`.

The `inotify_find_inode()` function as its name suggests, resolves a user-provided path to a specified inode.

The `inotify_update_watch()` updates the watch and returns its watch descriptor; if you're not watching that path already, it creates a new watch for you. Marks (`fsnotify_marks` to be precise) are objects associated with core inodes that allow `fsnotify` listeners to exclude or include events that match a specific mask. Each `group` has its own mark.

## Relation Between `VFS` and `fsnotify`

Whenever you call the `write()` or `read()` syscalls in your program, [kernel will eventually call `vfs_write()` or `vfs_read()` for you](https://elixir.bootlin.com/linux/v6.10.6/source/fs/read_write.c#L643). If you read more than 0 bytes using `vfs_read()`, `fsnotify_access(file)` gets called on that file, and for `vfs_write()`, `fsnotify_modify(file)` will notify the `fsnotify` subsystem. If the file is the `root` (e.g., you've marked a file only) `fsnotify()` will get called immediately; otherwise, the kernel will look for the parent and then call `fsnotify()` over that `dentry`. The real magic happens in the `fsnotify()` function. This is where the `fsnotify` sends the event to the `group` by `send_to_group()` function. Remember the `inotify_handle_inode_event()` that I mentioned earlier? It becomes handy here. It inserts the event into your `group->notification_list`. The user won't be aware of this event until they read from the inotify instance. By ignoring all of the locking and waitqueue stuff, `inotify_read()` looks like this:

```c
static ssize_t inotify_read(struct file *file, char __user *buf,
			    size_t count, loff_t *pos)
{
	struct fsnotify_group *group;
	struct fsnotify_event *kevent;
	char __user *start;
	int ret;

	start = buf;
	group = file->private_data;

	while (1) {
		kevent = get_one_event(group, count);

		if (kevent) {
			ret = copy_event_to_user(group, kevent, buf);
			fsnotify_destroy_event(group, kevent);
			buf += ret;
			count -= ret;
			continue;
		}

        if (start != buf)
			break;
	}
	return ret;
}
```

The `get_one_event()` fetches an event from the group if it's small enough to fit in `count`, and the `copy_event_to_user()` will write the event to the userspace buffer specified in your `read()` call.
