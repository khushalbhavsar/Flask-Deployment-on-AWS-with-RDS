# Interview Questions & Answers - AWS Flask + RDS Deployment

## Table of Contents
- [Flask Application Questions](#flask-application-questions)
- [AWS Elastic Beanstalk Questions](#aws-elastic-beanstalk-questions)
- [AWS RDS & Database Questions](#aws-rds--database-questions)
- [VPC & Networking Questions](#vpc--networking-questions)
- [Security Questions](#security-questions)
- [Load Balancing & Auto Scaling Questions](#load-balancing--auto-scaling-questions)
- [Scenario-Based Questions](#scenario-based-questions)

---

## Flask Application Questions

### Q1: Why is the Flask application file named `application.py` instead of `app.py`?
**Answer:** AWS Elastic Beanstalk by default looks for an application callable in a file named `application.py`. While you can configure it differently using a Procfile, using `application.py` follows the convention expected by Elastic Beanstalk's Python platform, reducing configuration overhead.

---

### Q2: Explain the purpose of `os.environ.get()` for database credentials in this project.
**Answer:** Using `os.environ.get()` retrieves database credentials from environment variables instead of hardcoding them. This approach:
- **Enhances security:** Credentials aren't stored in source code
- **Enables portability:** Different environments (dev, staging, prod) can have different configurations
- **Follows 12-factor app principles:** Configuration is stored in the environment
- **Works seamlessly with Elastic Beanstalk:** EB allows setting environment variables through the console

---

### Q3: What is the purpose of `cursorclass=pymysql.cursors.DictCursor` in the database connection?
**Answer:** `DictCursor` returns query results as dictionaries instead of tuples. This means:
```python
# With DictCursor
{'id': 1, 'name': 'John', 'policy_id': 'P001'}

# Without DictCursor (default)
(1, 'John', 'P001')
```
This makes code more readable and maintainable as you access values by column name (`row['name']`) rather than index (`row[0]`).

---

### Q4: Why does the application use `app.run(debug=True)` and is it safe for production?
**Answer:** `debug=True` is **NOT safe for production** because:
- It enables an interactive debugger that can execute arbitrary Python code
- It auto-reloads on code changes, which is inefficient
- It exposes detailed error information

In production with Elastic Beanstalk, the app is served by Gunicorn (WSGI server), and this line is never executed because `__name__ != '__main__'` when imported by Gunicorn.

---

### Q5: How does the application handle database connection failures?
**Answer:** The application implements basic error handling:
1. `get_db_connection()` wraps the connection in a try-except block
2. Returns `None` if connection fails, with error logged to console
3. The `/claim` route checks `if connection:` before proceeding
4. Returns appropriate error messages to users

**Improvement suggestion:** Implement connection pooling, retry logic, and proper logging instead of print statements.

---

## AWS Elastic Beanstalk Questions

### Q6: What is AWS Elastic Beanstalk and why use it for this project?
**Answer:** AWS Elastic Beanstalk is a PaaS (Platform as a Service) that:
- Automatically handles deployment, capacity provisioning, load balancing, and auto-scaling
- Provides managed EC2 instances with pre-configured platforms (Python, Java, etc.)
- Maintains full control over underlying resources if needed
- Simplifies application deployment while handling infrastructure

For this insurance claims project, it reduces operational overhead while maintaining scalability.

---

### Q7: What is the difference between Elastic Beanstalk environments: Web Server and Worker?
**Answer:**
| Feature | Web Server | Worker |
|---------|-----------|--------|
| Purpose | Handles HTTP requests | Processes background tasks |
| Connection | Load Balancer + Internet | SQS queue |
| Use Case | Flask API, web pages | Batch processing, async jobs |

This project uses a **Web Server** environment since it serves HTTP requests for claims submission.

---

### Q8: How does Elastic Beanstalk deploy application updates?
**Answer:** Elastic Beanstalk supports multiple deployment policies:
1. **All at once:** Deploy to all instances simultaneously (downtime risk)
2. **Rolling:** Deploy in batches, keeping some instances running
3. **Rolling with additional batch:** Launch new instances first, then deploy
4. **Immutable:** Launch new instances with new version, then swap
5. **Blue/Green:** Create entirely new environment and swap CNAMEs

For production, **Immutable** or **Blue/Green** are recommended for zero-downtime deployments.

---

### Q9: What files does Elastic Beanstalk require/expect in a Python application?
**Answer:**
- `application.py` (or configured entry point) - Main application file
- `requirements.txt` - Python dependencies
- `.ebextensions/` folder - Configuration files (optional)
- `Procfile` - Custom start commands (optional)
- `.ebignore` - Files to exclude from deployment (optional)

---

### Q10: How do you set environment variables in Elastic Beanstalk?
**Answer:** Multiple methods:
1. **AWS Console:** Configuration → Software → Environment properties
2. **EB CLI:** `eb setenv DB_HOST=myhost DB_USER=admin`
3. **Configuration files:** `.ebextensions/env.config`
4. **AWS CLI:** `aws elasticbeanstalk update-environment --option-settings`

Best practice is using AWS Secrets Manager for sensitive values.

---

## AWS RDS & Database Questions

### Q11: Why place RDS in a private subnet instead of a public subnet?
**Answer:** Placing RDS in a private subnet:
- **Prevents direct internet access:** Database isn't exposed to the internet
- **Reduces attack surface:** Only EC2 instances in the VPC can connect
- **Follows security best practices:** Defense in depth
- **Compliance requirements:** Many regulations require databases to be isolated

EC2 instances in public subnets connect to RDS via internal VPC networking (port 3306).

---

### Q12: What is RDS Multi-AZ deployment and when would you use it?
**Answer:** Multi-AZ deployment:
- Creates a synchronous standby replica in a different Availability Zone
- Provides automatic failover if the primary instance fails
- Enhances durability with redundant storage
- **Use cases:** Production databases requiring high availability

This project's architecture shows Multi-AZ with Primary in AZ-1 and Standby in AZ-2.

---

### Q13: Explain the difference between RDS Read Replicas and Multi-AZ.
**Answer:**
| Feature | Multi-AZ | Read Replicas |
|---------|----------|---------------|
| Purpose | High availability | Read scalability |
| Replication | Synchronous | Asynchronous |
| Failover | Automatic | Manual promotion |
| Read Traffic | No | Yes |
| Cross-Region | No | Yes |

For this project, Multi-AZ ensures the claims database is always available.

---

### Q14: How do you secure the connection between EC2 and RDS?
**Answer:**
1. **Security Groups:** RDS SG allows port 3306 only from EC2 security group
2. **Private Subnets:** RDS not accessible from internet
3. **IAM Authentication:** Optional database authentication via IAM
4. **SSL/TLS:** Encrypt data in transit
5. **Network ACLs:** Additional layer of subnet-level security

---

### Q15: What is an RDS Subnet Group and why is it required?
**Answer:** An RDS Subnet Group is a collection of subnets where RDS can place database instances. It's required because:
- Defines which subnets RDS can use for the primary and standby instances
- Must span at least 2 Availability Zones for Multi-AZ deployments
- Ensures RDS has network resources for failover
- In this project: Private Subnet A (10.0.3.0/24) and Private Subnet B (10.0.4.0/24)

---

## VPC & Networking Questions

### Q16: Explain the VPC architecture used in this project.
**Answer:** The VPC (10.0.0.0/16) contains:
- **Public Subnets (10.0.1.0/24, 10.0.2.0/24):** EC2 instances with Flask app, attached to Internet Gateway
- **Private Subnets (10.0.3.0/24, 10.0.4.0/24):** RDS MySQL instances, no internet access
- **Internet Gateway:** Enables internet access for public subnets
- **Route Tables:** Direct traffic appropriately
- **Security Groups:** Control inbound/outbound traffic

---

### Q17: Why are there subnets in multiple Availability Zones?
**Answer:**
- **High Availability:** If one AZ fails, application continues from another
- **Load Balancing:** ALB distributes traffic across AZs
- **RDS Multi-AZ:** Standby database in different AZ for failover
- **Compliance:** Many compliance frameworks require multi-AZ deployments
- **Reduced Latency:** Users can be served from nearest AZ

---

### Q18: What is the difference between Security Groups and Network ACLs?
**Answer:**
| Feature | Security Groups | NACLs |
|---------|----------------|-------|
| Level | Instance | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow and Deny |
| Evaluation | All rules | Rules in order |
| Default | Deny all inbound | Allow all |

This project uses Security Groups (SG-ALB, SG-EC2, SG-RDS) for fine-grained control.

---

### Q19: Why doesn't this architecture use a NAT Gateway?
**Answer:** This architecture is **cost-optimized** by avoiding NAT Gateway because:
- RDS doesn't need outbound internet access
- EC2 instances are in public subnets with direct internet access
- NAT Gateway costs ~$32/month + data processing fees
- **Trade-off:** If EC2 were in private subnets, NAT Gateway would be needed

---

### Q20: How does traffic flow from a user to the database in this architecture?
**Answer:**
1. User sends HTTPS request to ALB endpoint
2. ALB terminates SSL and forwards to healthy EC2 instance (port 80)
3. Flask application processes request
4. Application connects to RDS via private IP (port 3306)
5. RDS returns query results
6. Flask renders response
7. Response travels back through ALB to user

---

## Security Questions

### Q21: Explain the Security Group configuration for this project.
**Answer:**
1. **SG-ALB:** Allows inbound HTTP (80) and HTTPS (443) from anywhere (0.0.0.0/0)
2. **SG-EC2:** Allows inbound port 80 **only from SG-ALB** (no direct internet access to EC2)
3. **SG-RDS:** Allows inbound port 3306 **only from SG-EC2** (only app servers can access DB)

This creates a chain: Internet → ALB → EC2 → RDS

---

### Q22: How would you prevent SQL injection in this application?
**Answer:** The application already uses parameterized queries:
```python
sql = "INSERT INTO claims (policy_id, name, dob, mobile) VALUES (%s, %s, %s, %s)"
cursor.execute(sql, (policy_id, name, dob, mobile))
```
This prevents SQL injection because values are passed separately, not concatenated into the query string. Additional measures:
- Input validation
- Use ORM like SQLAlchemy
- Web Application Firewall (WAF)

---

### Q23: What AWS services would you use to store database credentials securely?
**Answer:**
1. **AWS Secrets Manager:** Automatic rotation, native RDS integration
2. **AWS Systems Manager Parameter Store:** For less frequently rotated secrets
3. **Environment Variables:** Basic approach, used in this project
4. **IAM Database Authentication:** Token-based authentication

Best practice: Use Secrets Manager with automatic rotation for RDS credentials.

---

### Q24: How would you implement HTTPS for this application?
**Answer:**
1. **Obtain SSL Certificate:** AWS Certificate Manager (free for AWS resources)
2. **Configure ALB:** Add HTTPS listener (443) with the certificate
3. **Redirect HTTP to HTTPS:** Add redirect rule on ALB
4. **Security Group:** Allow port 443 on SG-ALB
5. **Optional:** Enforce HTTPS in Flask using `flask-tls` or middleware

---

### Q25: What security headers should the Flask application implement?
**Answer:**
```python
@app.after_request
def add_security_headers(response):
    response.headers['X-Content-Type-Options'] = 'nosniff'
    response.headers['X-Frame-Options'] = 'DENY'
    response.headers['X-XSS-Protection'] = '1; mode=block'
    response.headers['Strict-Transport-Security'] = 'max-age=31536000'
    response.headers['Content-Security-Policy'] = "default-src 'self'"
    return response
```
Or use the `Flask-Talisman` extension.

---

## Load Balancing & Auto Scaling Questions

### Q26: What type of load balancer is used and why?
**Answer:** **Application Load Balancer (ALB)** because:
- Operates at Layer 7 (HTTP/HTTPS)
- Supports path-based and host-based routing
- Native integration with Elastic Beanstalk
- Health checks at application level
- WebSocket support

Alternatives: Network Load Balancer (Layer 4, high performance) or Classic Load Balancer (legacy).

---

### Q27: How does ALB health checking work in this project?
**Answer:** ALB performs periodic health checks:
1. Sends HTTP GET request to specified path (default: `/`)
2. Expects HTTP 200 response within timeout
3. Marks instance unhealthy after consecutive failures
4. Stops sending traffic to unhealthy instances
5. Elastic Beanstalk can replace unhealthy instances

Configuration: Path, interval, timeout, healthy/unhealthy thresholds.

---

### Q28: Explain Auto Scaling in the context of this project.
**Answer:** Elastic Beanstalk creates an Auto Scaling Group that:
- **Maintains desired capacity:** Keeps minimum number of instances running
- **Scales out:** Adds instances when CPU/memory exceeds threshold
- **Scales in:** Removes instances when load decreases
- **Replaces unhealthy instances:** Terminates and launches new ones

Example policy: Scale out when CPU > 70% for 5 minutes.

---

### Q29: What metrics would you use to trigger Auto Scaling?
**Answer:**
- **CPU Utilization:** Most common, scale at 70-80%
- **Memory Utilization:** For memory-intensive apps
- **Request Count:** Scale based on ALB RequestCount
- **Latency:** Scale when response time increases
- **Custom Metrics:** Application-specific metrics via CloudWatch

---

### Q30: How does session management work with multiple EC2 instances?
**Answer:** With multiple instances, sessions must be externalized:
1. **Sticky Sessions:** ALB routes user to same instance (not ideal)
2. **ElastiCache (Redis/Memcached):** Centralized session store
3. **DynamoDB:** Serverless session storage
4. **Database Sessions:** Store in RDS (adds latency)

This project appears stateless (form submission), so session management isn't critical.

---

## Scenario-Based Questions

### Q31: SCENARIO: Users report the application is slow. How do you troubleshoot?
**Answer:**
1. **Check CloudWatch Metrics:**
   - EC2: CPU, memory utilization
   - RDS: CPU, connections, read/write latency
   - ALB: Response time, request count
2. **Review Application Logs:** Elastic Beanstalk logs or CloudWatch Logs
3. **Database Performance:**
   - Check slow query log
   - Analyze query execution plans
   - Review connection pool usage
4. **Network:** Check for packet loss, DNS resolution time
5. **Solutions:**
   - Scale up instance types
   - Add read replicas
   - Implement caching
   - Optimize queries

---

### Q32: SCENARIO: The RDS primary instance fails. What happens?
**Answer:** With Multi-AZ deployment:
1. **AWS detects failure** (hardware, network, AZ outage)
2. **Automatic failover initiates** (60-120 seconds)
3. **DNS endpoint updated** to point to standby
4. **Standby promoted** to primary
5. **Application reconnects** using same endpoint
6. **New standby provisioned** in the original AZ

Application impact: Brief interruption during failover; connection retry logic helps.

---

### Q33: SCENARIO: You need to update the database schema without downtime. How?
**Answer:**
1. **Backward-compatible changes:**
   - Add new columns as nullable
   - Don't rename/remove columns immediately
2. **Blue/Green database approach:**
   - Create RDS snapshot
   - Restore to new instance
   - Apply schema changes
   - Update application to new endpoint
3. **Online schema change tools:**
   - `pt-online-schema-change` for MySQL
   - `gh-ost` from GitHub
4. **Application code:**
   - Deploy new code handling both schemas
   - Migrate data
   - Remove old column handling

---

### Q34: SCENARIO: The application suddenly starts returning 500 errors. Steps to diagnose?
**Answer:**
1. **Immediate checks:**
   - ALB health check status
   - EC2 instance status
   - RDS connectivity
2. **Review logs:**
   - `/var/log/eb-current-app/` on EC2
   - CloudWatch Logs
   - RDS error logs
3. **Common causes:**
   - Database connection exhaustion
   - Environment variable misconfiguration
   - Disk space full
   - Memory exhaustion
4. **Quick fixes:**
   - Restart application servers
   - Scale up instances
   - Check recent deployments (rollback if needed)

---

### Q35: SCENARIO: Database credentials are accidentally exposed. What do you do?
**Answer:**
1. **Immediate actions:**
   - Rotate RDS master password immediately
   - Update environment variables in Elastic Beanstalk
   - Revoke any compromised IAM credentials
2. **Investigation:**
   - Check CloudTrail for unauthorized access
   - Review RDS connection logs
   - Check for data exfiltration
3. **Prevention:**
   - Implement AWS Secrets Manager with rotation
   - Use IAM database authentication
   - Never commit credentials to Git
   - Enable git-secrets or similar tools
4. **Documentation:**
   - Document incident
   - Update security policies

---

### Q36: SCENARIO: How would you implement a disaster recovery plan?
**Answer:**
1. **Backup Strategy:**
   - RDS automated backups (point-in-time recovery)
   - Manual snapshots before major changes
   - Cross-region snapshot copy
2. **Multi-Region Setup:**
   - RDS Read Replica in another region
   - Elastic Beanstalk environment in DR region
   - Route 53 health checks and failover routing
3. **Recovery Procedures:**
   - Document RTO (Recovery Time Objective)
   - Document RPO (Recovery Point Objective)
   - Regular DR drills
4. **Infrastructure as Code:**
   - CloudFormation/Terraform templates
   - Version-controlled configurations

---

### Q37: SCENARIO: Traffic has increased 10x. How do you scale?
**Answer:**
1. **Immediate (Horizontal scaling):**
   - Increase Auto Scaling max capacity
   - Scale out manually if needed
2. **Database:**
   - Add RDS Read Replicas for read traffic
   - Consider RDS Proxy for connection pooling
   - Scale up instance class if needed
3. **Caching:**
   - Implement ElastiCache for frequently accessed data
   - Cache static content at CloudFront
4. **Architecture improvements:**
   - Implement API rate limiting
   - Queue non-urgent operations (SQS)
   - Consider Aurora Serverless for auto-scaling database

---

### Q38: SCENARIO: How would you deploy this application to multiple AWS regions?
**Answer:**
1. **Infrastructure per region:**
   - Identical VPC architecture
   - Elastic Beanstalk environment
   - RDS instance (or cross-region read replica)
2. **Global traffic management:**
   - Route 53 with latency-based or geolocation routing
   - Global Accelerator for improved performance
3. **Data synchronization:**
   - RDS cross-region read replicas
   - DynamoDB Global Tables if switching to NoSQL
4. **CI/CD:**
   - Pipeline deploys to all regions
   - Gradual rollout (one region first)

---

### Q39: SCENARIO: A deployment caused issues. How do you rollback?
**Answer:**
**Elastic Beanstalk rollback options:**
1. **Quick rollback:**
   - Console: Actions → Restore Environment
   - EB CLI: `eb deploy --version PREVIOUS_VERSION_LABEL`
2. **Blue/Green swap:**
   - If using Blue/Green, swap back to previous environment
3. **Database rollback:**
   - If schema changed: Restore from point-in-time backup
   - Note: May cause data loss for recent transactions
4. **Prevention:**
   - Always deploy to staging first
   - Use immutable deployments
   - Implement feature flags

---

### Q40: SCENARIO: How would you monitor application performance proactively?
**Answer:**
1. **CloudWatch Alarms:**
   - CPU utilization > 80%
   - Healthy host count < 2
   - HTTP 5xx errors > threshold
   - Database connections > 80% of max
2. **Application Monitoring:**
   - AWS X-Ray for distributed tracing
   - Custom CloudWatch metrics
   - APM tools (Datadog, New Relic)
3. **Logging:**
   - Centralized logging with CloudWatch Logs
   - Log insights for query/analysis
4. **Dashboards:**
   - CloudWatch dashboard with key metrics
   - Real-time visibility

---

### Q41: SCENARIO: The nginx reverse proxy returns 502 Bad Gateway. Cause and fix?
**Answer:**
**Possible causes:**
1. Flask/Gunicorn application crashed
2. Gunicorn socket file missing or permission issues
3. Application taking too long to respond2
4. Out of memory

**Troubleshooting:**
```bash
# Check Gunicorn status
sudo systemctl status gunicorn

# Check application logs
cat /var/log/web.stdout.log

# Check socket file
ls -la /var/run/gunicorn.sock

# Check memory
free -m
```

**Fixes:**
- Restart Gunicorn service
- Increase Gunicorn timeout
- Scale up instance (more memory)
- Fix application errors

---

### Q42: SCENARIO: Claims are being submitted but not appearing in the database.
**Answer:**
**Investigation steps:**
1. **Check application response:** Is success message shown?
2. **Check database directly:**
   ```sql
   SELECT * FROM claims ORDER BY created_at DESC LIMIT 10;
   ```
3. **Review application logs:** Connection errors? Query errors?
4. **Verify environment variables:** Are DB credentials correct?
5. **Check RDS connectivity:** From EC2, can you connect?
   ```bash
   mysql -h $DB_HOST -u $DB_USER -p$DB_PASSWORD $DB_NAME
   ```
6. **Common issues:**
   - `connection.commit()` missing (but it's present in code)
   - Security group not allowing connection
   - Wrong database name

---

### Q43: SCENARIO: How would you implement CI/CD for this project?
**Answer:**
```yaml
# GitHub Actions example
name: Deploy to Elastic Beanstalk
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.x'
      
      - name: Install dependencies
        run: pip install -r requirements.txt
      
      - name: Run tests
        run: pytest
      
      - name: Deploy to EB
        uses: einaregilsson/beanstalk-deploy@v21
        with:
          aws_access_key: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws_secret_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          region: us-east-1
          application_name: InsuranceClaimsApp
          environment_name: production
          version_label: ${{ github.sha }}
```

---

### Q44: SCENARIO: How would you implement rate limiting to prevent abuse?
**Answer:**
1. **Application level (Flask):**
   ```python
   from flask_limiter import Limiter
   
   limiter = Limiter(app, key_func=get_remote_address)
   
   @app.route('/claim', methods=['POST'])
   @limiter.limit("5 per minute")
   def claim():
       # ...
   ```

2. **AWS WAF:**
   - Create rate-based rule
   - Block IPs exceeding threshold
   - Attach to ALB

3. **API Gateway:**
   - Built-in throttling
   - Usage plans and API keys

---

### Q45: SCENARIO: The database is running out of storage. What do you do?
**Answer:**
1. **Immediate actions:**
   - Enable RDS Storage Auto Scaling (recommended)
   - Manually increase allocated storage
2. **Investigation:**
   - Identify large tables: `SHOW TABLE STATUS;`
   - Check binary logs: May need purging
   - Review slow-growing storage issues
3. **Long-term solutions:**
   - Implement data archival strategy
   - Partition large tables
   - Move to Aurora (auto-scaling storage)
   - Implement data lifecycle policies

---

### Q46: SCENARIO: How would you add a caching layer to reduce database load?
**Answer:**
1. **Set up ElastiCache (Redis):**
   - Create Redis cluster in private subnets
   - Update Security Groups to allow EC2 → Redis

2. **Implement in Flask:**
   ```python
   import redis
   import json
   
   cache = redis.Redis(host='redis-endpoint', port=6379)
   
   @app.route('/')
   def index():
       cached = cache.get('products')
       if cached:
           products = json.loads(cached)
       else:
           products = get_products_from_db()
           cache.setex('products', 300, json.dumps(products))
       return render_template('index.html', products=products)
   ```

3. **Cache strategies:**
   - Cache-aside (application manages cache)
   - Write-through (cache updated on writes)
   - TTL-based expiration

---

### Q47: SCENARIO: How would you implement logging and tracing for debugging?
**Answer:**
1. **Structured Logging:**
   ```python
   import logging
   import json
   
   class JSONFormatter(logging.Formatter):
       def format(self, record):
           return json.dumps({
               'timestamp': self.formatTime(record),
               'level': record.levelname,
               'message': record.getMessage(),
               'function': record.funcName
           })
   
   handler = logging.StreamHandler()
   handler.setFormatter(JSONFormatter())
   app.logger.addHandler(handler)
   ```

2. **AWS X-Ray:**
   ```python
   from aws_xray_sdk.core import xray_recorder
   from aws_xray_sdk.ext.flask.middleware import XRayMiddleware
   
   xray_recorder.configure(service='InsuranceClaims')
   XRayMiddleware(app, xray_recorder)
   ```

3. **CloudWatch Logs Insights:** Query and analyze logs

---

### Q48: SCENARIO: How do you ensure the application meets compliance requirements (e.g., HIPAA)?
**Answer:**
1. **Data encryption:**
   - Enable RDS encryption at rest
   - Enforce SSL/TLS for connections
   - HTTPS for all traffic
2. **Access control:**
   - IAM roles with least privilege
   - MFA for console access
   - Database audit logging
3. **Network security:**
   - VPC isolation
   - Security Groups restricting access
   - VPC Flow Logs
4. **Monitoring:**
   - CloudTrail for API auditing
   - GuardDuty for threat detection
   - Config for compliance checking
5. **Data handling:**
   - PII encryption
   - Data retention policies
   - Regular security assessments

---

### Q49: SCENARIO: A new developer needs to set up a local development environment. What's the process?
**Answer:**
1. **Prerequisites:**
   ```bash
   # Install Python 3.x
   python --version
   
   # Install MySQL locally or use Docker
   docker run -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=password mysql:8
   ```

2. **Project setup:**
   ```bash
   git clone <repository>
   cd AWS-Flask---RDS-Deployment
   
   # Create virtual environment
   python -m venv venv
   source venv/bin/activate  # or venv\Scripts\activate on Windows
   
   # Install dependencies
   pip install -r requirements.txt
   ```

3. **Configure environment:**
   ```bash
   export DB_HOST=localhost
   export DB_USER=root
   export DB_PASSWORD=password
   export DB_NAME=insured
   ```

4. **Initialize database:**
   ```bash
   mysql -u root -p < insured.sql
   ```

5. **Run application:**
   ```bash
   python application.py
   ```

---

### Q50: SCENARIO: How would you optimize costs for this AWS architecture?
**Answer:**
1. **EC2/Elastic Beanstalk:**
   - Use Reserved Instances for predictable workloads (up to 72% savings)
   - Implement proper scaling policies (scale down during low traffic)
   - Right-size instances based on actual usage
   - Use Spot Instances for non-critical environments

2. **RDS:**
   - Reserved Instances for production
   - Use Aurora Serverless for variable workloads
   - Schedule dev/test instances to stop during off-hours
   - Optimize storage type (gp3 vs io1)

3. **Network:**
   - Avoid NAT Gateway if possible (as in this architecture)
   - Use VPC endpoints for AWS services
   - Minimize cross-AZ data transfer

4. **Monitoring:**
   - Set up AWS Budgets with alerts
   - Use Cost Explorer to identify waste
   - Regularly review unused resources

5. **This project's approach:**
   - No NAT Gateway (cost-optimized)
   - Right-sized instances
   - Multi-AZ only for production

---

## Additional Technical Questions

### Bonus Q1: Why is Gunicorn used instead of Flask's built-in server?
**Answer:** Flask's built-in server:
- Single-threaded, can't handle concurrent requests
- Not secure or stable for production
- Development purpose only

Gunicorn provides:
- Multiple worker processes
- Production-grade stability
- Better resource utilization
- WSGI compliance

---

### Bonus Q2: What's the purpose of the nginx.conf in this project?
**Answer:** The nginx.conf configures Nginx as a reverse proxy:
1. **Upstream:** Routes requests to Gunicorn via Unix socket
2. **Proxy headers:** Passes client IP and original protocol
3. **Static files:** Serves static content directly without hitting Flask
4. **Benefits:** Better performance, SSL termination, load distribution

---

### Bonus Q3: How would you implement connection pooling for the database?
**Answer:**
```python
# Using SQLAlchemy with connection pooling
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    f'mysql+pymysql://{db_user}:{db_password}@{db_host}/{db_name}',
    poolclass=QueuePool,
    pool_size=5,
    max_overflow=10,
    pool_timeout=30,
    pool_recycle=1800
)

# Or use AWS RDS Proxy for managed connection pooling
```

---

### Bonus Q4: What would you add for a production-ready version of this application?
**Answer:**
1. **Security:** HTTPS, WAF, Secrets Manager, input validation
2. **Monitoring:** CloudWatch alarms, X-Ray, centralized logging
3. **Performance:** Caching, connection pooling, CDN for static assets
4. **Reliability:** Health checks, retries, circuit breakers
5. **CI/CD:** Automated testing, staged deployments
6. **Documentation:** API docs, runbooks, architecture diagrams

---

*End of Interview Questions Document*
