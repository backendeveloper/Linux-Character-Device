
Linux Character Device Driver
===============


> **Note:**

>   The post helps understand how to write a device driver, the significance of a device file and its role in the interaction between a user program and device driver.


Introduction and some basics
-----------------------------------
###*Introduction*
The devices in UNIX fall in two categories- Character devices and Block devices. Character devices can be compared to normal files in that we can read/write arbitrary bytes at a time (although for most part, seeking is not supported).They work with a stream of bytes. Block devices, on the other hand, operate on blocks of data, not arbitrary bytes. Usual block size is 512 bytes or larger powers of two. However, block devices can be accessed the same was as character devices, the driver does the block management. (Networking devices do not belong to these categories, the interface provided by these drivers in entirely different from that of char/block devices)

In our example, we will be developing a character device represented by the device file /dev/myDev.  The mechanisms for creating this file will be explained later.

###*Under the hood*

Now how does Linux know which driver is associated with which file? For that, each device and its device file has associated with it, a unique Major number and a Minor number. No two devices have the same major number.  When a device file is opened, Linux examines its major number and forwards the call to the driver registered for that device.  Subsequent calls for read/write/close too are processed by the same driver. As far as kernel is concerned, only major number is important. Minor number is used to identify the specific device instance if the driver controls more than one device of a type.

To know the major, minor number of devices, use the **ls – l** command as shown below.

```{r, engine='bash', count_lines}
$ ls -l /dev/random 
```

A Linux driver is a Linux module which can be loaded and linked to the kernel at runtime. The driver operates in **kernel space** and becomes part of the kernel once loaded, the kernel being monolithic. It can then access the symbols exported by the kernel.

When the device driver module is loaded, the driver first registers itself as a driver for a particular device specifying a particular Major number.

It uses the call ***register_chrdev*** function for registration. The call takes the Major number, Minor number, device name and an address of a structure of the type file_operations(discussed later) as argument. In our example, we will be using a major number of 89 . The choice of major number is arbitrary but it has to be unique on the system.

The syntax of register_chrdev is

***int register_chrdev(unsigned int major,const char *name,struct file_operations *fops)***

Driver is unregistered by calling the ***unregister_chrdev*** function.

Since device driver is a kernel module, it should implement ***init_module*** and ***cleanup_module*** functions. The register_chrdev call is done in the init_module function  and unregister_chrdev call is done in the cleanup_module function.

When register_chrdev call is done, the fourth argument is a structure that contains the addresses of these **callback functions**, callbacks for open, read, write, close system calls. The structure is of the type ***file_operations*** and has 4 main fields that should be set  **– read,write,open and release**. Each field must be assigned an address of a function that would be called when open, read,write , close system calls are called respectively.  For eg

```
// structure containing callbacks
static struct file_operations fops =
{
	.read = dev_read, // address of dev_read
	.open = dev_open, // address of dev_open
	.write = dev_write, // address of dev_write
	.relase = dev_rls, // address of dev_rls
};
```
----

Creating a device file
-------------------------
A device file is a special file. It can’t just be created using cat or gedit or shell redirection for that matter. The shell command mknod is usually used to create device file. The syntax of mknod is
mknod path type major minor

                             mknod path type major minor

> **Note:**

> - **path** :  path where the file to be created. It’s not necessary that the device file needs to be created in the /dev directory. It’s a mere convention. A device file can be created just about anywhere.
> - **type** :  'c' or 'b' . Whether the device being represented is a character device or a block device. In our example, we will be simulating a character device and hence we choose 'c'.
> - **major, minor** : the major and minor number of the device.

Heres how

```{r, engine='bash', count_lines}
root@ubuntu:/home/kaan# mknod /dev/myDev c 89 l
root@ubuntu:/home/kaan# chmod a+r+w /dev/myDev
```

> **Note:**

> - **chmod**, though not necessary is done because, if not done, only processes will root permission can read or write to our device file.

----

The driver code
------------------
Given below is the code of the device driver


![Imgur](http://i.imgur.com/t1tn3da.gif)
*Device Driver Code*


----
Compiling the driver
------------------------
A Linux module cannot just be compiled the way we compile normal C files.  cc filename.c won’t work. Compiling a Linux module is a separate process by its own. We use the help of kernel Makefile for compilation. The makefile we will be using is.

[Imgur](http://i.imgur.com/wAYM9fV.png)
*makefile for module compilation*

Here, we are making use of the **kbuild** mechanism used for compiling the kernel.
The result of compilation is a **ko** file (kernel object) which can then be loaded dynamically when required.

----
Loading and unloading the driver
---------------------------------------
Once the compilation is complete, we can use either **insmod** or **modprobe** command ( ***insmod myDev.ko***  or ***modprobe myDev.ko***, of course assuming the current directory contains the compiled module). The difference between insmod and modprobe is that modprobe automatically reads our module to find any dependencies on other modules and loads them before loading our module (these modules must be present in the standard path though!). insmod lacks this feature.

To test if the driver has been loaded successfully, do ***cat /proc/modules and cat /proc/devices***.  We should see our module name in the first case and device name in the second.

[Imgur](http://i.imgur.com/oabfAzy.png)
*cat /proc/modules*

[Imgur](http://i.imgur.com/yIeN5as.png)
*cat /proc/modules*

To unload the driver, use **rmmod** command. (***rmmod myDev.ko***)

----
Testing the driver
---------------------
To test the driver, we try writing something to the device file and then reading it. For example,

[Imgur](http://i.imgur.com/dpdSmHT.png)
*Testing the driver*

See the output. (The reason for the ‘ugly’ output is because echo automatically writes a newline character to the end of string. When the driver reverses the string, the newline is shifted to the front of the string and there is no newline at the end. Hence the result being ‘ugly’)

To see how this can be done from our program, I wrote a demo program given below

[Imgur](http://i.imgur.com/PncaPdn.png)
*Interacting with the driver*

Compile it normally(or run **make test**) and run **./test  some_string**  and see the output.

[Imgur](http://i.imgur.com/PncaPdn.png)
*Testing the driver*

> **Note:**

> - You need to be root to compile the module, load the module and unload the module.
>- This driver interface presented here is an old one, there is a newer one but the old one is still supported.


