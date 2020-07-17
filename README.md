# How to compress continuous intervals

Sometimes we run into the problem of being able to aggregate validity intervals.
However, this is not as trivial as it first appears. 

The following is a simple example and recursive selects to solve this problem.

-------------------------------------

**A table for demo**

```sql
    create table GA ( NAME character varying ( 100 ) 
                    , VFD  date
                    , VTD  date
                    );
    
    insert into GA (NAME, VFD, VTD) values ( 'NAME_1', to_date('2007-02-07', 'yyyy-mm-dd') , to_date('2009-06-30', 'yyyy-mm-dd'));
    
    insert into GA (NAME, VFD, VTD) values ( 'NAME_1', to_date('2016-10-01', 'yyyy-mm-dd') , to_date('2017-06-30', 'yyyy-mm-dd'));
    
    insert into GA (NAME, VFD, VTD) values ( 'NAME_2', to_date('2009-07-02', 'yyyy-mm-dd') , to_date('2014-02-06', 'yyyy-mm-dd'));
    insert into GA (NAME, VFD, VTD) values ( 'NAME_2', to_date('2014-02-07', 'yyyy-mm-dd') , to_date('2016-09-30', 'yyyy-mm-dd'));
    
    insert into GA (NAME, VFD, VTD) values ( 'NAME_2', to_date('2017-07-01', 'yyyy-mm-dd') , to_date('2018-05-17', 'yyyy-mm-dd'));
    
    insert into GA (NAME, VFD, VTD) values ( 'NAME_2', to_date('2019-01-03', 'yyyy-mm-dd') , to_date('2019-01-03', 'yyyy-mm-dd'));
    insert into GA (NAME, VFD, VTD) values ( 'NAME_2', to_date('2019-01-04', 'yyyy-mm-dd') , to_date('2019-02-05', 'yyyy-mm-dd'));
    insert into GA (NAME, VFD, VTD) values ( 'NAME_2', to_date('2019-02-06', 'yyyy-mm-dd') , to_date('2019-02-08', 'yyyy-mm-dd'));
    insert into GA (NAME, VFD, VTD) values ( 'NAME_2', to_date('2019-02-09', 'yyyy-mm-dd') , to_date('2019-02-10', 'yyyy-mm-dd'));
    
    insert into GA (NAME, VFD, VTD) values ( 'NAME_2', to_date('2019-03-01', 'yyyy-mm-dd') , to_date('2019-03-02', 'yyyy-mm-dd'));
```

-------------------------------------
**The Demo data**
The "-" shows the starting of an interval and "+" shows the continuation

    NAME_1	2007.02.07 00:00:00	2009.06.30 00:00:00 -
    NAME_1	2016.10.01 00:00:00	2017.06.30 00:00:00 -
    NAME_2	2009.07.02 00:00:00	2014.02.06 00:00:00 -
    NAME_2	2014.02.07 00:00:00	2016.09.30 00:00:00 +
    NAME_2	2017.07.01 00:00:00	2018.05.17 00:00:00 -
    NAME_2	2019.01.03 00:00:00	2019.01.03 00:00:00 -
    NAME_2	2019.01.04 00:00:00	2019.02.05 00:00:00 +
    NAME_2	2019.02.06 00:00:00	2019.02.08 00:00:00 +
    NAME_2	2019.02.09 00:00:00	2019.02.10 00:00:00 +
    NAME_2	2019.03.01 00:00:00	2019.03.02 00:00:00 -


-------------------------------------
**This is the result what we want to get**

    NAME_1	2007.02.07 00:00:00	2009.06.30 00:00:00
    NAME_1	2016.10.01 00:00:00	2017.06.30 00:00:00
    NAME_2	2009.07.02 00:00:00	2016.09.30 00:00:00
    NAME_2	2017.07.01 00:00:00	2018.05.17 00:00:00
    NAME_2	2019.01.03 00:00:00	2019.02.10 00:00:00
    NAME_2	2019.03.01 00:00:00	2019.03.02 00:00:00

## Here is the solution 

-------------------------------------
**Postgre**

```sql
    with recursive TT as
    ( select NAME
           , VFD
           , VTD
        from GA K
       where not exists ( select 1 from GA B where B.NAME = K.NAME and B.VTD + 1 = K.VFD )
      union all
      select TT.NAME
           , TT.VFD
           , GA.VTD
        from GA
        join TT on GA.NAME = TT.NAME and GA.VFD = TT.VTD + 1
    )
    select NAME
         , VFD
         , max( VTD )
      from TT
     group by NAME
         , VFD
    order by 1,2
```    
-----------------------------
**Oracle #1**
```sql
    with TT ( NAME, VFD, VTD ) as
    ( select NAME
           , VFD
           , VTD
        from GA K
       where not exists ( select 1 from GA B where B.NAME = K.NAME and B.VTD + 1 = K.VFD )
      union all
      select TT.NAME
           , TT.VFD
           , GA.VTD
        from GA
        join TT on GA.NAME = TT.NAME and GA.VFD = TT.VTD + 1
    )
    select NAME
         , VFD
         , max( VTD )
      from TT
     group by NAME
         , VFD
    order by 1,2
```
----------------------------
**Oracle #2**

```sql
    select NAME
         , VFD
         , max( VTD ) 
     from ( select NAME
                 , connect_by_root VFD as VFD
                 , VTD  
              from GA K 
            connect by prior VTD + 1 = VFD and prior NAME = NAME
              start with not exists ( select 1 from GA B where B.NAME = K.NAME and B.VTD + 1 = K.VFD )
          )
     group by NAME
            , VFD
    order by 1,2
```
----------------------------
    



