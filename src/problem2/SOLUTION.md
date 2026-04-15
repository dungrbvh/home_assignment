Provide your solution here:
Funtional requirements
Core Requirements
1. Users can see live prices of stocks
2. Users can manage orders for stocks(market/limit orders, create/cancel orders)
3. Ledger and settlement, keeping position

Non-functional 
1. High avilability > Consistency
2. Throughput 500 requests per second
3. Latency reponse time p99<100ms

Out-of-scope
1. Payment gateway assume using Stripe
2. Observability 

Overview Diagram
![alt text](image.png)
Cloud services and their roles:
AWS Kinesis Stream or Kafka: streaming market data sources such as NYSE, NASDAQ, other trading platforms
AWS Kinesis Firehorse, EMR, Spark: Clean, process raw stream, data normalization and data aggregation for best market price
AWS DynamoDB: to store aggregated market data, can handle millions of market event per second, read replica, sharding, multi-AZ.
AWS ElasticCache-Redis: To store some common stocks for trading such as BTC, ETH, BNC. Served user for low latency. Used as Redis cluster for high availability. Alternatives: Memcached (cache-only). Multi-AZ enabled
AWS Opensearch or ElasticSearch: serve as stock lookup, it read in Cache first, it hits to DynamoDB
AWS Cloudfront: CDN+WAF prevent bot attack, rate limiting, throttling 
AWS API Gateway: JWT validation, routing to services such as user, order, search. Dedicated WebSocket API to serve real-time tier.
AWS ECS, Fargate,EKS: serverless, event-driven. 
AWS Cognito: Auth, KYC, Temporary token authenticated from User database.
AWS Aurora or RDS: Store user data, order data, Balances, withdrawal states benefit from ACID and mature tooling. 
AWS SQS or EventBus: message queue for orders, prevent lost orders during high peak period. 
AWS VPC: VPC private subnets + NAT for egress for security and compliance

Scaling plan
Increase Kinesis stream partitions to handle more market events
Increase processing units in Firehorse 
Scaling ECS tasks or use EKS alternatively to handle much traffic.
Sharding and read-replica on Aurora or RDS to handle spike of orders during peak period
Increase more resources on Opensearch to handle search requests
Scaling DynamoDB with sharding, read-replica enabled. 
