## Crontab 是什么

crontab 是 Linux 系统中用于任务调度的命令工具，允许用户设定在特定时间间隔，自动执行指定的命令或脚本，极大提升运维效率。

### 特性

①无需人工操作，自动运行任务

②能同时安排多个任务，且各任务可独立设定执行时间

③既可以指定特定日期和时间，如每月最后一天，也能按固定间隔，像每小时运行任务

④即便系统重启，已设任务也会自动恢复执行

## Crontab 命令基本语法

crontab 命令基本格式如下：

```
*    *    *    *    *    command to be executed
-    -    -    -    -
|    |    |    |    |
|    |    |    |    +----- day of the week (0 - 6) (Sunday=0)
|    |    |    +------- month (1 - 12)
|    |    +--------- day of the month (1 - 31)
|    +----------- hour (0 - 23)
+------------- min (0 - 59)
```

从左至右，各字段含义依次为：分钟（0 - 59）、小时（0 - 23）、日期（1 - 31）、月份（1 - 12）、星期（0 - 6，周日为 0），最后是需执行的命令或脚本路径。

### 特殊字符与快捷方式

1. **特殊字符**：

- - `*`：代表所有可能的值，例如，* * * * * 表示每分钟执行一次。

- - `,`：用于指定多个值，例如，15,30 * * * * 表示每小时的第 15、30 分钟执行。

- - `-`：用于指定范围，例如，1-5 * * * * 表示每小时的前五分钟执行。

- - `/`：用于指定步长，例如，*/20 * * * * 表示每 20 分钟执行一次。

1. **快捷方式**：

- - `@reboot` ：在每次系统启动时执行一次任务。

- - `@yearly` 或 `@annually`：每年执行一次任务。

- - `@monthly`：每月执行一次任务。

- - `@weekly` ：每周执行一次任务。

- - `@daily` 或 `@midnight`：每天执行一次任务。

- - `@hourly`：每小时执行一次任务。

### 示例

1. **每周一、三、五的下午 3 点 30 分执行磁盘空间检查脚本**：

```
30 15 * * 1,3,5 /path/to/disk_check.sh
```

1. **每天晚上 10 点到次日早上 6 点之间，每隔 2 小时执行一次服务器负载监测脚本**：

```
0 22-6/2 * * * /path/to/server_monitor.sh
```

1. **每周日的下午 5 点执行系统日志归档脚本**：

```
0 17 * * 0 /path/to/log_archive.sh
```

1. **每 15 分钟执行一次网络连接测试脚本**：

```
*/15 * * * * /path/to/network_test.sh
```

1. **每周执行一次任务**：

```
@weekly command to execute
```

## 创建定时清理文件的任务

### 编写 Shell 脚本

首先编写用于清理备份文件的 shell 脚本。假设要清理 30 天前位于/backup目录下的所有文件，脚本内容如下：

```
#!/bin/bash
find /backup -type f -mtime +30 -exec rm -rf {} \;
```

将上述脚本保存为[[clean_backup_files.sh](http://clean_backup_files.sh/)](http://clean_backup_files.sh)，并赋予其执行权限：

```
chmod +x clean_backup_files.sh
```

### 配置 Crontab 任务

打开 crontab 编辑器：

```
crontab -e
```

首次执行该命令时，系统可能会提示选择文本编辑器，如select-editor，选择熟悉的编辑器即可。

![Image](https://github.com/user-attachments/assets/7537c200-8118-42ac-8230-ab89b0a6aafe)

在打开的编辑器中添加如下内容，实现每天凌晨 2 点执行文件清理脚本：

```
0 2 * * * /path/to/clean_backup_files.sh >/dev/null 2>&1
```

这里的>/dev/null 2>&1用于将脚本执行过程中的标准输出和标准错误输出重定向到/dev/null，避免产生多余输出。

保存并退出编辑器。新添加的任务将在下一个预定时间点自动执行。

### 查看和管理定时任务

若要查看当前用户的所有定时任务，可使用以下命令：

```
crontab -l
或者
cat /path/crontab
```

![Image](https://github.com/user-attachments/assets/dd5f529f-2230-4cf8-bf77-e171fa18da53)

若需删除某个定时任务，再次使用crontab -e命令进入编辑器，删除对应的任务行即可。

### Crontab 的调试

检查 crontab 文件权限，确保 cron 守护进程可读取。

在 crontab 命令中添加>& /tmp/cron.log来将输出重定向到日志文件，便于调试。

使用crontab -l命令列出当前用户的所有定时任务，检查是否有语法错误。

检查系统日志/var/log/syslog或/var/log/cron（根据系统不同而不同），查看 cron 守护进程的输出。



## Crontab 高级技巧

### 环境变量设置

在 crontab 中，环境变量可能不会像在普通 shell 中那样被自动设置。若需在 crontab 任务中使用特定环境变量，可在 crontab 文件中显式设置：

```
PATH=/usr/local/bin:/usr/bin:/bin
export PATH
```

确保这些设置位于 crontab 文件最顶部。

### 处理不稳定网络连接

若 crontab 任务依赖网络资源，且网络连接可能不稳定，可在脚本中添加重试逻辑。例如：

```
*/5 * * * * /usr/local/bin/[fail-safe-command.sh](http://fail-safe-command.sh/)
```

在[fail-safe-command.sh](http://fail-safe-command.sh)脚本中，添加如下重试逻辑：

```sh
#!/bin/bash
MAX_RETRIES=5
RETRY_INTERVAL=10
COUNTER=0

while [ $COUNTER -lt $MAX_RETRIES ]; do
    ping -c 1 google.com && break
    sleep $RETRY_INTERVAL
    COUNTER=$((COUNTER+1))
done

if [ $COUNTER -eq $MAX_RETRIES ]; then
    echo "Failed to connect to Google after $MAX_RETRIES attempts."
fi
```

这个示例脚本会尝试 ping `google.com`，如果失败则每隔 10 秒重试一次，最多重试 5 次。如果 5 次都失败，则输出一条失败信息。



## Crontab 任务排错与调试

### 检查语法

提交 crontab 文件前，使用crontab -e命令后，在退出编辑器前检查语法错误。此外，一些第三方工具如crontab.guru或vcron可提供语法校验功能。

### 日志记录

为 crontab 任务添加日志记录是调试关键。可在任务命令后添加重定向输出到日志文件的命令：

```
* * * * * command to execute >> /path/to/logfile.log 2>&1
```

这样，命令的标准输出和标准错误都会被记录到指定日志文件中。

### 检查环境变量

有时crontab 任务运行失败是因为环境变量未正确设置。可在 crontab 文件中显式设置所需环境变量，或在执行的脚本中设置。

### 使用set -x进行跟踪

在脚本开头添加set -x命令可跟踪脚本执行过程，有助于识别脚本何处失败：

```
#!/bin/bash
set -x
# Your script commands here
```

### 检查磁盘空间

磁盘空间不足可能导致 crontab 任务无法创建日志文件或执行脚本，可以定期清理日志。

## 示例

### 定时备份数据库

数据库定期备份是企业数据安全重要组成部分。以下是使用 crontab 定时备份 MySQL 数据库的例子：

```
0 2 * * * /usr/bin/mysqldump -u username -p'password' database_name > /path/to/backup/database_$(date +\%Y-\%m-\%d).sql
```

该任务会在每天凌晨 2 点执行，将数据库备份到指定路径，文件名包含日期，便于归档和恢复。

### 定时清理日志文件

日志文件若不定期清理，可能占用大量磁盘空间。以下是定时清理旧日志文件的例子：

```
0 * * * * find /var/log/nginx/ -name 'access_*.log' -mtime +7 -exec rm -f {} \;
```

此任务会每小时检查/var/log/nginx/目录下所有名称为access_*.log的文件，若超过 7 天未被修改，就会被删除。

### 定时同步文件到远程服务器

若需定时将文件同步到远程服务器，可使用rsync命令。示例如下：

```
30 23 * * * /usr/bin/rsync -avz /path/to/local/directory username@remotehost:/path/to/remote/directory
```

该任务会在每天晚上 11 点半执行，将本地目录内容同步到远程服务器指定目录。

### 定时检查系统更新

对于需保持系统更新的场景，可设置定时任务检查和安装更新。以下是基于 Debian 系统的例子：

```
0 3 * * * /usr/bin/apt-get update && /usr/bin/apt-get upgrade -y
```

此任务会在每天凌晨 3 点执行，更新系统包列表，并升级所有可用包。

### 定时执行自定义脚本

若有自定义脚本需定期执行，可直接在 crontab 中调用：

```
*/30 * * * * /path/to/[custom_script.sh](http://custom_script.sh/)
```

该任务会每 30 分钟执行一次[custom_script.sh](http://custom_script.sh)脚本。