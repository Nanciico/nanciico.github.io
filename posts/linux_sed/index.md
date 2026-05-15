# Linux sed 命令


## 基本操作命令

### 替换操作 (s命令)

``` bash
# 基本替换
sed 's/old/new/' file.txt                    # 每行第一个old替换为new
sed 's/old/new/g' file.txt                   # 全局替换

# 替换指定行
sed '2s/old/new/' file.txt                   # 只替换第2行
sed '2,5s/old/new/g' file.txt                # 替换2-5行
sed '2,$s/old/new/g' file.txt                # 替换第2行到最后一行

# 原地修改文件
sed -i 's/old/new/g' file.txt                # 直接修改原文件
sed -i.bak 's/old/new/g' file.txt            # 备份原文件为file.txt.bak

# 使用不同分隔符
sed 's|/usr/local|/opt|g' file.txt           # 使用|作为分隔符
sed 's#/old/path#/new/path#g' file.txt       # 使用#作为分隔符
```

### 删除操作 (d命令)

``` bash
# 删除指定行
sed '2d' file.txt                            # 删除第2行
sed '2,5d' file.txt                          # 删除2-5行
sed '$d' file.txt                            # 删除最后一行

# 条件删除
sed '/pattern/d' file.txt                    # 删除包含pattern的行
sed '/^#/d' file.txt                         # 删除注释行（以#开头）
sed '/^$/d' file.txt                         # 删除空行
sed '/^[[:space:]]*$/d' file.txt             # 删除空白行（包括只有空格/tab的行）

# 保留特定范围
sed '3,8!d' file.txt                         # 只保留3-8行，删除其他行
```

### 打印操作 (p命令)

``` bash
# 打印特定行（通常与-n一起使用）
sed -n '2p' file.txt                         # 只打印第2行
sed -n '2,5p' file.txt                       # 打印2-5行
sed -n '/pattern/p' file.txt                 # 打印匹配pattern的行

# 打印行号
sed -n '=' file.txt                          # 只打印行号
sed -n '2,5p;2,5=' file.txt                  # 同时打印内容和行号
```

### 插入和追加 (i/a命令)

``` bash
# 插入行（在指定行之前）
sed '2i\插入的内容' file.txt                  # 在第2行前插入
sed '3i\第一行\n第二行' file.txt              # 插入多行

# 追加行（在指定行之后）
sed '2a\追加的内容' file.txt                  # 在第2行后追加
sed '/pattern/a\追加的内容' file.txt          # 在匹配pattern的行后追加

# 在文件开头/结尾添加
sed '1i\文件开头' file.txt                    # 在文件开头插入
sed '$a\文件结尾' file.txt                    # 在文件结尾追加
```

---

## 高级操作

### 多点编辑

``` bash
# 多个编辑操作
sed -e 's/old/new/g' -e '/pattern/d' file.txt
sed 's/old/new/g; /pattern/d' file.txt

# 从文件读取多个命令
echo "s/old/new/g\n/pattern/d" > script.sed
sed -f script.sed file.txt
```

### 写文件操作

``` bash
# 将匹配的行写入新文件
sed -n '/pattern/w output.txt' file.txt      # 将匹配pattern的行写入output.txt
sed -n '2,5w segment.txt' file.txt           # 将2-5行写入segment.txt
```

