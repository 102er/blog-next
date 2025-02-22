# 背景

mysql5.7官方已经不再支持，要求业务升级mysql8.0

### 新版本功能特性

**mysql8.0提供新功能：**https://dev.mysql.com/doc/refman/8.0/en/mysql-nutshell.html

mysql5.7和mysql8不兼容列表：当前版本mysql5.7.44版本 --> 目标版本mysql8.0.32版本，两个版本存在一定的不兼容性，强行升级会引起问题，不兼容项列表：

- 不得有使用过时的数据类型或函数的表。

- 不得有孤立的 *.frm 文件。

- 触发器不得具有缺失的或空的定义程序或无效的创建上下文。

- 不得有使用不支持本机分区的存储引擎的分区表。

- 不得出现关键字或保留关键字违规情况。MySQL 8.0 中可能会保留一些以前未保留的关键字。有关更多信息，请参阅 MySQL 文档中的[关键字和保留关键字](https://dev.mysql.com/doc/refman/8.0/en/keywords.html)。参考mysql兼容性检查结果

- MySQL 5.7 `mysql` 系统数据库中不得有与 MySQL 8.0 数据字典使用的表同名的表。

- `sql_mode` 系统变量设置中不得定义过时的 SQL 模式。 --- 确认，可以通过配置解决

  | **SQL_MODE模式**           | **作用**                                                     | **mysql8.0.32(目标版本)** | **mysql5.7.44(在用版本)** | 解决方案                           |
  | -------------------------- | ------------------------------------------------------------ | ------------------------- | ------------------------- | ---------------------------------- |
  | STRICT_TRANS_TABLES        | 严格模式控制 MySQL 如何处理数据更改语句（如 INSERT或 ） 中的无效值或缺失值UPDATE。值可能由于多种原因而无效。例如，它可能具有错误的列数据类型，或者可能超出范围。当要插入的新行不包含定义中 NULL没有显式子句的非 列 的值时，会缺少值。（对于列，如果值缺失，则插入 。）严格模式还会影响 DDL 语句，例如。 DEFAULTNULLNULLCREATE TABLE 如果严格模式未生效，MySQL 将为无效或缺失值插入调整后的值并生成警告。 在严格模式下，您可以使用 INSERT IGNORE 或来产生此行为UPDATE IGNORE。 SELECT 对于不改变数据 的语句，无效值在严格模式下会生成警告，而不是错误。 如果尝试创建超出最大密钥长度的密钥，则严格模式会产生错误。如果未启用严格模式，则会导致警告，并将密钥截断为最大密钥长度。 严格模式不会影响是否检查外键约束。foreign_key_checks可用于此目的。 | 默认启用                  | 默认启用                  | **关闭**                           |
  | ONLY_FULL_GROUP_BY         | 拒绝选择列表、 `HAVING`条件或列表引用未在子句中命名且在功能上不依赖于（由） 列 唯一确定的`ORDER BY`非聚合列的查询。 | 默认启用                  | 不启用                    | 手动关闭                           |
  | NO_ZERO_IN_DATE            | 该NO_ZERO_IN_DATE模式会影响服务器是否允许年份部分非零但月份或日期部分为 0 的日期。（此模式影响 或 之类的日期'2010-00-01'， '2010-01-00'但不影响 '0000-00-00'。要控制服务器是否允许'0000-00-00'，请使用 NO_ZERO_DATE模式。） 其效果还取决于是否启用了严格 SQL 模式。 | 默认启用，弃用的模式      | 不启用                    | 手动关闭                           |
  | NO_ZERO_DATE               | 该NO_ZERO_DATE模式会影响服务器是否允许 '0000-00-00'作为有效日期。 其效果还取决于是否启用了严格 SQL 模式。 | 默认启用                  | 不启用                    |                                    |
  | ERROR_FOR_DIVISION_BY_ZERO | 该 ERROR_FOR_DIVISION_BY_ZERO 模式会影响除以零的处理，包括 。对于数据更改操作（， ） 其效果还取决于是否启用了严格 SQL 模式。 | 默认启用                  | 不启用                    |                                    |
  | NO_ENGINE_SUBSTITUTION     | 当语句（例如CREATE TABLE或）ALTER TABLE指定已禁用或未编译的存储引擎时，控制默认存储引擎的自动替换。 | 需要手动启用              | 启用                      | 新数据库实例，查看是否有开启该模式 |

- 不得有包含超过 255 个字符或 1020 个字节的单个 `ENUM` 或 `SET` 列元素的表或存储过程。

- 在升级到 MySQL 8.0.13 或更高版本之前，不得有驻留在共享 InnoDB 表空间中的表分区。
- MySQL 8.0.12 或更低版本中不得有对 `ASC` 子句使用 `DESC` 或 `GROUP BY` 限定符的查询和存储程序定义。
- 您的 MySQL 5.7 安装不得使用 MySQL 8.0 不支持的功能。有关更多信息，请参阅 MySQL 文档中的 [MySQL 8.0 中删除的功能](https://dev.mysql.com/doc/refman/8.0/en/mysql-nutshell.html#mysql-nutshell-removals)。--- 确认，暂无发现
- 不得有超过 64 个字符的外键约束名称。
- 要获得改进的 Unicode 支持，请考虑将使用 `utf8mb3` 字符集的对象转换为使用 `utf8mb4` 字符集。`utf8mb3` 字符集已弃用。此外，请考虑对字符集引用使用 `utf8mb4` 而不是 `utf8`，因为 `utf8` 当前是 `utf8mb3` 字符集的别名。--- 有些数据库使用`utf8，参考数据库版本兼容性检查`

#### 数据库版本兼容性检查

目前 MySQL 官方提供的 MySQL Shell 中有专门用作 MySQL 升级兼容性检查的函数 util.check_for_server_upgrade()，我们可以使用该检查函数在升级之前对相应实例进行初步检查，如果检查中出现升级不兼容项目，输出的日志中会有详细的信息记录，并且会在日志末尾对不兼容项目的数量按照 Error、Warnings 以及 Notices 进行分组统计其数量。

```
./mysqlsh  root@xxx:3306  --py -e "util.check_for_server_upgrade()" >> mysql8.32.log
```

目前对线上备份库进行检查，其中

-  Error 项即为可以直接导致升级失败的项目
- Warning 则是不会导致升级失败但是需要在升级或后续的使用过程额外关注或修改的项目 -- 安排调整

```
Errors:   0
Warnings: 3781
Notices:  0
```

| **检查项**                                                   | **异常项**                                                   | **解决方案**                                |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------- |
| 关键字冲突  Warning: The following objects have names that conflict with new reserved    keywords. Ensure queries sent by your applications use `quotes` when | scdnweb_cp.member - Table name  scdnweb_cp.attribute.member - Column name  scdnweb_cp.domain_action.rank - Column name  scdnweb_cp.domain_level.rank - Column name  scdnweb_cp.domain_level_v2.rank - Column name  scdnweb_cp_v4.cpv4_tjkd_web_domain.rank - Column name  scdnweb_zhulu.zlkj_arcrank.rank - Column name  scdnweb_zhulu.zlkj_quickentry.groups - Column name  wafrule_prod.auth_users.groups - Column name | 一般使用orm，会使用``转义，目前没有发现问题 |
| 字符集utfmb3(utf8)  Warning: The following objects use the utf8mb3 character set. It is    recommended to convert them to use utf8mb4 instead, for improved Unicode | 1000+个数据库、表、字段                                      | 上线前，先调整数据库字字符集                |
| 这些模式禁止在日期、日期时间、时间戳字段中使用无效的 "零" 值（例如 `'0000-00-00'` 或 `'0000-00-00 00:00:00'`），这些值通常表示无效或未定义的日期  Warning: By default zero date/datetime/timestamp values are no longer allowed    in MySQL, as of 5.7.8 NO_ZERO_IN_DATE and NO_ZERO_DATE are included in    SQL_MODE by default. These modes should be used with strict mode as they will    be merged with strict mode in a future release. If you do not include these    modes in your SQL_MODE setting, you are able to insert    date/datetime/timestamp values that contain zeros. It is strongly advised to    replace zero values with valid ones, as they may not work correctly in the    future. | scdnweb_admin_v5.adminv5_auth_apps.created_at - column has zero default    value: 0000-00-00 00:00:00  scdnweb_admin_v5.adminv5_auth_apps.updated_at - column has zero default    value: 0000-00-00 00:00:00  scdnweb_admin_v5.adminv5_auth_user.last_login - column has zero default    value: 0000-00-00 00:00:00  scdnweb_admin_v5.adminv5_auth_user.error_passwd_last - column has zero    default value: 0000-00-00 00:00:00  scdnweb_admin_v5.adminv5_auth_user_apps.created_at - column has zero default ... | 上线前，调整表结构                          |