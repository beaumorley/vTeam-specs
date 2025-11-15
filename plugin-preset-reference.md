# Perses Plugin Preset Reference Guide

Quick reference for all available chart plugin presets in the OpenShift AI Observability Dashboard.

---

## Time Series Charts

### Request Rate
```typescript
import { createRequestRateChart } from '@/plugins';
const chart = createRequestRateChart();
```
**Use Case**: Monitor requests per second for model serving
**Y-Axis**: ops (requests/sec)
**Best For**: API endpoints, model inference requests

### Latency Chart
```typescript
import { createLatencyChart } from '@/plugins';
const chart = createLatencyChart();
```
**Use Case**: Track response times and latency percentiles
**Y-Axis**: seconds (milliseconds)
**Thresholds**: Warning at 1s, Critical at 3s
**Best For**: P95/P99 latency tracking

### Error Rate Chart
```typescript
import { createErrorRateChart } from '@/plugins';
const chart = createErrorRateChart();
```
**Use Case**: Monitor error percentages over time
**Y-Axis**: percentage (%)
**Thresholds**: Warning at 1%, Critical at 5%
**Best For**: Model serving error rates, API failures

### Utilization Chart
```typescript
import { createUtilizationChart } from '@/plugins';
const cpuChart = createUtilizationChart('CPU');
const memChart = createUtilizationChart('Memory');
const gpuChart = createUtilizationChart('GPU');
```
**Use Case**: Track resource consumption
**Y-Axis**: cores / bytes / percentage
**Thresholds**: Warning at 70%, Critical at 90%
**Best For**: Pod resources, node capacity

### Training Metrics Chart
```typescript
import { createTrainingMetricsChart } from '@/plugins';
const chart = createTrainingMetricsChart();
```
**Use Case**: Visualize training loss and accuracy over epochs
**Y-Axis**: Metric value (4 decimals)
**Features**: Shows data points, thin lines
**Best For**: ML training progress tracking

---

## Gauge Charts

### CPU Utilization Gauge
```typescript
import { createCPUUtilizationGauge } from '@/plugins';
const gauge = createCPUUtilizationGauge();
```
**Range**: 0-100%
**Thresholds**: Green (0-70%), Yellow (70-90%), Red (90-100%)
**Best For**: Real-time CPU usage display

### Memory Utilization Gauge
```typescript
import { createMemoryUtilizationGauge } from '@/plugins';
const gauge = createMemoryUtilizationGauge();
```
**Range**: 0-100%
**Thresholds**: Green (0-70%), Yellow (70-90%), Red (90-100%)
**Best For**: Real-time memory usage display

### GPU Utilization Gauge
```typescript
import { createGPUUtilizationGauge } from '@/plugins';
const gauge = createGPUUtilizationGauge();
```
**Range**: 0-100%
**Thresholds**:
- Idle (0-30%): Gray
- Low (30-60%): Green
- Optimal (60-85%): Blue
- High (85-95%): Yellow
- Critical (95-100%): Red
**Best For**: Training job GPU monitoring

### Success Rate Gauge
```typescript
import { createSuccessRateGauge } from '@/plugins';
const gauge = createSuccessRateGauge();
```
**Range**: 0-100%
**Thresholds**: Critical (<95%), Warning (95-99%), Good (99-99.9%), Excellent (>99.9%)
**Best For**: Model serving SLA tracking

### Storage Capacity Gauge
```typescript
import { createStorageCapacityGauge } from '@/plugins';
const gauge = createStorageCapacityGauge();
```
**Range**: 0-100%
**Thresholds**: Green (0-80%), Yellow (80-95%), Red (>95%)
**Best For**: PVC usage, storage quotas

### Accuracy Gauge
```typescript
import { createAccuracyGauge } from '@/plugins';
const gauge = createAccuracyGauge();
```
**Range**: 0-1 (3 decimals)
**Thresholds**: Red (<0.7), Yellow (0.7-0.85), Green (0.85-0.95), Blue (>0.95)
**Best For**: Model accuracy/confidence scores

---

## Stat Panels

### Request Count
```typescript
import { createRequestCountStat } from '@/plugins';
const stat = createRequestCountStat();
```
**Calculation**: Sum
**Format**: Abbreviated numbers (K/M/B)
**Best For**: Total request counters

### Requests Per Second
```typescript
import { createRequestsPerSecondStat } from '@/plugins';
const stat = createRequestsPerSecondStat();
```
**Calculation**: Last value
**Features**: Sparkline trend
**Format**: ops (2 decimals)
**Best For**: Current throughput display

### Error Rate Stat
```typescript
import { createErrorRateStat } from '@/plugins';
const stat = createErrorRateStat();
```
**Calculation**: Last value
**Format**: Percentage (2 decimals)
**Thresholds**: Green (<1%), Yellow (1-5%), Red (>5%)
**Best For**: Current error rate KPI

### Latency Stat
```typescript
import { createLatencyStat } from '@/plugins';
const stat = createLatencyStat();
```
**Calculation**: Last value
**Format**: Milliseconds (1 decimal)
**Thresholds**: Green (<500ms), Yellow (500-1000ms), Red (>1000ms)
**Best For**: Current latency display

### Active Models Stat
```typescript
import { createActiveModelsStat } from '@/plugins';
const stat = createActiveModelsStat();
```
**Calculation**: Last value
**Format**: Integer
**Best For**: Count of running models

### GPU Count Stat
```typescript
import { createGPUCountStat } from '@/plugins';
const stat = createGPUCountStat();
```
**Calculation**: Last value
**Format**: Integer
**Best For**: Available/allocated GPU count

### Memory Usage Stat
```typescript
import { createMemoryUsageStat } from '@/plugins';
const stat = createMemoryUsageStat();
```
**Calculation**: Last value
**Format**: Bytes (abbreviated)
**Thresholds**: Based on GB thresholds
**Best For**: Current memory consumption

### Epochs Completed Stat
```typescript
import { createEpochsCompletedStat } from '@/plugins';
const stat = createEpochsCompletedStat();
```
**Calculation**: Max value
**Format**: Integer
**Best For**: Training progress tracking

### Accuracy Stat
```typescript
import { createAccuracyStat } from '@/plugins';
const stat = createAccuracyStat();
```
**Calculation**: Last value
**Format**: Percentage (2 decimals)
**Best For**: Model accuracy display

### Uptime Stat
```typescript
import { createUptimeStat } from '@/plugins';
const stat = createUptimeStat();
```
**Calculation**: Last value
**Format**: Percentage (3 decimals)
**Thresholds**: Red (<95%), Yellow (95-99%), Green (99-99.9%), Blue (>99.9%)
**Best For**: Service availability SLA

---

## Table Views

### Model Serving Table
```typescript
import { createModelServingTable } from '@/plugins';
const table = createModelServingTable();
```
**Columns**: Model Name, Version, Total Requests, Req/sec, Error Rate, P95 Latency
**Sorting**: By total requests (desc)
**Features**: Color-coded error rates and latencies
**Best For**: Model serving overview

### Training Jobs Table
```typescript
import { createTrainingJobsTable } from '@/plugins';
const table = createTrainingJobsTable();
```
**Columns**: Job Name, Status, Epoch, Loss, Accuracy, GPU Usage, Runtime
**Sorting**: By job name (asc)
**Features**: Status badges, color-coded GPU usage
**Best For**: Training job monitoring

### Notebook Servers Table
```typescript
import { createNotebookServersTable } from '@/plugins';
const table = createNotebookServersTable();
```
**Columns**: Notebook, Status, CPU, Memory, Disk, Uptime
**Features**: Color-coded disk usage warnings
**Best For**: Notebook resource tracking

### Resource Quotas Table
```typescript
import { createResourceQuotasTable } from '@/plugins';
const table = createResourceQuotasTable();
```
**Columns**: Resource, Used, Limit, Utilization, Available
**Sorting**: By utilization (desc)
**Features**: Color-coded utilization backgrounds
**Best For**: Project quota management

### Pod Metrics Table
```typescript
import { createPodMetricsTable } from '@/plugins';
const table = createPodMetricsTable();
```
**Columns**: Pod, Namespace, CPU, Memory, Restarts, Age
**Pagination**: 25 rows per page
**Features**: Color-coded restart counts
**Best For**: Pod-level resource analysis

---

## Bar Charts

### Model Request Volume
```typescript
import { createModelRequestVolumeChart } from '@/plugins';
const chart = createModelRequestVolumeChart();
```
**Mode**: Vertical columns
**Sorting**: By value (desc)
**Features**: Data labels shown
**Best For**: Comparing request volumes across models

### Training Duration
```typescript
import { createTrainingDurationChart } from '@/plugins';
const chart = createTrainingDurationChart();
```
**Mode**: Horizontal bars
**Unit**: Hours
**Color**: Green palette
**Best For**: Comparing training job runtimes

### Resource Utilization by Type
```typescript
import { createResourceUtilizationByTypeChart } from '@/plugins';
const chart = createResourceUtilizationByTypeChart('CPU'); // or 'Memory', 'GPU'
```
**Mode**: Stacked columns
**Features**: Multi-color palette
**Best For**: Resource distribution analysis

### Errors by Status Code
```typescript
import { createErrorsByStatusCodeChart } from '@/plugins';
const chart = createErrorsByStatusCodeChart();
```
**Mode**: Vertical columns
**Color**: Red palette
**Sorting**: By error count (desc)
**Best For**: HTTP error analysis

### Top Resource Consumers
```typescript
import { createTopResourceConsumersChart } from '@/plugins';
const chart = createTopResourceConsumersChart(10); // top N
```
**Mode**: Horizontal bars
**Features**: Threshold lines at 70% and 90%
**Best For**: Identifying resource-hungry workloads

### GPU Allocation
```typescript
import { createGPUAllocationChart } from '@/plugins';
const chart = createGPUAllocationChart();
```
**Mode**: Stacked percentage columns
**Features**: Data labels, multi-color
**Best For**: GPU utilization across jobs

### Latency Distribution
```typescript
import { createLatencyDistributionChart } from '@/plugins';
const chart = createLatencyDistributionChart();
```
**Mode**: Vertical columns
**Unit**: Milliseconds
**Features**: Threshold markers
**Best For**: Response time histograms

### Dataset Size
```typescript
import { createDatasetSizeChart } from '@/plugins';
const chart = createDatasetSizeChart();
```
**Mode**: Vertical columns
**Unit**: Bytes (abbreviated)
**Color**: Green palette
**Best For**: Data storage analysis

### Deployment Status
```typescript
import { createDeploymentStatusChart } from '@/plugins';
const chart = createDeploymentStatusChart();
```
**Mode**: Stacked normal columns
**Colors**: Green (success) + Red (failed)
**Best For**: Deployment success rates over time

### Batch Throughput
```typescript
import { createBatchThroughputChart } from '@/plugins';
const chart = createBatchThroughputChart();
```
**Mode**: Vertical columns
**Unit**: Items processed (abbreviated)
**Sorting**: By value (desc)
**Best For**: Batch job performance comparison

---

## Common PromQL Queries

### Model Serving
```typescript
import { OpenShiftAIQueries } from '@/plugins';

// Request rate
OpenShiftAIQueries.modelServingRequestRate('my-model', '5m')
// => rate(model_server_requests_total{model_name="my-model"}[5m])

// Error rate
OpenShiftAIQueries.modelServingErrorRate('my-model')
// => rate(model_server_requests_total{model_name="my-model",status=~"5.."}[5m])

// P95 latency
OpenShiftAIQueries.modelInferenceLatencyP95('my-model')
// => histogram_quantile(0.95, rate(model_server_request_duration_seconds_bucket{model_name="my-model"}[5m]))
```

### Training Jobs
```typescript
// GPU utilization
OpenShiftAIQueries.gpuUtilization('training-job-1')
// => nvidia_gpu_utilization{job="training-job-1"}

// GPU memory
OpenShiftAIQueries.gpuMemoryUsage('training-job-1')
// => nvidia_gpu_memory_used_bytes{job="training-job-1"} / nvidia_gpu_memory_total_bytes{job="training-job-1"} * 100

// Training loss
OpenShiftAIQueries.trainingLoss('training-job-1')
// => training_loss{job="training-job-1"}
```

### Notebooks
```typescript
// CPU usage
OpenShiftAIQueries.notebookCPUUsage('notebook-server')
// => rate(container_cpu_usage_seconds_total{pod=~"notebook-server.*"}[5m])

// Memory usage
OpenShiftAIQueries.notebookMemoryUsage('notebook-server')
// => container_memory_working_set_bytes{pod=~"notebook-server.*"}
```

### Infrastructure
```typescript
// Pod restarts
OpenShiftAIQueries.podRestartCount('my-pod')
// => kube_pod_container_status_restarts_total{pod="my-pod"}

// Container status
OpenShiftAIQueries.containerStatus('my-pod')
// => kube_pod_container_status_running{pod="my-pod"}
```

---

## Usage Examples

### Creating a Complete Dashboard Panel

```typescript
import {
  createTimeSeriesChartPlugin,
  OpenShiftAIQueries,
  OPENSHIFT_AI_COLORS
} from '@/plugins';

const requestRatePanel = {
  kind: 'Panel',
  spec: {
    display: {
      name: 'Request Rate',
      description: 'Requests per second for model serving'
    },
    plugin: {
      kind: 'TimeSeriesChart',
      spec: createTimeSeriesChartPlugin({
        queries: [{
          query: OpenShiftAIQueries.modelServingRequestRate('llama-2-7b'),
          legend: 'Llama 2 7B'
        }],
        visual: {
          axis: {
            yAxis: {
              label: 'Requests/sec',
              format: { unit: 'ops', decimals: 2 }
            }
          }
        }
      })
    }
  }
};
```

### Combining Multiple Presets

```typescript
import {
  createRequestRateChart,
  createLatencyChart,
  createErrorRateChart,
  createSuccessRateGauge,
  createModelServingTable
} from '@/plugins';

const modelServingDashboard = {
  charts: {
    requestRate: createRequestRateChart(),
    latency: createLatencyChart(),
    errorRate: createErrorRateChart(),
  },
  gauges: {
    successRate: createSuccessRateGauge()
  },
  tables: {
    models: createModelServingTable()
  }
};
```

---

## Color Reference

### Primary (Blue)
- `#06c` - Main brand blue
- `#004080` - Dark blue
- `#0088ce` - Light blue
- `#00659c` - Medium blue
- `#004d75` - Deep blue

### Secondary (Green)
- `#3e8635` - Main green
- `#2d7623` - Dark green
- `#1e6823` - Forest green
- `#0f5323` - Deep green
- `#006400` - Dark green

### Warning (Orange/Yellow)
- `#f0ab00` - Main warning
- `#ec7a08` - Orange
- `#c58c00` - Dark yellow
- `#f4c145` - Light yellow
- `#f9d67a` - Pale yellow

### Danger (Red)
- `#c9190b` - Main danger
- `#a30000` - Dark red
- `#7d1007` - Blood red
- `#470000` - Deep red
- `#3c0000` - Darkest red

### Neutral (Gray)
- `#6a6e73` - Main gray
- `#4f5255` - Dark gray
- `#72767b` - Medium gray
- `#8a8d90` - Light gray
- `#d2d2d2` - Pale gray

---

## Quick Decision Matrix

**Need to show...**

| Metric Type | Best Chart | Best Stat | Best Gauge |
|-------------|------------|-----------|------------|
| Request rate | Time Series | Requests/sec Stat | - |
| Latency | Time Series | Latency Stat | - |
| Error rate | Time Series | Error Rate Stat | - |
| CPU usage | Time Series | - | CPU Gauge |
| Memory usage | Time Series | Memory Stat | Memory Gauge |
| GPU usage | Time Series | - | GPU Gauge |
| Success rate | - | Uptime Stat | Success Gauge |
| Count metrics | - | Request Count | - |
| Model comparison | Bar Chart | - | - |
| Resource breakdown | Bar Chart (stacked) | - | - |
| Multiple metrics | Table View | - | - |
| Training progress | Time Series | Epochs Stat | Accuracy Gauge |

---

## Tips & Best Practices

1. **Use Time Series for trends** over time (last hour, day, week)
2. **Use Gauges for instant values** that need threshold visualization
3. **Use Stats for KPIs** on overview dashboards
4. **Use Tables for detailed data** that needs sorting/filtering
5. **Use Bar Charts for comparisons** across categories
6. **Combine chart types** for comprehensive views (e.g., gauge + time series)
7. **Apply consistent colors** using OpenShift AI palette
8. **Set meaningful thresholds** based on SLOs/SLAs
9. **Use abbreviations** for large numbers (K/M/B)
10. **Optimize step intervals** for query performance

---

## Support

For questions or issues:
- Check `/plugins` source code for implementation details
- Review `/types/dashboard.ts` for type definitions
- See `phase3-task-group-6-summary.md` for architecture overview
