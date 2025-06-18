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

