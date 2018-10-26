# blobfuse for Azure Mooncake

## Background
目前blobfuse只能在Global Azure上使用。使用blobfuse连接mooncake的blob container，会导致命令hang死等异常现象。此版本的blobfuse进行了定制，可以连接mooncake的blob container，但不可连global azure的blob了。

## 使用方法：
1. git clone项目到Linux VM
2. 需要编译，因此针对不同的Linux release，安装需要的依赖库：

Centos:

	sudo yum -y install epel-release
	
	sudo yum install git cmake3 fuse-devel libcurl-devel gcc gcc-c++ gnutls-devel fuse -y

Redhat:

	sudo yum install git cmake fuse-devel libcurl-devel gcc gcc-c++ gnutls-devel fuse -y

Ubuntu:

	sudo apt-get install pkg-config libfuse-dev cmake libcurl4-gnutls-dev libgnutls28-dev -y 

3. 编译：

	cd azure-storage-fuse/
	
	./build.sh
	
看到以下提示，表示编译成功：

	Linking CXX executable blobfuse

	[100%] Built target blobfuse

4. 把编译好的blobfuse放到path路径下

编译好的blobfuse可执行文件在编译输出的build目录下，把其下的blobfuse可执行文件copy到/usr/bin下。在任意目录中执行blobfuse，看到以下提示表示blobfuse安装就绪：

	Usage: blobfuse <mount-folder> --config-file=<config-file> --tmp-path=<temp-path> [--use-https=false] [--file-cache-timeout-in-seconds=120]

	Please see https://github.com/Azure/azure-storage-fuse for installation and configuration instructions.

5. 准备blobfuse连接的配置文件，内容类似如下：

	accountName azstorage001

	accountKey BWG2dfUuMAIQX+VNnslPgVGLR7zQ8KI6dv5B0lMwLQEgSjr4kGc3+FCexF9wKQ15AGjWtoT3eyS4t0T5Z8qpIA==

	containerName container01

6. 执行连接

	blobfuse <mount_folder> --config-file=<connection_config_file> --tmp-path=<blobfuse_cache_folder>
 
7. 检查连接成功

$ df -h

	Filesystem      Size  Used Avail Use% Mounted on

	/dev/sda2        30G   16G   14G  55% /

	devtmpfs        3.4G     0  3.4G   0% /dev

	tmpfs           3.4G   33M  3.4G   1% /run

	/dev/sda1       497M  105M  393M  22% /boot

	**/dev/sdb1        14G   41M   13G   1% /mnt/resource**   <-- cache folder

	**blobfuse         14G   41M   13G   1% /home/caikai/bmount**  <-- mount folder





/*******************************************************************************

* below are the README.md info of original repository                          *

*******************************************************************************/
## About

blobfuse is an open source project developed to provide a virtual filesystem backed by the Azure Blob storage. It uses the [libfuse](https://github.com/libfuse/libfuse) open source library to communicate with the Linux FUSE kernel module, and implements the filesystem operations using the Azure Storage Blob REST APIs.

Please note that this tool is currently in Preview, and need your feedback for improvements. Please submit an issue [here](https://github.com/azure/azure-storage-fuse/issues) for any issues/requests/questions.

## Features
- Mount a Blob storage container on Linux
- Basic file system operations such as mkdir, opendir, readdir, rmdir, open, read, create, write, close, unlink, truncate, stat, rename
- Local cache to improve subsequent access times
- Parallel download and upload features for fast access to large blobs
- Allows multiple nodes to mount the same container for read-only scenarios.

## Considerations
Please take careful note of the following points, before using blobfuse:
- In order to achieve reasonable performance, blobfuse requires a temp directory to use as a local cache. This directory will contain the full contents of any file (blob) read to or written from through blobfuse, and will continue to grow as long as blobfuse is running. You must ensure you have enough free space in this directory.
  - If space is not a concern, putting this directory on a SSD will greatly enhance performance.
  - In order to delete the cache, un-mount and re-mount blobfuse.
  - Do not use the same temp directory for multiple instances of blobfuse, or for any other purpose while blobfuse is running.
  
### If your workload is read-only:
- If the temp directory gets too large, it is fine to manually delete it.  Un-mounting and re-mounting blobfuse will also clear the temp directory.
- Because blobs get cached locally, if the blob on the service is modified, these changes may or may not be reflected when accessing the blob through blobfuse.  The cache will periodically refresh, but this is done on a best-effort basis.

### If your workload is not read-only:
- Do not edit, modify, or delete the contents of the temp directory while blobfuse is mounted. Doing so could cause data loss or data corruption.
- While a container is mounted, the data in the container should not be modified by any process other than blobfuse.  This includes other instances of blobfuse, running on this or other machines.  Doing so could cause data loss or data corruption.  Mounting other containers is fine.
- Modifications to files are not persisted to Azure Blob storage until the file is closed. If multiple handles are open to a file simultaneously, and data in the file has been modified, the close of each handle will flush the file to blob storage. 

## Current Limitations
- Some file system APIs have not been implemented: readlink, symlink, link, chmod, chown, fsync, lock and extended attribute calls.
- Not optimized for updating an existing file. blobfuse downloads the entire file to local cache to be able to modify and update the file
- Does not support SAS yet
- High latency compared to local filesystems. Further you are from the Azure region, higher the latency is.
- Currently does not implement data integrity checks. Always have your data backed up before using blobfuse. Use Https as a data integrity mechanism over the wire.
- Because Azure Block Blobs do not have file and directory semantics, certain directory operations may not behave entirely as expected. For example, deleting the last file in a directory may also delete the directory.

## Installation

In order to invoke blobfuse, you will need to install the dependencies (libfuse, libcurl, GnuTLS and cmake), clone this repository and build. The process is explained in the [blobfuse installation](https://github.com/Azure/azure-storage-fuse/wiki/Installation) page.

## Usage

### Mounting
Once you have built blobfuse, configure your account credentials in the template provided in blobfuse folder (connection.cfg).

Here is the format for connection.cfg:
```
accountName <account-name-here> 
accountKey <account-key-here> 
containerName <container-name-here>
```

By default, blobfuse will use the ephemeral disks in Azure VMs as the local cache (/mnt/blobfusetmp). Please make sure that your user has write access to this location. If not, create and `chown` to your user.

```
mkdir -p /mnt/blobfusetmp
chown <myuser> /mnt/blobfusetmp
```

Now you can mount using the provided mount script (mount.sh):
```
./mount.sh </path/to/mount>
```

### Mount Options
- You can modify the default FUSE options in mount.sh file. All options for FUSE is described in the [FUSE man page](http://manpages.ubuntu.com/manpages/xenial/man8/mount.fuse.8.html)
- In addition to the FUSE kernel module options; blobfuse offers following options:
	* --config-path=/path/to/connection.cfg : Configures the path for the file where the account credentials are provided
	* --tmp-path=/path/to/cache : Configures the tmp location for the cache. Always configure the fastest disk (SSD) for best performance. Note that the files in this directory are not purged automatically.
	* --use-https=true/false : Enables HTTPS communication with Blob storage. False by defaul. Enable it to protect against data corruption over the wire.
	* --file-cache-timeout-in-seconds=120 : Blobs will be cached in the temp folder for this many seconds. 120 seconds by default. During this time, blobfuse will not check whether the file is up to date or not.
	

### Notes
- blobfuse is currently in Preview. Expect significant performance (and stability) improvements in the upcoming months.
- Performance is varied with your setup. Specifically latency plays an important role in filesystems. Expect better performance within an Azure region (Azure VM and Azure Storage account in the same region)
- Rename operation on directories is not an atomic operation. blobfuse iterates over the entire directory and issue [Copy Blob API](https://docs.microsoft.com/en-us/rest/api/storageservices/copy-blob) one by one, hence expect poor performance for rename operations on directories.
- Files you open in blobfuse mount get stored in the local cache, and only persisted to Blob storage after a 'close' is called. 

## License
This project is licensed under MIT.
 
## Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.microsoft.com.

When you submit a pull request, a CLA-bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., label, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

