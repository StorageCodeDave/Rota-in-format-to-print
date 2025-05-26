Issue was the rota when print in the GUI was only printing in the format on the screen and only fitting 5 staff on each page. 
The customer wanted a table report that could be print on A3 and the rota pinned up in the office for every to read as they passed.
I advised I could create something they could use in PowerQuery on there PC and would connect to SQL and could refresh it everytime they wanted to print it. 
It's limitation was they could not choose the dates so I would set the report to return last 33 days and forward 14 days and they were happy be able to print this additional to the GUI.

I did run into issues with the date converted to more readable format then made the date order of the rota to order wrong. I create a temp table with dates in it before the date where 
converted to string and that kept the dates in order and better format for the report

![image](https://github.com/user-attachments/assets/91c9f2ff-f166-478d-8334-f24e59de73f8)


This was my first dynamic query as orginally design it for myself to be able to check data directly in the database and did not want to be limited by a date range size. 
This did make it easy to reuse with how many days the customer wanted to see in the past and shifts in the future,

DECLARE @starttime datetime, @enddate datetime, @cols AS NVARCHAR(MAX), @query AS NVARCHAR(MAX);

SELECT @starttime = getdate()-33, @enddate = getdate()+14
select @starttime as startdate,@enddate as enddate into #tempdates;



WITH DateCTE AS (
    SELECT DISTINCT clockingdate
    FROM rota
    WHERE clockingdate BETWEEN @starttime AND @enddate
    
)
SELECT @cols = STUFF((SELECT ',' + QUOTENAME(CONVERT(VARCHAR(6), c.clockingdate, 106))
                    FROM DateCTE c
					ORDER BY C.clockingdate
                    FOR XML PATH(''), TYPE).value('.', 'NVARCHAR(MAX)'), 1, 1, '');
SET @query = 
  N'
  DECLARE @starttime datetime, @enddate datetime
  set @starttime = (select startdate from #tempdates)
  set @enddate = (select enddate from #tempdates)
  Select * , ' + @cols + ' from(

SELECT 
	r.Empref, 
	Si.sitename,
	D.deptname,
	j.jobpostname,
	E.payrollno,
	e.forenames, 
	--e.surname, 
	CASE 
		WHEN r.Shiftref = -1 THEN ''DayOff''
               ELSE coalesce(dbo.fn_ConvertIntToTimeStr(S.starttime)+'' - ''+dbo.fn_ConvertIntToTimeStr(s.endtime),''Blank'')
          END AS Name,
	CONVERT(VARCHAR(6), r.clockingdate, 106) AS clockingdate_formatted
FROM 
rota r
LEFT JOIN shifts s ON r.Shiftref = s.Shiftref
left JOIN empdetails e ON r.Empref = e.Empref
left join deptments D on E.deptref = D.deptref
left join sites Si on E.siteref = Si.siteref
left join jobposts J on J.jobpostref = E.jobpostref
where r.clockingdate between @starttime and @enddate-- and R.empref = 161
	  UNION ALL
		Select 
		A.empref,
		Si.sitename,
		D.deptname,
		j.jobpostname,
		E.payrollno,
		e.forenames, 
		--e.surname,
		coalesce(AC.codename,''Blank'') as Name,
		CONVERT(VARCHAR(6), absdate, 106) AS clockingdate_formatted
		from absences A
		left join abscodes AC on AC.coderef = A.coderef
		left JOIN empdetails e ON A.Empref = e.Empref
		left join deptments D on E.deptref = D.deptref
		left join sites Si on E.siteref = Si.siteref
		left join jobposts J on J.jobpostref = E.jobpostref
		where A.absdate between @starttime and @enddate-- and R.empref = 161
		
  ) x
  PIVOT 
  (
      MAX(Name)
      FOR clockingdate_formatted IN (' + @cols + ' )
  ) p
  
  Order by P.forenames';
  
EXEC sp_executesql @query;
drop table #tempdates
