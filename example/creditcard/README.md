# The Credit Card Fraud Detection Example

The `downsample/creditcard.csv` file in this directory comes from [Kaggle](https://www.kaggle.com/mlg-ulb/creditcardfraud). 

## Down-sample

To make it an example that can be checked into a git repo, I randomly picked up 2000 out from the 284,808 lines from the original dataset using the following command:

```bash
gshuf -n 5000 /tmp/creditcard.csv  > ./downsample/creditcard.csv
```

where `/tmp/creditcard.csv` is the one downloaded from Kaggle.

If you want to down-sample by yourself on macOS, you might need to install `gshuf`.

```bash
brew install coreutils
```

## Import into MySQL

To develop/test/debug the TensorFlow program automatically generated by SQLFlow, which is supposed to be able to read data from MySQL, we'd import the `downsample/creditcard.csv` into MySQL.

### Run MySQL Server

For more about running MySQL in Docker containers, please refer to [Run MySQL Server and Client in Docker Containers](../../doc/mysql-setup.md).

The following command starts the MySQL server:

```bash
docker run --rm \
   -v $HOME/work/creditcarddb:/var/lib/mysql \
   --name mysql01 \
   -e MYSQL_ROOT_PASSWORD=root \
   -e MYSQL_ROOT_HOST='%' \
   -p 3306:3306 \
   -d \
   mysql/mysql-server:8.0 \
   mysqld --secure-file-priv=""
```

Please be aware that the above command doesn't use the default entrypoint of the Docker image; instead, it explicitly start `mysqld` in the container so that it could provide the command line option `--secure-file-priv=""`, which is required by `mysqlimport`.

Run the following command to start a MySQL client:

```bash
docker run --rm -it \
   -v $HOME/work/creditcarddb:/var/lib/mysql \
   mysql/mysql-server:8.0 \
   mysql -uroot -p
```

Remember to type the password `root` when prompted.  Then input the following SQL program in the client to create the table `creditcard.creditcard`:

```sql
CREATE DATABASE creditcard;
USE creditcard;
CREATE TABLE IF NOT EXISTS creditcard (
    time INT,
	v1 float,
	v2 float,
	v3 float,
	v4 float,
	v5 float,
	v6 float,
	v7 float,
	v8 float,
	v9 float,
	v10 float,
	v11 float,
	v12 float,
	v13 float,
	v14 float,
	v15 float,
	v16 float,
	v17 float,
	v18 float,
	v19 float,
	v20 float,
	v21 float,
	v22 float,
	v23 float,
	v24 float,
	v25 float,
	v26 float,
	v27 float,
	v28 float,
	amount float,
	class varchar(255));
```

Before we could import the CSV file, we must follow the security policy of MySQL, which is wired and hard to understand, and copy the CSV file into the database directory:

```bash
cp downsample/creditcard.csv ~/work/creditcarddb/creditcard/
```

The following command runs `mysqlimport` in the `mysql/mysql` Docker image to import the data:

```bash
docker run --rm -it \
    -v $HOME/work/creditcarddb:/var/lib/mysql \
	-v $HOME/work/creditcard:/var/lib/mysql-files \
    mysql:8.0 \
    mysqlimport \
	  --ignore-lines=1  \
      --fields-terminated-by=,  \
      --socket /var/lib/mysql/mysql.sock \
	  -u root -p \
	  creditcard creditcard.csv 
```

Please be aware that `mysqlimport` doesn't exist in the `mysql/mysql-server` image.

A message likes the following indicates the success of importing:

```
creditcard.creditcard: Records: 1999  Deleted: 0  Skipped: 0  Warnings: 0
```

We can verify the imported result by running a SELECT statement like the following

```bash
docker run --rm -it \
   -v $HOME/work/creditcarddb:/var/lib/mysql \
   mysql/mysql-server:8.0 \
   mysql -uroot -p \
   -e 'use creditcard; select time, v1, v28, amount, class from creditcard limit 10;'
```

which should print something like the following

```
+--------+-----------+------------+--------+-------+
| time   | v1        | v28        | amount | class |
+--------+-----------+------------+--------+-------+
| 123590 | -0.341154 |   0.130741 |     69 | "0"   |
| 135797 | -0.118853 |    0.11599 |  11.79 | "0"   |
| 161249 |  -1.04129 |  -0.362025 |   7.38 | "0"   |
|  72471 |  0.603151 |  0.0551408 | 352.04 | "0"   |
| 120182 |   2.10678 | -0.0566284 |   0.89 | "0"   |
| 160476 |   2.37223 | -0.0724277 |      6 | "0"   |
| 135023 |   1.87987 |  0.0214597 |  20.05 | "0"   |
| 117329 | -0.375213 |  -0.130152 |    159 | "0"   |
| 148637 |   1.83178 | -0.0378951 |  38.37 | "0"   |
|  48293 |   1.26094 |  0.0275234 |     12 | "0"   |
+--------+-----------+------------+--------+-------+
```