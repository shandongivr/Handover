# 出账说明
* [移网出账](#移网出账)
* [固网出账](#固网出账)
  - [97数据](#每月最后一天97用户数据导入)
  - [强制包月](#每月最后一天强制包月)
  - [322话单](#322话单转移)
  - [出账前处理](#每月1日出账前处理)
  - [报表统计](#报表统计)
### 移网出账
每月25号凌晨程序自动出账，白天上班后到`/q:/flow/bill/otherNet/`路径下取出账数据发送到地市，地市收件人可以查看上个月邮件里面发送的；
### 固网出账 
#### 每月最后一天97用户数据导入
```sql
--1、需要处理的97数据
----1)枣庄：由孙世红通过邮箱发送
----2)淄博：每月最后1日晚上19时左右登陆FTP,到目录q:/flow/zibo/97mpuser/97/下下载当月数据(看文件后缀)


--2、处理前提：
----1)97用户表bill_mp_97,表结构分别如下：
CREATE TABLE BILL.BILL_MP_97
(
    FD_USER VARCHAR2(32) NOT NULL, --用户号码,必须带区号,例如06323169799
    FD_ITEM VARCHAR2(32) NOT NULL, --包月扣费编码,必须带区号,例如06321684560
    FD_LDRDATE DATE DEFAULT SYSDATE NULL, --导入时间,不赋值则默认为当前系统时间
    FD_EXPDATE DATE NULL, --97用户过期时间,由地市提供
    PRIMARY KEY(FD_USER,FD_ITEM)
)
----2)ctl_97.ctl文件,内容如下：
load data
infile '97.txt'
badfile '97.bad'
append into table BILL_MP_97
fields terminated by "," 
trailing nullcols
(
    FD_USER,
    FD_ITEM,
    FD_EXPDATE Timestamp "YYYYMMDD"
  )
----3)将以下语句保存为ldr_97.bat文件,注意ctl文件存放路径必须正确。如下:
sqlldr bill/sdbill_jnoradba_0808@jnoradb control=q:/flow/bill/出账前处理/ctl_97.ctl errors=10
pause


--3、导入bill_mp_97表：
----1)将数据处理为：用户号码,计费编码,过期时间（注意格式为YYYYMMDD,如果说明2013年1月为最后一月,则此过期时间应写为下个月的一号：20130201）,并保存为97.txt，和ctl_97.ctl文件放在同一目录下
----2)备份上月97数据
INSERT INTO BILL.BILL_MP_97_BAK
SELECT FD_USER,FD_ITEM,FD_LDRDATE,FD_EXPDATE,SYSDATE
   FROM BILL.BILL_MP_97;
COMMIT;
----3)清空bill_mp_97表：
TRUNCATE TABLE BILL.BILL_MP_97;
COMMIT;
----4)执行ldr_97.bat,导入97用户数据


--4、导入bill_mp_user表
----1)删除bill_mp_user表中现有97用户数据
SELECT count(*) FROM BILL.BILL_MP_USER A
 WHERE EXISTS(SELECT 1 FROM BILL.BILL_MP_97 B
               WHERE B.FD_USER =A.FD_USERID
                 AND (B.FD_ITEM=A.FD_BILLITEM
                  OR A.FD_BILLITEM LIKE B.FD_ITEM||'%'));
                  
DELETE FROM BILL.BILL_MP_USER A
 WHERE EXISTS(SELECT 1 FROM BILL.BILL_MP_97 B
               WHERE B.FD_USER =A.FD_USERID
                 AND (B.FD_ITEM=A.FD_BILLITEM
                  OR A.FD_BILLITEM LIKE B.FD_ITEM||'%'));
COMMIT;
----2)将97数据插入bill_mp_user表(注意枣庄的业务插入时，需要给业务号加上后缀0,同时要保证导入的FD_ITEM存在于actinfo表中)：
INSERT INTO BILL.BILL_MP_USER(FD_USERID,FD_BILLPHONE,FD_BILLITEM,FD_STATUS,FD_BILLMONTH,FD_EXPDATE)
     SELECT FD_USER,FD_USER,
            DECODE(SUBSTR(FD_ITEM,1,4),'0632',CONCAT(FD_ITEM,'0'),FD_ITEM),
            0,TO_CHAR(ADD_MONTHS(SYSDATE,-1),'YYYYMM'),FD_EXPDATE
       FROM BILL.BILL_MP_97
      WHERE EXISTS(SELECT 1 FROM BILL.BILL_ACTINFO A
                    WHERE A.C_BILLITEM=FD_ITEM);
COMMIT;

97不入库！
```
#### 每月最后一天强制包月  
```sql
declare
c int :=0;
begin
  P_TAS_MPFORCE(c);
end;
```
#### 322话单转移
> 将9600322业务的当月话单数据从29服务器的bill_realtimebill中导出到13的bill_formalbill当中（数据保留切勿删除）。（两套平台都有一个bill_formalbill_testst表，用该表进行中转，使用insert语句导入数据，导入之前要先过滤一下红名单用户，只导入非红名单用户的话单）

##### 清空bill_formalbill_testst表（29，13机器上的数据都清空）
```sql
truncate table bill_formalbill_testst;
```
##### 322数据导入bill_formalbill_testst表（29机器上）
```sql
insert into bill_formalbill_testst (select * from bill_realtimebill where c_endtime between trunc(add_months(sysdate,-1),'mm') and trunc(sysdate,'mm'));
```
##### exp、imp数据（29机器上）
```
exp bill/sdbill_jnoradba_0808@jnivrapp  file=e:\data_test.dmp tables=bill_formalbill_testst
imp bill/sdbill_jnoradba_0808@192.168.53.13  file=e:\data_test.dmp tables=bill_formalbill_testst ignore=y
```
##### 剔除公免用户（20机器上连13的库）
```sql
select * from bill_formalbill_testst where c_billphone in(select c_phone from bill_reduser where c_redtype='7');
delete bill_formalbill_testst where c_billphone in(select c_phone from bill_reduser where c_redtype='7');
```
##### 导入正式话单表（20机器上连13的库）
```sql
insert into bill_formalbill (select * from bill_formalbill_testst where c_endtime between trunc(add_months(sysdate,-1),'mm') and trunc(sysdate,'mm'));
```
#### 每月1日出账前处理
```sql
--注意：以下所有update或delete语句,都必须先用select语句确定结果后再执行！！
莱芜包月扣费查询

--1、过期包月用户状态更新
SELECT count(*) FROM BILL.BILL_MP_USER
 WHERE FD_EXPDATE<SYSDATE
   AND FD_STATUS=0;
   
UPDATE BILL.BILL_MP_USER
   SET FD_STATUS=1,FD_UPDATEDATE=SYSDATE,FD_CANCELDATE=SYSDATE
 WHERE FD_EXPDATE<SYSDATE
   AND FD_STATUS=0;
COMMIT;

--2、97用户免单(必须在每月1日话单转储后、固网出账前执行)
 SELECT count(*) FROM BILL.BILL_FORMALBILL F
  WHERE F.C_FEESUM>0
    AND F.C_ENDTIME 
BETWEEN TRUNC(ADD_MONTHS(SYSDATE,-1),'MM')
    AND TRUNC(SYSDATE,'MM')
    AND EXISTS(SELECT 1 FROM BILL.BILL_MP_97 M
                WHERE M.FD_USER=F.C_BILLPHONE 
                  AND M.FD_ITEM = F.C_CALLED);
                               
 UPDATE BILL.BILL_FORMALBILL F
    SET F.C_FEESUM=0
  WHERE F.C_FEESUM>0
    AND F.C_ENDTIME 
BETWEEN TRUNC(ADD_MONTHS(SYSDATE,-1),'MM')
    AND TRUNC(SYSDATE,'MM')
    AND EXISTS(SELECT 1 FROM BILL.BILL_MP_97 M
                WHERE M.FD_USER=F.C_BILLPHONE 
                  AND M.FD_ITEM = F.C_CALLED);
COMMIT;

--3、公免用户免收9600322业务费用(固网出账前必须执行) 
 select count(*) from BILL.BILL_FORMALBILL F
  WHERE F.C_BILLITEM LIKE '9600322%'
    AND F.C_FEESUM>0
    AND F.C_ENDTIME 
BETWEEN TRUNC(ADD_MONTHS(SYSDATE,-1),'MM')
    AND TRUNC(SYSDATE,'MM')
    AND EXISTS(SELECT 1 FROM BILL.BILL_REDUSER M
                WHERE M.C_PHONE=F.C_BILLPHONE 
                  AND M.C_REDTYPE=7);

 UPDATE BILL.BILL_FORMALBILL F
    SET F.C_FEESUM=0
  WHERE F.C_BILLITEM LIKE '9600322%'
    AND F.C_FEESUM>0
    AND F.C_ENDTIME 
BETWEEN TRUNC(ADD_MONTHS(SYSDATE,-1),'MM')
    AND TRUNC(SYSDATE,'MM')
    AND EXISTS(SELECT 1 FROM BILL.BILL_REDUSER M
                WHERE M.C_PHONE=F.C_BILLPHONE 
                  AND M.C_REDTYPE=7);
COMMIT;


--4、删除办事处成员话单
 SELECT count(*) FROM BILL.BILL_FORMALBILL F
  WHERE F.C_ENDTIME 
BETWEEN TRUNC(ADD_MONTHS(SYSDATE,-1),'MM') AND TRUNC(SYSDATE,'MM')
    AND EXISTS(SELECT 1 FROM BILL.BILL_TEST_PHONE T WHERE T.HM=F.C_BILLPHONE);

 DELETE FROM BILL.BILL_FORMALBILL F
  WHERE F.C_ENDTIME 
BETWEEN TRUNC(ADD_MONTHS(SYSDATE,-1),'MM') AND TRUNC(SYSDATE,'MM')
    AND EXISTS(SELECT 1 FROM BILL.BILL_TEST_PHONE T WHERE T.HM=F.C_BILLPHONE);
 COMMIT;
 
/*--4、超大话单处理
 SELECT count(*) FROM BILL.BILL_FORMALBILL
  WHERE C_RATETYPE=1
    AND C_FEESUM>0
    AND C_DURATION>3500
    AND C_ENDTIME 
BETWEEN TRUNC(ADD_MONTHS(SYSDATE,-1),'MM')
    AND TRUNC(SYSDATE,'MM');

 UPDATE BILL.BILL_FORMALBILL
    SET C_FEESUM=60*C_RATE
  WHERE C_RATETYPE=1
    AND C_FEESUM>0
    AND C_DURATION>3500
    AND C_ENDTIME
BETWEEN TRUNC(ADD_MONTHS(SYSDATE,-1),'MM')
    AND TRUNC(SYSDATE,'MM');
 COMMIT;*/
 
--5、超短话单处理
 SELECT count(*) FROM BILL.BILL_FORMALBILL
  WHERE C_RATETYPE=1
    AND C_FEESUM>0
    AND C_DURATION<6
    AND C_ENDTIME 
BETWEEN TRUNC(ADD_MONTHS(SYSDATE,-1),'MM')
    AND TRUNC(SYSDATE,'MM');

 UPDATE BILL.BILL_FORMALBILL
    SET C_FEESUM=0
  WHERE C_RATETYPE=1
    AND C_FEESUM>0
    AND C_DURATION<6
    AND C_ENDTIME
BETWEEN TRUNC(ADD_MONTHS(SYSDATE,-1),'MM')
    AND TRUNC(SYSDATE,'MM');
 COMMIT;
 
--6、重单查询
 SELECT A.C_CALLER,A.C_BILLITEM,A.C_BILLPHONE,A.C_CALLED,
        A.C_DURATION,A.C_FEESUM,A.C_BEGINTIME,A.C_ENDTIME 
   FROM BILL.BILL_FORMALBILL A
  WHERE A.C_RATETYPE=1
    AND A.C_FEESUM>0
    AND A.C_ENDTIME
BETWEEN TRUNC(SYSDATE-1,'MM')
    AND TRUNC(SYSDATE,'MM')
    AND (A.C_BILLPHONE,A.C_CALLED,A.C_BEGINTIME,A.C_ENDTIME)
     IN (SELECT B.C_BILLPHONE,B.C_CALLED,B.C_BEGINTIME,B.C_ENDTIME 
           FROM BILL.BILL_FORMALBILL B
          WHERE B.C_RATETYPE=1
            AND B.C_FEESUM>0
            AND B.C_ENDTIME
        BETWEEN TRUNC(SYSDATE-1,'MM')
            AND TRUNC(SYSDATE,'MM')
          GROUP BY B.C_BILLPHONE,B.C_CALLED,B.C_BEGINTIME,B.C_ENDTIME
         HAVING COUNT(0)>1);
         
 SELECT A.C_CALLER,A.C_BILLITEM,A.C_BILLPHONE,A.C_CALLED,
        A.C_DURATION,A.C_FEESUM,A.C_BEGINTIME,A.C_ENDTIME 
   FROM BILL.BILL_FORMALBILL A
  WHERE A.C_RATETYPE=3
    AND A.C_FEESUM>0
    AND A.C_ENDTIME
BETWEEN TRUNC(SYSDATE-1,'MM')
    AND TRUNC(SYSDATE,'MM')
    AND (A.C_BILLPHONE,A.C_CALLED,A.C_BEGINTIME,A.C_ENDTIME)
     IN (SELECT B.C_BILLPHONE,B.C_CALLED,B.C_BEGINTIME,B.C_ENDTIME 
           FROM BILL.BILL_FORMALBILL B
          WHERE B.C_RATETYPE=3
            AND B.C_FEESUM>0
            AND B.C_ENDTIME
        BETWEEN TRUNC(SYSDATE-1,'MM')
            AND TRUNC(SYSDATE,'MM')
          GROUP BY B.C_BILLPHONE,B.C_CALLED,B.C_BEGINTIME,B.C_ENDTIME
         HAVING COUNT(0)>1);


```
#### 报表统计
##### 更新BILL_LAST_MONTH表
```sql
TRUNCATE TABLE BILL_LAST_MONTH;

SELECT * FROM BILL_LAST_MONTH;

INSERT INTO BILL_LAST_MONTH SELECT * FROM BILL_FORMALBILL 
WHERE C_ENDTIME BETWEEN TRUNC(ADD_MONTHS(SYSDATE,-1),'MM') AND TRUNC(SYSDATE,'MM');
```
##### 运行统计脚本并发送邮件
运行`C:\Users\DEVELOP\Desktop\pyscript\`路径下的`report.py`，待程序完成后，将在桌面`output`文件夹下生成上月报表（生成前请清空文件夹）;
邮件按上月已发送进行发送  
*ps:德州数据需要手动更新，进德州目录打开`声讯报表.xls`进行更新！*
