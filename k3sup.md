# k3sup to install k3s

## prepare

## install server

## install worker

## FAQ

### ssh problem

```text
Hello. This kind of problem often occurs with jenkins. There is a simple solution:
You need to create a file myuser, you can read in more detail here:
cat /etc/sudoers.d/README

Place the 0440 mode file in /etc/sudoers.d/myuser with the following content:
myuser ALL = (ALL) NOPASSWD: ALL
and don't forget to chmod 0440 /etc/sudoers.d/myuser
```