# Completely removing a file from git

We recently had a developer commit several large movie files into the git repository. Although this can be reverted with a simple call to ```git rm```, it doesn't entirely solve the problem. Since git is just tracking snapshots, the mp4 is still in the repository's history. Every time you initiate a ```git clone```, you will be pulling down that mp4 file. This is unnecessary bloat and should be removed. Here's a guide to completely torch a file from git so that it is as if it never existed in the first place.

## Prep Work

This is going to be a dangerous journey. You are rewriting history and irrevocably deleting data from your repository. Let's make sure to have some appropriate backups.

First, ensure that you are up to date with your remote:

```bash
> cd ~/my_large_repo
> git pull
Already up-to-date.
```

Once you're up to date take a full copy of the repo (this can also be achieved with a call to git clone, but a straight copy will make sure all your git remotes/hooks/etc. are preserved).

```bash
> cp -Rp ~/my_large_repo ~/my_large_repo.backup
```

## The Simple Option

As mentioned earlier, the easy way to remove the files from your filesystem is via a quick call to ```git rm```. For those who are newer to git, read on. For those who are experienced and just want to fully delete the files, jump to the next section.

Let's tease this out why git rm doesn't quite do the job. Here's what the filesystem looks like:

```bash
$ ls -altr videos
total 93768
-rw-r--r--  1 coldwd  staff  20581288 Nov  4 18:08 griffith.webm
-rw-r--r--  1 coldwd  staff  22814007 Nov  4 18:08 griffith.mp4
-rw-r--r--  1 coldwd  staff   2498504 Nov  4 18:08 abstract.webm
-rw-r--r--  1 coldwd  staff   2110649 Nov  4 18:08 abstract.mp4
drwxr-xr-x  8 coldwd  staff       272 Nov  4 18:08 ..
drwxr-xr-x  6 coldwd  staff       204 Nov  4 18:08 .
```

Those four files come in at just under 48MB of space on the filesystem. Let's also take note of our entire repository size:

```bash
$ du -sk my_large_repo/
251356  my_large_repo/
```

251MB. Let's try to chop this down by removing the video files with a call to ```git rm``` and also commit it using ```git commit```:

```bash
$ git rm -rf videos/*
rm 'videos/abstract.mp4'
rm 'videos/abstract.webm'
rm 'videos/griffith.mp4'
rm 'videos/griffith.webm'
$ git commit -m 'removing video assets'
```

How did this affect the repo's size?

```bash
$ du -sk my_large_repo/
204468  my_large_repo/
```

Sweet, that's what we were hoping for, the filesystem has dropped by same amount as the removed videos. We're done here, right? Not quite. Git actually still has a record of these files. All we have to do is checkout the previous SHA to bring the deleted files back to life. Here's what our git history looks like right now:

```bash
$ git log
commit a5cec74b28e5d0600e83360f3f4dad16df46d3cc
Author: Dennis Coldwell
Date:   Mon Nov 4 17:01:37 2013 -0800

    removing video assets

commit 046ebf0ddf90273090c193a8c18523ed5da1ca75
Author: Dennis Coldwell
Date:   Sun Nov 3 11:55:13 2013 -0800

    another day, another commit
```

Let's checkout the previous commit and see what happens to our repo.

```bash
$ git checkout 046ebf0ddf90273090c193a8c18523ed5da1ca75
Previous HEAD position was a5cec74... removing video assets
HEAD is now at 046ebf0... another day, another commit
```

Filesize goes back to 251MB now:

```bash
$ du -sk my_large_repo/
251356  my_large_repo/
```

So those files are still taking up space, even though you removed them.

## The Nuclear Option

There are a couple of ways to go about a full history deletion in git. The one that I prefer is `filter-branch`. The [git book](http://git-scm.com/book) refers to this technique as the *nuclear option* for good reason:

> There is another history-rewriting option that you can use if you need to rewrite a larger number of commits in some scriptable way — for instance, changing your e-mail address globally or removing a file from every commit. The command is filter-branch, and it can rewrite huge swaths of your history.[^1]

The command is powerful and is meant as a way to apply filters across the entire repository. Let's take a look at the command option syntax:

```bash
git filter-branch [--env-filter <command>] [--tree-filter <command>]
        [--index-filter <command>] [--parent-filter <command>]
        [--msg-filter <command>] [--commit-filter <command>]
        [--tag-name-filter <command>] [--subdirectory-filter <directory>]
        [--prune-empty]
        [--original <namespace>] [-d <directory>] [-f | --force]
        [--] [<rev-list options>…]
```

That's a lot of options. Here are the ones that we actually care about:

* `--index-filter`: does most of the work. It will rewrite the git index and apply the given command.
* `--prune-empty`: if there are any commits that become empty after applying the filter command, this option will prune the commit and make a cleaner history.
* `--tag-name-filter`: this option will take care of any tags that exist on your repository. We'll pass the identity command of `cat` to this option.
* `--`: indicates the end of the filter-branch options.
* `--all`: this will apply the changes to all refs.

### Removing from history

Now here's the command that will blow that sucker away from your git repo, **forever**!
```bash
git filter-branch \
    --index-filter "git rm --cached -f --ignore-unmatch videos/abstract.mp4 videos/abstract.webm videos/griffith.mp4 videos/griffith.webm" \
    --prune-empty --tag-name-filter cat -- --all
```

**Important** It's really important to give the full path to the files that you want to delete. The first time I ran filter-branch, I just gave the filenames, but git won't find a match and no changes will be made to your repository.

When you execute the command git will process *every* commit from the very first one made on your repository. You should see output like the following:

```bash
Rewrite aca02f66208ab310a30ac1bad747c44e8e8edbcf (277/403)rm 'videos/griffith.mp4'
rm 'videos/griffith.webm'
Rewrite ae0a6f00bcaa995946365486792ee5aa046f3293 (278/403)rm 'videos/griffith.mp4'
rm 'videos/griffith.webm'
Rewrite 2dde79b5bc88a0416a3d17c0879186993911ef86 (279/403)rm 'videos/abstract.mp4'
rm 'videos/abstract.webm'
rm 'videos/griffith.mp4'
rm 'videos/griffith.webm'
```

This tells us a couple of interesting things.

1. The first time one of these videos hit the repository was commit #277 (SHA: aca02f662).
2. The abstract videos didn't come into being until two commits later.
3. Git figures this out and only rewrites the files that match for each commit.

Once everything is finished, let's review the log again:

```bash
$ git log
commit 39c68f216b073e33220fdc6d658c16cad2999782
Author: Dennis Coldwell
Date:   Sun Nov 3 11:55:13 2013 -0800

    another day, another commit

commit 0b76a5e6081a75305f23675d04e561bc7bb31e85
Author: Dennis Coldwell
Date:   Sat Nov 2 17:27:53 2013 -0700

    work, work
```

Interesting! My original ```git rm``` commit a5cec74b28e5d0600e83360f3f4dad16df46d3cc is completely gone. Further, the prior commit 046ebf0ddf90273090c193a8c18523ed5da1ca75 has now been reset to 39c68f216b073e33220fdc6d658c16cad2999782. In fact *every* commit **after** the first time one of these videos hit the repository has been reindexed (commit #277/aca02f66208ab310a30ac1bad747c44e8e8edbcf).

This is exactly what we were hoping to have happen. But...the filesize hasn't dropped as dramatically as we may have hoped:

```bash
du -sk my_large_repo/
207192  my_large_repo/
```

This is because the objects will still exist in the local repository until they've been dereferenced/garbage collected. Github has a [handy reference](https://help.github.com/articles/remove-sensitive-data#cleanup-and-reclaiming-space) on cleaning up and reclaiming space. Here's a summary from that post:

```bash
$ rm -rf .git/refs/original/
$ git reflog expire --expire=now --all
$ git gc --prune=now
# Counting objects: 2437, done.
# Delta compression using up to 4 threads.
# Compressing objects: 100% (1378/1378), done.
# Writing objects: 100% (2437/2437), done.
# Total 2437 (delta 1461), reused 1802 (delta 1048)

$ git gc --aggressive --prune=now
# Counting objects: 2437, done.
# Delta compression using up to 4 threads.
# Compressing objects: 100% (2426/2426), done.
# Writing objects: 100% (2437/2437), done.
# Total 2437 (delta 1483), reused 0 (delta 0)
```

What does the filesystem look like now?

```bash
$ du -sk my_large_repo/
117004  my_large_repo/
```

**MUCH** better! We've dropped the overall repo size by more than half. We are in much better shape than just having run a ```git rm```.

## Some Helpful Links

I made use of help from the following posts:

* https://help.github.com/articles/remove-sensitive-data
* http://stackoverflow.com/questions/2100907/how-do-i-purge-a-huge-file-from-commits-in-git-history
* http://git-scm.com/book/ch6-4.html#The-Nuclear-Option%3a-filter-branch

[^1]: http://git-scm.com/book/ch6-4.html#The-Nuclear-Option%3a-filter-branch

