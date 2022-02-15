# TR Data challenge

## Data modelling and ETL

### Setup

The appraoch I follwed for this part of the project has been an ELT pipeline. I often prefer to ingest raw data as they come from the different sources, applying any kind of transformation on a second stage.

In order to solve this issue I decided to create a Postgresql database cluster node on DigitalOcean at the cost of 5$/month.

With the database available, and after having explored and understood the data received, I have created [an SQL file](https://github.com/dolphinxyz/TR_Challenge/blob/main/init.sql) to instanciate the tables necessary for the challenge.

Finally after having defined the environmental variables on my machine, I worked on a Golang script to run the SQL file above created, as well as pushing the data from the CSV files into the proper table.

I have decided to use Golang because of different reasons:

- It is extremely fast
- It can scale really well for large amount of data thanks to its natural predisposition for parallelism and concurrency jobs
- It does not require too many dependencies
- Structs are a great way to abstract data
- It is easy to read because automatically formatted, the code looks always the same

[Here](https://github.com/dolphinxyz/TR_Challenge/blob/main/load/load.go) the golang script.
First of all, I define all the data structures I need. Then within the main function I instanciate a database connection, I run the initial SQL script, and then finally I push the data from the three tables received.

During the last part of the job I noticed some inconciestences in the *trades.csv* file. For the purpose of this challenge I have decided to remove those problems manually:

- row 681620: missing ExecutionSize and ExecutionPrice
- row 681734: invalid InstrumentId (900)
- row 689718: missing InstrumentId

Many things could be improved:

- Better manage of timestamp
- More sophisticated way to detect unique values
- Dynamic chunck size allocation
- Creating strong unit tests

### Answering questions

After having explored the questions making some initial attempts, I relised how all the questions could have been answered at once with a proper query as well as a proper tool to explore the results.

For this reason I decided to run a [Metabase](https://www.metabase.com/) instance on DigitalOcean.

I downloaded the image from DockerHub and I uploaded it on DigitalOcean. After having created a database to host the queries, my instance is ready to go.

You can log in [here](https://metabase-kicdm.ondigitalocean.app/), credentials have been sent separately. You will find a single query saved on [this](https://metabase-kicdm.ondigitalocean.app/question/1-questions) link.

One thing I notice about the data is that it does not only contains 6 "accounts", but many more.

[Here](https://github.com/dolphinxyz/TR_Challenge/blob/main/questions.sql) the query I wrote, and below a snapshot of the solution:

![snapshot](https://github.com/dolphinxyz/TR_Challenge/blob/e2cd0fd6f56d685f59f8b0d6d1555b0e0fc1d2f7/questions.png)

It would be great if we could discuss the answers of your questions during our conversation.

An important detail is this condition of the query:

```sql
WHERE a.qty_sells <= a.qty_buys
```

Since the data I received are a mere snapshot, for some users the quantity of *sells* was bigger than the quanity of *buys*, which is of course impossible. So I added this condition to make everything more reasonable.

### Technical follow-up

#### Improving performance - loading

I assume that the huge amount of data available to TR has been acquired in several months/years of work, so I exclude the necessity to have in production a system than needs to injest terabyte of data at once.

Despite so, it is critically important to rely on a lean and efficent system to pull data from different sources whatever size they are.

In order to do so, I would create an agile microservice architecture, in which each job has a single specifical goal.

Ideally this can be achieved with a Kubernetes cluster to orchestrate all the containers necessary to execute the load of the data. With Grafana we could set up a monitoring system in order to always having a clear overview of the status of the jobs. Moreover, using a proper programming language, all those jobs could be optimized and divided into concurrent ones, in order to speed up the data load as much as possibile.

Lastly the choice of the database is critical, despite being a huge fan of Postgresql, other solutions could be evaluated, for example Cockroachdb or Snowflake.

#### Improving performance - transforming

Following the architecture created above, the whole activity of data transformation could be orchestreated by an army of microservices connected to each other. A system based on Lambda functions could be reasonable built, calling a specific lambda as soon as another one has been completed. To manage this aspect of the company, we could think about using Python and Apache Airflow, keeping everything in once place.

Relying on proper servers, as well as on efficient software, make the amount of data to transform less problematic. 

Moreover, ad-hoc activities could be tackled in separate environments.

#### Improving performance - Query

Industry has developed clever solutions to analyze massive amount of information. One of my favourite is using Google BigQuery. They are able to fully hide any complexity allowing analysts to simply write queries as they would do with small datasets.

Considering the costs connected with solutions as BigQuery, it is essential to rely on well-designed data structures. A cumbersome data architecture makes queries slow and inefficent, decreasing the work-quality of analysis and all the people depending on those data.

#### Testing correctness

For each job of the microservices architecture above described it is mandartory to write unit tests to ensure their correcteness. A job can be considered production ready only if a set of tests has been created. Moreover, also integration tests can be designed to fully test a whole data pipeline.

Moreover, another way to strenght the correctness of the architecture is to set up a smart alerting system. Whenever a job fails, get notified on different medium, specifying the urgency of a certain event (alert, warn, error, fatal). With those alerts in place it is much easier being confident that jobs have been correctely executed.

On top of tests and alerts, another component is to rely on a system to visualise the health of the whole architecture at once, using browser based application.

## Add Data API

For the second challenge I decided to use [this](https://freecurrencyapi.net/dashboard) service, offering 50'000 free API calls per months.

To fullfull the purpose of your request, I am creating a new table into the database, called *currencies*:

```SQL
CREATE TABLE currencies (
	date VARCHAR(10) NOT NULL,
	base_currency VARCHAR(5) NOT NULL,
	currency VARCHAR(5) NOT NULL,
	price numeric NOT NULL,
	UNIQUE(date, currency)
);
```

Moreover, to ease my query, I inserted also the following table called *prefixes*, adding on it the table presented in the document of this challenge:

```SQL
CREATE TABLE prefixes (
	last_digit VARCHAR(1) NOT NULL,
	currency VARCHAR(5) NOT NULL
);
```

Now, using Golang I have been able to fetch the API endpoint with the currency rates of each date, filling up the database's table.

You can find the Golang script [here](https://github.com/dolphinxyz/TR_Challenge/blob/main/api/api.go).

Now finally, using SQL, I can expand the query created for the previous step, showing the value of each account's portfolio in the requested currency.

The query is available [here](https://github.com/dolphinxyz/TR_Challenge/blob/main/api.sql) and also on [this link](https://metabase-kicdm.ondigitalocean.app/question/3-api-challenge) on metabase.