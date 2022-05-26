## autocommit
在autocommit=0的情況下，開啟事務會自動提交之前的語句
```sql
set autocommit=0;
insert into user values (null, 'patrick');
# 此時查詢table，patrick尚未插入表中

BEGIN;
# 此時查詢表中會發現patrick已插入表中，即使rollback也不會消失
```

* BEGIN 是 START TRANSACTION 的 Alias,但若搭配 BEGIN...END不是
* 結束session
經實測，直接結束會話事務也跟著結束


## 簡單insert情境
需要先確認表格是否有值在做後續處理
一開始的想法是先select確認是否有值，沒有做後續處理並且insert
但這樣select之後必須鎖表
可以直接改成
```php
try{
$db->startTransaction(); // 開啟事務
$db->insert($sql) // 直接insert進unique欄位

$res = sendCurl($url); // 處理php邏輯
if($res){
     $db->commit(); // 成功commit
   }else{
   	 $db->rollback(); // 失敗rollback
   }
}catch(){
	$db->rollback(); // 錯誤rollback;
}
```


## 賣超場景
```sql
# 加行寫鎖
select * from [table] where id = [xx] for update;

# if sotck > 0
update [table] set stock = stock - 1 where id = [xx];

# 成功
commit;

# 失敗
rollback;
```


## 鎖
* MyISAM都使用表鎖，併發不好，但也不會有死鎖的問題
  innodb才支援事務及行級鎖，要使用行鎖必須要有索引，否則等於是表鎖

* 樂觀鎖與悲觀鎖
  這是針對讀(select)的說法，不是實際的鎖

* 讀鎖又稱共享鎖(S鎖)，寫鎖又稱排他鎖(X鎖)

* 同個資源同事務，讀鎖與寫鎖不會互相排斥
* 同個資源不同事務，排他鎖不可與其他鎖共存
* 同個資源不同事務，共享鎖可以共存  
  ![lock_relation2](https://user-images.githubusercontent.com/24542187/170448775-12bcb9ad-e9a1-44a2-a8f8-9c4ba08c235d.png)  

* 資料庫的增刪改操作預設都會加排他鎖，而查詢不會加任何鎖
  即使不是在trasaction中操作，依然會加鎖

* lock table
  - `LOCK TABLE table READ;`  
    表讀鎖：所有終端都不能寫操作，但可以進行讀操作
  - `LOCK TABLE table WRITE;`  
    表加寫鎖：加鎖的終端機可以進行讀寫，其他終端機不能讀也不能寫
  - 注意這兩個語法並不需要放在transaction中，這裡的讀指的是一般的讀取，與行鎖不太一樣
    不過如果真的要鎖住整張表可以直接用`select * from table for update`應該也是一樣的意思
  - 若lock table 與 transactionu共同使用時，以下是官網所描述的
    > LOCK TABLES is not transaction-safe and implicitly commits any active transaction before attempting to lock the tables.
      UNLOCK TABLES implicitly commits any active transaction, but only if LOCK TABLES has been used to acquire table locks

* 意象鎖:參考連結

參考:   
https://blog.csdn.net/localhost01/article/details/78720727  
https://dev.mysql.com/doc/refman/8.0/en/lock-tables.html


## 事務隔離級別
修改該會話隔離級別
```sql
SET SESSION TRANSACTION ISOLATION LEVEL [READ UNCOMMITTED];
```

READ UNCOMMITTED 與READ COMMITTED兩者差不多  
差別在於READ UNCOMMITTED會出現髒讀
* 髒讀:讀取未commit的資料 

參考官方文件  
https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html
