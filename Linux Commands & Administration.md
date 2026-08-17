
# COMMANDS

## USER AND GROUP ADMINISTRATION

|                                                                  |                                                                                    |
| ---------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `usermod -a -G [new group] [user]`                               | add a user to a group                                                              |
| `chgrp -R [new group] [user]`                                    | change primary group of a user and reflect the ownership on all [user] directories |
| `useradd -g [group] -s  "[shell path]" -m -d [home path] [user]` | create a new user, give it a group, a shell to use, and define the home path       |
| `passwd [user]`                                                  | give/change the password of a user                                                 |

| deluser [user]                      | deletes a user                   |
| ----------------------------------- | -------------------------------- |
| `deluser --remove-all-files [user]` | deletes a user with at its files |
| `su -`                              | log in as root                   |
## PROCESS ADMINISTRATION


| kill -19 [pid] | pause/stop a process |
| -------------- | -------------------- |
|                |                      |

## SYSTEMCTL

| `systemctl list-units -type=service` | list all services |
| ------------------------------------ | ----------------- |
## Network
### SSH

| ssh user:ip | connect to another machine |
| ----------- | -------------------------- |
|             |                            |

# PATHS

| Path             | Description                                                                                            |
| ---------------- | ------------------------------------------------------------------------------------------------------ |
| `/etc/init`      | configuration on the init process                                                                      |
| `/etc/passwd`    | user information                                                                                       |
| `/etc/group`     | group information                                                                                      |
| `/etc/shadow`    | users passwords                                                                                        |
| `/etc/login.def` | configurations parameters for the shadow password suite - controls user account defaults for new users |
