For review



1. Select an Issue to work on
  * If there isn't an issue, make one
2. (desktop) switch to `master` branch in `medley` repo
  * see notes below
  * Command line: `git pull` `git checkout master`
3. Make a new branch. Call it `issue-`NNN where NNN is the issue number
   * Command line: `git branch -b issue-NNN`
4. _make edits starting with master's full_
5. Commit, adding changed files with an explanation of what you changed and why. Don't rely on seeing deltas
   * Command line: `git add file1 file2 ...` `git commit -m "(explanation here)"
   * Don't commit new loadups
6. push commits to GitHub branch
   * Command line `git push`
7. Make a pull request
   * Command line: `gh pr create`
   * Can also be done on the GitHub web site
   * If you fixed the issue, include "fixes `#`NNN" in the PR explanation. Otherwise include "See `#`NNN".

Notes:
* If you're working in a branch and discover a file you want to commit independently, then
1. Commit current branch
2. *  switch to master branch, Command `git checkout master`
3. checkout the file into this new branch `git checkout BRANCHNAME -- file file.LCOM file.TEDIT`
4. TBA
* Steps 4-6 can be repeated
* If you need previous fixes to be there, then at step 1, either
   * Wait for them to be merged
   * start with previous PR instead of master
