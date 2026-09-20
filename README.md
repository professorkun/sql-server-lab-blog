# SQL Server 上机实验：数据库创建、管理与 T-SQL 基础

> 本文按实训任务整理操作流程。代码中的 `E:\SqlLab` 是示例目录，请替换为本机已存在且 SQL Server 服务账户可访问的练习目录；所有结果应以实际执行和查询回读为准。
>
> 安全提示：`DROP DATABASE`、`WITH ROLLBACK IMMEDIATE`、数据库分离/脱机和 `DBCC SHRINK*` 可能删除数据、断开连接或影响性能。仅在可丢弃的练习数据库中按教师要求操作，不要直接用于生产数据库。

## 一、实训目的

本次实训主要使用 SQL Server 2017 和 SSMS 完成数据库的创建、管理和基础 T-SQL 操作。

主要内容包括：

1. 创建数据库及数据文件、日志文件；
2. 创建多个数据文件的数据库；
3. 添加文件组并扩展数据库存储空间；
4. 完成数据库收缩、分离和附加；
5. 使用自定义数据类型、变量及系统函数；
6. 使用 IF、WHILE 等 T-SQL 流程控制语句；
7. 完成数据库重命名、删除以及文件属性修改。

---

# 二、实验环境

- SQL Server 2017
- SQL Server Management Studio（SSMS）
- Windows 操作系统
- 数据库存储目录：

```text
E:\SqlLab
```

实验开始前先创建该文件夹。

---

# 三、实验过程

## 任务一：创建 ClubDB 数据库

首先创建校园社团管理数据库 `ClubDB`。

要求如下：

| 文件 | 逻辑名称 | 初始大小 | 增长方式 | 最大大小 |
|---|---|---:|---|---|
| 主数据文件 | ClubDB_Primary | 20MB | 每次 3MB | 无限制 |
| 日志文件 | ClubDB_Log | 8MB | 每次 10% | 100MB |

SQL：

```sql
USE master;
GO

CREATE DATABASE ClubDB
ON PRIMARY
(
    NAME = N'ClubDB_Primary',
    FILENAME = N'E:\SqlLab\ClubDB_Primary.mdf',
    SIZE = 20MB,
    FILEGROWTH = 3MB,
    MAXSIZE = UNLIMITED
)
LOG ON
(
    NAME = N'ClubDB_Log',
    FILENAME = N'E:\SqlLab\ClubDB_Log.ldf',
    SIZE = 8MB,
    FILEGROWTH = 10%,
    MAXSIZE = 100MB
);
GO
```

查看数据库文件：

```sql
USE ClubDB;
GO

SELECT
    name AS 逻辑文件名,
    physical_name AS 物理路径,
    size * 8.0 / 1024 AS 当前大小MB,
    growth,
    is_percent_growth
FROM sys.database_files;
GO
```

执行后可通过查询回读文件名、物理路径、大小和增长设置，核对数据库文件是否符合要求。

---

# 任务二：创建 ClubRecruitTemp 测试数据库

创建测试数据库 `ClubRecruitTemp`，其中包含两个数据文件和一个日志文件。

```sql
USE master;
GO

CREATE DATABASE ClubRecruitTemp
ON PRIMARY
(
    NAME = N'Recruit1',
    FILENAME = N'E:\SqlLab\Recruit1.mdf',
    SIZE = 10MB,
    FILEGROWTH = 2MB,
    MAXSIZE = 50MB
),
(
    NAME = N'Recruit2',
    FILENAME = N'E:\SqlLab\Recruit2.ndf',
    SIZE = 8MB,
    FILEGROWTH = 10%,
    MAXSIZE = UNLIMITED
)
LOG ON
(
    NAME = N'RecruitLog',
    FILENAME = N'E:\SqlLab\RecruitLog.ldf',
    SIZE = 5MB,
    FILEGROWTH = 1MB,
    MAXSIZE = 20MB
);
GO
```

查看文件：

```sql
USE ClubRecruitTemp;
GO

SELECT
    name,
    physical_name,
    size * 8.0 / 1024 AS 大小MB
FROM sys.database_files;
GO
```

---

# 任务三：扩展 ClubDB 存储空间

## 1. 创建新的文件组

建立名为 `Group_History` 的文件组：

```sql
USE master;
GO

ALTER DATABASE ClubDB
ADD FILEGROUP Group_History;
GO
```

然后向该文件组添加数据文件：

```sql
ALTER DATABASE ClubDB
ADD FILE
(
    NAME = N'ClubDB_History',
    FILENAME = N'E:\SqlLab\ClubDB_History.ndf',
    SIZE = 15MB,
    FILEGROWTH = 0
)
TO FILEGROUP Group_History;
GO
```

---

## 2. 修改数据文件大小

将 `ClubDB_History` 初始大小调整为 40MB，同时设置每次增长 5MB。

```sql
USE master;
GO

ALTER DATABASE ClubDB
MODIFY FILE
(
    NAME = N'ClubDB_History',
    SIZE = 40MB,
    FILEGROWTH = 5MB
);
GO
```

> 上一步的 `FILEGROWTH = 0` 表示关闭自动增长；本步骤再按练习要求改为每次增长 5MB。关闭自动增长后，文件空间耗尽时写入会失败，实际设置应符合课程要求和容量规划。

查看结果：

```sql
USE ClubDB;
GO

SELECT
    name,
    physical_name,
    size * 8.0 / 1024 AS 当前大小MB,
    growth * 8.0 / 1024 AS 增长大小MB
FROM sys.database_files;
GO
```

---

# 任务四：数据库基础维护

## 1. 收缩 ClubDB 数据库

要求收缩后保留 50% 空闲空间。

```sql
USE ClubDB;
GO

DBCC SHRINKDATABASE (ClubDB, 50);
GO
```

> `target_percent = 50` 表示收缩后目标空闲空间比例约为 50%，不是“缩小 50%”。数据库收缩不应作为常规维护；反复收缩可能增加索引碎片，并造成文件再次自动增长。

---

## 2. 收缩 ClubDB_History 文件

将文件大小收缩到约 25MB。

```sql
USE ClubDB;
GO

DBCC SHRINKFILE (N'ClubDB_History', 25);
GO
```

---

## 3. 分离数据库

为了移动数据库文件，首先将数据库分离。

```sql
USE master;
GO

-- 此操作会断开其他连接并回滚未提交事务；仅用于独立练习库。
ALTER DATABASE ClubDB
SET SINGLE_USER
WITH ROLLBACK IMMEDIATE;
GO

EXEC sp_detach_db N'ClubDB';
GO
```

数据库分离之后，SSMS 中的 `ClubDB` 会消失，但是数据库文件仍然存在。

---

## 4. 移动数据库文件

创建文件夹：

```text
E:\SqlLab\ClubDB_Backup
```

将以下文件移动进去：

```text
ClubDB_Primary.mdf
ClubDB_History.ndf
ClubDB_Log.ldf
```

---

## 5. 重新附加数据库

使用 SQL 将数据库恢复：

```sql
USE master;
GO

CREATE DATABASE ClubDB
ON
(
    FILENAME = N'E:\SqlLab\ClubDB_Backup\ClubDB_Primary.mdf'
),
(
    FILENAME = N'E:\SqlLab\ClubDB_Backup\ClubDB_History.ndf'
),
(
    FILENAME = N'E:\SqlLab\ClubDB_Backup\ClubDB_Log.ldf'
)
FOR ATTACH;
GO
```

查看：

```sql
USE ClubDB;
GO

SELECT
    name,
    physical_name
FROM sys.database_files;
GO
```

重新附加后，再通过 `sys.database_files` 核对文件路径，并确认数据库可正常访问。

---

# 任务五：自定义数据类型及基础 T-SQL

## 1. 创建自定义数据类型

创建 `ClubID_Type`，基础类型为 `CHAR(6)`，不允许为空。

```sql
USE ClubDB;
GO

CREATE TYPE ClubID_Type
FROM CHAR(6) NOT NULL;
GO
```

---

## 2. 局部变量

```sql
DECLARE @ClubName NVARCHAR(20);

SET @ClubName = N'计算机协会';

SELECT @ClubName AS 社团名称;
GO
```

整型变量：

```sql
DECLARE @MemberCount INT;

SET @MemberCount = 32;
SELECT @MemberCount AS 第一次结果;

SET @MemberCount = 128;
SELECT @MemberCount AS 第二次结果;
GO
```

---

## 3. 字符串变量

```sql
DECLARE @ClubNotice NVARCHAR(25);

SET @ClubNotice = N'欢迎新同学加入校园社团！';
SELECT @ClubNotice;

SET @ClubNotice = N'2026秋季社团招新将于本周六举行';
SELECT @ClubNotice;
GO
```

---

## 4. 字符串函数

### LOWER()

```sql
SELECT LOWER('WELCOME TO CAMPUS CLUB') AS 结果;
```

结果：

```text
welcome to campus club
```

### UPPER()

```sql
SELECT UPPER('student club activity') AS 结果;
```

结果：

```text
STUDENT CLUB ACTIVITY
```

### 去除空格并拼接

```sql
DECLARE @S1 NVARCHAR(20);
DECLARE @S2 NVARCHAR(30);

SET @S1 = N'  书法协会  ';
SET @S2 = N'秋季招新开始报名啦！';

SELECT LTRIM(RTRIM(@S1)) + @S2 AS 结果;
GO
```

---

## 5. SUBSTRING 截取字符串

```sql
DECLARE @Text NVARCHAR(50);

SET @Text = N'示例学院校园社团管理系统';

SELECT SUBSTRING(@Text, 5, 6) AS 结果;
GO
```

结果：

```text
校园社团管理
```

---

## 6. 日期函数

```sql
SELECT
    CONVERT(DATE, GETDATE()) AS 当前日期,
    DAY(GETDATE()) AS 当前日,
    MONTH(GETDATE()) AS 当前月份;
GO
```

---

# 任务六：T-SQL 流程控制

## 1. 判断社团等级

```sql
DECLARE @ClubLevel INT;

SET @ClubLevel = 3;

IF @ClubLevel = 1
BEGIN
    PRINT N'校级社团，可跨院招生';
END
ELSE IF @ClubLevel = 2
BEGIN
    PRINT N'院级社团，仅对本院学生开放';
END
ELSE IF @ClubLevel = 3
BEGIN
    PRINT N'兴趣社团，自由招生，无身份限制';
END
ELSE
BEGIN
    PRINT N'社团级别输入错误';
END;
GO
```

---

## 2. 使用 WHILE 计算报名费

共有 150 名成员，每人 20 元。

```sql
DECLARE @Count INT;
DECLARE @TotalMoney INT;

SET @Count = 1;
SET @TotalMoney = 0;

WHILE @Count <= 150
BEGIN
    SET @TotalMoney = @TotalMoney + 20;
    SET @Count = @Count + 1;
END;

SELECT @TotalMoney AS 总报名费;
GO
```

最终结果：

```text
3000
```

---

## 3. 输出斐波那契数列

输出所有小于 100 的斐波那契数。

```sql
DECLARE @A INT = 1;
DECLARE @B INT = 1;
DECLARE @C INT;

PRINT @A;
PRINT @B;

SET @C = @A + @B;

WHILE @C < 100
BEGIN
    PRINT @C;

    SET @A = @B;
    SET @B = @C;
    SET @C = @A + @B;
END;
GO
```

结果：

```text
1
1
2
3
5
8
13
21
34
55
89
```

---

## 4. 求最大公约数

计算 24 和 36 的最大公约数。

```sql
DECLARE @A INT = 24;
DECLARE @B INT = 36;
DECLARE @R INT;

WHILE @B <> 0
BEGIN
    SET @R = @A % @B;
    SET @A = @B;
    SET @B = @R;
END;

SELECT @A AS 最大公约数;
GO
```

运行结果：

```text
12
```

---

# 任务七：数据库修改与删除

## 1. 修改数据库名称

将：

```text
ClubRecruitTemp
```

修改为：

```text
ClubTemp_New
```

SQL：

```sql
USE master;
GO

ALTER DATABASE ClubRecruitTemp
MODIFY NAME = ClubTemp_New;
GO
```

---

## 2. 删除数据库

```sql
USE master;
GO

-- 此操作会断开其他连接并回滚未提交事务；仅用于独立练习库。
ALTER DATABASE ClubTemp_New
SET SINGLE_USER
WITH ROLLBACK IMMEDIATE;
GO

DROP DATABASE ClubTemp_New;
GO
```

执行前先确认当前上下文和数据库名称，并确保这只是可删除的练习库。

---

## 3. 修改日志文件增长方式

将 `ClubDB_Log` 修改为每次增长 2MB。

```sql
USE master;
GO

ALTER DATABASE ClubDB
MODIFY FILE
(
    NAME = N'ClubDB_Log',
    FILEGROWTH = 2MB
);
GO
```

---

## 4. 修改数据文件逻辑名称

将：

```text
ClubDB_History
```

修改为：

```text
ClubDB_HistoryData
```

代码：

```sql
USE master;
GO

ALTER DATABASE ClubDB
MODIFY FILE
(
    NAME = N'ClubDB_History',
    NEWNAME = N'ClubDB_HistoryData'
);
GO
```

---

## 5. 修改物理文件名称

先修改数据库记录的物理文件路径：

```sql
ALTER DATABASE ClubDB
MODIFY FILE
(
    NAME = N'ClubDB_HistoryData',
    FILENAME =
    N'E:\SqlLab\ClubDB_Backup\ClubDB_HistoryData.ndf'
);
GO
```

然后将数据库设置为离线：

```sql
ALTER DATABASE ClubDB
SET OFFLINE
WITH ROLLBACK IMMEDIATE;
GO
```

此操作会断开现有连接；执行前应确认已完成备份且文件移动目标正确。若 SQL Server 无法访问更新后的路径，数据库将不能正常上线。

在 Windows 文件资源管理器中，将：

```text
ClubDB_History.ndf
```

重命名为：

```text
ClubDB_HistoryData.ndf
```

最后重新上线：

```sql
ALTER DATABASE ClubDB
SET ONLINE;
GO
```

检查：

```sql
USE ClubDB;
GO

SELECT
    name AS 逻辑名称,
    physical_name AS 物理路径
FROM sys.database_files;
GO
```

---

# 四、常见问题与排查思路

以下是数据库创建与文件管理中常见的现象及处理思路：

### 1. 数据库分离后在 SSMS 中消失

分离 `ClubDB` 后，数据库会从 SSMS 的实例列表中消失。

原因是数据库分离操作只是解除 SQL Server 实例与数据库文件的关联，并不会删除 `.mdf`、`.ndf` 和 `.ldf` 文件。

解决方法是使用 `CREATE DATABASE ... FOR ATTACH` 将数据库重新附加。

---

### 2. 文件无法移动

数据库仍处于在线状态时，数据库文件可能被 SQL Server 占用，无法移动。

因此必须先执行数据库分离或者将数据库设置为离线，再移动文件。

---

### 3. CREATE DATABASE 提示数据库已存在

如果之前已经使用图形界面建立了同名数据库，再执行 SQL 创建会提示数据库已经存在。

解决方法是删除原数据库后重新创建，或者只进行其中一种创建方式。

---

### 4. 文件路径错误

数据库创建或附加时，如果：

```text
E:\SqlLab
```

目录不存在，会导致创建失败。

因此实验前需要提前建立对应目录。

---

# 五、检查点与结果分析

执行各项任务后，可用以下检查点确认操作结果：

- 查询 `sys.database_files`，核对逻辑名称、物理路径、文件大小和增长设置。
- 分离、移动并重新附加后，再次检查文件路径，并确认数据库能够访问。
- 执行变量、字符串、日期和流程控制语句后，对照输出检查表达式与循环边界。

通过数据库文件与文件组的操作，可以观察逻辑文件名、物理文件和数据库对象之间的对应关系。

T-SQL 部分涵盖局部变量、字符串函数、日期函数、`IF` 分支和 `WHILE` 循环，可用于练习基本程序控制。

这些基础操作为后续学习数据库表设计、数据查询和数据库程序设计提供了练习入口。

---

# 六、总结

本次实训涉及 SQL Server 数据库管理中非常基础且重要的操作，包括：

```text
CREATE DATABASE
ALTER DATABASE
DROP DATABASE
DBCC SHRINKDATABASE
DBCC SHRINKFILE
CREATE TYPE
IF...ELSE
WHILE
字符串函数
日期函数
数据库分离与附加
```

完成并回读检查各项任务后，可进一步梳理数据库创建、修改、维护与物理文件管理之间的关系。本文中的语句和输出仍应以实际练习环境验证为准。

## 参考资料

- [Microsoft Learn：创建数据库](https://learn.microsoft.com/en-us/sql/relational-databases/databases/create-a-database?view=sql-server-ver17)
- [Microsoft Learn：DBCC SHRINKDATABASE](https://learn.microsoft.com/en-us/sql/relational-databases/database-console-commands/dbcc-shrinkdatabase-transact-sql?view=sql-server-ver17)
- [Microsoft Learn：分离数据库](https://learn.microsoft.com/en-us/sql/relational-databases/databases/detach-a-database?view=sql-server-ver17)
- [Microsoft Learn：ALTER DATABASE 文件与文件组](https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-database-transact-sql-file-and-filegroup-options?view=sql-server-ver17)
