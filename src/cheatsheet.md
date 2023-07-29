# My cheatsheet

`.zshrc`, `.gitconfig` 등 내가 사용하는 설정 파일과 쉘 명령어 모음.

## .zshrc

```bash
alias gg="git lg --all"
alias gzxc="git checkout master && git pull"
alias gclear='git fetch --prune && git branch | grep -v "develop" | grep -v "master" | grep -v "main" | xargs git branch -D'
alias zxc="cd ~/Desktop/repos"
alias ll="ls -hl"
```

## .gitconfig

```bash
[user]
        name = Doowonee
        email = doowonee.com@gmail.com
[core]
        editor = vim
[alias]
        # show graph with colors
        lg = log --graph --pretty=format:\"%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%ci) %C(bold blue)<%an> %Creset\" --abbrev-commit
```

## .ssh/config

```bash
Host *
    IdentityFile /Users/doowonee/Desktop/something.pem
    StrictHostKeyChecking no
```

## Shell scripts

```bash
# remove lines from file
sed -i 1,2600445d something.log

# in macOS -i option needs backup extension
# remove line included "POST"
sed -i "" '/POST/d' wb-2022-03-11_10-46-14.log

# show files order by file size DESC
sudo du -h /var  | sort -h

# find removed file but opened
sudo lsof | grep "/var" | grep deleted

# find file with content
ack -l "Something" path/logs/

# jq
jq '.' 1.json >> 1-formatted.json

# crontab
# in crontab % = new line use `\` for escaping
17 22 * * * root rm -rf /home/logs/$(date -d 'now - 4 days' +'\%Y\%m\%d')

# stress-ng
stress-ng --cpu 4 -p 80 --timeout 12s
stress-ng --vm 3 --vm-bytes 4g -t 20s -v

# remove git branch with pattern
git branch | grep "<pattern>" | xargs git branch -D

# show longest line from file
awk 'length > max_length { max_length = length; longest_line = $0 } END { print longest_line }' ./file.log

# save stdout with stderror
cargo run -p payment --example m3_2254 |& tee run.log 

# remove terminal color
sed -r "s/\x1B\[([0-9]{1,3}(;[0-9]{1,2})?)?[mGK]//g"

# get infomation from aws instance metadata
curl -s http://169.254.169.254/latest/meta-data/instance-id

# make file size zero
truncate -s 0 /var/lib/docker/containers/**/*-json.log
```
