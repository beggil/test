```sql
set serveroutput on
declare
ENDDATE date;
STARTDATE date;
  N  INT;
  i  INT;
  v_date date;
  v_count  number;
  v_count0 number;
begin
    STARTDATE:='20180102';
    ENDDATE:='20180102';
    N:=ENDDATE-STARTDATE;
    for i in 0..N loop
    if STARTDATE<=TRUNC(SYSDATE) then
    v_date:=STARTDATE+i;
        merge into code_detail  co using(
         select distinct v_date busdate,c.organ,c.class,b.sort,c.code
         from commodity b  join wuerp.comm_shop c on b.code=c.code 
         where c.organ !='0000'and c.state in(0,1,2,5,6,7)
         and c.code not in('6902952880294', '6902952884308','6902952894093', '6902952880492', '6902952894420')
         group by c.organ,c.class,b.sort,c.code)b
         on ( b.busdate=co.busdate and  b.organ=co.organ  and b.code=co.code)
         WHEN NOT MATCHED THEN INSERT (co.busdate,co.organ,co.class,co.sort,co.code) 
         values(b.busdate,b.organ,b.class,b.sort,b.code);
     ----------------------------organ,class,sort,code,busdate
--        if sql%found then 
         merge into CODE_DETAIL co using(
                        select distinct  a.busdate,a.organ,a.class,trim(c.sort) sort,a.code,sum(sum_price) sale,sum(sum_cost) cost,sum(amount) amount
                                   from  account a,commodity c where c.code=a.code and a.resume like '销售%' and a.busdate=v_date
                                   and a.organ not like '%99%' and a.code not in('6902952880294','6902952884308','6902952894093','6902952880492','6902952894420')
                                   group by a.busdate,a.class,trim(c.sort),a.organ,a.code) b         
                                   on (b.busdate=co.busdate and b.organ=co.organ and b.code =co.code)
                                   WHEN MATCHED THEN UPDATE SET co.sale=b.sale,co.cost=b.cost,co.amount=b.amount  where co.BUSDATE =v_date
                                   WHEN NOT MATCHED THEN INSERT (co.busdate,co.organ,co.class,co.sort,co.code,co.amount,co.sale,co.cost) 
                                   values(b.busdate,b.organ,b.class,b.sort,b.code,b.amount,b.sale,b.cost);
    ----------------------------------- 销售额,销售成本,销量
                       dbms_output.put_line(v_date||','||'1');    
        merge into CODE_DETAIL co using(
                                select  a.busdate,a.organ,a.class,trim(c.sort) sort,a.code,sum(amount*cast (a.state as int)) accept_amount,sum(sum_cost*cast (a.state as int)) accept_cost
                                from account a,commodity c where c.code=a.code and a.resume in('进货','退货') and a.busdate=v_date
                                and a.organ not like '%99%' and a.code not in('6902952880294','6902952884308','6902952894093','6902952880492','6902952894420')
                                group by a.busdate,a.organ,a.class,trim(c.sort),a.code)b
                                on (b.busdate=co.busdate and b.organ=co.organ and b.code =co.code)
                                WHEN MATCHED THEN UPDATE SET co.accept_amount=b.accept_amount,co.accept_cost=b.accept_cost where v_date=co.busdate
                                WHEN NOT MATCHED THEN INSERT (co.busdate,co.organ,co.class,co.sort,co.code,co.accept_amount,co.accept_cost) 
                                values(b.busdate,b.organ,b.class,b.sort,b.code,b.accept_amount,b.accept_cost);
    --------------------------------------进货数量,进货成本
                       dbms_output.put_line(v_date||','||'2');    
        if v_date<'20171231' then 
               update   code_detail co set  co.amount_stock =0,co.cost_stock =0 where co.busdate=v_date;      
        elsif  to_char(v_date+1,'dd')='01' then 
                select count(*) into v_count0 from wuerp.account where resume='期初结存' and jyfs='自营'  and busdate=v_date; 
                if v_count0!=0 then
                merge into code_detail co using (
                         select distinct a.busdate,a.organ,a.class,trim(c.sort) sort,a.code,sum(a.sum_cost) cost1,sum(a.amount) amount1
                                    from  wuerp.account a,commodity c where c.code=a.code and  a.resume='期初结存' and a.jyfs='自营'  and a.busdate=v_date
                                    group by a.busdate,a.class,trim(c.sort),a.organ,a.code  having sum(a.amount)!=0)b
                            on( b.busdate=co.busdate and b.organ=co.organ  and b.code=co.code)
                            WHEN MATCHED THEN UPDATE SET co.amount_stock =b.amount1,co.cost_stock =b.cost1 where co.busdate=v_date
                            WHEN NOT MATCHED THEN INSERT (co.busdate,co.organ,co.class,co.sort,co.code,co.amount_stock,co.cost_stock) 
                                values(b.busdate,b.organ,b.class,b.sort,b.code,b.amount1,b.cost1);
                   dbms_output.put_line(v_date||','||'3');            
                else 
                merge into code_detail co using (
                        with a as(select * from code_detail where busdate=v_date-1),
                             b as (select distinct a.busdate,a.organ,a.class,trim(c.sort) sort,a.code,sum(a.sum_cost*cast(a.state as int)) cost2,sum(a.amount*cast(a.state as int))amount2 
                            from  wuerp.account a,commodity c where c.code=a.code and a.resume !='期初结存' and a.jyfs='自营' and a.busdate=v_date
                            group by a.busdate,a.organ,a.class,trim(c.sort),a.code  having sum(a.amount)!=0) 
                        select distinct b.busdate, b.organ,b.class,b.sort,b.code,(a.cost_stock+b.cost2) cost_stock,(a.amount_stock+b.amount2) amount_stock 
                                    from a join b on a.organ=b.organ and a.code=b.code)b
                        on(b.busdate=co.busdate and b.organ=co.organ  and b.code=co.code)
                        WHEN MATCHED THEN UPDATE SET co.amount_stock =b.amount_stock,co.cost_stock =b.cost_stock where co.busdate=v_date
                        WHEN NOT MATCHED THEN INSERT (co.busdate,co.organ,co.class,co.sort,co.code,co.amount_stock,co.cost_stock) 
                                values(b.busdate,b.organ,b.class,b.sort,b.code,b.amount_stock,b.cost_stock);    
               dbms_output.put_line(v_date||','||'4');                 
                end if;    
      -----------------------------------历史库存=期初结存+每日进销差额
        elsif v_date=trunc(sysdate) then 
                merge into code_detail co using(                                
                             select distinct c.organ,c.class,co.sort,c.code,c.indate,c.state,c.price
                             from code_detail co,wuerp.comm_shop c 
                             where co.organ=c.organ and co.code=c.code)b
                             on(co.organ=b.organ  and co.code=b.code )
                             WHEN MATCHED THEN UPDATE SET co.recdate =b.indate,co.state =b.state, co.price=b.price  where co.busdate=v_date;         
     -----------------------------recdate,state,price                              
                merge into code_detail co using(
                              select v_date busdate,a.organ,a.code,nvl(sum(a.amount),0) amount_s,nvl(sum(a.sum_cost),0) cost_s
                              from wuerp.cur_stock a,code_detail co
                              where a.organ=co.organ and a.code=co.code and co.busdate=v_date
                              group by co.busdate,a.organ,a.code) b
                              on(co.busdate=b.busdate and b.organ=co.organ  and b.code=co.code)
                              WHEN MATCHED THEN UPDATE SET co.amount_stock =b.amount_s,co.cost_stock =b.cost_s where co.busdate=v_date;             
        else 
                merge into code_detail co using (
                        with a as(select * from code_detail where busdate=v_date-1),----
                             b as (select distinct a.busdate,a.organ,a.class,trim(c.sort) sort,a.code,sum(a.sum_cost*cast(a.state as int)) cost2,sum(a.amount*cast(a.state as int))amount2 
                            from  wuerp.account a,commodity c where c.code=a.code and a.resume !='期初结存' and a.jyfs='自营' and a.busdate=v_date--- 
                            group by a.busdate,a.organ,a.class,trim(c.sort),a.code  having sum(a.amount)!=0) 
                        select distinct b.busdate, b.organ,b.class,b.sort,b.code,(a.cost_stock+b.cost2) cost_stock,(a.amount_stock+b.amount2) amount_stock 
                                    from a join b on a.organ=b.organ and a.code=b.code)b
                        on(b.busdate=co.busdate and b.organ=co.organ  and b.code=co.code)
                        WHEN MATCHED THEN UPDATE SET co.amount_stock =b.amount_stock,co.cost_stock =b.cost_stock where co.busdate=v_date
                        WHEN NOT MATCHED THEN INSERT (co.busdate,co.organ,co.class,co.sort,co.code,co.amount_stock,co.cost_stock) 
                                values(b.busdate,b.organ,b.class,b.sort,b.code,b.amount_stock,b.cost_stock);   
                                dbms_output.put_line(v_date||','||'5');                        
        end if;
        commit;

       -------------------------------------------库存数量和库存金额
        update CODE_DETAIL co
            set co.profit=sale-cost where co.BUSDATE =v_date;
    -----------------------------毛利
        update CODE_DETAIL co 
            set co.rate=round((case when sale!=0 then profit/sale else 0 end),4)*100 where busdate=v_date;
    ----------------------------毛利率    
        merge into CODE_DETAIL co using(
                            select distinct busdate,organ,code,
                            dense_rank() over(partition by organ order by sale desc nulls last) sale_rank ,
                            dense_rank()over(partition by organ order by profit desc nulls last) profit_rank,
                            dense_rank()over(partition by organ order by rate desc nulls last) rate_rank,
                            round(ratio_to_report(sale) over(partition by organ),6)*100  sale_ratio
                            from code_detail where busdate=v_date) b
                            on (b.busdate=co.busdate and  b.organ=co.organ and b.code=co.code)
                            WHEN MATCHED THEN UPDATE SET co.organ_sale_rank =b.sale_rank,co.organ_profit_rank =b.profit_rank,
                            co.organ_rate_rank =b.rate_rank,co.organ_sale_pro =b.sale_ratio where co.busdate=v_date;
           dbms_output.put_line(v_date||','||'6');                      
  -------------------------------店销售排名,店毛利排名,店毛利率排名,店销售占比  
        merge into CODE_DETAIL co using(
                               select organ,code,sum(sale) sale7,sum(amount) amount7,round(sum(amount)/7,2) avg7  from CODE_DETAIL
                               where  busdate between v_date-6 and v_date
                               group by organ,code)b
                               on (b.organ=co.organ and b.code=co.code)
                               WHEN MATCHED THEN UPDATE SET co.sale7= b.sale7 ,co.amount7= b.amount7,co.avg_sale7 =b.avg7 where v_date=co.busdate;                          
    -----------------------------7天销售额,销量,日均销量  
        merge into CODE_DETAIL co using(
                               select organ,code,sum(sale) sale15,sum(amount) amount15,round(sum(amount)/15,2) avg15  from CODE_DETAIL
                               where  busdate between v_date-14 and v_date
                               group by organ,code)b
                               on (b.organ=co.organ and  b.code=co.code)
                               WHEN MATCHED THEN UPDATE SET co.sale15= b.sale15 ,co.amount15= b.amount15,co.avg_sale15 =b.avg15 where v_date=co.busdate;                          
    -----------------------------15天销售额,销量,日均销量   
        merge into CODE_DETAIL co using(
                               select organ,code,sum(sale) sale30,sum(amount) amount30,round(sum(amount)/30,2) avg30  from CODE_DETAIL
                               where  busdate between v_date-29 and v_date
                               group by organ,code)b
                               on (b.organ=co.organ  and b.code=co.code)
                               WHEN MATCHED THEN UPDATE SET co.sale30= b.sale30 ,co.amount30= b.amount30,co.avg_sale30 =b.avg30 where v_date=co.busdate;                          
    -----------------------------30天销售额,销量,日均销量 
        merge into CODE_DETAIL co using(
                               select organ,code,sum(sale) sale90,sum(amount) amount90,round(sum(amount)/90,2) avg90  from CODE_DETAIL
                               where  busdate between v_date-89 and v_date
                               group by organ,code)b
                               on (b.organ=co.organ  and b.code=co.code)
                               WHEN MATCHED THEN UPDATE SET co.sale90= b.sale90 ,co.amount90= b.amount90,co.avg_sale90 =b.avg90 where v_date=co.busdate;   
    -----------------------------90天销售额,销量,日均销量
        merge into CODE_DETAIL co using(
                               select organ,code,sum(sale) sale180,sum(amount) amount180,round(sum(amount)/180,2) avg180  from CODE_DETAIL
                               where  busdate between v_date-179 and v_date
                               group by organ,code)b
                               on (b.organ=co.organ  and b.code=co.code)
                               WHEN MATCHED THEN UPDATE SET co.sale180= b.sale180 ,co.amount180= b.amount180,co.avg_sale180 =b.avg180 where v_date=co.busdate;   
    -----------------------------180天销售额,销量,日均销量
        merge into CODE_DETAIL co using(
                        select  distinct busdate,organ,code,
                                dense_rank() over(partition by organ order by sale7 desc nulls last) sale7_rank, 
                                dense_rank() over(partition by organ order by sale15 desc nulls last) sale15_rank,
                                dense_rank() over(partition by organ order by sale30 desc nulls last) sale30_rank,
                                dense_rank() over(partition by organ order by sale90 desc nulls last) sale90_rank,
                                dense_rank() over(partition by organ order by sale180 desc nulls last) sale180_rank
                                from CODE_DETAIL where busdate=v_date)b
                                on (b.busdate=co.busdate and b.organ=co.organ and b.code=co.code)
                                WHEN MATCHED THEN UPDATE SET co.organ_sale7_rank =b.sale7_rank,co.organ_sale15_rank =b.sale15_rank, 
                                co.organ_sale30_rank =b.sale30_rank,co.organ_sale90_rank =b.sale90_rank, 
                                co.organ_sale180_rank =b.sale180_rank where co.busdate=v_date;                        
     --------------------------------------7天店销售排名,15天店销售排名,30天店销售排名,90天店销售排名,180天店销售排名
               commit;  
                                      dbms_output.put_line(v_date||','||'7'); 
             merge into CODE_DETAIL co using(
                              select distinct a.selldate busdate,a.organ,a.code,sum(a.sum_sell) batchsale from sell_waste_day a,commodity c 
                              where a.code=c.code and a.type in ('单品销售','退货')  and a.receid in 
                                (select distinct code from wuerp.receiver where name like '%大宗%') and a.selldate=v_date
                                group by a.selldate,a.organ,a.code)b
                                on (b.busdate=co.busdate and b.organ=co.organ  and b.code=co.code )
                                WHEN MATCHED THEN UPDATE SET co.batchsale=b.batchsale where co.busdate=v_date;                     
   ----------------------------------大宗销售 
            select count(nosale_days) into v_count from code_detail where  busdate=v_date-1; 
            if v_count=0 then 
            merge into CODE_DETAIL co using(                    
                    select co.organ,co.code,nvl((v_date-max(a.busdate)),0) da 
                    from account a,CODE_DETAIL co where a.busdate<=v_date and resume like '销售%'
                    and a.organ=co.organ  and a.code=co.code 
                    group by co.organ,co.code) b
                    on (b.organ=co.organ and b.code = co.code  )
                    WHEN MATCHED THEN UPDATE SET co.nosale_days =b.da where co.busdate=v_date;     
            else
            merge into CODE_DETAIL co using(                    
                    select a.busdate,a.organ,a.class,a.sort,a.code,a.nosale_days da
                                from CODE_DETAIL a where a.busdate=v_date-1) b
                                on ( b.busdate+1=co.busdate and b.organ=co.organ and b.code = co.code)
                                WHEN MATCHED THEN UPDATE SET co.nosale_days=(case when sale!=0 then 0 else b.da+1 end)  where co.busdate=v_date;  
                   dbms_output.put_line(v_date||','||'8');                      
            end if;
    -----------------------------不动销天数 (null表示无销售记录)
        if sql%found then
            merge into  code_Detail co using (
                select distinct a.busdate,a.organ,a.code from code_detail a,commodity b
                where  b.code=a.code and a.avg_sale30!=0 and ((a.amount_stock/a.avg_sale30) >=case when a.class <'20' then 30 when a.class>'20'and a.class <'30' then 45 end) 
                and a.cost_stock>1000  and a.amount_stock>b.conversion*3 and a.busdate=v_date) b --and a.state !=4
            on (co.busdate=b.busdate and b.organ=co.organ  and b.code = co.code)
            WHEN MATCHED THEN UPDATE SET co.if_high_stock = 1  where co.busdate=v_date ;  
        
            merge into  code_Detail co using (
                select distinct a.busdate,a.organ,a.code from code_detail a,commodity b
                where  b.code=a.code and avg_sale30=0
                and cost_stock>1000 and a.amount_stock>b.conversion*3 and a.busdate=v_date) b ---and a.state !=4 
            on  (co.busdate=b.busdate and b.organ=co.organ and b.code = co.code)
            WHEN MATCHED THEN UPDATE SET co.if_high_stock = 1  where co.busdate=v_date; 
                                   dbms_output.put_line(v_date||','||'9'); 
        end if;
    -----------------------------判断高库存 
--       end if;     
    end if;
    commit;
    end loop;
    EXCEPTION
    WHEN OTHERS THEN
      ROLLBACK;
                         dbms_output.put_line(v_date||','||'99');    
end;
```

