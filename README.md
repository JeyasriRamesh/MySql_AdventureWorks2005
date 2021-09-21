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



