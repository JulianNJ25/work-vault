
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

|                  |                                                                  |
| ---------------- | ---------------------------------------------------------------- |
| `kill -20 [pid]` | pause/stop a process gracefully                                  |
| `kill -19 [pid]` | pause/stop a process forcefully                                  |
| `kill -2 [pid]`  | terminate process gracefully                                     |
| `kill -l`        | list all process signals                                         |
| `ps -p [pid]`    | find a process name and details by its pid                       |
| `bf %[n]`        | bring a stopped process to the background and continue execution |
|                  |                                                                  |

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

# Git


|                                                                          |                          |
| ------------------------------------------------------------------------ | ------------------------ |
| git remote set-url origin https://github.com/USERNAME/REPOSITORY.git<br> | set a new origin to push |
|                                                                          |                          |


