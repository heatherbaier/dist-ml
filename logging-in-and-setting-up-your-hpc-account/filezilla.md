# How to get your files on the HPC
There are a wide range of tools you can use to get files onto the HPC, including command line options such as [scp](https://haydenjames.io/linux-securely-copy-files-using-scp/) or [rsync](https://www.samba.org/rsync/).  While necessary for large file transfers (i.e., binary files ~>100GB), for smaller-scale data transfers you can use the SFTP protocol coupled with the nice GUI of [filezilla](https://filezilla-project.org/), a free FTP platform.  This brief guide shows how to do just that.

# Install
FileZilla is predominantly a GUI-based way to access FTP and SFTP sites, and as such has a simple graphical installer you can download for any platform here:
    [https://filezilla-project.org/](https://filezilla-project.org/)

# Basic Use

![step1](https://user-images.githubusercontent.com/7882645/190205224-068f3893-07a9-4bf5-8165-9e61e248266e.png)

From a very high level, we seek to connect our local files (shown on the left) to some remote server, and then choose what to upload or download with the UI.  The first thing we need to do is tell filezilla where the remote server (i.e., SciClone) is located.  To do that, first click on "Configure new connection" as shown in the above image.

![step2](https://user-images.githubusercontent.com/7882645/190205264-be9bdf45-cf4b-4d09-90ca-3de70e33f77c.png)

Once clicked, you'll be shown an interface that looks something like this (though you likely will not have any sites pre-configured).  First you want to click New Site, and then name your site something you'll remember (mine is called 'sciclone' in the image).  You then need to select protocol `SFTP - SSH File Transfer Protocol`, and type in the host for sciclone (i.e., `vortex.sciclone.wm.edu`).  Under logon type select Normal, and then type in your William & Mary username and password.  Finally, click connect (next time, you'll be able to simply click connect, as the remote server location itself will be saved).  Note you may be prompted that the server's host key is unknown - this is a security check; generally, you'll want to check "Always trust this host, add this key to the cache" and then click OK.

![step3](https://user-images.githubusercontent.com/7882645/190205296-529fe01c-84dc-4349-a825-9b31f51d4b9b.png)

If everything went well, you will now see an interface that looks like the above.  Dragging files initiates uploads and downloads, and you can right-click to create new folders as you desire.  What you see on the right-hand side (the remote host) will match what you see (except hidden files that start with a '.') when you type "ls" in after you login to SciClone via command line.
