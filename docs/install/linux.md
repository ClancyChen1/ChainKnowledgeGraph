# ChainKnowledgeGraph Linux 安装与运行手册

本文档说明如何在 Linux 上安装项目依赖、启动 Neo4j、导入图谱数据，以及后续如何查看和重启。

当前项目默认连接配置在 `build_graph.py` 中：

```text
NEO4J_URI      默认 bolt://127.0.0.1:7687
NEO4J_USER     默认 neo4j
NEO4J_PASSWORD 默认 12345678
```

如果你的 Neo4j 地址、用户名、密码不同，可以用环境变量覆盖。

## 通用说明

项目主要文件：

```text
build_graph.py     构建 Neo4j 图数据库的脚本
data/              公司、行业、产品、关系数据
readme.md          原项目说明
```

本项目依赖：

```text
Python 3.10+，推荐 Python 3.12
uv
py2neo==2021.2.4
Neo4j Community 5.x，本文使用 5.26.26
Java 17 或 Java 21
```

Neo4j 5 要求密码至少 8 位，因此本文统一使用：

```text
username: neo4j
password: 12345678
```

注意：当前 `build_graph.py` 使用 `CREATE` 创建节点和关系，不是完全幂等脚本。不要在已有数据的库里反复执行 `uv run build_graph.py`，否则可能产生重复节点，或者触发唯一约束报错。需要重新导入时，先看本文最后的“重新导入数据”。

## Linux 版

以下命令以项目路径 `/home/nanxian/other_projects/llm/group_project/ChainKnowledgeGraph` 为例。

### 安装 Python 依赖

```bash
cd /home/nanxian/other_projects/llm/group_project/ChainKnowledgeGraph

uv venv --python 3.12
uv pip install py2neo==2021.2.4
```

验证：

```bash
uv run python - <<'PY'
import py2neo
print(py2neo.__version__)
PY
```

期望输出：

```text
2021.2.4
```

### 安装 Java 17

先检查：

```bash
java -version
```

如果已经是 Java 17 或 Java 21，可以跳过本节。

用户目录安装 Java 17，不需要 `sudo`：

```bash
mkdir -p ~/.local/opt
cd ~/.local/opt

wget -c 'https://api.adoptium.net/v3/binary/latest/17/ga/linux/x64/jdk/hotspot/normal/eclipse' -O temurin-jdk17.tar.gz
tar -xzf temurin-jdk17.tar.gz

JDK_DIR=$(tar -tzf temurin-jdk17.tar.gz | head -1 | cut -d/ -f1)
ln -sfn "$JDK_DIR" jdk-17
```

验证：

```bash
~/.local/opt/jdk-17/bin/java -version
```

### 安装 Neo4j

```bash
mkdir -p ~/.local/opt
cd ~/.local/opt

NEO4J_VERSION=5.26.26
wget -c "https://s3-eu-west-1.amazonaws.com/dist.neo4j.org/neo4j-community-${NEO4J_VERSION}-unix.tar.gz" -O "neo4j-community-${NEO4J_VERSION}-unix.tar.gz"

tar -xzf "neo4j-community-${NEO4J_VERSION}-unix.tar.gz"
ln -sfn "neo4j-community-${NEO4J_VERSION}" neo4j
```

说明：Neo4j 官网常见下载地址 `https://dist.neo4j.org/...` 在部分网络或代理环境下可能返回 `403 Forbidden`，上面的 S3 地址可以作为可用替代。

### 配置环境变量

```bash
cat >> ~/.bashrc <<'EOF'

# User-local Java and Neo4j.
export JAVA_HOME="$HOME/.local/opt/jdk-17"
export NEO4J_HOME="$HOME/.local/opt/neo4j"
export PATH="$JAVA_HOME/bin:$NEO4J_HOME/bin:$PATH"
EOF

source ~/.bashrc
```

验证：

```bash
java -version
neo4j version
```

期望看到 Java 17 和 Neo4j 5.26.26。

### 第一次设置 Neo4j 密码

在第一次启动 Neo4j 之前执行：

```bash
neo4j-admin dbms set-initial-password 12345678
```

如果提示密码已经设置过，说明数据库已经初始化过。可以继续用现有密码，或者删除 Neo4j 数据目录后重新初始化。

危险操作，只有在确认不需要当前数据库数据时才执行：

```bash
neo4j stop
rm -rf ~/.local/opt/neo4j/data
neo4j-admin dbms set-initial-password 12345678
```

### 第一次启动 Neo4j

```bash
neo4j start
neo4j status
```

检查端口：

```bash
ss -ltnp | rg ':7474|:7687'
```

期望看到：

```text
127.0.0.1:7687   Bolt 连接端口
127.0.0.1:7474   Web Browser 端口
```

如果 `neo4j start` 显示启动成功但随后又退出，使用前台模式查看详细日志：

```bash
neo4j console
```

如果 `neo4j console` 前台模式能正常启动，但希望关闭当前终端后 Neo4j 仍然运行，可以用：

```bash
setsid env JAVA_HOME="$HOME/.local/opt/jdk-17" \
  PATH="$HOME/.local/opt/jdk-17/bin:$HOME/.local/opt/neo4j/bin:$PATH" \
  neo4j console > "$HOME/.local/opt/neo4j/logs/console.out" 2>&1 < /dev/null &
```

检查：

```bash
ss -ltnp | rg ':7474|:7687'
tail -f ~/.local/opt/neo4j/logs/console.out
```

### 测试项目连接 Neo4j

```bash
cd /home/nanxian/other_projects/llm/group_project/ChainKnowledgeGraph

uv run python - <<'PY'
from build_graph import MedicalGraph
h = MedicalGraph()
print(h.g.run("RETURN 1 AS ok").data())
PY
```

期望输出：

```text
[{'ok': 1}]
```

### 导入图谱数据

```bash
cd /home/nanxian/other_projects/llm/group_project/ChainKnowledgeGraph
uv run build_graph.py
```

### Linux 无图形界面查看图谱

Neo4j Browser 是 Web 页面，服务器本身不需要图形界面。

如果在服务器本机浏览器查看：

```text
http://127.0.0.1:7474
```

如果服务器无图形界面，在自己的电脑上使用 SSH 端口转发：

```bash
ssh -L 7474:127.0.0.1:7474 -L 7687:127.0.0.1:7687 用户名@服务器IP
```

保持 SSH 窗口不要关闭，然后在自己电脑浏览器打开：

```text
http://127.0.0.1:7474
```

登录：

```text
username: neo4j
password: 12345678
```

### Linux 后续启动

```bash
source ~/.bashrc
neo4j start
neo4j status
```

停止：

```bash
neo4j stop
```

如果只是查看已有图谱，不需要重新执行 `build_graph.py`。

## 常用 Cypher 查询

进入 Neo4j Browser 后可以执行：

查看少量节点：

```cypher
MATCH (n) RETURN n LIMIT 50;
```

查看公司：

```cypher
MATCH (n:company) RETURN n LIMIT 25;
```

查看产品：

```cypher
MATCH (n:product) RETURN n LIMIT 25;
```

查看行业：

```cypher
MATCH (n:industry) RETURN n LIMIT 25;
```

查看关系图：

```cypher
MATCH p=(a)-[r]->(b) RETURN p LIMIT 100;
```

统计节点数：

```cypher
MATCH (n) RETURN labels(n) AS labels, count(n) AS count;
```

统计关系数：

```cypher
MATCH ()-[r]->() RETURN type(r) AS type, count(r) AS count ORDER BY count DESC;
```

## 重新导入数据

当前 `build_graph.py` 使用 `CREATE` 创建节点和关系，不是完全幂等脚本。不要在已有数据的库里反复执行 `uv run build_graph.py`。

如果要从零重新导入，先清空数据库：

```bash
uv run python - <<'PY'
from build_graph import MedicalGraph
h = MedicalGraph()
h.g.run("MATCH (n) DETACH DELETE n")
print("database cleared")
PY
```

然后重新导入：

```bash
uv run build_graph.py
```

也可以在 Neo4j Browser 里执行：

```cypher
MATCH (n) DETACH DELETE n;
```

## 常见报错

### `http_port` 不支持

报错：

```text
ValueError: The following settings are not supported: {'http_port': 7474}
```

原因：当前 `py2neo==2021.2.4` 不支持旧写法：

```python
Graph(host="127.0.0.1", http_port=7474, user="neo4j", password="123456")
```

应使用 URI 写法：

```python
Graph("bolt://127.0.0.1:7687", auth=("neo4j", "12345678"))
```

本项目当前代码已经使用新写法。

### 无法连接 `127.0.0.1:7687`

报错：

```text
Cannot open connection to ConnectionProfile('bolt://127.0.0.1:7687')
ConnectionRefusedError: [Errno 111] Connection refused
```

原因：Neo4j 没启动，或者 Bolt 端口没监听。

检查：

```bash
neo4j status
```

Linux/macOS 还可以检查端口：

```bash
ss -ltnp | rg ':7687|:7474'
```

Windows PowerShell 可以检查：

```powershell
netstat -ano | findstr "7474 7687"
```

### Neo4j Browser 认证失败

报错：

```text
The client is unauthorized due to authentication failure.
```

原因：用户名或密码不对。

本项目默认：

```text
username: neo4j
password: 12345678
```

不要输入 `123456`，Neo4j 5 不接受 6 位初始密码。

### Java 版本不对

报错：

```text
Unsupported Java 11 detected. Please use Java 17 or Java 21.
```

原因：Neo4j 5 不能使用 Java 11。

处理：安装 Java 17 或 Java 21，并确认 `java -version` 输出正确。

### Neo4j 密码太短

报错：

```text
InvalidPasswordException: A password must be at least 8 characters.
```

原因：Neo4j 5 要求密码至少 8 位。

本项目建议使用：

```text
12345678
```

### `Schema.ConstraintValidationFailed`

报错：

```text
Node already exists with label `product` and property `name` = ...
```

原因：数据库里已经存在节点，又重复执行了 `build_graph.py`。当前脚本使用 `CREATE`，重复运行不安全。

处理：如果确认要重新导入，先清库，再运行脚本。

### `defaultdict` 未定义

报错：

```text
NameError: name 'defaultdict' is not defined
```

原因：脚本里使用了 `defaultdict(list)`，但没有导入。

修复：

```python
from collections import defaultdict
```

本项目当前代码已经修复。

## 参考链接

```text
Neo4j Browser:     http://127.0.0.1:7474
Neo4j Bolt:        bolt://127.0.0.1:7687
Neo4j 官网:        https://neo4j.com/
Neo4j 下载页:      https://neo4j.com/deployment-center/
Adoptium Java:     https://adoptium.net/
Microsoft OpenJDK: https://learn.microsoft.com/java/openjdk/download
py2neo 文档:       https://py2neo.org/
uv:                https://docs.astral.sh/uv/
```
