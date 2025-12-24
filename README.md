CUDOS is a prebuilt AWS QuickSight dashboard that shows your cost, usage, and savings insights using data from the AWS Cost & Usage Report (CUR). It works by pulling CUR data from S3, processing it via Glue, querying it with Athena, and visualizing it in QuickSight. You deploy it using a CloudFormation stack, which auto-sets up everything — S3, Glue, Athena, and dashboards — in one go. It helps you easily spot cost anomalies, top services, and optimization opportunities. No coding needed — just launch and use.
https://catalog.workshops.aws/awscid/en-US/dashboards/foundational/cudos-cid-kpi/deploy


Athena Query
CREATE OR REPLACE VIEW cid_data_export.vw_cost31 AS
SELECT
  CAST(line_item_usage_account_id AS VARCHAR) AS account_id,
 
  -- Standardized service name
  COALESCE(
    CASE
      WHEN line_item_product_code = 'AmazonEC2' THEN 'Amazon EC2'
      WHEN line_item_product_code = 'AmazonRDS' THEN 'Amazon RDS'
      WHEN line_item_product_code = 'AWSELB' THEN 'Elastic Load Balancing'
      WHEN line_item_product_code = 'AWSConfig' THEN 'AWS Config'
      WHEN line_item_product_code = 'AmazonCloudWatch' THEN 'Amazon CloudWatch'
      WHEN line_item_product_code = 'AmazonVPC' THEN 'Amazon VPC'
      WHEN line_item_product_code = 'AmazonS3' THEN 'Amazon S3'
      WHEN line_item_product_code = 'AmazonDynamoDB' THEN 'Amazon DynamoDB'
      WHEN line_item_product_code = 'AmazonRedshift' THEN 'Amazon Redshift'
      WHEN line_item_product_code = 'AmazonEKS' THEN 'Amazon EKS'
      WHEN line_item_product_code = 'AmazonES' THEN 'Amazon OpenSearch'
      WHEN line_item_product_code = 'AmazonGuardDuty' THEN 'Amazon GuardDuty'
      WHEN line_item_product_code = 'AWSGlue' THEN 'AWS Glue'
      WHEN line_item_product_code = 'AmazonQuickSight' THEN 'Amazon QuickSight'
      ELSE line_item_product_code
    END,
    product['product_name'],
    product['productName'],
    'unknown'
  ) AS service,
 
  -- Region
  COALESCE(product['region_code'], product_region_code, 'unknown') AS region,
 
  -- Resource type logic
  CASE
    WHEN COALESCE(product['product_name'], product['productName'], 'unknown') = 'Amazon Virtual Private Cloud'
         AND line_item_usage_type LIKE '%PublicIPv4%' THEN 'Public/Elastic IPs'
    WHEN COALESCE(product['product_name'], product['productName'], 'unknown') = 'Amazon Virtual Private Cloud'
         AND line_item_usage_type LIKE '%TransitGateway%' THEN 'TransitGateway'
    WHEN COALESCE(product['product_name'], product['productName'], 'unknown') = 'Amazon Virtual Private Cloud'
         AND line_item_usage_type LIKE '%VpcEndpoint%' THEN 'VpcEndpoint'
    ELSE COALESCE(NULLIF(product_product_family, ''), 'unknown')
  END AS resource_type,
 
  -- Product family
  COALESCE(NULLIF(product_product_family, ''), 'unknown') AS product_family,
 
  -- Full usage type
  line_item_usage_type AS instance_type,
 
  -- Resource ID
  line_item_resource_id AS resource_id,
 
  -- Smart instance size extraction
  CASE
    WHEN line_item_product_code = 'AmazonEC2' AND REGEXP_LIKE(line_item_usage_type, '.*[:/]([a-z0-9.]+)$') 
         AND REGEXP_LIKE(line_item_usage_type, '.*(t|m|c|r|i|x|z|a)[0-9].*\..*') THEN 
      REGEXP_EXTRACT(line_item_usage_type, '.*[:/]([a-z0-9.]+)$')
 
    WHEN line_item_product_code = 'AmazonRDS' AND REGEXP_LIKE(line_item_usage_type, '.*[:/](db\.[a-z0-9.]+)$') THEN 
      REGEXP_EXTRACT(line_item_usage_type, '.*[:/](db\.[a-z0-9.]+)$')
 
    WHEN line_item_product_code = 'AmazonES' AND REGEXP_LIKE(line_item_usage_type, '.*[:/]([a-z0-9.]+\.elasticsearch)$') THEN 
      REGEXP_EXTRACT(line_item_usage_type, '.*[:/]([a-z0-9.]+\.elasticsearch)$')
 
    ELSE NULL
  END AS instance_size,
 
  -- Created by tag
  COALESCE(resource_tags['aws_created_by'], 'N/A') AS created_by,
 
  -- Corrected Custom Tags
  COALESCE(resource_tags['user_cost_center'], 'N/A') AS cost_center,
  COALESCE(resource_tags['user_owner'], 'N/A') AS owner_tag,
  COALESCE(resource_tags['user_name'], 'N/A') AS name_tag,
  COALESCE(resource_tags['user_team'], 'N/A') AS team_tag,
 
  -- Charge type
  line_item_line_item_type AS charge_type,
 
  -- Cost
  ROUND(COALESCE(line_item_unblended_cost, 0.0), 5) AS total_cost,
 
  -- Date fields
  DATE(line_item_usage_start_date) AS usage_start_date,
  DATE(line_item_usage_end_date) AS usage_end_date
 
FROM cid_data_export.cur2
WHERE 
  line_item_line_item_type <> 'Tax';

Screenshots of Quicksight dashboard
 


 


 
 
 
 



