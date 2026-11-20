# 00 – Linux basics

New to Linux? These are the commands you need to follow this series. All examples work on any Ubuntu server (for example, an AWS EC2 instance).

> Tip: don't try to memorise this page. Keep it open while you follow the episodes, and come back when you need a command.

---

## 1. Connect to a server

```shell
chmod 400 my-key.pem                       # SSH refuses a key file that others can read
ssh -i my-key.pem ubuntu@<SERVER-IP>       # on Ubuntu EC2, the default user is "ubuntu"
exit                                       # disconnect
```

Admin commands need `sudo`:

```shell
sudo apt update     # run one command as root (the admin user)
sudo -i             # become root until you type: exit
```

## 2. Move around

```shell
pwd                 # where am I?
ls                  # what is here?
ls -l               # long list: permissions, owner, size, date
ls -la              # also show hidden files (names starting with .)
cd /var/log         # go to a folder (absolute path, starts with /)
cd ..               # one folder up
cd ~                # your home folder
cd -                # back to the previous folder
```

Important folders:

| Folder | What is inside |
|---|---|
| `/etc` | configuration files |
| `/var/log` | log files |
| `/home/<user>` | each user's personal folder |
| `/root` | the root user's home folder |
| `/tmp` | temporary files, cleared on reboot |
| `/opt` | extra software you install yourself |
| `/usr/bin` | programs (commands) |

## 3. Files and folders

```shell
mkdir practice                 # create a folder
mkdir -p a/b/c                 # create nested folders in one go
touch notes.txt                # create an empty file
cp notes.txt copy.txt          # copy a file
cp -r practice/ backup/        # copy a folder (-r = everything inside)
mv copy.txt renamed.txt        # move or rename
rm renamed.txt                 # delete a file (no recycle bin!)
rm -r backup/                  # delete a folder and everything inside – double-check first
```

Read files:

```shell
cat notes.txt               # print the whole file
less /var/log/syslog        # scroll through a long file (q to quit, /word to search)
head -20 /var/log/syslog    # first 20 lines
tail -20 /var/log/syslog    # last 20 lines
tail -f /var/log/syslog     # keep showing new lines as they arrive (Ctrl+C to stop)
```

## 4. Edit files with vim (survival guide)

```shell
vim notes.txt
```

| Key | What it does |
|---|---|
| `i` | start typing (insert mode) |
| `Esc` | stop typing |
| `:wq` then Enter | save and quit |
| `:q!` then Enter | quit **without** saving |
| `/word` then Enter | search for "word" |

Prefer something simpler? Use `nano notes.txt` (Ctrl+O to save, Ctrl+X to quit).

## 5. Find things

```shell
grep "error" /var/log/syslog          # lines that contain "error"
grep -i "error" /var/log/syslog       # same, ignoring upper/lower case
grep -rn "hello" practice/            # search every file in a folder, with line numbers
find /home -name "*.txt"              # find files by name
find /var/log -name "*.log" -mtime +7 # .log files not changed for more than 7 days
```

## 6. Users and groups

```shell
whoami                                   # which user am I?
id                                       # my user ID and groups
sudo useradd -m -s /bin/bash devuser     # create a user with a home folder and bash
sudo passwd devuser                      # set its password
sudo usermod -aG sudo devuser            # add to a group (-a = append, keep existing groups)
su - devuser                             # switch to that user (exit to come back)
sudo -u devuser whoami                   # run one command as another user
sudo userdel -r devuser                  # delete the user and its home folder
cat /etc/passwd                          # list of all users
cat /etc/group                           # list of all groups
```

## 7. Permissions

`ls -l` shows something like `-rwxr-x--- 1 ubuntu ubuntu ... script.sh`:

```
-   rwx    r-x    ---
|   owner  group  others
file type (- = file, d = folder)
```

`r` = read (4), `w` = write (2), `x` = execute (1). Add them up for each part:

```shell
chmod 755 script.sh      # owner: rwx (7), group and others: r-x (5) – typical for scripts
chmod 644 notes.txt      # owner: rw- (6), others: r-- (4) – typical for normal files
chmod 600 secrets.txt    # only the owner can read and write
chmod 400 my-key.pem     # only the owner can read – SSH keys
chmod +x script.sh       # just add "execute"
sudo chown devuser:devuser notes.txt    # change owner and group
sudo chown -R devuser:devuser practice/ # change it for a folder and everything inside
```

## 8. Install software (apt)

```shell
sudo apt update              # refresh the list of available packages – always do this first
sudo apt install -y tree     # install a package (-y = don't ask "are you sure?")
tree practice/               # try it
sudo apt remove -y tree      # uninstall
dpkg -l | grep tree          # is it installed?
```

## 9. Services (systemctl)

A service is a program that runs in the background. `cron` (the task scheduler) is on every Ubuntu server, so practise with it:

```shell
systemctl status cron            # is it running? (q to quit)
sudo systemctl stop cron         # stop now
sudo systemctl start cron        # start now
sudo systemctl restart cron      # stop + start – needed after changing its config
sudo systemctl enable cron       # start automatically after a reboot
sudo systemctl disable cron      # don't start after a reboot
sudo systemctl enable --now cron # enable + start, in one command
sudo systemctl daemon-reload     # after creating or editing a .service file
```

`enable` does **not** start a service, and `start` does **not** survive a reboot. That's why `enable --now` is so common.

## 10. Logs

```shell
journalctl -u cron -n 50         # last 50 log lines of one service
journalctl -u cron -f            # follow new lines live (Ctrl+C to stop)
sudo tail -f /var/log/syslog     # general system log
sudo tail -f /var/log/auth.log   # logins and sudo usage
```

When something fails, the log almost always tells you why. Read the error before changing anything.

## 11. Processes and ports

```shell
ps aux                     # every running process
ps aux | grep cron         # find one process
top                        # live CPU and memory usage (q to quit)
kill <PID>                 # stop a process by its ID (PID = the number in ps aux)
sudo ss -tlnp              # which ports are open, and which program owns each
curl -I https://example.com   # call a website and show only the response headers
```

## 12. Disk and memory

```shell
df -h                   # free disk space (-h = human sizes: G, M)
du -sh /var/log/*       # size of each item in a folder
free -h                 # memory usage
```

## 13. Networking

```shell
ip a                    # this server's IP addresses
curl -s ifconfig.me     # this server's public IP
ping -c 3 google.com    # can I reach the internet? (-c 3 = three tries)
nslookup google.com     # which IP does a name point to?
```

On AWS, servers in the same network (VPC) talk to each other over **private IPs**. Your laptop reaches a server through its **public IP**. A public IP changes when you stop and start an instance; a private IP does not.

## 14. Shell basics we use in scripts

```shell
NAME="world"                    # a variable (no spaces around =)
echo "Hello $NAME"              # use it
echo $?                         # exit code of the last command: 0 = success, anything else = failure
mkdir test && cd test           # run the second command only if the first succeeded
cd /nowhere || echo "failed"    # run the echo only if the first command failed
ls /etc | grep host             # | sends the output of one command into the next
echo "line 1" > file.txt        # write to a file (overwrites it)
echo "line 2" >> file.txt       # append to a file
sudo apt install -y tree &>> install.log   # append output AND errors to a log file
```

## 15. Help and shortcuts

```shell
man ls              # the manual for a command (q to quit)
ls --help           # short help
history             # commands you ran before
clear               # clean the screen
```

Keys: `Tab` completes names, `↑` shows the previous command, `Ctrl+C` stops a command, `Ctrl+R` searches your history.
