# Git

## Config

```sh
# user setup
git config --global user.email firstname.lastname@example.com
git config --global user.name 'Matthew Dowdell'

# url aliases
git config --global url.git@github.com.insteadof gh
git config --global url.git@github.com:.insteadof https://github.com/

# command aliases
git config --global alias.co checkout
git config --global alias.st status

# pull config
git config --global pull.rebase false
git confif --global push.autosetupremote true
```

## Empty commit

```sh
git commit --allow-empty -m "message"
```

## Modify git commit date

```sh
GIT_COMMITTER_DATE="$(date)" git commit --amend --no-edit --date "$(date)
```

## Soft reset commits on branch

```sh
# merge in default branch
git pull origin main

# soft reset all branch-specific commits
git reset $(git merge-base origin/main $(git branch --show-current))
```
