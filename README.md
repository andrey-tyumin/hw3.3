#  ДЗ 3.3
1. Какой системный вызов делает команда cd? В прошлом ДЗ мы выяснили, что cd не  
является самостоятельной программой, это shell builtin, поэтому запустить  
strace непосредственно на cd не получится. Тем не менее, вы можете запустить  
strace на /bin/bash -c 'cd /tmp'.  
В этом случае вы увидите полный список системных вызовов, которые делает сам bash при  
старте. Вам нужно найти тот единственный, который относится именно к cd.  
  
Ответ:  
chdir("/tmp")  

---
2. Попробуйте использовать команду file на объекты разных типов на файловой системе.  
Например:
```
vagrant@netology1:~$ file /dev/tty  
/dev/tty: character special (5/0)  
vagrant@netology1:~$ file /dev/sda  
/dev/sda: block special (8/0)  
vagrant@netology1:~$ file /bin/bash  
/bin/bash: ELF 64-bit LSB shared object, x86-64  
```

Используя strace выясните, где находится база данных file на основании которой она  
делает свои догадки.  
Ответ:  
Последовательно ищет след. файлы с описанием:  
"/home/user_name/.magic.mgc"  
"/home/use_name/.magic"  
"/etc/magic"  
"/usr/share/misc/magic.mgc"  
В моем случае инфа о файле /dev/tty и др. была найдена в последнем по списку.  

---

3. Предположим, приложение пишет лог в текстовый файл. Этот файл оказался  
удален (deleted в lsof), однако возможности сигналом сказать приложению  
переоткрыть файлы или просто перезапустить приложение – нет. Так как  
приложение продолжает писать в удаленный файл, место на диске постепенно  
заканчивается. Основываясь на знаниях о перенаправлении потоков предложите  
способ обнуления открытого удаленного файла (чтобы освободить место на файловой системе).  
Ответ:  
echo "0" > /proc/process_id/fd/1  

---

4. Занимают ли зомби-процессы какие-то ресурсы в ОС (CPU, RAM, IO)?  
Ответ: Процесс при завершении освобождает все свои ресурсы (за исключением PID)  
и становится «зомби» — пустой записью в таблице процессов, хранящей код завершения  
для родительского процесса.  
Система уведомляет родительский процесс о завершении дочернего с помощью сигнала  
SIGCHLD. Предполагается, что после получения SIGCHLD он считает код возврата с помощью  
системного вызова wait(), после чего запись зомби будет удалена из списка процессов.  
Если родительский процесс игнорирует SIGCHLD (а он игнорируется по умолчанию), то  
зомби остаются до его завершения.

---

5. В iovisor BCC есть утилита opensnoop:
```
root@vagrant:~# dpkg -L bpfcc-tools | grep sbin/opensnoop
/usr/sbin/opensnoop-bpfcc
```
На какие файлы вы увидели вызовы группы open за первую секунду работы утилиты?  
Воспользуйтесь пакетом bpfcc-tools для Ubuntu 20.04. Дополнительные сведения по установке.  
Ответ:  
На /proc/stat  

---

6. Какой системный вызов использует uname -a? Приведите цитату из man по этому  
системному вызову, где описывается альтернативное местоположение в /proc, где  
можно узнать версию ядра и релиз ОС.  

Ответ: вызов uname.  
Part of the utsname information is also accessible via /proc/sys/kernel/{ostype, hostname, osrelease, version,   domainname}  

---

7.Чем отличается последовательность команд через ; и через && в bash? Например:  
```
root@netology1:~# test -d /tmp/some_dir; echo Hi
Hi
root@netology1:~# test -d /tmp/some_dir && echo Hi
root@netology1:~#
```

Есть ли смысл использовать в bash &&, если применить set -e?  

Ответ: команды разделенные ; будут выполнены друг за другом, не зависимо от кода возврата.  
Команды разделенные && : каждая след. команда будет выполнена при успешном завершении предыдущей.  
Как я понял из прочтения mаn set, при установленном set -e, любая команда с ненулевым кодом  
возврата вызывает завершение работы оболочки, так что смысл использовать && есть.  
 ```
 -e
    A simple command that has a non-zero exit status causes the shell to exit unless the simple command is:
            o contained in an && or || list 
            o the command immediately following if, while, or until 
            o contained in the pipeline following ! 
```

8. Из каких опций состоит режим bash set -euxo pipefail и почему его хорошо было бы 
использовать в сценариях?
```
 -e
    A simple command that has a non-zero exit status causes the shell to exit unless the simple command is:
            o contained in an && or || list 
            o the command immediately following if, while, or until 
            o contained in the pipeline following ! 
 -u
    If enabled, the shell displays an error message when it tries to expand a variable that is unset. 
 -x
    Execution trace. The shell displays each command after all expansion and before execution preceded by the expanded value of the PS4 parameter. 
  o pipefail
 	A pipeline does not complete until all components of the pipeline have completed, and the exit status of the pipeline is the value of the last command to exit with non-zero exit status, or is zero if all commands return zero exit status. 
```

Получается следующее: 
е - Завершение работы при ненулевом коде возврата простой команды(не в цикле, не в цепочке)  
u - показывать ошибки неопределенных переменных  
х - в процессе отображаются команды со всеми определенными переменными.  
o pipefail - статус последовательности команд зависит от стутуса всех команд в цепочке, т.е. если у всех 0, то   статус = 0, если какая-то команда завершилась с <> 0, то и статус такой же.  
С этими опциями удобнее искать баги при тестировании и в работе.  

---

9. Используя -o stat для ps, определите, какой наиболее часто встречающийся статус у процессов в  
системе. В man ps ознакомьтесь (/PROCESS STATE CODES) что значат дополнительные к основной  
заглавной буквы статуса процессов. Его можно не учитывать при расчете (считать S, Ss или  
Ssl равнозначными).

Самый часто встречающийся код S - процесс чего-то ждет (Interruptible sleep (waiting for an event to complete).  
Дополнителные:  
```
       For BSD formats and when the stat keyword is used, additional characters may be displayed:

               <    high-priority (not nice to other users) / с повышенным приоритетом
               N    low-priority (nice to other users)  / с пониженным приоритетом
               L    has pages locked into memory (for real-time and custom IO) / Страницы памяти заблокированы(ввод-вывод)
               s    is a session leader / родительский процесс сессии
               l    is multi-threaded (using CLONE_THREAD, like NPTL pthreads do) / многопоточный
               +    is in the foreground process group / запущен в терминале
```
---
