# MySQL queries on Adventure works 2005 database

![AdvWorksOLTPSchemaVisio](https://user-images.githubusercontent.com/47832124/133079946-0649136d-8c02-4301-80e9-8d0ce2e20215.png)

Credit: Questions are from ["REAL SQL QUERIES 50 CHALLENGES by BRIAN COHEN, NEIL PEPI and NEERJA MISHRA"](https://www.amazon.in/Real-SQL-Queries-50-Challenges/dp/1517290708). 

Credit: [adventureworks2005 Mysql database](https://github.com/tapsey/AdventureWorksMYSQL) 

Questions are provided in shorted notation. The full descriptive questions can be found on the book above. Some changes are made to the questions to comply version differences in the DB used in book(2012) and the database I used(2005). Following are my solutions in MySQL. Line breaks may differ in markdown and hence the queries may look unformatted.

#### Question 1: 
An executive requests data concerning fiscal quarter sales by salesperson. She’d like to see comparisons from the fiscal quarters of 2001 to the same fiscal quarters of 2002.

#### Answer 1:

-- drop temporary table Q1

-- Creating temporary table with aggregated results

/*Create temporary table Q1 <br/>
SELECT C.LastName, S.SalesPersonID, <br/>
case <br/>
when month(S.orderdate) > 6 then year(S.orderdate) <br/>
else year(S.orderdate) - 1<br/>
end as FY,<br/>
quarter(S.orderdate) as FYQ,<br/>
sum(TotalDue) as Sales<br/>
FROM salesorderheader as S<br/>
left join Employee as E on S.SalesPersonID = E.employeeID<br/>
left join Contact as C on C.ContactID = E.ContactID<br/>
where S.onlineorderflag = 0<br/>
group by S.SalesPersonID, FY, FYQ<br/>
*/<br/>

-- Duplicating temporary table since a temporary table cannot be called twice on the same query
 
/*Create temporary table Q11<br/>
select * from Q1<br/>
*/

-- Final query to get results

select <br/>
Q1.LastName, Q1.SalesPersonID, Q1.FY, Q1.FYQ , Q11.FY as PFY, <br/>
Q1.Sales, Q11.Sales as PSales, <br/>
round((Q1.Sales - Q11.Sales),2) as SalesChange,<br/>
round(((Q1.Sales - Q11.Sales)/Q1.Sales)*100,2) as PerSalesChange   <br/>
from Q1 inner join Q11 on Q11.FY = Q1.FY-1 and Q1.SalesPersonID = Q11.SalesPersonID and Q1.FYQ = Q11.FYQ<br/>

**Sample output:**

<img width="693" alt="Screenshot 2021-09-13 at 5 41 18 PM" src="https://user-images.githubusercontent.com/47832124/133081571-5337261f-1a36-419a-9944-8b5c1fc426f1.png">

<hr style="border:2px solid gray"> </hr>

#### Question 2:

A marketing manager devised the “2/22” promotion, in which orders subtotaling at least $2,000 ship for $0.22. The strategy assumes that freight losses will be offset by gains from higher value orders. According to the marketing manager, orders between $1,700 and $2,000 will likely boost to $2,000 as customers feel compelled to take advantage of bargain freight pricing. You are asked to test the 2/22 promotion for hypothetical profitability based on the marketing manager’s assumption about customer behavior. Examine orders shipped to California during fiscal year 2004 for net gains or losses under the promotion.

#### Answer 2:

-- Creating temporary table to aggregate results

/*<br/>
create temporary table Q2<br/>
select <br/>
S.salesorderID, P.name, date(S.orderdate), S.freight, S.totaldue,<br/> 
case <br/>
when S.totaldue between 1700 and 2000 then "Increase order to 2000 and pay $0.22 freight charge"<br/>
when S.totaldue <= 1700 then "No change in order pay old freight charge"<br/>
else "No change in order pay $0.22 freight charge"<br/>
end as Promostatus,<br/>
case <br/>
when S.totaldue between 1700 and 2000 then freight - 0.22<br/>
when S.totaldue <= 1700 then 0<br/>
else freight - 0.22<br/>
end as freightloss, <br/>
case <br/>
when S.totaldue between 1700 and 2000 then 2000 - S.totaldue<br/>
else 0<br/>
end as ordergain <br/>
from salesorderheader as S <br/>
left join customeraddress as C on S.CustomerID = C.CustomerID<br/>
left join Address as A on A.addressid = C.AddressID<br/>
left join stateProvince as P on P.stateprovinceid = A.stateprovinceid<br/>
where P.name = 'California' and year(S.orderdate) = 2004 <br/>
*/

-- drop temporary table Q2

-- Aggregating final results
select Promostatus, sum(ordergain) as Gain, sum(freightloss) as loss, (sum(ordergain) - sum(freightloss)) as netGain from Q2 group by Promostatus

**Sample output:** Inference -> 2/22 promotion is a not a profitable strategy

<img width="609" alt="Screenshot 2021-09-13 at 5 52 41 PM" src="https://user-images.githubusercontent.com/47832124/133082670-0cafb13b-397e-4dbf-aeb2-07f2101d767d.png">

<hr style="border:2px solid gray"> </hr>


#### Question 3:

Ten million dollars of revenue is a common benchmark for Adventure Works. For each fiscal year (2007 and 2008), find the first dates when the cumulative running revenue total hit $10 million.

#### Answer 3:

-- Creating temporary tables for the FY 2002, 2003 individually. <br/>
/*<br/>
create temporary table Q3_2002<br/>
(id INTEGER NOT NULL AUTO_INCREMENT PRIMARY KEY)<br/>
select 2002 as FY, orderdate, totaldue, salesorderID <br/>
from salesorderheader <br/>
where ((year(orderdate) = 2002 and month(orderdate) > 6) or<br/>
(year(orderdate) = 2003 and month(orderdate) <= 6))<br/>
*/<br/>

-- drop temporary table Q3_2003<br/>


-- Creating while function <br/>
-- creating stored procedure to compute aggregate amount for 2002 and 2003<br/>
/*<br/>
 DELIMITER $$<br/>
 
create procedure Q3_compute_amount_2002()<br/>
begin<br/>
		declare amount float;<br/>
        declare total_amount float default 0;<br/>
        declare order_number int;<br/>
        declare order_date date;<br/>
        declare year_number int;<br/>
        declare cur1 cursor for select id, OrderDate, totaldue from Q3_2002 order by id;<br/>
        set year_number = (select distinct(FY) from Q3_2002);<br/>
        open cur1;<br/>
        compute_amount : loop<br/>
			fetch cur1 into order_number, order_date, amount;<br/>
            set total_amount = total_amount + amount;<br/>
            if total_amount > 10000000 then <br/>
				leave compute_amount;<br/>
			end if;<br/>
		end loop compute_amount;<br/>
        close cur1;<br/>
        create temporary table 2002_result select year_number as FiscalYear, order_date as Orderdate,<br/> order_number as FYOrder, total_amount as RunningTotal;<br/>
end$$<br/>
<br/>
*/<br/>
<br/>

-- drop procedure Q3_compute_amount_2002<br/>

-- call Q3_compute_amount_2002()<br/>
-- call Q3_compute_amount_2003()<br/>

select * from 2002_result union select * from 2003_result;<br/>

<hr style="border:2px solid gray"> </hr>

#### Question 4:

Tuesday’s are “upsell” days for sales people at Adventure Works. Management wants to compare sales from Tuesday to other days of the week to see if the initiative is working. Help monitor the upsell initiative by creating a query to calculate average revenue per order by day of week in 2004.


#### Answer 4:

select dayofweek(orderdate), count(orderdate), sum(totaldue) , avg(totaldue) from salesorderheader <br/>
where year(orderdate) = 2002 and onlineorderflag = 0<br/>
group by dayofweek(orderdate)<br/>
order by dayofweek(orderdate)<br/>

**Sample output:** 

<img width="470" alt="Screenshot 2021-09-30 at 3 22 55 PM" src="https://user-images.githubusercontent.com/47832124/135430458-83990023-a88a-4927-92aa-775750fc7ac3.png">

<hr style="border:2px solid gray"> </hr>


#### Question 5:

The Accounting department found instances where expired credit cards were used with sales orders. You are asked examine all credit cards and report the extent of such activity.

#### Answer 5:

-- checking if a customer holds/ uses only one creditcard for transaction <br/>
/*<br/>
select count(distinct(CustomerID)), count(distinct(creditcardid)) from salesorderheader where creditcardid is not null<br/>
*/<br/>

-- Fetching expirationd date for all creditcards used in trasaction<br/>
/*<br/>
create temporary table Q5 <br/>
select S.CreditCardID, c.Cardtype, max(S.OrderDate), last_day(concat(C.ExpYear,'-',C.ExpMonth,'-1')) as ExpirationDate,<br/>
case <br/>
when max(S.OrderDate) > last_day(concat(C.ExpYear,'-',C.ExpMonth,'-1')) then 'yes'<br/>
else 'no'<br/>
end as 'CreditCardExpirationFlag'<br/>
from salesorderheader S <br/>
left join creditcard C on S.creditcardid = C.creditcardid <br/>
where S.CreditCardID is not null<br/>
group by  S.CreditCardID<br/>


create temporary table Q5_1 <br/>
select S.CreditCardID, c.Cardtype, S.OrderDate, last_day(concat(C.ExpYear,'-',C.ExpMonth,'-1')) as ExpirationDate<br/>
from salesorderheader S <br/>
left join creditcard C on S.creditcardid = C.creditcardid <br/>
where S.CreditCardID is not null<br/>
*/<br/>

-- Based on credit card ID<br/>
select Creditcardid, Cardtype, count(*) as OrdersBeforeExpiring from Q5_1 <br/>
where OrderDate <= ExpirationDate<br/>
group by Creditcardid<br/>
<br/>
-- Based on credit card type<br/>
/*<br/>
select Cardtype, count(*) as OrdersBeforeExpiring from Q5_1 <br/>
where OrderDate <= ExpirationDate<br/>
group by Cardtype<br/>
*/<br/>

<hr style="border:2px solid gray"> </hr>

#### Question 6:

Adventure Works will feature one product for the cover of its print catalog. Help select a list of products for consideration.
Your list should contain products which meet all of the following conditions:
* Finished goods (not products utilized to make other products) List price at least $1,500
* At least 150 in inventory
* Currently available for sale

#### Answer 6:

select P.ProductID,P.Name, P.Color, P.listprice, I.Quantity, P.DiscontinuedDate  , P.FinishedGoodsFlag <br/>
from product P<br/>
left join productinventory I<br/>
on P.ProductID = I.ProductID<br/>
where P.FinishedGoodsFlag = 1 and P.DiscontinuedDate is null and I.Quantity >=150 and P.listprice >=1500<br/>

<hr style="border:2px solid gray"> </hr>


<!--------------------

#### Question :


#### Answer :


<hr style="border:2px solid gray"> </hr>

-->
