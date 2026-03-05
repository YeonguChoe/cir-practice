# Github command

## Update PR
1. Make alias for repository URL named `upstream`

```bash
git remote add upstream https://github.com/llvm/llvm-project.git
```

2. Bring latest commit history from `upstream` repository

```bash
git fetch upstream
```

3. Change Branch to PR branch

```bash
git checkout <branch>
```

4. Reflect updated code to PR branch

```bash
git rebase upstream/main
```

5. Fix Code

6. Stage fixed code

```bash
git add <file1> <file2> ...
```

- remove staged files: `git restore --staged .`

7. Make snapshot of staged files and empty staged area

```bash
git commit -m "<message>"
```

- cancel commit and move back fixed code to staged area: `git reset --soft HEAD~1`

8. push to my repository
