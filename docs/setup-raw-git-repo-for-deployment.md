# Use your server as a remote git repository and automate deployment with git hooks

**_sources_**:
- https://macarthur.me/posts/deploying-code-with-a-git-hook
- https://stackoverflow.com/questions/59838/how-can-i-check-if-a-directory-exists-in-a-bash-shell-script
- `$ man git`

This two-part guide will walk you through setting up a remote git repository on your server and, optionally, configure it to execute build scripts or other jobs every time you push to a specific branch.

## Create a remote repository

First, login to your server as your non-root user (I'll use one I named `charlie`).
```
$ ssh charlie@<my_server_ip>
```
In your user's home directory (`/home/charlie`), create a bare git repository with
```
$ git init --bare my-project.git
```
And that's it! You now have a fully functioning remote git repository [caveat](#a-note-on---bare)

I'll configure a local git remository (on my laptop) to push to my new remote (on my server)
```
$ git remote add server charile@<my_server_ip>:/home/charlie/my-project.git
```
Now I can push and pull from my new remote (which I named `server`) like I would from GitHub or GitLab! For example:
```
$ git push -u server main
```
### A note on `--bare`
I used the `--bare` flag When we created the git repo on our server. This creates a `bare` git repository – one without a working directory. Since a bare repository contains no working directory, you can't make changes to code inside it. This helps to prevent conflicts. It also means that we'll need to take extra steps if we want to be able to build/deploy our code when we push to our remote.

Another small note: it's customary to add the `.git` suffix to the end of bare git repository name


## Automate build and deployment jobs with git hooks
`git` exposes hooks (or events) that fire at various points during execution of various `git` commands. We'll only use the `post-receive` hook but you can read a more comprehensive description of available hooks and their behavior [here](https://www.digitalocean.com/community/tutorials/how-to-use-git-hooks-to-automate-development-and-deployment-tasks).

I'll setup a `post-receive` hook to copy my code into another location on my server (where it can be served as a static website) whenever I push to the `main` branch.

To configure the `post-receive` hook, create a file called `post-receive` inside `<my-project>/hooks`
```
$ sudo nano hooks/post-receive
```
Add the following content
```bash
#!/bin/bash

# The GIT_DIR environment variable specifies a path to use instead of the default .git for the base
# of the repository. The --git-dir command-line option also sets this value. It can be an absolute
# path or relative path to current working directory. The default is "GIT_DIR=.git".  Since we are
# in a bare git repo (with no .git directory), set it to "."
GIT_DIR=.

# Save the location we wish to use as our working tree in an environment variable called TARGET
# Since we are not at the top-level directory of the working tree (we're in a bare repo with no
# working tree), you should tell Git where the top-level of the working tree is, with the
# --work-tree=<path> option. This is where Git will recreate our projects files and folders
# and where we can execute other scripts if we wish.
TARGET=/home/$USER/my-project

# Setup a bash "while read" loop to read the arguments passed to this script by Git
while read oldrev newrev ref; do
  # Get the name of the banch that was just pushed
  BRANCH=$(git rev-parse --symbolic --abbrev-ref $ref)

  # Check if we pushed to the "main" branch
  if [[ $BRANCH == "main" ]]; then
    # Send a message to the machine that's doing the push
    echo "Push received! Deploying to branch: '${BRANCH}'"

    # Ensure the location where we wish to checkout our working tree exists
    if [[ ! -d $TARGET ]]; then
      mkdir $TARGET
    fi

    # Checkout the branch we just pushed. Since we set the GIT_WORK_TREE environment variable above
    # to /home/$USER/my-project, this will recreate our project's directory structure just as it was
    # when in our local repo when we pushed our changes. The  --force flag will throw away any local
    # modifications made in our GIT_WORK_TREE directory
    git --work-tree $TARGET checkout --force $BRANCH
  else
    echo "Not on main branch. Skipping deployment"
  fi

done

```

Finally, make the `post-reveive` hook executable. Run this command from the root of the bare git repo
```
$ sudo chmod +x hooks/post-receive
```

As a bonus. You can create this file your work laptop (or equivelant) and then use `rsync` to copy it to your new git remote
```
rsync -chv --progress hooks/post-receive charile@<my_server_ip>:/home/charlie/my-project.git/hooks/
```