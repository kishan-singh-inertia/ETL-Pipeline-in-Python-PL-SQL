## ETL Pipeline in Python & Oracle SQL.
---
Extracted data from expenses.xlsx and 
```
https://www.bankofcanada.ca/valet/observations/FXUSDCAD/json?start_date={start_date}
```
---
Code:
```python
import petl
import requests
import datetime
import json
import decimal
import oracledb
import sys



url = 'https://www.bankofcanada.ca/valet/observations/FXUSDCAD/json?start_date='
start_date = '2020-01-01'

res = requests.get(url+start_date)

dates = []
rates = []



if (res.status_code == 200):
    res_json = json.loads(res.text)

    for row in res_json['observations']:
        dates.append(datetime.datetime.strptime(row['d'],'%Y-%m-%d'))
        rates.append(decimal.Decimal(row['FXUSDCAD']['v']))
    
    exch_rates = petl.fromcolumns([dates,rates],header=['date','rate'])
    expenses = petl.io.xlsx.fromxlsx('Expenses.xlsx',sheet='Github')

    ex_exp = petl.outerjoin(exch_rates,expenses,key='date')
    ex_exp = petl.filldown(ex_exp,'rate')
    ex_exp = petl.select(ex_exp,lambda rec: rec.USD != None)
    ex_exp = petl.addfield(ex_exp,'CAD', lambda rec: decimal.Decimal(rec.USD) * rec.rate)


    connection = oracledb.connect(
    user="user",
    password='pass',
    dsn="localhost/FREEPDB1")

    cursor = connection.cursor()

    cursor.execute("""
        DROP TABLE IF EXISTS Expenses
                   """)

    cursor.execute("""
        CREATE TABLE Expenses(
	        dates DATE,
	        USD NUMBER(10,5),
	        rate NUMBER(10,5),
	        CAD NUMBER(10,5))""")
    
    ex_exp_array = list(petl.toarray(ex_exp))
    cursor.executemany("INSERT INTO expenses (dates, USD , rate, CAD) values(:1, :2, :3, :4)", ex_exp_array)
    print(cursor.rowcount, "Rows Inserted")

    connection.commit()
else:
    sys.exit()
```
---
Required package:
```
pip install oracledb
pip install petl
pip install openpyxl
```
---
