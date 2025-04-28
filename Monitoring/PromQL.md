Of course! Here's a full **Markdown** (`.md`) formatted guide about **PromQL Queries in Prometheus**:

---

# 📈 PromQL (Prometheus Query Language) - Complete Guide

PromQL (Prometheus Query Language) is a powerful, flexible query language used to retrieve, filter, and process time series data stored in Prometheus.

---

## 📚 Table of Contents

- [Introduction](#introduction)
- [Core Concepts](#core-concepts)
- [Data Types in PromQL](#data-types-in-promql)
- [Selectors](#selectors)
- [Operators](#operators)
- [Functions](#functions)
- [Aggregation Operators](#aggregation-operators)
- [Subqueries](#subqueries)
- [Recording Rules](#recording-rules)
- [Best Practices](#best-practices)
- [Useful Examples](#useful-examples)
- [References](#references)

---

## 📝 Introduction

PromQL is designed for **powerful yet simple** extraction and analysis of time series data. It allows you to:

- Fetch raw or aggregated data.
- Apply mathematical calculations.
- Perform filtering and grouping.
- Visualize trends over time.

---

## 🧠 Core Concepts

Prometheus data is a collection of **time series** identified by:

- A **metric name** (e.g., `http_requests_total`)
- **Labels** (key-value pairs like `{method="GET", handler="/api"}`)

Each time series is uniquely identified by its combination of metric name and label set.

---

## 📊 Data Types in PromQL

| Type           | Description |
|----------------|-------------|
| **Instant Vector** | Set of time series containing a single sample for each series at a given timestamp. |
| **Range Vector**   | Set of time series containing a range of data points over time for each series. |
| **Scalar**         | Single numeric floating-point value (e.g., `5.2`). |
| **String**         | Single string value (very rarely used in queries). |

---

## 🎯 Selectors

### 1. **Instant Vector Selector**

```promql
http_requests_total
```

Fetches the latest value of all series with the name `http_requests_total`.

### 2. **Label Matchers**

You can filter metrics by labels:

```promql
http_requests_total{method="GET", handler="/api"}
```

Supported operators:

- `=` (equal)
- `!=` (not equal)
- `=~` (regex match)
- `!~` (regex not match)

### 3. **Range Vector Selector**

Fetches values over a time window:

```promql
http_requests_total{method="POST"}[5m]
```
Fetches 5 minutes of data.

---

## ➗ Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `+` | Addition | `metric_a + metric_b` |
| `-` | Subtraction | `metric_a - metric_b` |
| `*` | Multiplication | `metric_a * 2` |
| `/` | Division | `metric_a / 2` |
| `%` | Modulo | `metric_a % 2` |
| `==` | Equals | `metric_a == 100` |
| `!=` | Not Equals | `metric_a != 200` |
| `>` | Greater Than | `metric_a > 10` |
| `<` | Less Than | `metric_a < 10` |

> **Note:** Prometheus automatically aligns time series based on their labels for binary operations.

---

## 🛠️ Functions

PromQL includes many built-in functions:

| Function | Purpose | Example |
|----------|---------|---------|
| `rate(v range-vector)` | Calculate per-second average rate | `rate(http_requests_total[5m])` |
| `irate(v range-vector)` | Instantaneous per-second rate | `irate(http_requests_total[30s])` |
| `increase(v range-vector)` | Total increase over time | `increase(http_requests_total[1h])` |
| `avg_over_time(v range-vector)` | Average over time window | `avg_over_time(http_requests_total[5m])` |
| `sum_over_time(v range-vector)` | Sum over time window | `sum_over_time(http_requests_total[5m])` |
| `max_over_time(v range-vector)` | Max value over time | `max_over_time(http_requests_total[5m])` |
| `min_over_time(v range-vector)` | Min value over time | `min_over_time(http_requests_total[5m])` |
| `abs(v instant-vector)` | Absolute value | `abs(metric)` |

**Example:**

```promql
rate(node_cpu_seconds_total{mode="idle"}[5m])
```
This shows the CPU idle rate over the past 5 minutes.

---

## 🧮 Aggregation Operators

Aggregation groups metrics by labels and performs operations:

| Operator | Description |
|----------|-------------|
| `sum()`  | Sum across labels |
| `avg()`  | Average across labels |
| `max()`  | Maximum value |
| `min()`  | Minimum value |
| `count()`| Count of series |
| `stddev()`| Standard deviation |
| `stdvar()`| Standard variance |
| `topk(k, vector)`| Top k elements by value |
| `bottomk(k, vector)`| Bottom k elements by value |

**Example:**

```promql
sum(rate(http_requests_total[5m])) by (method)
```
Sums the request rates, grouped by `method`.

---

## 🌀 Subqueries

Subqueries allow you to select a range vector over a computed period inside a larger query:

```promql
avg_over_time(rate(http_requests_total[5m])[30m:])
```

- `rate(http_requests_total[5m])`: Computes 5m rates.
- `[30m:]`: Takes these rates over the past 30 minutes.
- `avg_over_time(...)`: Averages them.

---

## 📝 Recording Rules

To **precompute** expensive queries and **save** them as a new time series:

Example in `rules.yml`:

```yaml
groups:
- name: example
  rules:
  - record: job:http_inprogress_requests:sum
    expr: sum(http_inprogress_requests) by (job)
```

This saves the computed value as `job:http_inprogress_requests:sum`.

---

## 💡 Best Practices

- **Use specific labels** to avoid fetching unnecessary data.
- **Precompute** frequently used heavy queries using **recording rules**.
- **Alert** on **rates** or **aggregations** rather than raw counts.
- Use **`rate()`** for counters (e.g., `http_requests_total`), not `increase()` for short durations.
- **Avoid** unbounded label matches like `=~".*"`, which can be very slow.
- **Group by appropriate labels** to make aggregations meaningful.

---

## 📚 Useful Examples

| Task | Query |
|------|-------|
| CPU usage % | `100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` |
| Memory used | `node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes` |
| HTTP 5xx errors per second | `rate(http_requests_total{status=~"5.."}[5m])` |
| Pod restarts in Kubernetes | `increase(kube_pod_container_status_restarts_total[5m])` |
| Disk usage % | `(node_filesystem_size_bytes - node_filesystem_free_bytes) / node_filesystem_size_bytes * 100` |

---

## 📖 References

- [Official PromQL Documentation](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Prometheus Functions](https://prometheus.io/docs/prometheus/latest/querying/functions/)
- [Prometheus Aggregation Operators](https://prometheus.io/docs/prometheus/latest/querying/operators/#aggregation-operators)

---

# 🚀 Bonus: Useful Cheatsheet

```promql
# Basic metric
up

# Select metric with labels
up{job="api-server"}

# Rate over 5 minutes
rate(http_requests_total[5m])

# Sum by label
sum(rate(http_requests_total[5m])) by (job)

# Error rate
sum(rate(http_requests_total{status=~"5.."}[5m])) by (job) / sum(rate(http_requests_total[5m])) by (job)

# Top 5 CPU using pods
topk(5, rate(container_cpu_usage_seconds_total[5m]))
```

---

Would you also like a **visual diagram** of PromQL components or a **cheat sheet image**? 🎯  
I can create that if you want!
