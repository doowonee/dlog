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
# 실행 stdout를 파일로 저장
cargo run -p payment --example m3_2254 |& tee run.log 

# remove terminal color
# 터미널 색상 코드 제거
sed -r "s/\x1B\[([0-9]{1,3}(;[0-9]{1,2})?)?[mGK]//g"

# EC2 instance ID 조회하기 AWS EC2에서만 동작
curl -s http://169.254.169.254/latest/meta-data/instance-id

# make file size zero
# 파일 내용 비우기
truncate -s 0 /var/lib/docker/containers/**/*-json.log
```

## linux disk management

```bash
sudo lsblk -f

# hot storage
sudo file -s /dev/nvme1n1
sudo mkfs -t ext4 /dev/nvme1n1
sudo mkdir /hot-data 
sudo mount /dev/nvme1n1 /hot-data
sudo chown -R ubuntu:ubuntu /hot-data

# cold storage
# 데이터 용량이 클수록 mkfs가 오래 걸린다 2TB면 1시간 걸린다는데?ㄷ
sudo file -s /dev/nvme2n1
sudo mkfs -t ext4 /dev/nvme2n1
sudo mkdir /cold-data
sudo mount /dev/nvme2n1 /cold-data
sudo chown -R ubuntu:ubuntu /cold-data

# UUID 확인
sudo blkid

# 32GB 할당 했는데 30G 사용 가능이고  256GB 할당 했는데 239G 사용 가능 무엇?
# 이유는 https://askubuntu.com/questions/79981/df-h-shows-incorrect-free-space 참고

# /etc/fstab에 아래 행 추가해서 재부팅해도 자동으로 마운트 되도록 설정
# defaults는 https://wiki.debian.org/fstab 참고
UUID=9cd1fc7b-1423-4833-909b-9f9119cd2f9e /hot-data  ext4 defaults,nofail  0  2
UUID=84464544-0f11-4968-a79b-cf675edb5000 /cold-data  ext4 defaults,nofail  0  3

# fstab 변경한 설정이 잘되나 확인 
# 참고로 현재 워킹 디렉토리에 들어 간채로 umount 하면 busy라고 안된다 다른 디렉토리로 이동 하고 해야 한다
sudo umount /cold-data && sudo umount /hot-data && sudo mount -a
```
