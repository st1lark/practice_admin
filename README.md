1. **Как можно перевести какой-нибудь процесс в linux в статус R, S, T, D, Z. (В ответе требуется: описать команды с помощью которых это можно осуществить. Команды должны быть воспроизводимы. Если состояние какими-либо командами невозможно получить (или очень трудно), то можно описать - при каких условиях оно возникает)**

   R - этим статусом обозначатся процесс, который сейчас запущен 

   *S- этим статусом обозначается процесс, которые ожидает какого-то события.*

   По первому и второму статусу не могу назвать специальных действий, которые необходимо проделать, чтобы привести процесс к ним. 

   *T - процесс был остановлен по сигналу. Перевести процесс в данное состояние можно отправив ему 19-ый сигнал (SIGSTOP).*

   Пример:

   Запущен процесс `./process`

   ```bash
   root@stlark:~# ps aux | grep ./process 
   root     13654  0.0  0.3  30428  2956 ?        Ss   17:52   0:00 SCREEN -s ./process
   root     13655  0.0  0.1   4516  1580 pts/4    Ss+  17:52   0:00 ./process
   root     13709  0.0  0.1  13140  1040 pts/3    S+   17:54   0:00 grep --color=auto ./process
   root@stlark:~# 
   ```

   Отправляем сигнал SIGSTOP и видим, что процесс остановлен:

   ```bash
   root@stlark:~# kill -19 13655
   root@stlark:~# ps aux | grep ./process 
   root     13654  0.0  0.3  30428  2956 ?        Ss   17:52   0:00 SCREEN -s ./process
   root     13655  0.0  0.1   4516  1580 pts/4    Ts+  17:52   0:00 ./process
   root     13774  0.0  0.1  13140  1036 pts/3    S+   17:57   0:00 grep --color=auto ./process
   root@stlark:~# 
   ```

   

   *D - процесс в состоянии ожидания ввода/вывода. Процесс с данным статусом игнорирует сигналы, которые ему отправляются, сами же сигналы встают в очередь.*

   Привести процесс к данном состоянию можно следующим образом:

   ***Первый вариант:***

   1. С помощью sshfs монтируем директорию c удаленного сервера на свой хост:

      ```bash
      sshfs root@45.90.33.31:/root/altdocker /root/sshfs
      ```

   2. На стороне удаленного сервера прописываем правило `iptables`

      ```bash
      iptables -A INPUT -p tcp -s 193.168.46.148 -j DROP
      ```

   3. На стороне сервера `193.168.46.148`, например, выполняем команду `ls`, которая повиснет в ожидании:

      ```bash
      root@stlark:~/sshfs# ls
      
      ```

   4. Смотрим текущее состояние данного процесса:

      ```
      root@stlark:~# ps a | grep ls
      16266 pts/2    S+     0:00 ls --color=auto
      16284 pts/5    S+     0:00 grep --color=auto ls
      root@stlark:~# 
      ```

   5.  Чтобы перевести данный процесс в состояние `D`  отправяем ему, наример, сигнал SIGTERM:

      ```bash
      root@stlark:~# kill 16266
      root@stlark:~# ps a | grep ls
      16266 pts/2    D+     0:00 ls --color=auto
      16330 pts/5    S+     0:00 grep --color=auto ls
      root@stlark:~# 
      ```

   ***Второй вариант переведения процесса в статус `D`***

   1. Создаем директорию `group1` в папке `/sys/fs/cgroup/freezer`

   2. В файлу `/sys/fs/cgroup/freezer/group1/freezer.state` передаем значение `FROZEN`

   3. В файл `/sys/fs/cgroup/freezer/group1/tasks`передаем  PID процесса, который хотим "заморозить". В мое случае это PID bash, запущенного в соседней консоли

   4. Проверяем статус процесса:

      ```bash
      root@stlark:/sys/fs/cgroup/freezer/group1# ps a | grep bash
      16205 pts/0    Ds+    0:00 -bash
      16681 pts/2    Ss     0:00 -bash
      17214 pts/2    S+     0:00 grep --color=auto bash
      root@stlark:/sys/fs/cgroup/freezer/group1# 
      ```

      

   Z - статус дочернего процесса, который завершился, но не был удален из списка процессов. В такое состояние процесс можно привести, например, отправкой SIGSTOP родительскому процессу и отправкой сигнала. Приведу пример с процессами Apache. 

   Нормальное состояние процессов:

   ![](/home/stanislav/Загрузки/apache1.png)

   Приведение одного из дочерних процессов в состояние zombie:

   ![](/home/stanislav/Загрузки/apache2.png)

2. **Напишите правило в iptables для логирования исходящих запросов на порт 3306.**

   ```
   iptables -A OUTPUT -p tcp --dport 3306 -j LOG
   ```

3. **Напишите bash скрипт, которые на logstorage обходит директории серверов хостинга и парсит exim логи за предыдущий день. (учтите - день, месяц и год в момент запуска скрипта может быть разным). Результатом работы скрипта должен быть вывод отправителей, письма с которых за предыдущий день отклонялись более 400 раз.**

   Написал два варианта скрипта.

   Первый вариант (четко слеудет задаче), увидеть его можно также по адресу http://193.168.46.148/search.sh:

   ```bash
   #!/bin/bash
   
   dir_log=/home/beget/support/log/`date -d 'yesterday' +%Y.%m`
   
   servers=`find $dir_log -type d -name exim 2> /dev/null | awk -F/ '{print $7}'`
   
   IFS=$'\n'
   
   for server in $servers
   do
   echo $server
   rej_senders=`rg '\*\*' $dir_log/$server/exim/$(date -d 'yesterday' +%d).log.gz | rg -o " F=<.+@.+> " | awk '{print $1}' | sort | uniq -c`
   
           for rej_sender in $rej_senders
           do
                   count=`echo $rej_sender | awk '{print $1}'`
   
                   if [ $count -ge 400 ]
                   then
                           echo " $rej_sender "
                   fi
           done
   done
   ```

   Запуск можно произвести командой:

   ````
   curl 193.168.46.148/search.sh 2> /dev/null | nice -n 19 ionice -c 3 bash
   ````

   

   Второй вариант, не так сильно отличается от первого, но добавляет немного интерактивности в виде выбора сервера и количества отклоненных писем (доступен по ссылке http://193.168.46.148/search_arr.sh ):

   ```bash
   #!/bin/bash
   
   function is_number() {
   	if [ -z `echo $1 | grep -E '(^[1-9][0-9]+$)'`  ]
   	then
   		echo 0
   	else
   		echo 1
   	fi
   }
   
   dir_log=/home/beget/support/log/`date -d 'yesterday' +%Y.%m`
   
   read < /dev/tty -p "Enter servers separated by a space[default all]: " -a servers
   
   ##Checking servers
   if [[ ${#servers[@]} -eq 0 || ${servers[0]} == 'all' ]]
   then	
   	servers=( $(find $dir_log -type d -name exim 2> /dev/null | awk -F/ '{print $7}') )
   fi
   
   read < /dev/tty -p "Enter count reject mails[default 400]: " counts
   
   
   ##Сhecking counts 
   if [[ $(is_number "$counts") -eq 0 && -n $counts ]]
   then
   	echo "Enter number, not string"
   	exit 0
   elif [ -z $counts ] 
   then
   	counts=400
   fi
   
   ##Print rej_senders
   IFS=$'\n'
   
   for server in ${servers[@]}
   do
   	echo $server
   	rej_senders=`rg '\*\*' $dir_log/$server/exim/$(date -d 'yesterday' +%d).log.gz | rg -o " F=<.+@.+> " | awk '{print $1}' | sort | uniq -c`
   
   	for rej_sender in $rej_senders
   	do
   		count=`echo $rej_sender | awk '{print $1}'`
   		
   		if [ $count -ge $counts ]
   		then 
   			echo " $rej_sender "
   		fi
   	done
   done
   ```

   Запуск можно произвести командой:

   ````
   curl 193.168.46.148/search_arr.sh 2> /dev/null | nice -n 19 ionice -c 3 bash
   ````

   

4. **Можно ли вывести какой-нибудь текст в другую консоль на том же хосте? Если да, то как?  Продемонстрируйте это, запустив команду `sleep infinity | grep 123` и в grep отправив `123`. (можно сделать как минимум 2-мя способами, для ответа достаточно указать хотя бы один)**

   Общение между консолями между двумя хостами можно выполнить, например, через утилиту `write` и через перенаправелние вывод в `/dev/pts/<номер>`. 

   Для демонстрации же я нашел один вариант решения задачи, это перенаправление вывод в файловый дескриптор потока ввода процесса `grep`:

   ![](/home/stanislav/Изображения/redir.png)

   

5. **Поднять второй докер-демон https://github.com/mlosev/altdocker и запустить на нем контейнер с adminer-ом. Сделать так, чтобы можно было открыть web-интерфейс adminer-a введя http://localhost:8080 в браузере. Подключиться из него к базе на хостинге (предварительно создав ее на тестовом аккаунте).**

   Докер разворачивл на Ubuntu 18.04. 

   Прикладываю команды и их вывод при развертывании altdocker и запуске контейнера с `adminer`:

   ````bash
   root@stlark:~# git clone https://github.com/mlosev/altdocker.git
   Cloning into 'altdocker'...
   remote: Enumerating objects: 22, done.
   remote: Total 22 (delta 0), reused 0 (delta 0), pack-reused 22
   Unpacking objects: 100% (22/22), done.
   root@stlark:~# cd altdocker/
   root@stlark:~/altdocker# ./get_docker.sh 
   + DOCKER_VERSION=1.12.1
   + '[' '!' -s docker-1.12.1.tgz ']'
   + curl -o docker-1.12.1.tgz https://get.docker.com/builds/Linux/x86_64/docker-1.12.1.tgz
     % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed
   100 27.5M  100 27.5M    0     0  13.1M      0  0:00:02  0:00:02 --:--:-- 13.1M
   + tar -xzvf docker-1.12.1.tgz
   docker/
   docker/docker-containerd-ctr
   docker/docker
   docker/docker-containerd
   docker/dockerd
   docker/docker-proxy
   docker/docker-runc
   docker/docker-containerd-shim
   root@stlark:~/altdocker# screen -s ./dae1
   [detached from 12748.pts-2.stlark]
   root@stlark:~/altdocker# screen -ls
   There is a screen on:
   	12748.pts-2.stlark	(08/15/2022 05:27:13 PM)	(Detached)
   1 Socket in /run/screen/S-root.
   root@stlark:~/altdocker# ./cli-dae1 run -d -v /sys/fs/cgroup/:/sys/fs/cgroup/ -p 8080:8080 --name adminer adminer
   Unable to find image 'adminer:latest' locally
   latest: Pulling from library/adminer
   213ec9aee27d: Pull complete 
   a600fdbc30cc: Pull complete 
   0cdd6cb15c0d: Pull complete 
   8a4c40d8aee7: Pull complete 
   77e67522f4fd: Pull complete 
   d181492ef8e9: Pull complete 
   0d5c09a73378: Pull complete 
   8bb8e21282b9: Pull complete 
   fdd7ab990e10: Pull complete 
   d59940a6c65c: Pull complete 
   8af31b44c60c: Pull complete 
   8f57d1664c2a: Pull complete 
   eafa31c9e999: Pull complete 
   d99ebf4176fd: Pull complete 
   dd175257ae8d: Pull complete 
   Digest: sha256:ffef52f1cdc49e9877f8e73b62afe358396241fd3d018dd6e55bc865c2d58c87
   Status: Downloaded newer image for adminer:latest
   2bdce8211d4f8d2b0f3bc5ec26ec7f1094c5a656092a242109b42cdd384e4b68
   root@stlark:~/altdocker# 
   root@stlark:~/altdocker# ./cli-dae1 ps
   CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS                    NAMES
   2bdce8211d4f        adminer             "entrypoint.sh docker"   About a minute ago   Up About a minute   0.0.0.0:8080->8080/tcp   adminer
   root@stlark:~/altdocker# 
   ````

   При запуске `adminer` был прокинут  volume из-за того, что возникала следующая ошибка:

   ```bash
   94e43bb1881d00600df386e3725c1d216b21ef3e4b50573ae9c1151fd79aa957
   docker: Error response from daemon: oci runtime error: rootfs_linux.go:53: mounting "/sys/fs/cgroup" to rootfs "/root/altdocker/altdocker/dae1/graph/overlay2/7f632588479b3c6c2e6f966c1cdbb6e6ea3c9dc9030b19c3954c360475be39e3/merged" caused "no subsystem for mount".
   root@stlark:~/altdocker# 
   ```

   Кроме того, ее можно обойти, как вариант, увеличением версии Docker в файле `get_docker.sh` , например, до версии `20.10.7`

   Проверить, что adminer досутпен можно по ссылке http://193.168.46.148:8080/. Прикладываю доступы к базе, по которым можно проверить работу `adminer`:

   ```
   Сервер: stlark.beget.tech
   Имя пользовтаеля: stlark_1 
   Пароль: ob*mvVN5
   База данных: stlark_1
   ```

   Также прикладываю скришот с окном админера:

   ![](/home/stanislav/Изображения/adminer.png)
