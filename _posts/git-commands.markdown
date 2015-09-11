git clean -f -n

-n shows what would be done

# Revert changes to modified files.
git reset --hard

# Remove all untracked files and directories.
git clean -fd

# Revert changes in specific file

git checkout HEAD file/to/restore

## Git stash

git stash

git stash list

git stash apply
git stash apply stash@{2}
