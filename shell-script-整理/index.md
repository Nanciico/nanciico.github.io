# Shell Script 整理


## Shell 参数处理特殊字符

``` bash
#!/bin/bash

echo "Starting program at $(date)"                                  # 1. $( CMD ) 运行 CMD 命令

echo "Running program $0 with $# arguments with pid $$"             # 2. 见特殊字符表

for file in "$@"; do                                                # 3. 见特殊字符表
    grep foobar "$file" > /dev/null 2> /dev/null                    # 4. "2> /dev/null" 把标准错误重定向到/dev/null

    if [[ $? -ne 0 ]]; then                                         # 5. 见特殊字符表；在比较操作中使用双中括号
        echo "File $file does not have any foobar, adding one"
        echo "# foobar" >> "$file"
    fi
done
```

### 特殊字符表

| 参数处理 | 说明 |
| :----: | :----: |
| $0 | Shell Script 名字 |
| $# | 参数个数 |
| $$ | 进程 PID |
| $@ | 所有参数 |
| $? | 上一条命令的返回码 |

### Test 命令的双中括号

[Differences Between Single and Double Brackets in Bash.](https://www.baeldung.com/linux/bash-single-vs-double-brackets)

## Safe-bgsave 脚本整理

``` bash
#!/bin/bash

function help()                                                     # 1. 函数声明
{
    echo "usage: safe-bgsave -t THRESHOLD -i INTERVAL -p PORT."
    echo "       safe-bgsave -t 20        -i 1        -p 7002"
    exit
}

# success return 0; fail return 1.
function safe_bgsave()
{
    # check security
    local available=`free -g | grep Mem | awk '{print $7}'`         # 2. awk 选取第 7 列
    if [ ${available} -le ${THRESHOLD} ]
    then
        echo `date` " Available memory: ${available} <= threshold: ${THRESHOLD}. Do not process bgsave." >> ${LOG_PATH} 2>&1
        return 1
    fi

    # bgsave
    local bgsave=`${REDIS_CLI} -p ${PORT} bgsave`
    echo `date` " ${bgsave}." >> ${LOG_PATH} 2>&1                   # 3. 字符串拼接；"2>&1" 将标准错误重定向到标准输出

    sleep ${INTERVAL}

    local pid=`ps -ef | grep "${PROCESS_NAME}" | grep -v grep | awk '{print $2}'`
    while [ -n "${pid}" ]
    do
        available=`free -g | grep Mem | awk '{print $7}'`

        if [ ${available} -le ${THRESHOLD} ]
        then
            echo `date` " bgsave failed. Available memory: ${available} <= threshold: ${THRESHOLD}. Trying to kill process." >> ${LOG_PATH} 2>&1
            kill "${pid}"
            return 1
        fi

        sleep ${INTERVAL}

        pid=`ps -ef | grep "${PROCESS_NAME}" | grep -v grep | awk '{print $2}'`
    done

    echo `date` " bgsave success." >> ${LOG_PATH} 2>&1
    return 0
}

# Begin
[ $# -ne 6 ] && help                                                # 4. 检查参数个数；调用 help 函数

while [ -n "$1" ]                                                   # 5. 参数赋值的写法
do
    case "$1" in
        -t) THRESHOLD=$2
            shift 2                                                 # 6. 将参数数组向左移动两位
            ;;
        -i) INTERVAL=$2
            shift 2
            ;;
        -p) PORT=$2
            shift 2
            ;;
         *) help
            ;;
    esac
done

REDIS_CLI="redis-cli"
PROCESS_NAME="redis-rdb-bgsave"
LOG_PATH="/opt/data/redis/safe_bgsave.${PORT}.log"

safe_bgsave

if [ $? -eq 1 ]
then
    exit 1
fi

exit 0
```

