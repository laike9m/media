When I started wirting my blog, one thing that confused me is different versions of `settings.py`. As we know, different `settings.py` should be kept for developement and deployment, however when it comes to using Git, things get messy. The main difficulties lies on two things:

1. Should I upload `settings.py` from developement environment to Git server? If I did, does that mean I have to change the file every time after doing `Git pull` on deployment server?  
2. If I choose not to upload `settings.py`, there is a risk of losing data.

My purpose is to **keep different versions of `settings.py`**. Here's what I did:  

1. First, on the computer I do developing, `commit` and `push` everything as you normally do, including `settings.py`.
2. Create a new branch called `deploy` on Github.
3. Then, on deployment server, `git pull` from branch `master`.
4. **Modify `settings.py` for deployment**, e.g. set `DEBUG=False`, change `MEDIA_ROOT`, `STATIC_ROOT`, etc. The ONLY file you should change is `settings.py`, **DO NOT TOUCH OTHER FILES !**  
5. `commit` and `push` to branch `deploy`.
6. Back to developing environment, type this command:  
```bash
git update-index --assume-unchanged my_blog/settings.py
```
Done :)

<br>
The key part is step 5. First you should understand what `git update-index --assume-unchanged` is doing, basically it assumes that the file does not change. From [git-scm][g]: 

> --[no-]assume-unchanged  
> When these flags are specified, the object names recorded for the paths are not updated. Instead, these options set and unset the "assume unchanged" bit for the paths. When the "assume unchanged" bit is on, Git stops checking the working tree files for possible modifications, so you need to manually unset the bit to tell Git when you change the working tree file. This is sometimes helpful when working with a big project on a filesystem that has very slow lstat(2) system call (e.g. cifs).
> 
> This option can be also used as a coarse file-level mechanism to ignore uncommitted changes in tracked files (akin to what .gitignore does for untracked files). Git will fail (gracefully) in case it needs to modify this file in the index e.g. when merging in a commit; thus, in case the assumed-untracked file is changed upstream, you will need to handle the situation manually.

[g]:http://git-scm.com/docs/git-update-index

So next time you `commit` your work, though `settings.py` may has been changed, those changes won't get uploaded to Git server, which means there is no confict when you `pull` from master branch on deployment server no matter how many times your `settings.py` has been edited there. You could also achieve this by adding `my_blog/settings.py` to `.git/info/exclude`.

Finally, workflow is like this:

**Coding, Push to `master` —> On deployment server, pull from `master`**

**Edit `settings.py` on deployment server (if needed) —> Push to `deploy`**

Be careful, NEVER push to branch `master` from deployment server.  
I'm open to changes and suggestions, if you have better ideas, feel free to comment.