Lista de comandos e saídas

```bash

root@ubuntu:~$ pwd
/root
root@ubuntu:~$ cd ~
root@ubuntu:~$ ls -la
total 32
drwx------  4 root root 4096 Oct  5 12:30 .
drwxr-xr-x 22 root root 4096 Oct  5 12:29 ..
-rw-r--r--  1 root root 3234 Oct  5 12:29 .bashrc
-rw-r--r--  1 root root  161 Apr 22  2024 .profile
drwx------  2 root root 4096 Oct  5 12:28 .ssh
drwxr-xr-x  7 root root 4096 Oct  9 00:54 .theia
-rw-r--r--  1 root root  109 Oct  5 12:29 .vimrc
-rw-r--r--  1 root root  165 Oct  5 12:30 .wget-hsts
lrwxrwxrwx  1 root root    1 Oct  5 12:29 filesystem -> /
root@ubuntu:~$ ^C
root@ubuntu:~$ mkdir Projeto_A
root@ubuntu:~$ cd Projeto_A
root@ubuntu:~/Projeto_A$ mkdir documentos logs scripts
root@ubuntu:~/Projeto_A$ ls -l
total 12
drwxr-xr-x 2 root root 4096 Oct  9 01:03 documentos
drwxr-xr-x 2 root root 4096 Oct  9 01:03 logs
drwxr-xr-x 2 root root 4096 Oct  9 01:03 scripts
root@ubuntu:~/Projeto_A$ cd .
root@ubuntu:~/Projeto_A$ cd ..
root@ubuntu:~$ touch documentos/relatorio_inicial.txt
touch: cannot touch 'documentos/relatorio_inicial.txt': No such file or directory
root@ubuntu:~$ cd Projeto_A
root@ubuntu:~/Projeto_A$ touch documentos/relatorio_inicial.txt
root@ubuntu:~/Projeto_A$ #echo "Log de inicializa
root@ubuntu:~/Projeto_A$ echo "Log de inicializacao do sistema - Projeto A" > logs/sistema.log
root@ubuntu:~/Projeto_A$ cat logs/sistema.log
Log de inicializacao do sistema - Projeto A
root@ubuntu:~/Projeto_A$ cp documentos/relatorio_inicial.txt logs/relatorio_backup.txt
root@ubuntu:~/Projeto_A$ mv logs/sistem.log documentos/
mv: cannot stat 'logs/sistem.log': No such file or directory
root@ubuntu:~/Projeto_A$ mv logs/sistema.log documentos/
root@ubuntu:~/Projeto_A$ ls -l documentos
total 4
-rw-r--r-- 1 root root  0 Oct  9 01:06 relatorio_inicial.txt
-rw-r--r-- 1 root root 44 Oct  9 01:08 sistema.log
root@ubuntu:~/Projeto_A$ ls -l logs
total 0
-rw-r--r-- 1 root root 0 Oct  9 01:09 relatorio_backup.txt
root@ubuntu:~/Projeto_A$ 
