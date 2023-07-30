# Shell

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
# 파일 특정 라인 삭제
sed -i 1,2600445d something.log

# in macOS -i option needs backup extension
# remove line included "POST"
# 파일중 특정 단어가 포함된 라인 삭제
sed -i "" '/POST/d' wb-2022-03-11_10-46-14.log

# show files order by file size DESC
# 파일 크기 큰 순서대로 보기
sudo du -h /var  | sort -h

# find removed file but opened
# 삭제된 파일중 프로세스가 접근 중인 파일 찾기
sudo lsof | grep "/var" | grep deleted

# find file with content
# 파일 내용으로 파일 찾기
ack -l "Something" path/logs/

# jq
# JSON 유틸 jq
jq '.' 1.json >> 1-formatted.json

# crontab
# in crontab % = new line use `\` for escaping
# 크론 탭 사용법 % 문자는 개행 취급이라 \로 이스케이프 필요
17 22 * * * root rm -rf /home/logs/$(date -d 'now - 4 days' +'\%Y\%m\%d')

# stress-ng
# 스트레스 테스트 툴
stress-ng --cpu 4 -p 80 --timeout 12s
stress-ng --vm 3 --vm-bytes 4g -t 20s -v

# remove git branch with pattern
# 특정 git 브렌치 삭제
git branch | grep "<pattern>" | xargs git branch -D

# show longest line from file
# 파일에서 가장 긴 라인 보기
awk 'length > max_length { max_length = length; longest_line = $0 } END { print longest_line }' ./file.log

# save stdout with stderror
# 실행 결좌를 파일로 저장
cargo run -p payment --example m3_2254 |& tee run.log 

# remove terminal color
# 터미널 색상 코드 제거
sed -r "s/\x1B\[([0-9]{1,3}(;[0-9]{1,2})?)?[mGK]//g"

# get infomation from aws instance metadata
# EC2 정보 읽어오기
curl -s http://169.254.169.254/latest/meta-data/instance-id

# make file size zero
# 파일 내용 비우기
truncate -s 0 /var/lib/docker/containers/**/*-json.log
```
