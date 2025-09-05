# Эффективное использование LLM в командной строке

## Зачем

Использование ИИ проникло уже во все возможные сферы в ИТ. Мы применяем LLM модели как через Web так и подключая через специальный API в программном коде. Запускаем агентов кодогенерации и реализации других практических задач через специальные плагины или отдельные приложения. Рассмотрим ещё один интерфейс работы с GenAI — командную строку. Сразу предупрежу, мы не будем рассматривать код-агенты вроде Claude Code или AI-терминалы вроде Warp.

Сказанное ниже в основном применимо к командной строке типичной UNIX like системы Linux, MacOS, хотя использование WSL добавляет сюда и Windows пользователей. Покажем на доступных и простых примерах, как LLM помогают решать как типичные задачи для cli (command line interface), так и дают дополнительное удобство, преимущество в работе с моделями. Минимум текста и максимум конкретных вызовов команд расскажут сами за себя.

Для практического использования не принципиальна конкретная модель, но рекомендую брать `instruct`, НЕ reasoning и с бОльшим размером контексте. Для достижения лучшего эффекта лучше использовать "большие" модели с числом параметров от 24B. Нижеописанное я опробовал в основном на `meta-llama/llama-3.3-70b-instruct` с размером контекста 64k и на  `mistral-small-3.2-24b-instruct` с  128k когда его не хватало , в командной строке MacOS X 15 с инференсом на [openrouter](https://openrouter.ai/).  Примеры несколько простые и местами синтетические, но которые всё же базируются на личном практическом опыте. Цель показать возможные области применимости, а не e2e кейсы.

## Подключение и настройка клиента

Для работы будем использовать этот клиент [aichat](https://github.com/sigoden/aichat). У него много разных возможностей, но пока мы воспользуемся самым минимумом которого достаточно для текущей демонстрации. Можно выбрать другой клиент с хорошей CLI-интеграцией - [llm](https://github.com/simonw/llm) или [ollama](https://github.com/ollama/ollama?tab=readme-ov-file#pass-the-prompt-as-an-argument) если запускать локально.

Итак, после установки и первоначального запуска конфигурируем клиент для использования имеющегося у вас доступа к LLM и интерфейс готов.

```console
$ aichat hi
How's it going? Is there something I can help you with or would you like to chat?
```

Так как aichat — полноценная консольная команда, мы можем работать с ним по тем же принципам, что и с другими утилитами UNIX  реализующий принцип "все есть текст", те передавать текст в поток ввода, через параметры командной строки и перенаправлять ответ или в файл или в поток ввода другой команды. Если кратко то все потенциальные возможности реализуются такой конструкцией.

```bash
command1 | aichat "Message to prompt. $(command2 2>&1)" >file.out 
```

- перенаправлять вывод `command1` в поток ввода `aichat` и это сразу пойдет в промт модели
- добавлять в промпт вывод команды `command2`, `$()` запускает команду в дочернем шелле и подставляет её вывод в место вызова, а конструкция `2>&1` сразу перенаправит поток ошибок если они будут
- вывод модели будет записан в `file.out`, или тут же можно было бы перенаправить его с помощью pipe `|` на ввод другой команде

### Системный промпт

Модели по умолчанию слишком _разговорчивы_ для командой строки, поэтому следующее что я рекомендую сразу настроить - это системный промпт.  В терминологии aichat это делается путем создания роли. Ниже мой вариант, который подобрал экспериментально.

```text
You are a AI tool working in command line interface - shell. Process the given query and output a direct, plain-text answer to STDOUT. No extra explanation or for formatting (no backticks or language tags) unless asked
```

Польза собственного промпта:

```bash
$ aichat "Show used shells for users $(cat /etc/passwd)"
The used shells for users are:

1. `/usr/bin/false` (used by most system users)
2. `/bin/sh` (used by the `root` user)
3. `/bin/bash` (used by the `mbsetupuser` user)

Note that `/usr/bin/false` is a special shell that does nothing when invoked, and is often used for system users that do not need to log in interactively.

#cli role with special system prompt
$ aichat -r cli "Show used shells for users $(cat /etc/passwd)"
/bin/sh 
/usr/bin/false 
/bin/bash

```

### Сессии

Чатясь с моделью в браузере мы чаще делаем это в рамках какой-то конкретной задачи, сессии. Тоже самое можно организовать и в cli, потому что по умолчанию каждый запрос в рамках aichat это новый диалог.

```bash
$ aichat "My name Sergey" 
Hello Sergey! It's nice to meet you. Where are you from, Sergey?
$ aichat "What is my name? "
I don't know your name!...
```

Для коммуникации в рамках сессии используем следующий набор ключей:

```bash
$ aichat -s session1 --save-session "My name Sergey"
Nice to meet you, Sergey! How are you doing today? Is there something I can help you with or would you like to chat?
$ aichat -s session1 "What is my name ?"
Your name is Sergey! You told me that at the beginning of our conversation.

```

Объединив 2 подхода с помощью алиаса таким образом, каждая bash процесс это новая сессия с LLM с сохранением истории.

```bash
alias aichats='aichat -r cli -s $$ --save-session'

aichats "I am Sergey"

aichats "What is my name"
```

`$$` - pid текущего процесса. Соответственно нужно следить, добавить свою автоматизаци, что вы явно используете новую сесссию, а не переиспользуете старую с таким же pid. Как вариант можно затюнить приглашение bash - переменную `PS1` чтобы отражать ID текущей сессии если она есть.

## Дополнение, замена классических UNIX команд

Всю мощь LLM можно применять для расширения возможностей или замены привычных команд работы с текстом и файлами.

Замена grep, sort:

```bash
$ ps aux | aichat -r cli 'Print processes owned by root sorted by CPU usage '
USER               PID  %CPU %MEM      VSZ    RSS   TT  STAT STARTED      TIME COMMAND
root               328   1.7  0.5 443841552  87392   ??  Ss   18Jun25  53:25.91 /System/Library/Frameworks/CoreServices.framework/Frameworks/Metadata.framework/Versions/A/Support/mds_stores
root             64705   1.6  0.0 410484848   2192 s014  R+   12:54PM   0:00.01 ps aux
root               147   0.2  0.0 426932576   3648   ??  Ss   18Jun25  38:12.47 /usr/sbin/notifyd
...
```

sed:

```bash
cat file | aichat -r cli "replace oldtext to newtext" >file.new
```

file:

```bash
aichat -r cli "What type of information it is: $(cat file)"
```

diff:

```bash
aichat -r cli "Provide key difference between files $(cat file1) $(cat file2)"
```

Учитываем, что конструкция вида `$(command)` после выполнения будет заменена на содержимое вывода команды, БЕЗ её имени. Если для задачи важно сохранить имя выполняемой команды или команд `$(cmd1| cmd2 )`, необходимо сделать это явно.

```bash
aichat "Assess output of cmd1 | cmd2 $(cmd1| cmd2 )"
```

Более комплексный пример. Очистка свежесобранных не нужных образов докера

```bash
docker images | aichat -r cli "Print id of images created for last 3 hours" | xargs docker rm
```

## Анализ, Дебаг, Работа c конфигами

При работе с cli, вместо того чтобы копипастить необходимый текст для анализа в web окно чата модели, намного удобнее сразу перенаправлять его из соответствующей команды.

Начнем с типичного сценария по поиску причины проблемы и решения. Базовым паттерном будет следующая конструкция:

```bash
command_with_error 2>&1 | aichat -r cli
```

Это удобно: после ошибки достаточно добавить к команде вызов aichat — и получить пояснение.

Не стартует под в k8s:

```bash
$ kubectl logs debug-training-deployment-6d9dd7cb7f-dgjp2 | aichat -r cli "Find root cause and provide solution"
The root cause of the issue is that the command to start nginx is being executed with an invalid option "u" which is likely due to attempting to start nginx as a user 'fakeuser' using the -u option, which is not a valid option for the nginx command.

The solution is to use the correct command to start nginx as the desired user.
...
```

Анализируем производительность локальной системы:

```bash
$ ps aux | aichat -r cli "identify the most resource consuming processes"
The most resource consuming processes are:

1. Visual Studio Code (PID 47091) - CPU: 2.6%, MEM: 2.6%
2. Google Chrome (PID 44623) - CPU: 0.0%, MEM: 1.6%
3. Google Chrome (PID 34221) - CPU: 0.0%, MEM: 0.4%
...
```

Изменения конфигурационных файлов. На примере конфигурации деплоймента k8s:

```bash
$ aichat -r cli "Update $(cat deployment.yaml) to have 3 replicas. Output generate in diff format" >deployment.patch
$ cat deployment.patch
...
$ patch -i deployment.patch  deployment.yaml
patching file deployment.yaml

# после некоторого "принятия" можно будет сразу применять изменения
$ aichat -r cli "Update $(cat deployment.yaml) to have 3 replicas. Output generate in diff format" | patch -b deployment.yaml
```

В cli удобно комбинировать сразу несколько источников данных для наполнения контекста и анализа. В рамках сессии мы можем обогощать запрос новыми данными и уточнять поиск решения.

```bash
# получаем рекомендации по командам для конкретной системы с определенным набором ПО
$ aichats "Need to identify root cause of performance issue on system $(uname -a) \ 
$(cat /etc/os-release) $(ps aux) $(free) with packages $(rpm -qa). \
Provide a list of commands with parameters to run to collect debug info"
...

# в рамках этой же сессии предоставляем запрошенные данные модели для анализа. Получаем ответ.
$ aichats "$(top -c -b -n 1) $(iostat -xd 1 5) $(free -m) ..."
...

# получили дополнительную рекомендацию или увидели что каких то данных не хватает
$ aichats "check also $(df -h)"
...

```

В примере выше весь аутпут будет частью одной сессию работы с LLM, одним запросом по сути. Тут важен как хороший размер контекста модели, так и релевантные запросу данные (используем `grep`, `head`, `tail` чтобы отфильтровать то что не имеет отношения к решаемой задачи).

Отдельно стоит отметить паттерн работы в сессионном режиме - `aichat -s debug --save-session`. Он особенно полезен при анализе сложных проблем, многошаговом поиске решений или любых задачах, требующих нескольких итераций. К сессии можно возвращаться в любое время и продолжать работу. При этом `aichat` сохраняет историю в YAML-файл, который становится удобным артефактом для документирования и последующего использования.

## Внешние источники. Работа с документами

Дополнительный уровень возможностей даёт способность комбинировать в едином LLM контексте информацию из разных источников. Тут важно помнить что LLM работает с текстом и необходимо привести данные к единому текстовому формату. Markdown прекрасно подходит как универсальный формат текстовых данных которой хорошо понимают модели. Для конвертации разнородных форматов документов в Markdown, использую 2 утилиты:

- [trafilatura](https://trafilatura.readthedocs.io/en/latest/) больше для обработки Web страниц
- [markitdown](https://github.com/microsoft/markitdown) для конвертации разнообразных документов MS Office
  
Быстрая оценка соответствия best practise'ам локального minikube кластера:

```bash
$ curl https://spacelift.io/blog/kubernetes-best-practices | trafilatura --markdown  | \
aichat "Check requirements compliance for the provided cluster.  $(kubectl get all -A -o yaml) "

Based on the provided cluster state and the Kubernetes best practices you've listed, I'll analyze the compliance of your cluster with these best practices:

1. **Use namespaces**:
   - Your cluster has multiple namespaces (default, kube-system, etc.)
   - The `debug-training` deployment is running in the default namespace
   - Compliance: Partial (using namespaces, but could use more logical separation)

...   

Key recommendations for improvement:

1. Implement readiness and liveness probes for all critical applications

...

The core system components (etcd, kube-apiserver, coredns) appear to be properly configured with appropriate resources and monitoring, but the application workloads (debug-training) show significant gaps in following best practices.
```

Такой способ конечно не подойдет в лоб для больших кластеров и многостраничных документов, и его нужно творчески модифицировать под свои нужды. Например, не передавать весь аутпут `kubectl` кластера, а попросить модель на базе стандарта и базовой информации о кластере `kubectl get/describe node`, сформировать совместимые `kubectl get` проверки, аутпут которых потом и передавать частями в LLM.

Диаграммы, представленные в текстовом виде (Plantuml, Mermaid), также можно обрабатывать схожим образом. Несколько примеров обработки архитектурных диаграм:

```bash
cat arch.plantuml | aichat -r cli "add miscroservice C in the same manner as A. Add DB replication link" >arch_2.plantuml
...
aichat -r cli "Assess architecture $(cat arch.plantuml) compatible with $(markitdown corporate_arch_guide.doc) and provide updated version in the same format" > arch_2.plantuml
...
cat arch.plantuml | aichat -r cli "Convert plantuml to drawio xml code" >arch.drawio
...
```

Обработка нескольких документов и структурированный вывод:

```bash
aichat -r cli "Assess architecture $(cat arch.plantuml) $(cat arch.doc | markitdown) complience with Security standard \ 
$(curl https://local_site.com/sec_requirements | trafilatura --markdown ) and provide results in csv format using structure \ 
$(cat Summary_table.xls| markitdown) " > arch_security_complience.csv

```

Последний пример показывает потенциал комбинирования разных источников для решения конкретной практической задачи. Конечно при огромном размере документов здесь необходимо будет прибегать к технике саммаризации и другим ухищрениям. Также помним, что сами вызовы к LLM тоже можно объединять в pipe и получать ещё более комплексные сценарии: `aichat | aichat ...`

## Ограничения

Важно помнить об ограничениях:

- вероятностная природа моделей и возможность галлюцинаций
  - верифицируйте аутпут LLM прежде чем его использовать
  - установите `temperature` и `top_p` в `0`  настройках роли для используемой модели
- размер контекстного окна ограничен как и длина команды с параметрами командной строки. Соответственно что то подобное делать обдуманно:

  ```bash
  cat huge_file | aichat "replace string1 with string2" >huge_file.new

  aichat "Describe the content of $(cat huge_file)"
  ```

## Выводы

Интеграция LLM в командную строку открывает новые сценарии работы: от замены утилит до анализа логов и обработки документов. Это гибкий подход, позволяющий напрямую управлять контекстом и строить цепочки команд через pipe.

Уже сегодня такие решения можно дополнять подключением собственных данных через RAG и использованием function calling для вызова утилит. Всё это превращает LLM в полноценного партнёра инженера. Лично я уже не могу представить работу в командной строке без такого помощника.
