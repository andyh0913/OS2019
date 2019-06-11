# OS2019 Project2
![](https://i.imgur.com/SprVWWI.png)

PROGRAMMING DESIGN
----
### ksocket
#### `ksocket.c`
- Mostly done by the author, here we beiefly describe what each function does
- `ksocket_t ksocket(int domain, int type, int protocol)`
    - creates a socket and returns a pointer to the created socket 
- `int kbind(ksocket_t socket, struct sockaddr *address, int address_len)`
    - binds a socket to an address, used in master side
- `int klisten(ksocket_t socket, int backlog)`
    - connect to an address, used in master side
- `ksocket_t kaccept(ksocket_t socket, struct sockaddr *address, int *address_len)`
    - accept a connection on a socket, then create a new socket and return a pointer to it
- `ssize_t krecv(ksocket_t socket, void *buffer, size_t length, int flags)`
    - receive message on a socket with TCP
- `ssize_t ksend(ksocket_t socket, const void *buffer, size_t length, int flags)`
    - send message on a socekt with TCP
- 

### User program
#### `master.c`
- Implemented the `mmap` method
```clike=
while (offset < file_size) {
    size_t length = MAP_SIZE;            // size to copy

    if ((file_size - offset) < length)   // if rest of file < MAP_SIZE
        length = file_size - offset;     // assign the rest to length
        
    // get file address to read from
    file_address = mmap(NULL, length, PROT_READ, MAP_SHARED, file_fd, offset);
    // get device address to write to
    kernel_address = mmap(NULL, length, PROT_WRITE, MAP_SHARED, dev_fd, offset);
    // write data from file to device
    memcpy(kernel_address, file_address, length);
    // increment offset
    offset += length;
    ioctl(dev_fd, 0x12345678, length);
}
```
#### slave.c
- Implemented the `mmap` method
```clike=
while (true) {
    ret = ioctl(dev_fd, 0x12345678);
    if (ret == 0) {    // no bytes to read
        file_size = offset;
        break;
    }
    // allocate space
    posix_fallocate(file_fd, offset, ret);
    // get file address to write to
    file_address = mmap(NULL, ret, PROT_WRITE, MAP_SHARED, file_fd, offset);
    // get device address to read from
    kernel_address = mmap(NULL, ret, PROT_READ, MAP_SHARED, dev_fd, offset);
    // write data from device to file
    memcpy(file_address, kernel_address, ret);
    // increment offset
    offset += ret;
}
```

### Devices
#### master_device.c
- implemented mmap_fault
    - it keeps on generating error messages when the function takes more than one parameter, so we changed the method from parsing `vma` as one of the parameter from the outside, into `vma = vmf->vma` and it worked.
```clike=
static int mmap_fault(struct vm_fault *vmf)
{
	struct vm_area_struct *vma = vmf->vma;
	vmf->page = virt_to_page(vma->vm_private_data);
	get_page(vmf->page);
	return 0;
}

```
- implemented my_mmap
```c=
static int my_mmap(struct file *file, struct vm_area_struct *vma)
{
	io_remap_pfn_range(vma,
		vma->vm_start,
		virt_to_phys(file->private_data) >> PAGE_SHIFT,
                vma->vm_end - vma->vm_start,
		vma->vm_page_prot);
	vma->vm_ops = &my_vm_ops;
	vma->vm_flags |= VM_RESERVED;
	vma->vm_private_data = file->private_data;
	mmap_open(vma);
	return 0;
}
```
#### slave_device.c
- implemented mmap_fault and my_mmap <b>(almost same as master_device.c</b>


THE RESULT
----
### mmap
- file 1
![](https://i.imgur.com/yft9pJb.png)
- file 2
![](https://i.imgur.com/wNSVyIt.png)
- file 3
![](https://i.imgur.com/mtMFkmC.png)
- file 4
![](https://i.imgur.com/VsXu7fc.png)


### fcntl
- file 1
![](https://i.imgur.com/Z04zUaT.png)
- file 2
![](https://i.imgur.com/HmEo0z9.png)
- file 3
![](https://i.imgur.com/jUZMzAs.png)
- file 4
![](https://i.imgur.com/rqjQOGe.png)

### dmesg
![](https://i.imgur.com/Ln3mn1P.png)



COMPARISON OF FILE I/O AND MMAP
----

![](https://i.imgur.com/6m7NyDE.png)


- Assumption of the reason why `average byte transmission time` decreases as file enlarges
    - The total transmission time actually includes not only the transmission time, but also the execution time of the function
        - We proved this by using two terminals. The result will include the time between switching windows and not pressing enter at the same time.
    - So the overhead(time consumption for executing functions not involved in transmitting files) of this function could be approximately the same no matter what file it is transmitting.
    - Thus increassing bytes makes the`average byte transmission time` less.

- <b>mmap is slower than fcntl</b>
    - Because the cost of creating map pages is relatively expensive.
    - Also it really depends on how fast the hard disk works.
    - And file I/O operation doesn't cache the file. Since we only read each file once in an operation, caching a read-only file is kind of meaningless.

WORK LIST
----
- b05901015 陳培威
    - user program
    - debugging
- b05901063 黃世丞
    - debugging
    - environment
- b05901179 詹欣玥
    - master_device/slave_device
    - report: comparison
- b03201012 林松逸
