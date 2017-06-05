一、执行计划
1.1关于执行计划：
方法1：
F5/explain plan for  
explain plan for +SQL语句
再执行select * table(dbms_xplain.display());
可能是错误的执行计划




方法2:
autotrace开关
set autot on    --开启 显示结果+执行计划+统计信息
set autot off    --关闭
set autot trace  --不显示结果，只显示执行计划+统计信息（一般用这个）
set autot trace exp
set autot trace stat   
在SQLPLUS中打开开关执行SQL语句，是对explain plan的一个封装，执行计划也可能是错误的




方法3:
alter session set statistics_level=all;--显示所有
select * from table(dbms_xplan.display_cursor(null,null,'ALLSTATS LAST'));--查看最近一下执行的SQL的执行计划
这是真实的执行计划


还有10046等其他方法，涉及到追踪，个人不常用。
总结：plan_table出来的执行计划可能是错误的，v$sql_plan出来的执行计划才是真实的。




1.2如何看执行计划：
先从最开头一直持续往右看，直到看到最右边的并列地方；对于不并列的，靠右的先执行；如果看到并列的，就从上往下看，对于并列的，靠上的先执行。
个人：最右上最先执行。







二、访问数据的方法
2.1全表扫描:
不是说全表扫描不好，事实上ORACLE全表扫描时会使用多块读，这在目标表的数据量不高是效率是非常高的，但全表扫描最大的问题就在于全表扫描的目标SQL的执行时间不稳定、不可控，这个执行时间一定会随着目标表数据量的递增而递增。高水位线这种特性所带来的副作用是，即使使用DELETE语句删光了目标表中所有的数据，高水位还是会在原来的位置，这意味着全表扫描该表时ORACLE还是需要扫描该高水位线下所有的数据块，所以消耗的时间还是不会有明显改观，I/O资源还是很高。
2.2 ROWID扫描
select empno,
      ename,
      rowid,
      dbms_rowid.rowid_relative_fno(rowid),--几号文件
      dbms_rowid.rowid_block_number(rowid),--第几个数据块
      dbms_rowid.rowid_row_number(rowid)--第几行记录
 from emp;


三、访问索引的方法
索引分支块包含指向相应索引分支块/叶子块的指针和索引键值列(这里的指针是指相关分支块/叶子快的地址RDBA。每个索引分支块都会有两种类型的指针，一种是lmc，另一种是索引分支块的索引记录所记录的指针。lmc是Left Most Child的缩写，每个索引分支块都只有一个lmc，这个lmc指向的分支块/叶子快中的所有索引键值列中的最大值一定小于该lmc所在索引分支块的所有索引键值列中的最小值;而索引分支块的索引行记录的指针所指向的分支块/叶子快的所有索引键值列中的最小值一定大于或等于该行记录的索引键值列的值)


访问相关的B树索引会标都需要消耗I/O,这意味着在ORACLE中访问索引的成本由两部分组成:一部分是访问相关B树索引的成本(从根节点定位到相关的分支块,再定位到相关的叶子快,最后对这些叶子快执行扫描操作);另外一部分是回表的成本(根据得到的ROWID再回表去扫描对应的数据行所在的数据块)


3.1索引唯一性扫描
索引唯一性扫描(INDEX UNIQUE SCAN)是针对唯一性索引(UNIQUE INDEX)的扫描,它仅仅适用于WHERE条件里等值查询的目标SQL.因为扫描的对象是唯一性索引,所以索引唯一性扫描的结果至多只会返回一条记录.
create unique index idx_emp on emp(empno);
为了避免Buffer Cache和数据字典缓存对逻辑的统计结果的影响，清空Buffer Cache
alter system flush shared_pool;--不要在生产上使用
alter system flush buffer_cache;--不要在生产上使用
consistent gets ---消耗的逻辑读
3.2索引范围扫描
索引范围扫描(INDEX RANGE SCAN)适用于所有类型的B树索引,当扫描的对象是唯一性索引时，此时目标SQL的where条件一定是范围查询(谓词条件为BETWEEN,<>等);当索引扫描的对象是非唯一性的索引时,对目标的SQL的where条件没有限制(可以是等值查询，也可以是范围查询)。索引范围扫描的结果可能是返回多条记录。为了确定扫描终点,在同等条件下,索引范围扫描会比索引唯一性扫描的逻辑读多1。因为分支和叶子的物理地址是不连续的，不能多块读。那么就需要多读一块来确定是不是最后一个逻辑读。（个人观点）
3.3索引全扫描
索引全扫描(INDEX FULL SCAN)适用于所有类型的B树索引，是从左至右依此顺序扫描目标索引所有叶子块的所有索引行，而索引是有序的，所以索引全扫描的执行结果也是有序的，并且按照该索引的索引键值列来排序，这也就意味着走索引全扫描既能达到排序的效果，又同时避免了对该索引的索引键值的真正排序操作。ORACLE中能做索引全扫描的前提条件是目标索引至少有一个索引键值列的属性是NOT NULL。
3.4索引快速全扫描
索引快速全扫描(INDEX FAST FULL SCAN)适用于所有类型的B树索引,与索引全扫描类似，区别在于:1.索引全扫描只适用于CBO 2.索引快速扫描可以使用多块读，也可以并行


执行 3.索引快速全扫描的执行结果不一定是有序的。这是因为索引快速全扫描时ORACLE是根据索引行在磁盘上的物理存储顺序来扫描的，而不是根据索引行和逻辑顺序来扫描的。注意:叶子快里面的物理位置是一致的，叶子块与叶子块之间的物理位置不一定是有序的。
alter table emp_test add contraint pk_emp_test primary key(empbo,col1,col2,col3)
添加复合主键索引
select /*+ INDEX_FFS(emp_test pk_emp_test)*/empno from emp;走索引快速全扫描
3.5索引跳跃式扫描
索引跳跃式扫描(INDEX SKIP SCAN)适用于所有类型的B树索引,ORACLE中的索引跳跃式扫描仅仅适用于那么目标索引前导列的DISTINCT值数量少、后续非前导列的可选择性又非常好的情形，因为索引跳跃式扫描的执行效率一定会随着目标索引前导列的DISTINCT数量的递增而递减。




四、表连接


不管目标SQL中有多少个表做表连接，ORACLE在实际执行该SQL时只能先两两做表连接，再依次执行这样的两两表连接过程，直到目标SQL中所有的表都已经连接完毕。
JOIN ON、JOIN USING(不能用别名)、NATURAL JOIN(不推荐)
FULL OUTER JOIN 类似UNION 但实际上ORACLE的执行计划是:左右连接结果的UNION
关键字(+)出现在哪个表的连结列后面。就表明哪个表会以NULL值来填充那些不满足连接条件并位于该表中的查询列，此时应该以关键字(+)对面的表做驱动表。


4.1排序合并连接
排序合并连接(Sort Merge Join):
1.首先以目标SQL中指定的谓词条件(如果没有的话)去访问第一个表，然后通过对访问结果中的连接列来排序。
2.以目标SQL中指定的谓词条件(如果没有的话)去访问第二个表，然后通过对访问结果表中的连接列来排序。
3.最后对结果集1和结果集2执行合并操作，从中取出匹配记录来作为排序合并连接的最终结果。





合并方式:1中的第1条去2中匹配,2匹配上的记录或者删除，不需要遍历所有的记录(不确定是否正确)。


*通常情况下，排序合并连接的执行效率会远不如哈希连接，但前者的使用范围更广，因为哈希连接通常只能用于等值连接条件，而排序合并连接还能用于其他连接条件(例如<、>、<=、>=)。
*通常情况下，排序合并连接并不适合OLTP类型的系统，其本质原因是因为对于OLTP类型的系统而言，排序是非常昂贵的操作，当然，如果能避免排序操作，那么即使是OLTP类型的系统，也还是可以用排序合并连接的。


从严格意义上说，排序合并连接并不存在驱动表的概念，在执行合并的过程中，实际上还是存在驱动表和被驱动表的。(不确定是否正确)


4.2嵌套循环连接
嵌套循环连接(NESTED LOOPS):
1.优化器会按照一定的规则来决定表T1和T2中谁是驱动表、谁是被驱动表。驱动表用于外层循环，被驱动表用于内层循环。假设驱动表是T1，被驱动表是T2。
2.以目标SQL中指定的谓词条件(如果有的话)去访问驱动表T1，访问驱动表T1后得到的结果集为1
3.用结果集1的一条记录去遍历被驱动表T2，外层循环所对应的驱动结果集1有多少条记录，遍历被驱动表T2内层循环就要做多少次。


*如果驱动表所对应的驱动结果集的记录数较少，同时被驱动表的连结列上有存在唯一性索引(或者在被驱动表的连接列上存在选择性很好的非唯一性索引)，那么此时使用嵌套循环连接的执行效率就会非常高；但如果驱动表所对应的驱动结果集的记录数很多，即便在被驱动表的连接列上存在索引，此时使用嵌套循环连接的执行效率也不会高。
*大表也可以作为嵌套循环连接的驱动表，关键是看目标SQL中指定的谓词条件(如果有的话)能否将驱动结果集的数据量降下来。
*嵌套循环连接有其他连接方法所没有的一个优点:嵌套循环连接可以实现快速响应，即它可以第一时间返回已经连接过且满足连接条件的记录，而不必等待所有连接操作全部完后才返回连接结果。虽然排序合并连接和哈希连接也可以先返回已经连接过且满足连接条件的记录，而不必等待所有连接操作完成，但是它们并不是第一时间返回，因为排序合并连接要等到排完序后做合并操作时才能开始返回结果，而哈希连接则要等到驱动结果集所对应的Hash Table全部建完后才能开始返回数据。


为了提高嵌套循环连接的执行效率，在ORACLE 11g中，ORACLE引入了向量I/O(Vector I/O)。在引入向量I/O后，ORACLE就可以将原先一批单块读所需要消耗物理I/O组合起来，然后用一个向量I/O去批量处理它们，这样就实现了在单块读的数量不降低的情况下减少这些单块读所需要消耗的物理I/O数量，也就是提高了嵌套循环连接的执行效率。
select * from v$version;查看数据库版本
/*+ ordered use_nl(t2)*//*+ optimizer_features_enabl('9.2.0') eordered use_nl(t2)*/回到9的版本执行
10版本有两个NESTED LOOPS 这是ORACLE为了体现用ROWID回表访问时一批ROWID组合起来通过一个向量I/O批量回表。  


4.3哈希连接
哈希连接(HASH JOIN),官方解释比较复杂，这里用我个人理解的方式简单说明。
1.HASH的算法：
把驱动表放到PGA对JOIN的列进行HASH计算，两个表都只扫描了一次。
2.被驱动表每读一条数据，就到PGA进行匹配。（HASH不能做不等值连接）
3.
HASH是消耗内存的。
内存：SGA共有的，PGA私有的


4.4笛卡尔连接
一种特殊的‘合并连接’，不需要排序，连接条件漏了。




4.5连接的选择
重点：该如何选择表连接的方式
首先总结一下3种连接的特点：NESTED LOOPS,嵌套循环，返回多少条数据被驱动表就要扫描多少次。
                         HASH JOIN,哈希连接，两个表都只扫描一次，但是要消耗内存，属于资源换性能的方式，只适用于等值连接。
                          Sort Merge Join，排序合并连接，适用于等值连接，有排序。


总结：两表关联返回数据量少的走NESTED LOOPS,嵌套循环。在被驱动表建索引。
     两表关联返回数据量多的走HASH JOIN,哈希连接。




方法：select /*+ use_nl(a,b)*/* from tab1 a,tab2b b where a.id=b.id;  加HINT改变执行计划
         /*+ use_nl(a,b)*/  嵌套   /*+ use_hash(a,b)*/ 哈希



         五、平时写SQL注意的一些细节
         1.对于多张数据量大的表（这里几百条就算大了）JOIN，建议先分页再JOIN，不然逻辑读会很差。
         分页方法：
         分页查询格式1：在查询的最外层控制分页的最小值和最大值：
         select *  
          from (select a.*,rownum rn
                  from (select * from table_name)a)  
         where rn between 21 and 40;
         分页查询格式2：
         select *
          from (select a.*,rownum rn
                  from(select * from table_name)a)
         where rn>=21;
         分页查询格式3:考虑到多表联合的情况，如果不介意在系统中使用HINT的话，可以将分页的查询语句改为：
         select /*+ FIRST_ROWS*/*  
          from (select a.*,rownum rn
                  from(select *  
                         from table_name)a
                 where rownum<=40)
         where rn>=21;
         绝在多数情况下方法2比方法1效率高
         多表联合中分页查询可以考虑NESTED LOOPS，前面说到了返回数据量少的话选择NESTED LOOPS在被驱动表中建索引的情况下效率有所改观。




         2.尽量避免全表扫描，where和order by的列上建索引




         3.尽量避免对列的NULL判断，会导致无法走索引




         4.尽量避免在where子句中使用or的连接条件，如果一个字段有索引，一个字段没有索引，将放弃索引走全表扫描。

         例子：
         select id from t where num=10 or name='xxx';
         改为：
         select id from t where num=10
         union all
         select id from t where name='xxx'


         5.in和not in可考虑改为between
         例子:select id from t where num in(1,2,3);
         改为：select id from t where num between 1 and 3;




         6.like '%xxx%'的模糊查询也会导致全表扫描




         7.where子句加入参数也会导致全表扫描
         例子:select id from t where num=@num;
         改为:select id from t with(index(索引名) where num)




         8.尽量避免在where子句中使用!= <>，会走全表扫描




         9.对于where里面的条件，oracle是自下往上读的，尽量把能过滤掉大量数据的条件放在最下面，表连接条件放在最上面




         10.SQL的标量子查询可改为外连接




         11.自定义函数改为外连接




         12.表被访问多次的话可以考虑用分析函数改了










         with as的应用：
         1.相当于一个虚拟视图，对于union all的语句。每部分执行成本高，换成union all中就能减低成本。
         2.读取一次结果集可多次使用。
         3.用with as优化嵌套

         select * from tab1 where id in(select id from tab2 where name like '%aaa%')
         改为：
         with temp1 as(select id from tab2 where name like '%aaa%')
         select * from tab1 where id in(select * from temp1);




         开发时在用MERGE INTO更新插入发现很慢，看了执行计划走了MERGE,中间又走了NESTED LOOPS,很慢，可能是因为写了内参引起的（个人观点）。
         加了/*+ use_merge(a,b)*/   a为merge into后面的表，b为using的表，性能明显提高。










         六、脚本
         6.1查询一天中跑的慢的SQL
         SELECT S.SQL_TEXT,
               S.SQL_FULLTEXT,
               S.SQL_ID,
               ROUND(ELAPSED_TIME / 1000000 / (CASE
                       WHEN (EXECUTIONS = 0 OR NVL(EXECUTIONS, 1 ) = 1) THEN
                        1
                       ELSE
                        EXECUTIONS
                     END),
                     2) "执行时间'S'",
               S.EXECUTIONS "执行次数",
               S.OPTIMIZER_COST "COST",
               S.SORTS,
               S.MODULE, --连接模式（JDBC THIN CLIENT：程序）
               -- S.LOCKED_TOTAL,
               S.PHYSICAL_READ_BYTES "物理读",
               -- S.PHYSICAL_READ_REQUESTS "物理读请求",
               S.PHYSICAL_WRITE_REQUESTS "物理写",
               -- S.PHYSICAL_WRITE_BYTES "物理写请求",
               S.ROWS_PROCESSED      "返回行数",
               S.DISK_READS          "磁盘读",
               S.DIRECT_WRITES       "直接路径写",
               S.PARSING_SCHEMA_NAME,
               S.LAST_ACTIVE_TIME
FROM GV$SQLAREA S
WHERE ROUND(ELAPSED_TIME / 1000000 / (CASE
            WHEN (EXECUTIONS = 0 OR NVL(EXECUTIONS, 1 ) = 1) THEN
             1
            ELSE
             EXECUTIONS
          END),
          2) > 5 --100 0000微秒=1S
AND S.PARSING_SCHEMA_NAME = USER
AND TO_CHAR(S.LAST_LOAD_TIME, 'YYYY-MM-DD') =
    TO_CHAR( SYSDATE, 'YYYY-MM-DD' )
AND S.COMMAND_TYPE IN (2 , 3, 5, 6 , 189)
ORDER BY "执行时间'S'" DESC;




select b.owner,b.object_name,a.session_id,a.locked_mode
from v$locked_object a,dba_objects b
where b.object_id = a.object_id


6.2获取执行次数最多的10个SQL
select sql_text,executions
from (
select sql_text,executions,rank() over(order by executions desc) exec_rank
from v$sql
)
where exec_rank <=10;


获取单次执行时间最长的10个SQL
select sql_id,sql_text,round(exec_time/1000000,0) exec_time
from(
select sql_id,sql_text,exec_time,rank() over (order by exec_time desc) exec_rank
from
(
select sql_id,sql_text,cpu_time,elapsed_time,executions,round(elapsed_time/executions,0) exec_time
from v$sql
where executions>1
)
)
where exec_rank <=10;



6.3收集统计信息
begin  
 DBMS_STATS.gather_table_stats(ownname => 'ISSETL',
                               tabname => 'AAA_f_jdlkh_pkhjbxx',
                               estimate_percent => 100,
                               method_opt => 'for all columns size 1',
                               no_invalidate => FALSE,
                               degree => 1,
                               cascade =>TRUE);
end;






6.4查看统计信息
select a.column_name 列名,
       b.num_rows 列数,
       a.num_distinct 基数,
       round(a.num_distinct/b.num_rows*100,2) 选择性,
       a.histogram 是否收集直方图,
       a.num_buckets 桶数
  from dba_tab_col_statistics a,
       dba_tables b
 where a.owner=b.owner
   and a.table_name=b.table_name
   and a.owner='ISSETL'
   and a.table_name='AAA_F_JDLKH_PKH






6.5自动抓出错误NL的SQL
select *
 from (select parsing_schema_name owner,
              sql_text,
              sql_fulltext,
              sql_id,
              rows_processed/executions rows_processed
        from v$sql
       where parsing_schema_name='ISSETL'
         and executions>0
         and rows_processed/executions>2000
       order by 5 desc)a
where a.sql_id in (select sql_id
                     from v$sql_plan
                    where operation='NESTED LOOPS'
                      and id<=5);  
