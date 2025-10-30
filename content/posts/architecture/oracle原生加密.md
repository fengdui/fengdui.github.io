---
title: "oracle原生加密"
date: "2024-04-01"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

TDE column encryption was first introduced in Oracle Database 10g release 2 (10.2). To use this feature, you must be running Oracle Database 10g release 2 (10.2) or higher.
TDE tablespace encryption was introduced in Oracle Database 11g release 1 (11.1). To use this feature, you must be running Oracle Database 11g release 1 (11.1) or higher.

生成钱夹
sqlnet.ora文件 参数ENCRYPTION_WALLET_LOCATION  指定钱夹的位置
ALTER SYSTEM SET ENCRYPTION KEY ["certificate_ID"] IDENTIFIED BY "password"
这里rac多节点的情况 需要到每一个节点，即实例都做一遍。
由Oracle Wallet Manager或orapki创建的wallets使用标准PKCS12格式来存储X.509证书和私钥。wallets存储在名为"ewallet.p12"的文件中。如果您在wallets中启用自动登录，则会在文件"cwallet.sso"中创建wallets的混淆副本，然后无需提供密码即可使用。请注意，您必须使用名为"OraclePKI"的Oracle PKI提供程序从Java访问Oracle wallets。
钱包的作用是自动对具体的列进行加密，当钱包为打开状态，则可以正常访问加密列的数据，当钱包为关闭状态，则无法访问加密列的数据，设置加密秘钥后，会在钱包目录下产生一个.p12的加密秘钥文件，一定要进行备份并妥善保管。
Oracle 11gR2中 RAC节点能够共享钱包。Oracle建议在共享文件系统上创建钱包，这样允许所有实例访问相同的共享钱包，无需手动复制和同步所有节点上的钱包。
Oracle RAC中一个实例对钱包进行操作（如打开或关闭钱包），它会为Oracle RAC中所有实例打开或关闭。
12c
select CDB from v$database 是否是cdb

设置主密钥
alter system set key identified by "oracle";

https://docs.oracle.com/cd/E11882_01/network.112/e40393/asotrans.htm#ASOAG9522

一、初始化流程
1、为CDB创建ewallet.p12
在CDB中执行：
ADMINISTER KEY MANAGEMENT CREATE KEYSTORE '/oracle/12c/product/12.2.0/dbhome_1/network/admin' IDENTIFIED BY "123456" ;
命令成功后，将生成ewallet.p12文件，大小2400字节左右。V$ENCRYPTION_WALLET.status列状态值由NOT_AVAILABLE变为CLOSED。

2、创建cwallet.sso，自动打开ewallet
在CDB中执行：
ADMINISTER KEY MANAGEMENT CREATE AUTO_LOGIN KEYSTORE FROM KEYSTORE '/oracle/12c/product/12.2.0/dbhome_1/network/admin' IDENTIFIED BY "123456";
成功后生成cwallet.sso，大小比ewallet.p12大几十字节。

3、打开密码文件介绍：
如已经创建cwallet.sso，密码文件将自动打开，此步骤可跳过。
在CDB中执行：
ADMINISTER KEY MANAGEMENT SET KEYSTORE OPEN IDENTIFIED BY "123456" CONTAINER=ALL;
如只想针对某一PDB，可以去掉“CONTAINER=ALL”。
打开成功后，状态由CLOSED变为OPEN_NO_MASTER_KEY。同时WALLET_TYPE列值为PASSWORD。
需要注意：
（1）、在CDB中关闭密码文件，所有其他PDB的自动关闭。但在CDB打开，其他PDB不会自动打开。
（2）、CONTAINER=ALL选项，只会作用于正在Open状态的PDB，如某PDB是MOUNT状态，CONTAINER=ALL对它不起作用。

4、创建MastKey。
每个PDB一个（CDB也有一个），必须在密码文件打开后才可创建。
ADMINISTER KEY MANAGEMENT SET KEY FORCE KEYSTORE IDENTIFIED BY "123456" WITH BACKUP USING 'pdb4';

二、12CR1 初始化流程
1、为CDB创建ewallet.p12
在CDB中执行：
ADMINISTER KEY MANAGEMENT CREATE KEYSTORE '/oracle/12c/product/12.2.0/dbhome_1/network/admin' IDENTIFIED BY "123456" ;
命令成功后，将生成ewallet.p12文件，大小2400字节左右。V$ENCRYPTION_WALLET.status列状态值由NOT_AVAILABLE变为CLOSED。

2、打开密码文件介绍：
如已经创建cwallet.sso，密码文件将自动打开，此步骤可跳过。
在CDB中执行：
ADMINISTER KEY MANAGEMENT SET KEYSTORE OPEN IDENTIFIED BY "123456" CONTAINER=ALL;
如只想针对某一PDB，可以去掉“CONTAINER=ALL”。
打开成功后，状态由CLOSED变为OPEN_NO_MASTER_KEY。同时WALLET_TYPE列值为PASSWORD。
需要注意：
（1）、在CDB中关闭密码文件，所有其他PDB的自动关闭。但在CDB打开，其他PDB不会自动打开。
（2）、CONTAINER=ALL选项，只会作用于正在Open状态的PDB，如某PDB是MOUNT状态，CONTAINER=ALL对它不起作用。
3、创建cwallet.sso，自动打开ewallet
在CDB中执行：
ADMINISTER KEY MANAGEMENT CREATE AUTO_LOGIN KEYSTORE FROM KEYSTORE '/oracle/12c/product/12.2.0/dbhome_1/network/admin' IDENTIFIED BY "123456";
成功后生成cwallet.sso，大小比ewallet.p12大几十字节。

4、创建MastKey。
每个PDB一个（CDB也有一个），必须在密码文件打开后才可创建。
ADMINISTER KEY MANAGEMENT SET KEY IDENTIFIED BY "123456" WITH BACKUP USING 'pdb4';

三、12CR1  新增PDB初始化流程
1、删除cwallet.sso文件

2、在CDB中关闭密钥文件
ADMINISTER KEY MANAGEMENT SET KEYSTORE CLOSE;

3、在CDB中手动打开密钥文件
ADMINISTER KEY MANAGEMENT SET KEYSTORE OPEN IDENTIFIED BY "123456" CONTAINER=ALL;
在所有PDB中打开密钥

2、3步是为了将密钥的打开方式，由自动打开，转为手动打开。转为手动打开后，就可以再次创建cwallet.sso文件了。

4、在CDB中再次创建cwallet.sso文件：
ADMINISTER KEY MANAGEMENT CREATE AUTO_LOGIN KEYSTORE FROM KEYSTORE '/oracle/app/oracle/product/12.1.0/db_1/network/admin' IDENTIFIED BY "123456";

5、切换到新建的PDB，在ewallet.p12中创建它的密钥：
ADMINISTER KEY MANAGEMENT SET KEY IDENTIFIED BY "123456" WITH BACKUP USING 'pdb2';


手动操作TDE流程
1、administer key management create keystore '/oracle/app/product/12.1.0/db_1/network/admin' identified by "3644E830E2@aaa.com"
2、administer key management set keystore open identified by "3644E830E2@aaa.com" container=all;
3、administer key management create auto_login keystore from keystore '/oracle/app/product/12.1.0/db_1/network/admin' identified by "3644E830E2@aaa.com"
4、
#if 12.1
ADMINISTER KEY MANAGEMENT SET KEY IDENTIFIED BY "3644E830E2@aaa.com" WITH BACKUP USING 'aaa'
#else
ADMINISTER KEY MANAGEMENT SET KEY FORCE KEYSTORE IDENTIFIED BY "3644E830E2@aaa.com" WITH BACKUP USING 'aaa';

清理流程
清理流
1、
set lin 1000
col WRL_PARAMETER for a40
col status for a20
col WRL_TYPE for a10
col WALLET_TYPE for a20
select * from v$encryption_wallet;
2、
#autologin
administer key management set keystore close;
rm cwallet.sso
#else
administer key management set keystore close identified by "3644E830E2@aaa.com" container=all;
3、rm ewallet.p12
4、echo > sqlnet.ora
5 12cr2
alter system set "_db_discard_lost_masterkey"=TRUE SCOPE=MEMORY;

              oracle 11g 
                 打开加密
                      ALTER SYSTEM SET ENCRYPTION KEY IDENTIFIED BY "3644E830E2@aaa.com"
                 手动创建cwallet
                   1、orapki wallet create -wallet Oracle安装路径/network/admin -auto_login
                   2、要在操作系统中手动执行这个命令，会要求输入密码，密码是这个：3644E830E2@aaa.com

注意事项
新增PDB添加
一、Oracle 12C R1版本：（操作过程业务离线)
1、登录PDB服务名
2、查询加密状态：select STATUS from v$encryption_wallet;
3、若STATUS字段为CLOSED，则打开密钥
administer key management set keystore open identified by "3644E830E2@aaa.com";若STATUS字段为OPEN_NO_MASTER_KEY，进入如下流程
1）查找cwallet.sso所在目录
select WRL_PARAMETER from v$encryption_wallet;
rm cwallet.sso
2）进入CDB，关闭密钥文件
ADMINISTER KEY MANAGEMENT SET KEYSTORE CLOSE;
3）在CDB中手动打开密钥文件
administer key management set keystore open identified by "3644E830E2@aaa.com" container=all;
4）在CDB中再次创建cwallet.sso文件(标红部分实际环境注意)
administer key management create auto_login keystore from keystore '/oracle/app/product/12.1.0/db_1/network/admin' identified by "3644E830E2@aaa.com";
4、页面重新注册新增PDB，初始化
二、Oracle 12C R2版本(新增PDB业务离线)
支持新增PDB
三、Oracle 18C 版本(新增PDB业务离线)
支持新增PDB
CDB和PDB分离注册，注意账户权限，CDB使用公共账号，PDB使用本地账号，CDB初始化为前提，才能初始化PDB
所需权限
初始化
CREATE USER mcinituser IDENTIFIED BY password;
GRANT EXECUTE on sys.DBMS_SYSTEM TO mcinituser;
GRANT EXECUTE on sys.UTL_FILE TO mcinituser;
GRANT CREATE SESSION,ALTER SYSTEM,CREATE ANY DIRECTORY,DROP ANY DIRECTORY,CREATE PROCEDURE,SELECT ANY DICTIONARY TO mcinituser;
12C及以上版本，公共用户请修改为如下权限：
CREATE USER c##mcinituser IDENTIFIED BY password;
GRANT EXECUTE on sys.DBMS_SYSTEM TO c##mcinituser CONTAINER=ALL;
GRANT EXECUTE on sys.UTL_FILE TO c##mcinituser CONTAINER=ALL;
GRANT CREATE SESSION,ALTER SYSTEM,CREATE ANY DIRECTORY,DROP ANY DIRECTORY,CREATE PROCEDURE,SELECT ANY DICTIONARY,ADMINISTER KEY MANAGEMENT,SET CONTAINER TO c##mcinituser CONTAINER=ALL;
12C及以上版本，本地用户请修改为如下权限：
ALTER SESSION SET CONTAINER = mcinituserpdb;
CREATE USER mcinituser IDENTIFIED BY password;
GRANT EXECUTE on sys.DBMS_SYSTEM TO mcinituser;
GRANT EXECUTE on sys.UTL_FILE TO mcinituser;
GRANT CREATE SESSION,ALTER SYSTEM,CREATE ANY DIRECTORY,DROP ANY DIRECTORY,CREATE PROCEDURE,SELECT ANY DICTIONARY,ADMINISTER KEY MANAGEMENT,SET CONTAINER TO mcinituser;
加密权限
CREATE USER myuser IDENTIFIED BY password;
GRANT CREATE SESSION,CREATE TABLESPACE,ALTER TABLESPACE,ALTER DATABASE,UNLIMITED TABLESPACE,SELECT ANY DICTIONARY,ALTER ANY INDEX,CREATE ANY INDEX,ALTER ANY TABLE TO myuser;
12C及以上版本，公共用户请修改为如下权限：
CREATE USER c##myuser IDENTIFIED BY password;
GRANT CREATE SESSION,CREATE TABLESPACE,ALTER TABLESPACE,ALTER DATABASE,UNLIMITED TABLESPACE,SELECT ANY DICTIONARY,ALTER ANY INDEX,CREATE ANY INDEX,ALTER ANY TABLE,SET CONTAINER TO c##myuser CONTAINER=ALL;
12C及以上版本，本地用户请修改为如下权限：
ALTER SESSION SET CONTAINER = myuserpdb;
CREATE USER myuser IDENTIFIED BY password;
GRANT CREATE SESSION,CREATE TABLESPACE,ALTER TABLESPACE,ALTER DATABASE,UNLIMITED TABLESPACE,SELECT ANY DICTIONARY,ALTER ANY INDEX,CREATE ANY INDEX,ALTER ANY TABLE,SET CONTAINER TO myuser;
 