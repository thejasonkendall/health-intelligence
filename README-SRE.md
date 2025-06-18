# Health Intelligence App - Observability & SLO Strategy

## 🎯 Service Level Objectives (SLOs) & Service Level Indicators (SLIs)

### **API Service SLOs**
| SLO | Target | SLI Measurement | Error Budget |
|-----|--------|-----------------|--------------|
| **Availability** | 99.5% uptime | HTTP 200-299 responses / total requests | 0.5% (3.6h/month) |
| **Latency** | 95% of requests < 500ms | P95 response time for `/intelligence` endpoints | 5% can exceed 500ms |
| **Data Freshness** | 90% of health intelligence < 24h old | Articles with `created_at` within 24h / total articles | 10% can be stale |
| **AI Analysis Success** | 95% successful analysis | Successful Claude API calls / total attempts | 5% can fail |

### **Dashboard Service SLOs**
| SLO | Target | SLI Measurement | Error Budget |
|-----|--------|-----------------|--------------|
| **Page Load Time** | 95% loads < 2s | Browser timing API measurements | 5% can exceed 2s |
| **Dashboard Availability** | 99.9% uptime | HTTP 200 responses from dashboard | 0.1% (43m/month) |

### **Data Pipeline SLOs**
| SLO | Target | SLI Measurement | Error Budget |
|-----|--------|-----------------|--------------|
| **Collection Success** | 90% successful runs | Successful pipeline executions / total runs | 10% can fail |
| **News Processing** | 80% of articles analyzed | Articles with AI analysis / total fetched | 20% can be unprocessed |

## 📊 Monitoring Strategy

### **Golden Signals to Track**

#### 1. **Latency**
- API response times (P50, P95, P99)
- Database query execution times
- Claude API response times
- Dashboard load times

#### 2. **Traffic**
- Requests per second to API endpoints
- Database connections
- Dashboard page views
- News articles processed per hour

#### 3. **Errors**
- HTTP 4xx/5xx error rates
- Database connection failures
- Claude API errors
- Failed news collection attempts

#### 4. **Saturation**
- CPU utilization across EC2 instances
- Database connection pool usage
- Memory consumption
- Disk I/O for logs

### **Application-Specific Metrics**

#### **Business Metrics**
```yaml
Health Intelligence Metrics:
  - total_articles_processed_daily
  - high_significance_intelligence_count
  - countries_covered_daily
  - confidence_score_distribution
  - health_topics_diversity_index
  - time_to_intelligence (news publish → analysis complete)

API Usage Metrics:
  - endpoint_usage_distribution
  - filter_usage_patterns
  - search_query_frequency
  - user_session_duration

Data Quality Metrics:
  - duplicate_article_rate
  - analysis_confidence_scores
  - failed_geocoding_rate
  - incomplete_intelligence_records
```

## 🔧 Implementation Plan

### **Phase 1: Basic Observability (Week 1-2)**

#### **CloudWatch Setup**
```terraform
# Add to terraform/main.tf

# CloudWatch Log Groups
resource "aws_cloudwatch_log_group" "api_logs" {
  name              = "/aws/health-intelligence/api"
  retention_in_days = 30
}

resource "aws_cloudwatch_log_group" "dashboard_logs" {
  name              = "/aws/health-intelligence/dashboard"
  retention_in_days = 14
}

resource "aws_cloudwatch_log_group" "data_pipeline_logs" {
  name              = "/aws/health-intelligence/data-pipeline"
  retention_in_days = 7
}

# Custom Metrics
resource "aws_cloudwatch_dashboard" "health_intelligence" {
  dashboard_name = "HealthIntelligence-Overview"
  
  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 12
        height = 6
        
        properties = {
          metrics = [
            ["AWS/ApplicationELB", "RequestCount", "LoadBalancer", aws_lb.main.arn_suffix],
            [".", "ResponseTime", ".", "."],
            [".", "HTTPCode_Target_2XX_Count", ".", "."],
            [".", "HTTPCode_Target_4XX_Count", ".", "."],
            [".", "HTTPCode_Target_5XX_Count", ".", "."]
          ]
          period = 300
          stat   = "Sum"
          region = var.aws_region
          title  = "API Health Overview"
        }
      }
    ]
  })
}
```

#### **Application Instrumentation**
```python
# Add to api/main.py

import time
import boto3
from datetime import datetime
import logging
from contextlib import asynccontextmanager

# CloudWatch client
cloudwatch = boto3.client('cloudwatch')

# Custom metrics middleware
@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    start_time = time.time()
    
    response = await call_next(request)
    
    process_time = time.time() - start_time
    
    # Send custom metrics to CloudWatch
    try:
        cloudwatch.put_metric_data(
            Namespace='HealthIntelligence/API',
            MetricData=[
                {
                    'MetricName': 'ResponseTime',
                    'Value': process_time * 1000,  # Convert to milliseconds
                    'Unit': 'Milliseconds',
                    'Dimensions': [
                        {
                            'Name': 'Endpoint',
                            'Value': request.url.path
                        },
                        {
                            'Name': 'Method',
                            'Value': request.method
                        }
                    ]
                },
                {
                    'MetricName': 'RequestCount',
                    'Value': 1,
                    'Unit': 'Count',
                    'Dimensions': [
                        {
                            'Name': 'StatusCode',
                            'Value': str(response.status_code)
                        },
                        {
                            'Name': 'Endpoint',
                            'Value': request.url.path
                        }
                    ]
                }
            ]
        )
    except Exception as e:
        logger.warning(f"Failed to send metrics: {e}")
    
    return response

# Health Intelligence specific metrics
async def track_intelligence_metrics():
    """Track business metrics for health intelligence"""
    try:
        # Get daily intelligence count
        today_count_query = """
            SELECT COUNT(*) as count 
            FROM health_intelligence 
            WHERE DATE(created_at) = CURRENT_DATE
        """
        today_count = db.execute_query(today_count_query)[0]['count']
        
        # Get high significance count
        high_sig_query = """
            SELECT COUNT(*) as count 
            FROM health_intelligence 
            WHERE significance_level IN ('high', 'critical')
            AND DATE(created_at) = CURRENT_DATE
        """
        high_sig_count = db.execute_query(high_sig_query)[0]['count']
        
        # Send to CloudWatch
        cloudwatch.put_metric_data(
            Namespace='HealthIntelligence/Business',
            MetricData=[
                {
                    'MetricName': 'DailyIntelligenceCount',
                    'Value': today_count,
                    'Unit': 'Count'
                },
                {
                    'MetricName': 'HighSignificanceDaily',
                    'Value': high_sig_count,
                    'Unit': 'Count'
                }
            ]
        )
        
    except Exception as e:
        logger.error(f"Failed to track intelligence metrics: {e}")

# Add scheduled metric collection
import asyncio
from apscheduler.schedulers.asyncio import AsyncIOScheduler

scheduler = AsyncIOScheduler()
scheduler.add_job(
    track_intelligence_metrics,
    'interval',
    minutes=15,
    id='intelligence_metrics'
)
scheduler.start()
```

### **Phase 2: Advanced Monitoring (Week 3-4)**

#### **X-Ray Distributed Tracing**
```python
# Add to requirements.txt
aws-xray-sdk==2.12.1

# Instrument FastAPI with X-Ray
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

# Patch AWS services
patch_all()

# Add X-Ray middleware
from aws_xray_sdk.ext.fastapi import XRayMiddleware

app.add_middleware(XRayMiddleware, recorder=xray_recorder)

# Trace database calls
@xray_recorder.capture('database_query')
def execute_query_traced(query, params=None):
    return db.execute_query(query, params)

# Trace Claude API calls  
@xray_recorder.capture('claude_analysis')
def analyze_with_tracing(self, article):
    subsegment = xray_recorder.current_subsegment()
    subsegment.put_metadata('article_title', article.get('title', 'Unknown'))
    
    result = self.analyze_health_article(article)
    
    if result:
        subsegment.put_metadata('analysis_confidence', result.get('confidence_score'))
        subsegment.put_metadata('health_topic', result.get('health_topic'))
    
    return result
```

#### **Structured Logging**
```python
# Enhanced logging setup
import structlog
import json

# Configure structured logging
structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.PositionalArgumentsFormatter(),
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.UnicodeDecoder(),
        structlog.processors.JSONRenderer()
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    wrapper_class=structlog.stdlib.BoundLogger,
    cache_logger_on_first_use=True,
)

logger = structlog.get_logger()

# Usage in endpoints
@app.get("/intelligence")
async def get_health_intelligence(request: Request):
    logger.info(
        "intelligence_request",
        endpoint="/intelligence",
        user_agent=request.headers.get("user-agent"),
        query_params=dict(request.query_params)
    )
    
    try:
        result = # ... your existing logic
        
        logger.info(
            "intelligence_response",
            endpoint="/intelligence", 
            result_count=len(result.get('intelligence', [])),
            processing_time_ms=process_time
        )
        
        return result
        
    except Exception as e:
        logger.error(
            "intelligence_error",
            endpoint="/intelligence",
            error_type=type(e).__name__,
            error_message=str(e)
        )
        raise
```

### **Phase 3: Alerting & SLO Monitoring (Week 5-6)**

#### **CloudWatch Alarms**
```terraform
# Error Rate Alarms
resource "aws_cloudwatch_metric_alarm" "api_error_rate" {
  alarm_name          = "health-intelligence-api-error-rate"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "RequestCount"
  namespace           = "HealthIntelligence/API"
  period              = "300"
  statistic           = "Sum"
  threshold           = "10"
  alarm_description   = "API error rate is too high"
  
  dimensions = {
    StatusCode = "5XX"
  }
  
  alarm_actions = [aws_sns_topic.alerts.arn]
}

# Response Time Alarm
resource "aws_cloudwatch_metric_alarm" "api_latency" {
  alarm_name          = "health-intelligence-api-latency"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "3"
  metric_name         = "ResponseTime"
  namespace           = "HealthIntelligence/API"
  period              = "300"
  statistic           = "Average"
  threshold           = "500"
  alarm_description   = "API response time is too high"
  
  alarm_actions = [aws_sns_topic.alerts.arn]
}

# Data Freshness Alarm
resource "aws_cloudwatch_metric_alarm" "data_freshness" {
  alarm_name          = "health-intelligence-data-freshness"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = "1"
  metric_name         = "DailyIntelligenceCount"
  namespace           = "HealthIntelligence/Business"
  period              = "3600"
  statistic           = "Maximum"
  threshold           = "5"
  alarm_description   = "Not enough fresh intelligence data"
  
  alarm_actions = [aws_sns_topic.alerts.arn]
}

# SNS Topic for alerts
resource "aws_sns_topic" "alerts" {
  name = "health-intelligence-alerts"
}

resource "aws_sns_topic_subscription" "email_alerts" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = var.alert_email
}
```

#### **SLO Monitoring Dashboard**
```python
# Add SLO calculation endpoint
@app.get("/slo/metrics")
async def get_slo_metrics():
    """Calculate current SLO compliance"""
    
    # Calculate availability SLI (last 30 days)
    availability_query = """
        SELECT 
            COUNT(*) FILTER (WHERE status_code BETWEEN 200 AND 299) as success_count,
            COUNT(*) as total_count
        FROM api_requests 
        WHERE timestamp >= NOW() - INTERVAL '30 days'
    """
    
    # Calculate latency SLI
    latency_query = """
        SELECT 
            COUNT(*) FILTER (WHERE response_time_ms < 500) as fast_requests,
            COUNT(*) as total_requests
        FROM api_requests 
        WHERE timestamp >= NOW() - INTERVAL '30 days'
        AND endpoint = '/intelligence'
    """
    
    # Calculate data freshness SLI
    freshness_query = """
        SELECT 
            COUNT(*) FILTER (WHERE created_at >= NOW() - INTERVAL '24 hours') as fresh_count,
            COUNT(*) as total_count
        FROM health_intelligence
        WHERE created_at >= NOW() - INTERVAL '30 days'
    """
    
    # Execute queries and calculate SLIs
    availability_result = db.execute_query(availability_query)[0]
    latency_result = db.execute_query(latency_query)[0]
    freshness_result = db.execute_query(freshness_query)[0]
    
    availability_sli = availability_result['success_count'] / max(availability_result['total_count'], 1)
    latency_sli = latency_result['fast_requests'] / max(latency_result['total_requests'], 1)
    freshness_sli = freshness_result['fresh_count'] / max(freshness_result['total_count'], 1)
    
    # Calculate error budgets
    availability_error_budget = max(0, 0.995 - availability_sli)
    latency_error_budget = max(0, 0.95 - latency_sli)
    freshness_error_budget = max(0, 0.90 - freshness_sli)
    
    return {
        "slos": {
            "availability": {
                "target": 0.995,
                "current": availability_sli,
                "status": "healthy" if availability_sli >= 0.995 else "at_risk",
                "error_budget_remaining": availability_error_budget
            },
            "latency": {
                "target": 0.95,
                "current": latency_sli,
                "status": "healthy" if latency_sli >= 0.95 else "at_risk",
                "error_budget_