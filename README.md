# Azure Data Engineering Challenge

I developed this challenge using **Microsoft Azure**, the cloud platform I have the most experience with. The goal was to automate the loading, processing, and querying of information related to employees, job positions, and departments. The data was provided in CSV files, which I integrated into an SQL database and exposed via REST APIs.

Since the tasks were relatively simple and time was limited, I chose a **low-code/no-code** approach. This allowed me to move quickly using Azure's visual tools and managed services, without writing much custom code.

## Azure Components Used

Here are the main resources I created and used throughout the challenge:

- **challengemigrationfiles (Storage Account)**: used to upload the CSV files.
- **challengepipeline (Azure Data Factory)**: orchestrates reading and loading the data.
- **challengedb (Azure SQL Database)**: where the structured data is stored.
- **challengeserverdb (Azure SQL Server)**: SQL server hosting the database.
- **ChallengeSqlApi and ChallengeSqlApi2 (Logic Apps)**: expose queries as REST APIs.

## Development Steps

### 1. Uploading files to blob storage

I used the following Postman endpoints:

1. First, I ran a `POST` request called **ApiLogin** to obtain the authentication token. This step is mandatory, as Azure Blob Storage requires a valid token for the next requests.

2. Then, I used three `PUT` requests to upload the files:
   - **ApiUploadJobs**
   - **ApiUploadDepartments**
   - **ApiUploadEmployees**

The files uploaded were `jobs.csv`, `departments.csv`, and `hired_employees.csv`, and the requests were secured using OAuth2.

### 2. Orchestration with Data Factory

To automate the process, I designed a pipeline in Azure Data Factory that triggers whenever a file is uploaded to blob storage and loads the data into SQL.

- There’s one copy activity per file.
- Each file has a specific dataset and linked service.
- The target tables in the database also have their own datasets.

The pipeline is event-triggered, meaning it runs automatically upon new file uploads.

Full pipeline definition is available here:  
[https://github.com/yrociorm/challengedatafactory](https://github.com/yrociorm/challengedatafactory)

### 3. Database Modeling

I created a simple and consistent relational model with three tables:

- **departments**: `id`, `name`
- **jobs**: `id`, `title`
- **employees**: `id`, `name`, `hire_date`, `job_id`, `department_id`

The `employees` table uses foreign keys to reference `departments` and `jobs`, ensuring referential integrity and enabling consistent joins across the model.

### 4. Query Exposure with Logic Apps

I created two Logic Apps to expose SQL queries as REST APIs:

- The first returns the number of hires by department and quarter for a given year:

```sql
SELECT
    d.name AS department,
    j.title AS job,
    COUNT(CASE WHEN DATEPART(QUARTER, he.hire_date) = 1 THEN 1 END) AS Q1,
    COUNT(CASE WHEN DATEPART(QUARTER, he.hire_date) = 2 THEN 1 END) AS Q2,
    COUNT(CASE WHEN DATEPART(QUARTER, he.hire_date) = 3 THEN 1 END) AS Q3,
    COUNT(CASE WHEN DATEPART(QUARTER, he.hire_date) = 4 THEN 1 END) AS Q4
FROM
    employees he
JOIN
    departments d ON he.department_id = d.id
JOIN
    jobs j ON he.job_id = j.id
WHERE
    YEAR(he.hire_date) = @{triggerOutputs()['queries']['year']}
GROUP BY
    d.name,
    j.title
ORDER BY
    d.name,
    j.title;
```

- The second returns departments with above-average hires in the same year:

```sql
DECLARE @avg_hires FLOAT;

SELECT
    @avg_hires = AVG(hired_count * 1.0)
FROM (
    SELECT
        department_id,
        COUNT(*) AS hired_count
    FROM
        employees
    WHERE
        YEAR(hire_date) = @{triggerOutputs()['queries']['year']}
    GROUP BY
        department_id
) AS sub;

SELECT
    he.department_id id,
    d.name AS department,
    COUNT(*) AS hired
FROM
    employees he
JOIN
    departments d ON he.department_id = d.id
WHERE
    YEAR(he.hire_date) = @{triggerOutputs()['queries']['year']}
GROUP BY
    he.department_id, d.name
HAVING
    COUNT(*) > @avg_hires
ORDER BY
    hired DESC;
```

Both APIs are accessible via HTTP.

### 5. Validation and Testing

I created a Postman collection to run the full workflow, from authentication to file upload and query testing.

The collection is included so recruiters can easily reproduce the process and verify the results.

## Conclusion

- The low-code/no-code approach allowed me to deliver a working solution quickly.
- I used modular, reusable, and fully managed Azure services.
- The entire pipeline is automated from file upload to data querying.
- The APIs are production-ready and easy to integrate with external systems.

This solution demonstrates how to build a robust and scalable data pipeline in a short time using the right tools in Azure.
