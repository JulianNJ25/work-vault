
# COMMANDS
## Administration

|           |                                             |
| --------- | ------------------------------------------- |
| `sudo -i` | log in as sudo and get an interactive shell |
|           |                                             |

## File administration

|                                                                |                                      |
| -------------------------------------------------------------- | ------------------------------------ |
| `chmod 4xxx [file]`                                            | set the setuid bit to a file         |
| `find [path] -name [file_name]`                                | search for a file name in a location |
| `find [path] -name [file_name] -type d`                        | search for a directory in a location |
| tar -I pigz -cf archive_name.tar.gz directory_to_compress/<br> | compress a file                      |
| tar -I pigz -xf archive_name.tar.gz<br>                        | decompress a file                    |

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

|                          |                                                                  |
| ------------------------ | ---------------------------------------------------------------- |
| `kill -20 [pid]`         | pause/stop a process gracefully                                  |
| `kill -19 [pid]`         | pause/stop a process forcefully                                  |
| `kill -2 [pid]`          | terminate process gracefully                                     |
| `kill -l`                | list all process signals                                         |
| `ps -p [pid]`            | find a process name and details by its pid                       |
| `bf %[n]`                | bring a stopped process to the background and continue execution |
| sudo lsof -i :[port]     | verify a process using a port                                    |
| sudo fuser -k [port]/tcp | kill a process using a certain port                              |

## SYSTEMCTL

| `systemctl list-units -type=service` | list all services |
| ------------------------------------ | ----------------- |
## Network
### Configuration

|            |                                   |
| ---------- | --------------------------------- |
| `dhclient` | obtain automatic IP configuration |

### SSH

| ssh user@ip                         | connect to another machine |
| ----------------------------------- | -------------------------- |
| ssh-keygen -t ed25519 -C "username" | generate an ssh key-pair   |
### SFTP

| `help`                    | get help on all available commands      |
| ------------------------- | --------------------------------------- |
| `sftp [user]@[remote-ip]` | connect to a host                       |
| `get [file]`              | download a file from the remote machine |

# PATHS

| Path             | Description                                                                                            |
| ---------------- | ------------------------------------------------------------------------------------------------------ |
| `/etc/default`   | configuration of the scripts that the init process uses                                                |
| `/etc/init`      | configuration on the init process                                                                      |
| `/etc/passwd`    | user information                                                                                       |
| `/etc/group`     | group information                                                                                      |
| `/etc/shadow`    | users passwords                                                                                        |
| `/etc/login.def` | configurations parameters for the shadow password suite - controls user account defaults for new users |
|                  |                                                                                                        |

# Docker

| docker [command] --help                                                                                                                                  | get help on a command                                      |
| -------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| docker exec -it [container-id / name]] sh                                                                                                                | shell into the container                                   |
| `docker compose up -d`                                                                                                                                   | create and start a container from a compose file           |
| `docker compose down`                                                                                                                                    | removes all running containers                             |
| `docker system prune -a`                                                                                                                                 | removes all container and images that are no longer in use |
| `docker network inspect $(docker network ls -q) --format 'Network: {{.Name}} -> Containers: {{range .Containers}}{{.Name}} ({{.IPv4Address}}), {{end}}'` | find all the networks and their related containers         |
| docker run [container] [command]                                                                                                                         | run a command inside the container                         |
| docker run -it [container]                                                                                                                               | start an interactive shell to interact with the container  |
| docker ps                                                                                                                                                | currently running containers                               |
| docker ps -a                                                                                                                                             | look at current and previosly run containers               |
| docker rm [container]                                                                                                                                    | remove containers                                          |
| docker rmi [image-id]                                                                                                                                    | remove images                                              |
| docker system prune                                                                                                                                      | remove all stopped containers                              |
| docker port [container]                                                                                                                                  | check the open ports in a container                        |
| docker stop [container]                                                                                                                                  | stop a container                                           |
| docker images                                                                                                                                            | list all current images in the docker cache                |
| docker **volume** ls                                                                                                                                     | list created volumes                                       |
| docker volume create --name [volume name]<br>                                                                                                            | create a new volume                                        |
| docker compose -f production-stack.yaml up -d<br>                                                                                                        | run a compose file with a non-standard name                |
| docker run --rm [image-name-or-id] [program/command]                                                                                                     | test a command from a docker image                         |
|                                                                                                                                                          |                                                            |

# Git


|                                                                          |                                                                                                                                                  |
| ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| git remote set-url origin https://github.com/USERNAME/REPOSITORY.git<br> | set a new origin to push                                                                                                                         |
| git checkout origin/master -- .                                          | match from another branch in case of merge, and git does not recognize changes                                                                   |
| git fetch origin                                                         | download latests commits and branchs for remote                                                                                                  |
| git rebase -i origin/branch-name                                         | updates local branch without creating unnecesarry merge commits                                                                                  |
| git push -u origin my-clean-feature --force                              | creates a new branch in origin                                                                                                                   |
| git reset HEAD~1                                                         | undoes the current commit to the previous one                                                                                                    |
| git revert HEAD^                                                         | reverts the current commit changes to the previous one, however, this does not eliminates the previous commit git history in the tree            |
| git cherry-pick [hash] [hashes...]                                       | pick up only a small commit                                                                                                                      |
| git restore --staged <\file>                                             | move a file out of the staging area                                                                                                              |
| git restore <\file>                                                      | discards all changes in a file, retores it to its previous version                                                                               |
| git rebase -i HEAD~[n]                                                   | interactive rebase with a set of commits                                                                                                         |
| git branch -f [branch] [commit]                                          | make a branch point to a different commit (you must be in a different branch than the moved branch)                                              |
| git tag [tag] [branch]                                                   | tags a commit<br>if the branch name is not specified, the tag will be created where HEAD is point at                                             |
| git describe [commit]                                                    | describe where you are relatevely to the last tag<br><br>if the commit hash is not passed, describe will use where you are checked at the moment |
| <br>git push <remote_name> --delete <.branchname> <br>                   | delete a remote branch                                                                                                                           |

# GO

|                                                                                          |                                  |
| ---------------------------------------------------------------------------------------- | -------------------------------- |
| CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-s -w" -o [binary-name] main.go | **Compile statically for Linux** |

# GITHUB

<<<<<<< HEAD
# OPENBAO


|                                        |             |
| -------------------------------------- | ----------- |
| bao list auth/approle/role             | check roles |
| bao read auth/approle/role/{role-name} | read        |

=======
```bash
# CHANGE REMOTE ORIGIN TO SSH
git remote set-url origin git@github.com:JulianNJ25/work-vault.git
```
>>>>>>> 8277d739dbdb0155ba40ce587f8eead454cdf438
