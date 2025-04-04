# End-to-End Data Pipeline with Iceberg, Trino, MinIO, and PostgreSQL

This project implements an end-to-end data pipeline using:
- **MinIO** as S3-compatible object storage
- **PostgreSQL** for catalog metadata and final data storage
- **Iceberg** tables for data format
- **Trino** as the query engine
- **dbt Core** for transformations
- **Cron jobs** for scheduling

## Architecture Overview

![Architecture Diagram](https://i.imgur.com/7vStcn7.png)

### Components

1. **Storage Layer**: 
   - MinIO (S3-compatible) for object storage
   - Organized into raw, stage, and mart buckets

2. **Metadata Layer**: 
   - PostgreSQL for Iceberg catalog metadata
   - Also used for storing final transformed data

3. **Transformation Layer**: 
   - dbt Core for running transformations
   - Follows the raw → stage → mart pattern

4. **Query Layer**: 
   - Trino for querying both PostgreSQL and Iceberg tables
   - Used for performance testing against direct source access

5. **Orchestration**: 
   - Cron jobs for scheduling dbt runs
   - Python wrapper for job management and encapsulation

## Data Flow

The pipeline supports two scenarios:

### Scenario 1: PostgreSQL source data
1. Source data is read from PostgreSQL tables
2. dbt transforms and writes to S3 (raw layer)
3. Further transformations create the stage layer in S3
4. Final mart layer is written back to PostgreSQL

### Scenario 2: S3 source data
1. Source data is read from S3 parquet files
2. dbt transforms and writes to S3 (raw layer)
3. Further transformations create the stage layer in S3
4. Final mart layer is written back to PostgreSQL

## Getting Started

### Prerequisites

- Docker and Docker Compose
- At least 8GB of RAM for the container stack
- 20GB of free disk space

### Installation

1. Clone this repository:
   ```bash
   git clone <repository-url>
   cd <repository-directory>
   ```

2. Run the setup script:
   ```bash
   ./setup.sh
   ```

The setup script will:
- Create all necessary directories
- Copy configuration files to the right locations
- Build Docker images
- Start the container stack
- Generate test data
- Run the full dbt pipeline
- Run performance analysis
- Create encapsulated models

### Accessing Components

- **MinIO UI**: http://localhost:9001 (minioadmin/minioadmin)
- **Trino UI**: http://localhost:8080 (user: trino)
- **PostgreSQL**: localhost:5432 (postgres/postgres)

## Project Structure

```
├── config/
│   └── trino/
│       ├── catalog/
│       │   ├── iceberg.properties
│       │   └── postgresql.properties
│       └── config.properties
├── dbt/
│   ├── models/
│   │   ├── raw/
│   │   │   ├── raw_customers.sql
│   │   │   ├── raw_orders.sql
│   │   │   └── s3_products.sql
│   │   ├── stage/
│   │   │   ├── stg_customers.sql
│   │   │   ├── stg_orders.sql
│   │   │   └── stg_products.sql
│   │   ├── mart/
│   │   │   ├── customer_orders.sql
│   │   │   └── product_sales.sql
│   │   ├── encapsulated/
│   │   │   ├── enc_customer_orders.sql
│   │   │   └── enc_product_sales.sql
│   │   └── schema.yml
│   ├── dbt_project.yml
│   ├── requirements.txt
│   ├── profiles.yml
│   ├── dbt-runner.sh
│   ├── dbt_wrapper.py
│   └── crontab
├── data-generator/
│   ├── Dockerfile
│   └── generate_data.py
├── performance-analyzer/
│   ├── Dockerfile
│   └── analyze_performance.py
├── docker-compose.yml
└── setup.sh
```

## DBT Models

### Raw Layer
- `raw_customers.sql`: Raw customer data from PostgreSQL
- `raw_orders.sql`: Raw order data from PostgreSQL
- `s3_products.sql`: Raw product data from S3

### Stage Layer
- `stg_customers.sql`: Transformed customer data
- `stg_orders.sql`: Transformed order data
- `stg_products.sql`: Transformed product data

### Mart Layer
- `customer_orders.sql`: Customer order analytics
- `product_sales.sql`: Product sales analytics

### Encapsulated Models
- `enc_customer_orders.sql`: Encapsulated version of customer_orders
- `enc_product_sales.sql`: Encapsulated version of product_sales

## Running Jobs Manually

### Full Pipeline
```bash
docker-compose exec dbt ./dbt-runner.sh full
```

### Individual Layers
```bash
docker-compose exec dbt ./dbt-runner.sh raw-postgres
docker-compose exec dbt ./dbt-runner.sh raw-s3
docker-compose exec dbt ./dbt-runner.sh stage
docker-compose exec dbt ./dbt-runner.sh mart
```

### Using the Python Wrapper
```bash
# Run specific models
docker-compose exec dbt python dbt_wrapper.py run --models model1 model2

# Run tests
docker-compose exec dbt python dbt_wrapper.py test

# Get model lineage
docker-compose exec dbt python dbt_wrapper.py lineage --model-name customer_orders

# Create encapsulated model
docker-compose exec dbt python dbt_wrapper.py encapsulate --model-name source_model --encap-name encapsulated_model
```

## Performance Analysis

The performance analyzer compares:
1. Direct PostgreSQL queries
2. Trino queries on PostgreSQL
3. Trino queries on Iceberg tables

Results are stored in:
- `performance/benchmark_results.csv`: Raw results
- `performance/benchmark_results.png`: Chart visualizations
- `performance/benchmark_report.md`: Detailed report with findings

## Scheduling Jobs

Jobs are scheduled using cron:
- Raw layer: Daily at 1:00 AM and 1:30 AM
- Stage layer: Daily at 2:00 AM
- Mart layer: Daily at 3:00 AM
- Full pipeline: Weekly on Sunday at 12:00 AM
- Tests: Daily at 4:00 AM

To modify the schedule, edit `dbt/crontab` and restart the container.

## Model Encapsulation

The project includes functionality to encapsulate dbt models. This provides:
- A stable interface for downstream consumers
- The ability to hide implementation details
- Consistent documentation

Use the Python wrapper to create encapsulated models:
```bash
docker-compose exec dbt python dbt_wrapper.py encapsulate --model-name source_model --encap-name encapsulated_model
```

## Troubleshooting

### Common Issues

1. **Services are not starting**
   - Check Docker logs: `docker-compose logs`
   - Ensure ports are not already in use

2. **dbt jobs failing**
   - Check logs in `dbt/logs/`
   - Try running with debug: `docker-compose exec dbt ./dbt-runner.sh debug`

3. **Performance is slow**
   - Increase Docker resource allocation
   - Check MinIO and Trino logs for bottlenecks

4. **Data not appearing in queries**
   - Verify data was generated correctly: `docker-compose exec postgres psql -U postgres -c "SELECT COUNT(*) FROM customers"`
   - Check MinIO buckets for Parquet files

### Getting Help

For issues related to specific components:
- MinIO: https://min.io/docs
- Trino: https://trino.io/docs/current/
- Iceberg: https://iceberg.apache.org/docs/
- dbt: https://docs.getdbt.com/

## Extending the Project

### Adding New Data Sources

1. Add a new source to `dbt/models/schema.yml`
2. Create raw models in `dbt/models/raw/`
3. Add transformation models in `dbt/models/stage/`
4. Update mart models as needed

### Adding New Transformations

1. Create new models in the appropriate layer directory
2. Update schema.yml with column definitions
3. Update the dependencies in your mart layer models

### Custom Scheduling

Edit `dbt/crontab` to customize the job schedule according to your needs.

## Conclusion

This project demonstrates a full-featured data pipeline using modern cloud-native technologies. The architecture is designed to be:

- **Scalable**: Can handle large datasets with distributed processing
- **Flexible**: Supports multiple data sources and formats
- **Maintainable**: Well-organized code structure and encapsulated models
- **Performant**: Optimized for query performance across different engines

By leveraging the combination of Iceberg, Trino, MinIO, and PostgreSQL, you get the benefits of a data lakehouse architecture without the complexity of a full Hadoop/Spark stack.
