---
layout: post
title: Incremental Rotating Backups With Rsync
---

I recently needed to come up with a quick and easy and hopefully elegant way to keep incremental rotating backups of the code I was working on.
After a little research and testing I came up with the following bash script.
The magic in this is taking advantage of rsync’s link-dest option. The link-dest option allows you to specify a folder to hard link. This means that when you rsync it only pulls the difference between the link-dest version and the latest version of the code. Thus you will only ever take up disk space equal to a copy of your code plus a weeks worth of changes instead of eight copies of your code.
When you combine this with a simple rotating directory structure you have a pretty quick and easy incremental rotating backup solution.
{% highlight bash %}
    #!/bin/bash
    # This script creates a week long rotating backup of the work
    # at the PATH for the USER on the HOST that you specify.
    # Can be run by hand, but would suggest creating a cron job.

    # Vars
    # Your user name on the remote machine (e.g. bob).
    USERNAME='username'
    # The remote machines hostname (e.g. www.yoursite.com).
    HOSTNAME='hostname'
    # The path to the folder you want to back up (e.g. /home/bob)
    DIRPATH='path' 

    # Check to make sure the folders exist, if not creates them.
    /bin/mkdir -p backup.{0..7}

    # Delete the oldest backup folder.
    /bin/rm -rf backup.7

    # Shift all the backup folders up a day.
    for i in {7..1}
    do
      /bin/mv backup.$[${i}-1] backup.${i}
    done

    # Create the new backup hard linking with the previous backup.
    # This allows for the least amount of data possible to be
    # transfered while maintaining a complete backup.
    /usr/bin/rsync -a -e ssh -z --delete --link-dest=../backup.1 ${USERNAME}@${HOSTNAME}:${DIRPATH} backup.0/
{% endhighlight %}
