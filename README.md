# mysql-docker
Examples for MySQL Docker deployments

--> Make a csv file accessible inside of a mysql database (goal of ODBC connections from RStudio to MySQL for querying data on an ASC MySQL DB)
- https://www.mysqltutorial.org/import-csv-file-mysql-table/
- https://dev.mysql.com/doc/refman/8.0/en/mysql.html


## Getting Started

``` bash
# clone repo & navigate to this directory
cd mysql-docker

# for volume mounted persistent data:
mkdir data

# run your mysql db container
# be sure to set absolute path for volume mount
# change the password if you want! this is for dev of course ...
docker run --name mysql-test -v /Users/etiennejacquot/Documents/GitHub/mysql-docker/data/:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=password123 -d mysql:latest
```

### Preparing our example CSV data for SQL:

We want to create our table for a sample csv file... to start we need our datatypes, for this we run:

```python
import pandas as pd
df = pd.read_csv('./data/alvinzhou_example.csv')
df.dtypes()
    website      object
    source       object
    month         int64
    category     object
    measure      object
    value       float64
    dtype: object
```

Thus our SQL syntax probably should look something like this:

``` SQL
CREATE TABLE name_of_database:name_of_table_to_create (
    website TEXT,
    source TEXT,
    month INT,
    category TEXT,
    measure TEXT,
    value_ FLOAT
);
```
_______________

## Connect to mysql-test container and create our database & table:

We attach directly to the mysql container... there of course are alternatives for using another client to make a remote connection.

``` bash
# connect to container
docker exec -it mysql-test bash

# authenticate to mysql
mysql -u root -p # <--- password123

# (UPDATED) for local infile load if needed:
mysql --local-infile=1 -u root -p
```

Make sure the example csv file is loaded in `./data/` (in this case, provided by ASC Graduate student Alvin Zhou.
- load our file from volume mount, into table `csv_testing.example_table`, comma separated & delineated by line break, ignoring the header row
- Please note this is a LOCAL file, on secure MySQL deployments loading darta is not as simple

``` sql
-- Confirm that local infile upload is set to ON, 
-- if not you will likely get errors when loading the file
SHOW global variables like 'local_infile';

-- Create your database
CREATE DATABASE csv_testing;

-- Create your table for example csv file
CREATE TABLE IF NOT EXISTS csv_testing.example_table (
    website TEXT,
    source TEXT,
    month INT,
    category TEXT,
    measure TEXT,
    value_ FLOAT
);

-- Load the local example csv file
LOAD DATA LOCAL INFILE '/var/lib/mysql/alvinzhou_example.csv' 
INTO TABLE csv_testing.example_table 
FIELDS TERMINATED BY ',' 
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;

-- Sample the table to confirm data is loaded!
SELECT * FROM csv_testing.example_table limit 10;
```
s
At this point you should now have data successfully loaded in a MySQL database! If you get an error on loading, see below.

### Finally, making an ODBC connection from RStudio (or Python) to our local docker MySQL container:

End users want RStudio, I am going to try with Python since I have this readily available on my local system.

- more info here: https://pypi.org/project/pyodbc/
- `pip install pyodbc`

``` python
import pandas as pd
import pyodbc

### TO BE CONTINUED ... 

```
___________

## Errors for importing file to MySQL:

I got the following error, described [here](https://stackoverflow.com/questions/32737478/how-should-i-tackle-secure-file-priv-in-mysql):
    
        The MySQL server is running with the --secure-file-priv option so it cannot execute this statement

this recommends the `secure_file_priv` but I don't want to restart my container ... this could be editing in cnf configuration as well

``` sql
SHOW VARIABLES LIKE "secure_file_priv";
```

To resolve I just just enabled `LOCAL` file loading (remember our data file is volume mounted into the container), not sure how this would get imported for a production host
