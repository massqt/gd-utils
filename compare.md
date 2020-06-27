# Compare the speed difference of this tool and other similar tools in server side copy

Take copy [https://drive.google.com/drive/folders/1W9gf3ReGUboJUah-7XDg5jKXKl5XwQQ3](https://drive.google.com/drive/folders/1W9gf3ReGUboJUah-7XDg5jKXKl5XwQQ3) as an example ([File Statistics](https: //gdurl.viegg.com/api/gdrive/count?fid=1W9gf3ReGUboJUah-7XDg5jKXKl5XwQQ3))
A total of 242 files and 26 folders

Unless otherwise specified, the following operating environment is on the local command line (hang agent)

## This tool takes 40 seconds
<!-- ![](https://viegg.oss-cn-shenzhen.aliyuncs.com/1592732262296.png) -->
![](static/gdurl.png)

In addition, I executed the same command on a Los Angeles vps, which took 23 seconds.
This speed is derived from the default configuration of this project **20 parallel requests**, this value can be modified by yourself (there are methods below), the greater the number of parallel requests, the faster the total speed.

## AutoRclone takes 4 minutes and 57 seconds (4 minutes and 6 seconds to verify after removing the copy)
<!-- ![](https://viegg.oss-cn-shenzhen.aliyuncs.com/1592732547295.png) -->
![](static/autorclone.png)

## gclone takes 3 minutes and 7 seconds
<!-- ![](https://viegg.oss-cn-shenzhen.aliyuncs.com/1592732597593.png) -->
![](static/gclone.png)

## Why is there such a big difference in speed
First of all, it is necessary to clarify the principle of server side copy (hereinafter referred to as ssc).

As far as Google Drive itself is concerned, it won’t really copy it on its own file system because you ssc copied a file (otherwise no matter how big the hard drive will be filled), it just adds it to the database A record.

Therefore, no matter whether ssc is a large file or a small file, in theory, it takes the same time.
When you use these tools, you can also feel that copying a bunch of small files is much slower than copying a few large files.

The official Google Drive API only provides the function of copying a single file, and cannot directly copy the entire folder. You can't even read the entire folder, you can only read the first-level subfile (folder) information of a folder, similar to the `ls` command in the Linux command line.

The ssc functions of these three tools are essentially calls to [official file copy api] (https://developers.google.com/drive/api/v3/reference/files/copy).

Then talk about the principle of this tool, the approximate steps are as follows:

-First, it will recursively read the information of all files and folders in the directory to be copied and save it locally.
-Then, filter out all the folder objects, and then create a new folder with the same name according to the parent-child relationship, and restore the original structure. (Keeping the original folder structure unchanged while ensuring speed, it really took a lot of work)
-According to the correspondence between the old and new folder IDs left when creating the folder in the previous step, call the official API to copy the files.

Thanks to the existence of the local database, it can continue execution from the breakpoint after the task is interrupted. For example, after the user presses `ctrl+c`, he can execute the same copy command again. The tool will give three options:
<!-- ![](https://viegg.oss-cn-shenzhen.aliyuncs.com/1592735608511.png) -->
![](static/choose.png)

The other two tools also support breakpoint resume, how do they do it? AutoRclone is a layer of encapsulation of rclone command with python, gclone is a magic change based on rclone.
By the way-it is worth mentioning that-this tool is the official API called directly, does not depend on rclone.

I haven't read the source code of rclone carefully, but I can probably guess its working principle from its execution log.
First add a little background knowledge: for all the files (folders) objects that exist in Google drive, their lives are accompanied by a unique ID, even if one file is a copy of another, their ID is also different.

So how does rclone know which files have been copied and which ones have not? If it does not save the records in the local database like me, then it can only search for the existence of files with the same name in the same path, and if so, compare their size/modification time/md5 value to determine whether they have been copied.

That is to say, in the worst case (assuming that it is not cached), before copying a file, it must first call the official API to search and determine whether the file already exists!

In addition, although AutoRclone and gclone both support automatic switching of service accounts, but they perform a copy task when a single SA is calling the API, which is destined that they cannot adjust the request frequency too high-otherwise it may trigger a limit.

The tool also supports automatic switching of service accounts, the difference is that each request is randomly selected an SA, my [file statistics] (https://gdurl.viegg.com/api/gdrive/count?fid=1W9gf3ReGUboJUah -7XDg5jKXKl5XwQQ3) The interface uses 20 SA tokens, and the number of requests is set to 20, that is, on average, the number of concurrent requests for a single SA is only once.

So the bottleneck is not the frequency limit of SA, but on the running vps or proxy, you can adjust the value of PARALLEL_LIMIT appropriately (in `config.js`) according to your situation.

Of course, if the daily traffic of a certain SA exceeds 750G, it will automatically switch to another SA and filter out the SA with exhausted traffic. When all SA traffic is used up, it will switch to a personal access token until the traffic is also used up, and the process eventually exits.

*Restrictions for using SA: In addition to the daily traffic limit, in fact, each SA has a **15G personal disk space limit**, which means that you can copy up to 15G files to your personal disk per SA, but There is no such limitation when copying to team disk. *
