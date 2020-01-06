### Create a new repository on GitHub. 

- Open Git Bash
- Change the current working directory to your local project.
- Initialize the local directory as a Git repository.
  - `git init`
- Add the files in your new local repository. This stages them for the first commit.
  - `git add .`
- Commit the files that you’ve staged in your local repository.
  - `git commit -m "initial commit"``
- Copy the https url of your newly created repo
- In the Command prompt, add the URL for the remote repository where your local repository will be pushed.

  -`git remote add origin remote repository URL`

  - `git remote -v`
- If this returns with no output then do the following

  - `$git remote add origin ssh://git@example.com:1234/myRepo.git`

  - `# this will now work as expected`
  - `$git push origin master`

