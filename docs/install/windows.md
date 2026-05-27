# ChainKnowledgeGraph Windows 安装与运行手册

本文档说明如何在 Windows 上安装项目依赖、启动 Neo4j、导入图谱数据，以及后续如何查看和重启。

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

## Windows 版

Windows 建议使用 PowerShell。以下命令只是示例，实际路径可以换成你自己的目录。

推荐目录：

```text
C:\tools\neo4j
C:\tools\jdk-17
D:\projects\ChainKnowledgeGraph
```

### 安装 Python 和 uv

安装 Python 3.10+，推荐 Python 3.12。

可以从 Python 官网下载安装包，安装时勾选：

```text
Add python.exe to PATH
```

安装 uv：

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

重新打开 PowerShell 后验证：

```powershell
python --version
uv --version
```

### 安装项目 Python 依赖

```powershell
cd D:\projects\ChainKnowledgeGraph

uv venv --python 3.12
uv pip install py2neo==2021.2.4
```

验证：

```powershell
uv run python -c "import py2neo; print(py2neo.__version__)"
```

### 安装 Java 17 或 Java 21

Neo4j 5 需要 Java 17 或 Java 21。

Windows 下载方式可能随官网页面变化而变化，因此这里不写死具体安装包链接。请自行下载满足以下要求的 JDK：

```text
版本：Java 17 或 Java 21
架构：Windows x64
类型：JDK，不是 JRE
来源：Eclipse Temurin、Microsoft OpenJDK、Oracle JDK 均可
```

可选下载入口：

```text
https://adoptium.net/
https://learn.microsoft.com/java/openjdk/download
https://www.oracle.com/java/technologies/downloads/
```

安装完成后设置环境变量：

```text
JAVA_HOME=C:\Program Files\Eclipse Adoptium\jdk-17.x.x
PATH 增加 %JAVA_HOME%\bin
```

如果你解压到了用户目录，例如 `C:\tools\jdk-17`，则设置：

```text
JAVA_HOME=C:\tools\jdk-17
PATH 增加 %JAVA_HOME%\bin
```

PowerShell 验证：

```powershell
java -version
```

期望看到 Java 17 或 Java 21。

### 安装 Neo4j

Windows 下载方式也可能随 Neo4j 官网页面变化而变化。请从 Neo4j 官方下载 Community Edition：

```text
https://neo4j.com/deployment-center/
```

建议选择：

```text
Edition: Community
Version: 5.x，推荐 5.26.26 或同一大版本的 5.x LTS/稳定版本
Platform: Windows
Package: ZIP 或 Windows executable
```

如果下载的是 ZIP，解压到例如：

```text
C:\tools\neo4j-community-5.26.26
```

为了路径简单，可以把目录重命名为：

```text
C:\tools\neo4j
```

设置环境变量：

```text
NEO4J_HOME=C:\tools\neo4j
PATH 增加 %NEO4J_HOME%\bin
```

重新打开 PowerShell 后验证：

```powershell
neo4j version
```

如果 `neo4j` 命令找不到，也可以进入 Neo4j 安装目录执行：

```powershell
cd C:\tools\neo4j
.\bin\neo4j.bat version
```

### 第一次设置 Neo4j 密码

在第一次启动 Neo4j 之前执行：

```powershell
neo4j-admin dbms set-initial-password 12345678
```

如果命令找不到：

```powershell
cd C:\tools\neo4j
.\bin\neo4j-admin.bat dbms set-initial-password 12345678
```

### 第一次启动 Neo4j

建议先用前台模式启动，方便看日志：

```powershell
neo4j console
```

或：

```powershell
cd C:\tools\neo4j
.\bin\neo4j.bat console
```

看到类似信息表示启动成功：

```text
Bolt enabled on localhost:7687.
HTTP enabled on localhost:7474.
Remote interface available at http://localhost:7474/
Started.
```

保持这个 PowerShell 窗口运行，不要关闭。

### 测试项目连接 Neo4j

另开一个 PowerShell：

```powershell
cd D:\projects\ChainKnowledgeGraph

uv run python - <<'PY'
from build_graph import MedicalGraph
h = MedicalGraph()
print(h.g.run("RETURN 1 AS ok").data())
PY
```

如果 PowerShell 不支持这里的 heredoc 写法，可以改用一行命令：

```powershell
uv run python -c "from build_graph import MedicalGraph; h=MedicalGraph(); print(h.g.run('RETURN 1 AS ok').data())"
```

期望输出：

```text
[{'ok': 1}]
```

### 导入图谱数据

```powershell
cd D:\projects\ChainKnowledgeGraph
uv run build_graph.py
```

### Windows 查看图谱

浏览器打开：

```text
http://127.0.0.1:7474
```

登录：

```text
username: neo4j
password: 12345678
```

连接实例时填写：

```text
Protocol: bolt://
Connection URL: localhost:7687
Database user: neo4j
Password: 12345678
```

### Windows 后续启动

普通使用时：

```powershell
neo4j console
```

如果你把 Neo4j 安装成了 Windows 服务，可以使用服务方式启动和停止。服务安装方式与具体安装包有关，本文不强行写死；请参考 Neo4j 官方 Windows 安装说明。

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
