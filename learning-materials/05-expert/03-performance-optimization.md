# Module 19: Performance Optimization

## Learning Objectives

By the end of this module, you will be able to:

- Implement advanced pipeline performance optimization techniques
- Design and implement caching strategies for CI/CD pipelines
- Optimize build and deployment processes for speed and efficiency
- Implement parallel execution and resource optimization
- Monitor and analyze pipeline performance metrics
- Implement advanced testing optimization strategies
- Design scalable CI/CD architectures
- Troubleshoot performance bottlenecks

## Prerequisites

- Completion of Modules 1-18
- Understanding of CI/CD pipeline architecture
- Experience with containerization and orchestration
- Knowledge of monitoring and observability tools
- Familiarity with cloud infrastructure

---

## 1. Pipeline Performance Analysis

### Performance Metrics Collection

```python
# scripts/pipeline-analyzer.py
import json
import time
import statistics
from datetime import datetime, timedelta
from typing import Dict, List, Any, Optional
import requests
import boto3
from dataclasses import dataclass
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@dataclass
class PipelineMetrics:
    pipeline_id: str
    start_time: datetime
    end_time: datetime
    duration: float
    status: str
    stages: List[Dict[str, Any]]
    resource_usage: Dict[str, Any]
    cache_hits: int
    cache_misses: int
    artifacts_size: int
    test_count: int
    test_duration: float

class PipelineAnalyzer:
    def __init__(self, github_token: str, aws_region: str = 'us-west-2'):
        self.github_token = github_token
        self.github_headers = {
            'Authorization': f'token {github_token}',
            'Accept': 'application/vnd.github.v3+json'
        }
        self.cloudwatch = boto3.client('cloudwatch', region_name=aws_region)
        self.s3 = boto3.client('s3', region_name=aws_region)
        
    def analyze_github_workflows(self, owner: str, repo: str, days: int = 30) -> List[PipelineMetrics]:
        """Analyze GitHub Actions workflow performance"""
        logger.info(f"Analyzing workflows for {owner}/{repo} over {days} days")
        
        # Get workflow runs
        since = (datetime.now() - timedelta(days=days)).isoformat()
        url = f'https://api.github.com/repos/{owner}/{repo}/actions/runs'
        params = {
            'created': f'>={since}',
            'per_page': 100
        }
        
        response = requests.get(url, headers=self.github_headers, params=params)
        response.raise_for_status()
        
        workflow_runs = response.json()['workflow_runs']
        metrics = []
        
        for run in workflow_runs:
            try:
                metric = self._analyze_workflow_run(owner, repo, run)
                if metric:
                    metrics.append(metric)
            except Exception as e:
                logger.warning(f"Failed to analyze run {run['id']}: {e}")
        
        return metrics
    
    def _analyze_workflow_run(self, owner: str, repo: str, run: Dict[str, Any]) -> Optional[PipelineMetrics]:
        """Analyze individual workflow run"""
        run_id = run['id']
        
        # Get job details
        jobs_url = f'https://api.github.com/repos/{owner}/{repo}/actions/runs/{run_id}/jobs'
        jobs_response = requests.get(jobs_url, headers=self.github_headers)
        jobs_response.raise_for_status()
        
        jobs = jobs_response.json()['jobs']
        
        # Calculate metrics
        start_time = datetime.fromisoformat(run['created_at'].replace('Z', '+00:00'))
        end_time = datetime.fromisoformat(run['updated_at'].replace('Z', '+00:00')) if run['updated_at'] else start_time
        duration = (end_time - start_time).total_seconds()
        
        stages = []
        total_test_duration = 0
        total_test_count = 0
        
        for job in jobs:
            job_start = datetime.fromisoformat(job['started_at'].replace('Z', '+00:00')) if job['started_at'] else start_time
            job_end = datetime.fromisoformat(job['completed_at'].replace('Z', '+00:00')) if job['completed_at'] else job_start
            job_duration = (job_end - job_start).total_seconds()
            
            # Get job steps
            steps = job.get('steps', [])
            step_details = []
            
            for step in steps:
                step_start = datetime.fromisoformat(step['started_at'].replace('Z', '+00:00')) if step['started_at'] else job_start
                step_end = datetime.fromisoformat(step['completed_at'].replace('Z', '+00:00')) if step['completed_at'] else step_start
                step_duration = (step_end - step_start).total_seconds()
                
                step_details.append({
                    'name': step['name'],
                    'duration': step_duration,
                    'status': step['conclusion'],
                    'number': step['number']
                })
                
                # Identify test steps
                if any(keyword in step['name'].lower() for keyword in ['test', 'spec', 'unit', 'integration']):
                    total_test_duration += step_duration
                    total_test_count += 1
            
            stages.append({
                'name': job['name'],
                'duration': job_duration,
                'status': job['conclusion'],
                'steps': step_details,
                'runner_name': job.get('runner_name', 'unknown'),
                'runner_group_name': job.get('runner_group_name', 'default')
            })
        
        # Estimate resource usage (simplified)
        resource_usage = {
            'cpu_minutes': duration / 60,  # Simplified calculation
            'memory_gb_minutes': (duration / 60) * 2,  # Assume 2GB average
            'storage_gb': 0.1  # Simplified
        }
        
        # Simulate cache metrics (in real implementation, parse logs)
        cache_hits = len([s for stage in stages for s in stage['steps'] if 'cache' in s['name'].lower() and s['status'] == 'success'])
        cache_misses = len([s for stage in stages for s in stage['steps'] if 'cache' in s['name'].lower() and s['status'] != 'success'])
        
        return PipelineMetrics(
            pipeline_id=str(run_id),
            start_time=start_time,
            end_time=end_time,
            duration=duration,
            status=run['conclusion'] or 'in_progress',
            stages=stages,
            resource_usage=resource_usage,
            cache_hits=cache_hits,
            cache_misses=cache_misses,
            artifacts_size=0,  # Would need to query artifacts API
            test_count=total_test_count,
            test_duration=total_test_duration
        )
    
    def generate_performance_report(self, metrics: List[PipelineMetrics]) -> Dict[str, Any]:
        """Generate comprehensive performance report"""
        if not metrics:
            return {'error': 'No metrics available'}
        
        # Calculate statistics
        durations = [m.duration for m in metrics if m.status == 'success']
        failed_runs = [m for m in metrics if m.status == 'failure']
        
        # Stage analysis
        stage_stats = self._analyze_stage_performance(metrics)
        
        # Resource analysis
        resource_stats = self._analyze_resource_usage(metrics)
        
        # Cache analysis
        cache_stats = self._analyze_cache_performance(metrics)
        
        # Test analysis
        test_stats = self._analyze_test_performance(metrics)
        
        # Trend analysis
        trend_stats = self._analyze_trends(metrics)
        
        report = {
            'summary': {
                'total_runs': len(metrics),
                'successful_runs': len(durations),
                'failed_runs': len(failed_runs),
                'success_rate': len(durations) / len(metrics) * 100 if metrics else 0,
                'average_duration': statistics.mean(durations) if durations else 0,
                'median_duration': statistics.median(durations) if durations else 0,
                'p95_duration': self._percentile(durations, 95) if durations else 0,
                'p99_duration': self._percentile(durations, 99) if durations else 0,
                'min_duration': min(durations) if durations else 0,
                'max_duration': max(durations) if durations else 0
            },
            'stage_analysis': stage_stats,
            'resource_analysis': resource_stats,
            'cache_analysis': cache_stats,
            'test_analysis': test_stats,
            'trend_analysis': trend_stats,
            'bottlenecks': self._identify_bottlenecks(metrics),
            'recommendations': self._generate_recommendations(metrics)
        }
        
        return report
    
    def _analyze_stage_performance(self, metrics: List[PipelineMetrics]) -> Dict[str, Any]:
        """Analyze performance by stage"""
        stage_durations = {}
        
        for metric in metrics:
            for stage in metric.stages:
                stage_name = stage['name']
                if stage_name not in stage_durations:
                    stage_durations[stage_name] = []
                stage_durations[stage_name].append(stage['duration'])
        
        stage_stats = {}
        for stage_name, durations in stage_durations.items():
            if durations:
                stage_stats[stage_name] = {
                    'average_duration': statistics.mean(durations),
                    'median_duration': statistics.median(durations),
                    'p95_duration': self._percentile(durations, 95),
                    'min_duration': min(durations),
                    'max_duration': max(durations),
                    'total_runs': len(durations)
                }
        
        return stage_stats
    
    def _analyze_resource_usage(self, metrics: List[PipelineMetrics]) -> Dict[str, Any]:
        """Analyze resource usage patterns"""
        cpu_usage = [m.resource_usage['cpu_minutes'] for m in metrics]
        memory_usage = [m.resource_usage['memory_gb_minutes'] for m in metrics]
        storage_usage = [m.resource_usage['storage_gb'] for m in metrics]
        
        return {
            'cpu_minutes': {
                'total': sum(cpu_usage),
                'average': statistics.mean(cpu_usage) if cpu_usage else 0,
                'peak': max(cpu_usage) if cpu_usage else 0
            },
            'memory_gb_minutes': {
                'total': sum(memory_usage),
                'average': statistics.mean(memory_usage) if memory_usage else 0,
                'peak': max(memory_usage) if memory_usage else 0
            },
            'storage_gb': {
                'total': sum(storage_usage),
                'average': statistics.mean(storage_usage) if storage_usage else 0,
                'peak': max(storage_usage) if storage_usage else 0
            }
        }
    
    def _analyze_cache_performance(self, metrics: List[PipelineMetrics]) -> Dict[str, Any]:
        """Analyze cache performance"""
        total_hits = sum(m.cache_hits for m in metrics)
        total_misses = sum(m.cache_misses for m in metrics)
        total_requests = total_hits + total_misses
        
        return {
            'total_requests': total_requests,
            'cache_hits': total_hits,
            'cache_misses': total_misses,
            'hit_rate': (total_hits / total_requests * 100) if total_requests > 0 else 0,
            'miss_rate': (total_misses / total_requests * 100) if total_requests > 0 else 0
        }
    
    def _analyze_test_performance(self, metrics: List[PipelineMetrics]) -> Dict[str, Any]:
        """Analyze test performance"""
        test_durations = [m.test_duration for m in metrics if m.test_duration > 0]
        test_counts = [m.test_count for m in metrics if m.test_count > 0]
        
        return {
            'total_test_runs': len(test_durations),
            'average_test_duration': statistics.mean(test_durations) if test_durations else 0,
            'median_test_duration': statistics.median(test_durations) if test_durations else 0,
            'average_test_count': statistics.mean(test_counts) if test_counts else 0,
            'total_test_time': sum(test_durations),
            'test_efficiency': {
                'avg_time_per_test': statistics.mean([d/c for d, c in zip(test_durations, test_counts) if c > 0]) if test_counts else 0
            }
        }
    
    def _analyze_trends(self, metrics: List[PipelineMetrics]) -> Dict[str, Any]:
        """Analyze performance trends over time"""
        # Sort by time
        sorted_metrics = sorted(metrics, key=lambda m: m.start_time)
        
        # Group by day
        daily_stats = {}
        for metric in sorted_metrics:
            day = metric.start_time.date().isoformat()
            if day not in daily_stats:
                daily_stats[day] = []
            daily_stats[day].append(metric.duration)
        
        # Calculate daily averages
        daily_averages = {}
        for day, durations in daily_stats.items():
            daily_averages[day] = statistics.mean(durations)
        
        # Calculate trend
        days = list(daily_averages.keys())
        values = list(daily_averages.values())
        
        trend = 'stable'
        if len(values) >= 2:
            if values[-1] > values[0] * 1.1:
                trend = 'degrading'
            elif values[-1] < values[0] * 0.9:
                trend = 'improving'
        
        return {
            'daily_averages': daily_averages,
            'trend': trend,
            'trend_percentage': ((values[-1] - values[0]) / values[0] * 100) if len(values) >= 2 and values[0] > 0 else 0
        }
    
    def _identify_bottlenecks(self, metrics: List[PipelineMetrics]) -> List[Dict[str, Any]]:
        """Identify performance bottlenecks"""
        bottlenecks = []
        
        # Analyze stage durations
        stage_durations = {}
        for metric in metrics:
            for stage in metric.stages:
                stage_name = stage['name']
                if stage_name not in stage_durations:
                    stage_durations[stage_name] = []
                stage_durations[stage_name].append(stage['duration'])
        
        # Find slowest stages
        for stage_name, durations in stage_durations.items():
            if durations:
                avg_duration = statistics.mean(durations)
                if avg_duration > 300:  # More than 5 minutes
                    bottlenecks.append({
                        'type': 'slow_stage',
                        'stage': stage_name,
                        'average_duration': avg_duration,
                        'severity': 'high' if avg_duration > 600 else 'medium'
                    })
        
        # Check cache performance
        total_hits = sum(m.cache_hits for m in metrics)
        total_misses = sum(m.cache_misses for m in metrics)
        if total_hits + total_misses > 0:
            hit_rate = total_hits / (total_hits + total_misses)
            if hit_rate < 0.7:  # Less than 70% hit rate
                bottlenecks.append({
                    'type': 'poor_cache_performance',
                    'hit_rate': hit_rate * 100,
                    'severity': 'high' if hit_rate < 0.5 else 'medium'
                })
        
        # Check test performance
        test_durations = [m.test_duration for m in metrics if m.test_duration > 0]
        if test_durations:
            avg_test_duration = statistics.mean(test_durations)
            if avg_test_duration > 600:  # More than 10 minutes
                bottlenecks.append({
                    'type': 'slow_tests',
                    'average_duration': avg_test_duration,
                    'severity': 'high' if avg_test_duration > 1200 else 'medium'
                })
        
        return bottlenecks
    
    def _generate_recommendations(self, metrics: List[PipelineMetrics]) -> List[str]:
        """Generate optimization recommendations"""
        recommendations = []
        bottlenecks = self._identify_bottlenecks(metrics)
        
        for bottleneck in bottlenecks:
            if bottleneck['type'] == 'slow_stage':
                recommendations.append(
                    f"Optimize '{bottleneck['stage']}' stage - consider parallelization, caching, or resource scaling"
                )
            elif bottleneck['type'] == 'poor_cache_performance':
                recommendations.append(
                    f"Improve cache strategy - current hit rate is {bottleneck['hit_rate']:.1f}%"
                )
            elif bottleneck['type'] == 'slow_tests':
                recommendations.append(
                    "Optimize test execution - consider parallel testing, test selection, or faster test environments"
                )
        
        # General recommendations
        if len(metrics) > 0:
            avg_duration = statistics.mean([m.duration for m in metrics if m.status == 'success'])
            if avg_duration > 1800:  # More than 30 minutes
                recommendations.append("Consider breaking down the pipeline into smaller, parallel jobs")
            
            # Check for resource optimization opportunities
            cpu_usage = [m.resource_usage['cpu_minutes'] for m in metrics]
            if cpu_usage and statistics.mean(cpu_usage) > 60:
                recommendations.append("Consider using more powerful runners or optimizing resource-intensive tasks")
        
        if not recommendations:
            recommendations.append("Pipeline performance is within acceptable ranges")
        
        return recommendations
    
    def _percentile(self, data: List[float], percentile: int) -> float:
        """Calculate percentile"""
        if not data:
            return 0
        sorted_data = sorted(data)
        index = (percentile / 100) * (len(sorted_data) - 1)
        if index.is_integer():
            return sorted_data[int(index)]
        else:
            lower = sorted_data[int(index)]
            upper = sorted_data[int(index) + 1]
            return lower + (upper - lower) * (index - int(index))
    
    def export_metrics_to_cloudwatch(self, metrics: List[PipelineMetrics], namespace: str = 'CI/CD/Performance'):
        """Export metrics to CloudWatch"""
        logger.info(f"Exporting {len(metrics)} metrics to CloudWatch")
        
        metric_data = []
        
        for metric in metrics:
            timestamp = metric.start_time
            
            # Pipeline duration
            metric_data.append({
                'MetricName': 'PipelineDuration',
                'Value': metric.duration,
                'Unit': 'Seconds',
                'Timestamp': timestamp,
                'Dimensions': [
                    {'Name': 'Status', 'Value': metric.status}
                ]
            })
            
            # Resource usage
            metric_data.append({
                'MetricName': 'CPUMinutes',
                'Value': metric.resource_usage['cpu_minutes'],
                'Unit': 'Count',
                'Timestamp': timestamp
            })
            
            # Cache performance
            if metric.cache_hits + metric.cache_misses > 0:
                hit_rate = metric.cache_hits / (metric.cache_hits + metric.cache_misses) * 100
                metric_data.append({
                    'MetricName': 'CacheHitRate',
                    'Value': hit_rate,
                    'Unit': 'Percent',
                    'Timestamp': timestamp
                })
            
            # Test performance
            if metric.test_duration > 0:
                metric_data.append({
                    'MetricName': 'TestDuration',
                    'Value': metric.test_duration,
                    'Unit': 'Seconds',
                    'Timestamp': timestamp
                })
        
        # Send metrics in batches
        batch_size = 20
        for i in range(0, len(metric_data), batch_size):
            batch = metric_data[i:i + batch_size]
            try:
                self.cloudwatch.put_metric_data(
                    Namespace=namespace,
                    MetricData=batch
                )
            except Exception as e:
                logger.error(f"Failed to send metrics batch: {e}")

def main():
    import argparse
    
    parser = argparse.ArgumentParser(description='Pipeline Performance Analyzer')
    parser.add_argument('--github-token', required=True, help='GitHub token')
    parser.add_argument('--owner', required=True, help='Repository owner')
    parser.add_argument('--repo', required=True, help='Repository name')
    parser.add_argument('--days', type=int, default=30, help='Days to analyze')
    parser.add_argument('--output-file', default='performance-report.json', help='Output file')
    parser.add_argument('--export-cloudwatch', action='store_true', help='Export to CloudWatch')
    parser.add_argument('--aws-region', default='us-west-2', help='AWS region')
    
    args = parser.parse_args()
    
    analyzer = PipelineAnalyzer(args.github_token, args.aws_region)
    
    # Analyze workflows
    print(f"Analyzing workflows for {args.owner}/{args.repo}...")
    metrics = analyzer.analyze_github_workflows(args.owner, args.repo, args.days)
    
    if not metrics:
        print("No workflow data found")
        return
    
    # Generate report
    print("Generating performance report...")
    report = analyzer.generate_performance_report(metrics)
    
    # Save report
    with open(args.output_file, 'w') as f:
        json.dump(report, f, indent=2)
    
    # Export to CloudWatch if requested
    if args.export_cloudwatch:
        print("Exporting metrics to CloudWatch...")
        analyzer.export_metrics_to_cloudwatch(metrics)
    
    # Print summary
    print(f"\n{'='*60}")
    print("PERFORMANCE ANALYSIS SUMMARY")
    print(f"{'='*60}")
    print(f"Total Runs: {report['summary']['total_runs']}")
    print(f"Success Rate: {report['summary']['success_rate']:.1f}%")
    print(f"Average Duration: {report['summary']['average_duration']:.1f}s")
    print(f"P95 Duration: {report['summary']['p95_duration']:.1f}s")
    
    print(f"\nBottlenecks:")
    for bottleneck in report['bottlenecks']:
        print(f"- {bottleneck['type']}: {bottleneck.get('stage', 'N/A')}")
    
    print(f"\nRecommendations:")
    for i, rec in enumerate(report['recommendations'][:5], 1):
        print(f"{i}. {rec}")
    
    print(f"\nDetailed report saved to: {args.output_file}")

if __name__ == "__main__":
    main()
```

---

## 4. Resource Optimization

### Container Resource Management

```yaml
# .github/workflows/resource-optimized-pipeline.yml
name: Resource Optimized Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  resource-analysis:
    runs-on: ubuntu-latest
    outputs:
      cpu-intensive: ${{ steps.analysis.outputs.cpu-intensive }}
      memory-intensive: ${{ steps.analysis.outputs.memory-intensive }}
      io-intensive: ${{ steps.analysis.outputs.io-intensive }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Analyze resource requirements
        id: analysis
        run: |
          # Analyze codebase for resource requirements
          cpu_intensive="false"
          memory_intensive="false"
          io_intensive="false"
          
          # Check for CPU-intensive operations
          if grep -r "crypto\|hash\|compress\|image.*process" --include="*.js" --include="*.py" .; then
            cpu_intensive="true"
          fi
          
          # Check for memory-intensive operations
          if grep -r "large.*array\|big.*data\|stream" --include="*.js" --include="*.py" .; then
            memory_intensive="true"
          fi
          
          # Check for I/O intensive operations
          if grep -r "database\|file.*upload\|download" --include="*.js" --include="*.py" .; then
            io_intensive="true"
          fi
          
          echo "cpu-intensive=$cpu_intensive" >> $GITHUB_OUTPUT
          echo "memory-intensive=$memory_intensive" >> $GITHUB_OUTPUT
          echo "io-intensive=$io_intensive" >> $GITHUB_OUTPUT
  
  optimized-build:
    runs-on: ${{ matrix.runner-type }}
    needs: resource-analysis
    strategy:
      matrix:
        include:
          - component: frontend
            runner-type: ubuntu-latest
            cpu-limit: "2"
            memory-limit: "4Gi"
          - component: backend
            runner-type: ${{ needs.resource-analysis.outputs.cpu-intensive == 'true' && 'ubuntu-latest-4-cores' || 'ubuntu-latest' }}
            cpu-limit: ${{ needs.resource-analysis.outputs.cpu-intensive == 'true' && '4' || '2' }}
            memory-limit: ${{ needs.resource-analysis.outputs.memory-intensive == 'true' && '8Gi' || '4Gi' }}
          - component: api
            runner-type: ubuntu-latest
            cpu-limit: "1"
            memory-limit: "2Gi"
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        with:
          driver-opts: |
            image=moby/buildkit:master
            network=host
      
      - name: Build with resource constraints
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./docker/${{ matrix.component }}/Dockerfile
          push: false
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/${{ matrix.component }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            CPU_LIMIT=${{ matrix.cpu-limit }}
            MEMORY_LIMIT=${{ matrix.memory-limit }}
          platforms: linux/amd64
  
  performance-test:
    runs-on: ubuntu-latest
    needs: optimized-build
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Run performance benchmarks
        run: |
          echo "Running performance benchmarks..."
          # Performance testing logic would go here
          python scripts/performance-benchmark.py
      
      - name: Upload performance results
        uses: actions/upload-artifact@v3
        with:
          name: performance-results
          path: performance-results.json
```

### Performance Monitoring Script

```python
# scripts/performance-benchmark.py
import json
import time
import psutil
import subprocess
import threading
from typing import Dict, List, Any, Optional
from dataclasses import dataclass, asdict
import logging
from datetime import datetime

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@dataclass
class ResourceMetrics:
    timestamp: float
    cpu_percent: float
    memory_percent: float
    memory_mb: float
    disk_io_read: int
    disk_io_write: int
    network_sent: int
    network_recv: int

@dataclass
class BenchmarkResult:
    test_name: str
    duration: float
    success: bool
    peak_cpu: float
    peak_memory: float
    avg_cpu: float
    avg_memory: float
    resource_metrics: List[ResourceMetrics]
    error_message: Optional[str] = None

class PerformanceBenchmark:
    def __init__(self):
        self.monitoring = False
        self.metrics = []
        self.monitor_thread = None
        
    def start_monitoring(self, interval: float = 1.0):
        """Start resource monitoring"""
        self.monitoring = True
        self.metrics = []
        
        def monitor():
            while self.monitoring:
                try:
                    # Get system metrics
                    cpu_percent = psutil.cpu_percent(interval=None)
                    memory = psutil.virtual_memory()
                    disk_io = psutil.disk_io_counters()
                    network_io = psutil.net_io_counters()
                    
                    metric = ResourceMetrics(
                        timestamp=time.time(),
                        cpu_percent=cpu_percent,
                        memory_percent=memory.percent,
                        memory_mb=memory.used / 1024 / 1024,
                        disk_io_read=disk_io.read_bytes if disk_io else 0,
                        disk_io_write=disk_io.write_bytes if disk_io else 0,
                        network_sent=network_io.bytes_sent if network_io else 0,
                        network_recv=network_io.bytes_recv if network_io else 0
                    )
                    
                    self.metrics.append(metric)
                    time.sleep(interval)
                    
                except Exception as e:
                    logger.warning(f"Monitoring error: {e}")
                    time.sleep(interval)
        
        self.monitor_thread = threading.Thread(target=monitor, daemon=True)
        self.monitor_thread.start()
        logger.info("Resource monitoring started")
    
    def stop_monitoring(self) -> List[ResourceMetrics]:
        """Stop resource monitoring and return collected metrics"""
        self.monitoring = False
        if self.monitor_thread:
            self.monitor_thread.join(timeout=5)
        
        logger.info(f"Resource monitoring stopped. Collected {len(self.metrics)} data points")
        return self.metrics.copy()
    
    def run_benchmark(self, test_name: str, command: str, 
                     timeout: int = 300) -> BenchmarkResult:
        """Run a single benchmark test"""
        logger.info(f"Running benchmark: {test_name}")
        
        # Start monitoring
        self.start_monitoring()
        
        start_time = time.time()
        success = False
        error_message = None
        
        try:
            # Run the command
            result = subprocess.run(
                command,
                shell=True,
                capture_output=True,
                text=True,
                timeout=timeout
            )
            
            success = result.returncode == 0
            if not success:
                error_message = result.stderr
                
        except subprocess.TimeoutExpired:
            error_message = f"Test timed out after {timeout} seconds"
        except Exception as e:
            error_message = str(e)
        
        end_time = time.time()
        duration = end_time - start_time
        
        # Stop monitoring and get metrics
        metrics = self.stop_monitoring()
        
        # Calculate statistics
        if metrics:
            cpu_values = [m.cpu_percent for m in metrics]
            memory_values = [m.memory_percent for m in metrics]
            
            peak_cpu = max(cpu_values) if cpu_values else 0
            peak_memory = max(memory_values) if memory_values else 0
            avg_cpu = sum(cpu_values) / len(cpu_values) if cpu_values else 0
            avg_memory = sum(memory_values) / len(memory_values) if memory_values else 0
        else:
            peak_cpu = peak_memory = avg_cpu = avg_memory = 0
        
        return BenchmarkResult(
            test_name=test_name,
            duration=duration,
            success=success,
            peak_cpu=peak_cpu,
            peak_memory=peak_memory,
            avg_cpu=avg_cpu,
            avg_memory=avg_memory,
            resource_metrics=metrics,
            error_message=error_message
        )
    
    def run_build_benchmarks(self) -> List[BenchmarkResult]:
        """Run comprehensive build benchmarks"""
        benchmarks = [
            ("npm_install", "npm ci --prefer-offline"),
            ("typescript_compile", "npx tsc --noEmit"),
            ("webpack_build", "npm run build:production"),
            ("unit_tests", "npm run test:unit"),
            ("integration_tests", "npm run test:integration"),
            ("lint_check", "npm run lint"),
            ("docker_build", "docker build -t test-image ."),
        ]
        
        results = []
        
        for test_name, command in benchmarks:
            try:
                result = self.run_benchmark(test_name, command)
                results.append(result)
                
                # Log immediate results
                status = "✅ PASSED" if result.success else "❌ FAILED"
                logger.info(f"{test_name}: {status} ({result.duration:.1f}s, "
                           f"Peak CPU: {result.peak_cpu:.1f}%, Peak Memory: {result.peak_memory:.1f}%)")
                
            except Exception as e:
                logger.error(f"Benchmark {test_name} failed: {e}")
                results.append(BenchmarkResult(
                    test_name=test_name,
                    duration=0,
                    success=False,
                    peak_cpu=0,
                    peak_memory=0,
                    avg_cpu=0,
                    avg_memory=0,
                    resource_metrics=[],
                    error_message=str(e)
                ))
        
        return results
    
    def analyze_performance(self, results: List[BenchmarkResult]) -> Dict[str, Any]:
        """Analyze benchmark results and generate insights"""
        total_duration = sum(r.duration for r in results)
        successful_tests = [r for r in results if r.success]
        failed_tests = [r for r in results if not r.success]
        
        # Resource usage analysis
        peak_cpu_usage = max((r.peak_cpu for r in results), default=0)
        peak_memory_usage = max((r.peak_memory for r in results), default=0)
        avg_cpu_usage = sum(r.avg_cpu for r in results) / len(results) if results else 0
        avg_memory_usage = sum(r.avg_memory for r in results) / len(results) if results else 0
        
        # Identify bottlenecks
        bottlenecks = []
        for result in results:
            if result.duration > 60:  # More than 1 minute
                bottlenecks.append({
                    'test': result.test_name,
                    'duration': result.duration,
                    'type': 'slow_execution'
                })
            
            if result.peak_cpu > 80:  # High CPU usage
                bottlenecks.append({
                    'test': result.test_name,
                    'peak_cpu': result.peak_cpu,
                    'type': 'cpu_intensive'
                })
            
            if result.peak_memory > 80:  # High memory usage
                bottlenecks.append({
                    'test': result.test_name,
                    'peak_memory': result.peak_memory,
                    'type': 'memory_intensive'
                })
        
        # Generate recommendations
        recommendations = []
        
        if peak_cpu_usage > 90:
            recommendations.append("Consider using runners with more CPU cores for CPU-intensive tasks")
        
        if peak_memory_usage > 90:
            recommendations.append("Consider using runners with more memory for memory-intensive tasks")
        
        if len(failed_tests) > 0:
            recommendations.append(f"{len(failed_tests)} test(s) failed - investigate and fix before optimizing")
        
        slow_tests = [r for r in results if r.duration > 120]  # More than 2 minutes
        if slow_tests:
            recommendations.append(f"Optimize slow tests: {', '.join(r.test_name for r in slow_tests)}")
        
        if total_duration > 600:  # More than 10 minutes total
            recommendations.append("Consider parallelizing tests to reduce total execution time")
        
        return {
            'summary': {
                'total_duration': total_duration,
                'successful_tests': len(successful_tests),
                'failed_tests': len(failed_tests),
                'success_rate': len(successful_tests) / len(results) * 100 if results else 0,
                'peak_cpu_usage': peak_cpu_usage,
                'peak_memory_usage': peak_memory_usage,
                'avg_cpu_usage': avg_cpu_usage,
                'avg_memory_usage': avg_memory_usage
            },
            'test_results': [asdict(r) for r in results],
            'bottlenecks': bottlenecks,
            'recommendations': recommendations,
            'performance_grade': self._calculate_performance_grade(results)
        }
    
    def _calculate_performance_grade(self, results: List[BenchmarkResult]) -> str:
        """Calculate overall performance grade"""
        if not results:
            return 'F'
        
        # Calculate score based on multiple factors
        success_rate = len([r for r in results if r.success]) / len(results)
        avg_duration = sum(r.duration for r in results) / len(results)
        peak_cpu = max((r.peak_cpu for r in results), default=0)
        peak_memory = max((r.peak_memory for r in results), default=0)
        
        score = 100
        
        # Deduct points for failures
        score -= (1 - success_rate) * 40
        
        # Deduct points for slow execution
        if avg_duration > 60:
            score -= min((avg_duration - 60) / 60 * 20, 20)
        
        # Deduct points for high resource usage
        if peak_cpu > 80:
            score -= (peak_cpu - 80) / 20 * 15
        
        if peak_memory > 80:
            score -= (peak_memory - 80) / 20 * 15
        
        # Assign grade
        if score >= 90:
            return 'A'
        elif score >= 80:
            return 'B'
        elif score >= 70:
            return 'C'
        elif score >= 60:
            return 'D'
        else:
            return 'F'
    
    def export_results(self, results: List[BenchmarkResult], 
                      analysis: Dict[str, Any], filename: str = 'performance-results.json'):
        """Export results to JSON file"""
        export_data = {
            'timestamp': datetime.now().isoformat(),
            'analysis': analysis,
            'raw_results': [asdict(r) for r in results]
        }
        
        with open(filename, 'w') as f:
            json.dump(export_data, f, indent=2)
        
        logger.info(f"Performance results exported to {filename}")

def main():
    import argparse
    
    parser = argparse.ArgumentParser(description='Performance Benchmark Tool')
    parser.add_argument('--output-file', default='performance-results.json', help='Output file')
    parser.add_argument('--custom-tests', help='JSON file with custom test definitions')
    
    args = parser.parse_args()
    
    benchmark = PerformanceBenchmark()
    
    print("🚀 Starting Performance Benchmarks...")
    print(f"{'='*60}")
    
    # Run benchmarks
    results = benchmark.run_build_benchmarks()
    
    # Analyze results
    analysis = benchmark.analyze_performance(results)
    
    # Export results
    benchmark.export_results(results, analysis, args.output_file)
    
    # Print summary
    print(f"\n{'='*60}")
    print("📊 PERFORMANCE BENCHMARK SUMMARY")
    print(f"{'='*60}")
    print(f"Total Duration: {analysis['summary']['total_duration']:.1f}s")
    print(f"Success Rate: {analysis['summary']['success_rate']:.1f}%")
    print(f"Performance Grade: {analysis['performance_grade']}")
    print(f"Peak CPU Usage: {analysis['summary']['peak_cpu_usage']:.1f}%")
    print(f"Peak Memory Usage: {analysis['summary']['peak_memory_usage']:.1f}%")
    
    if analysis['bottlenecks']:
        print(f"\n🔍 Bottlenecks Detected:")
        for bottleneck in analysis['bottlenecks'][:5]:
            print(f"  - {bottleneck['test']}: {bottleneck['type']}")
    
    if analysis['recommendations']:
        print(f"\n💡 Recommendations:")
        for i, rec in enumerate(analysis['recommendations'][:5], 1):
            print(f"  {i}. {rec}")
    
    print(f"\n📄 Detailed results saved to: {args.output_file}")

if __name__ == "__main__":
    main()
```

## 5. Build Optimization

### Multi-stage Docker Optimization

```dockerfile
# Dockerfile.optimized
# Multi-stage build for optimal performance
FROM node:18-alpine AS base
WORKDIR /app

# Dependencies stage
FROM base AS deps
COPY package*.json ./
RUN npm ci --only=production --prefer-offline

# Build stage
FROM base AS build
COPY package*.json ./
RUN npm ci --prefer-offline
COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine AS production
WORKDIR /app

# Install dumb-init for proper signal handling
RUN apk add --no-cache dumb-init

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

# Copy built application
COPY --from=deps --chown=nextjs:nodejs /app/node_modules ./node_modules
COPY --from=build --chown=nextjs:nodejs /app/dist ./dist
COPY --from=build --chown=nextjs:nodejs /app/package.json ./package.json

USER nextjs

EXPOSE 3000

ENTRYPOINT ["dumb-init", "--"]
CMD ["npm", "start"]
```

### Build Cache Optimization

```yaml
# .github/workflows/optimized-build.yml
name: Optimized Build Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-optimization:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Cache node modules
        uses: actions/cache@v3
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-
      
      - name: Cache build artifacts
        uses: actions/cache@v3
        with:
          path: |
            .next/cache
            dist
            build
          key: ${{ runner.os }}-build-${{ hashFiles('**/package-lock.json') }}-${{ hashFiles('**/*.js', '**/*.ts', '**/*.jsx', '**/*.tsx') }}
          restore-keys: |
            ${{ runner.os }}-build-${{ hashFiles('**/package-lock.json') }}-
            ${{ runner.os }}-build-
      
      - name: Install dependencies
        run: npm ci --prefer-offline
      
      - name: Build application
        run: |
          # Enable build optimization flags
          export NODE_OPTIONS="--max-old-space-size=4096"
          export GENERATE_SOURCEMAP=false
          npm run build
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        with:
          driver-opts: |
            image=moby/buildkit:master
            network=host
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile.optimized
          push: false
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: |
            type=gha
            type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache
          cache-to: |
            type=gha,mode=max
            type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache,mode=max
          build-args: |
            BUILDKIT_INLINE_CACHE=1
          platforms: linux/amd64
```

### Build Performance Analyzer

```python
# scripts/build-analyzer.py
import json
import os
import time
import subprocess
import re
from typing import Dict, List, Any, Optional
from dataclasses import dataclass, asdict
from pathlib import Path
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@dataclass
class BuildMetrics:
    stage: str
    duration: float
    cache_hit: bool
    size_mb: Optional[float] = None
    files_processed: Optional[int] = None
    warnings: List[str] = None
    errors: List[str] = None

@dataclass
class BuildAnalysis:
    total_duration: float
    stages: List[BuildMetrics]
    cache_efficiency: float
    recommendations: List[str]
    performance_score: int

class BuildAnalyzer:
    def __init__(self):
        self.build_log = []
        self.start_time = None
        
    def analyze_webpack_build(self, build_output: str) -> BuildAnalysis:
        """Analyze webpack build output"""
        stages = []
        
        # Parse webpack output
        lines = build_output.split('\n')
        current_stage = None
        stage_start = None
        
        for line in lines:
            # Detect build stages
            if 'webpack compiled' in line.lower():
                if current_stage:
                    duration = time.time() - stage_start if stage_start else 0
                    stages.append(BuildMetrics(
                        stage=current_stage,
                        duration=duration,
                        cache_hit=False
                    ))
                current_stage = 'compilation'
                stage_start = time.time()
            
            # Detect cache hits
            elif 'cached' in line.lower():
                if current_stage:
                    stages[-1].cache_hit = True
            
            # Extract file counts
            elif re.search(r'(\d+) modules', line):
                match = re.search(r'(\d+) modules', line)
                if match and current_stage:
                    stages[-1].files_processed = int(match.group(1))
            
            # Extract warnings and errors
            elif 'warning' in line.lower():
                if current_stage and stages:
                    if not stages[-1].warnings:
                        stages[-1].warnings = []
                    stages[-1].warnings.append(line.strip())
            
            elif 'error' in line.lower():
                if current_stage and stages:
                    if not stages[-1].errors:
                        stages[-1].errors = []
                    stages[-1].errors.append(line.strip())
        
        return self._generate_analysis(stages)
    
    def analyze_docker_build(self, dockerfile_path: str) -> BuildAnalysis:
        """Analyze Docker build performance"""
        stages = []
        
        try:
            # Run docker build with timing
            cmd = f"docker build --progress=plain -f {dockerfile_path} ."
            start_time = time.time()
            
            result = subprocess.run(
                cmd,
                shell=True,
                capture_output=True,
                text=True
            )
            
            total_duration = time.time() - start_time
            
            # Parse docker build output
            lines = result.stdout.split('\n') + result.stderr.split('\n')
            
            current_step = None
            step_start = None
            
            for line in lines:
                # Detect build steps
                step_match = re.search(r'#(\d+) (\w+)', line)
                if step_match:
                    if current_step:
                        duration = time.time() - step_start if step_start else 0
                        stages.append(BuildMetrics(
                            stage=f"Step {current_step}",
                            duration=duration,
                            cache_hit='CACHED' in line
                        ))
                    
                    current_step = step_match.group(1)
                    step_start = time.time()
                
                # Detect cache usage
                elif 'CACHED' in line and current_step:
                    if stages:
                        stages[-1].cache_hit = True
            
            return self._generate_analysis(stages, total_duration)
            
        except Exception as e:
            logger.error(f"Docker build analysis failed: {e}")
            return BuildAnalysis(
                total_duration=0,
                stages=[],
                cache_efficiency=0,
                recommendations=["Docker build analysis failed"],
                performance_score=0
            )
    
    def analyze_npm_build(self) -> BuildAnalysis:
        """Analyze npm build performance"""
        stages = []
        
        # Analyze package.json scripts
        try:
            with open('package.json', 'r') as f:
                package_data = json.load(f)
            
            scripts = package_data.get('scripts', {})
            
            # Run and time each build-related script
            build_scripts = ['build', 'compile', 'bundle', 'dist']
            
            for script_name in build_scripts:
                if script_name in scripts:
                    start_time = time.time()
                    
                    try:
                        result = subprocess.run(
                            f"npm run {script_name}",
                            shell=True,
                            capture_output=True,
                            text=True,
                            timeout=300
                        )
                        
                        duration = time.time() - start_time
                        
                        # Check for cache usage
                        cache_hit = 'cache' in result.stdout.lower() or 'cached' in result.stdout.lower()
                        
                        # Extract warnings and errors
                        warnings = [line for line in result.stdout.split('\n') if 'warning' in line.lower()]
                        errors = [line for line in result.stderr.split('\n') if line.strip()]
                        
                        stages.append(BuildMetrics(
                            stage=script_name,
                            duration=duration,
                            cache_hit=cache_hit,
                            warnings=warnings if warnings else None,
                            errors=errors if errors else None
                        ))
                        
                    except subprocess.TimeoutExpired:
                        stages.append(BuildMetrics(
                            stage=script_name,
                            duration=300,
                            cache_hit=False,
                            errors=["Build timed out after 5 minutes"]
                        ))
            
            return self._generate_analysis(stages)
            
        except Exception as e:
            logger.error(f"NPM build analysis failed: {e}")
            return BuildAnalysis(
                total_duration=0,
                stages=[],
                cache_efficiency=0,
                recommendations=["NPM build analysis failed"],
                performance_score=0
            )
    
    def _generate_analysis(self, stages: List[BuildMetrics], 
                          total_duration: Optional[float] = None) -> BuildAnalysis:
        """Generate build analysis from stages"""
        if not total_duration:
            total_duration = sum(stage.duration for stage in stages)
        
        # Calculate cache efficiency
        cached_stages = [stage for stage in stages if stage.cache_hit]
        cache_efficiency = len(cached_stages) / len(stages) * 100 if stages else 0
        
        # Generate recommendations
        recommendations = []
        
        # Slow stages
        slow_stages = [stage for stage in stages if stage.duration > 60]
        if slow_stages:
            recommendations.append(f"Optimize slow stages: {', '.join(s.stage for s in slow_stages)}")
        
        # Low cache efficiency
        if cache_efficiency < 50:
            recommendations.append("Improve caching strategy - current cache efficiency is low")
        
        # High total duration
        if total_duration > 300:  # 5 minutes
            recommendations.append("Consider parallelizing build steps to reduce total time")
        
        # Warnings and errors
        total_warnings = sum(len(stage.warnings or []) for stage in stages)
        total_errors = sum(len(stage.errors or []) for stage in stages)
        
        if total_warnings > 0:
            recommendations.append(f"Address {total_warnings} build warnings")
        
        if total_errors > 0:
            recommendations.append(f"Fix {total_errors} build errors")
        
        # Calculate performance score
        score = 100
        
        # Deduct for slow builds
        if total_duration > 180:  # 3 minutes
            score -= min((total_duration - 180) / 60 * 10, 30)
        
        # Deduct for low cache efficiency
        score -= (100 - cache_efficiency) * 0.2
        
        # Deduct for errors
        score -= total_errors * 10
        
        # Deduct for warnings
        score -= total_warnings * 2
        
        performance_score = max(0, int(score))
        
        return BuildAnalysis(
            total_duration=total_duration,
            stages=stages,
            cache_efficiency=cache_efficiency,
            recommendations=recommendations,
            performance_score=performance_score
        )
    
    def export_analysis(self, analysis: BuildAnalysis, filename: str = 'build-analysis.json'):
        """Export analysis to JSON file"""
        export_data = {
            'timestamp': time.time(),
            'analysis': asdict(analysis)
        }
        
        with open(filename, 'w') as f:
            json.dump(export_data, f, indent=2)
        
        logger.info(f"Build analysis exported to {filename}")

def main():
    import argparse
    
    parser = argparse.ArgumentParser(description='Build Performance Analyzer')
    parser.add_argument('--type', choices=['webpack', 'docker', 'npm'], 
                       default='npm', help='Build type to analyze')
    parser.add_argument('--dockerfile', default='Dockerfile', help='Dockerfile path for Docker analysis')
    parser.add_argument('--output', default='build-analysis.json', help='Output file')
    
    args = parser.parse_args()
    
    analyzer = BuildAnalyzer()
    
    print(f"🔍 Analyzing {args.type} build performance...")
    
    if args.type == 'npm':
        analysis = analyzer.analyze_npm_build()
    elif args.type == 'docker':
        analysis = analyzer.analyze_docker_build(args.dockerfile)
    elif args.type == 'webpack':
        # For webpack, we'd need to capture build output
        print("Webpack analysis requires build output. Run with npm build first.")
        return
    
    # Export analysis
    analyzer.export_analysis(analysis, args.output)
    
    # Print summary
    print(f"\n{'='*60}")
    print("📊 BUILD PERFORMANCE ANALYSIS")
    print(f"{'='*60}")
    print(f"Total Duration: {analysis.total_duration:.1f}s")
    print(f"Cache Efficiency: {analysis.cache_efficiency:.1f}%")
    print(f"Performance Score: {analysis.performance_score}/100")
    print(f"Stages Analyzed: {len(analysis.stages)}")
    
    if analysis.recommendations:
        print(f"\n💡 Recommendations:")
        for i, rec in enumerate(analysis.recommendations, 1):
            print(f"  {i}. {rec}")
    
    print(f"\n📄 Detailed analysis saved to: {args.output}")

if __name__ == "__main__":
    main()
```

---

## 6. Performance Monitoring Dashboard

### Real-time Performance Dashboard

```yaml
# .github/workflows/performance-dashboard.yml
name: Performance Dashboard

on:
  schedule:
    - cron: '0 */6 * * *'  # Every 6 hours
  workflow_dispatch:

env:
  GRAFANA_URL: ${{ secrets.GRAFANA_URL }}
  GRAFANA_API_KEY: ${{ secrets.GRAFANA_API_KEY }}

jobs:
  update-dashboard:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install requests grafana-api pandas matplotlib
      
      - name: Update performance dashboard
        run: python scripts/update-dashboard.py
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Dashboard Update Script

```python
# scripts/update-dashboard.py
import json
import requests
import os
from datetime import datetime, timedelta
from typing import Dict, List, Any
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class PerformanceDashboard:
    def __init__(self):
        self.grafana_url = os.getenv('GRAFANA_URL')
        self.grafana_api_key = os.getenv('GRAFANA_API_KEY')
        self.github_token = os.getenv('GITHUB_TOKEN')
        self.repo = os.getenv('GITHUB_REPOSITORY')
        
    def get_workflow_metrics(self, days: int = 30) -> List[Dict[str, Any]]:
        """Get workflow performance metrics from GitHub API"""
        headers = {
            'Authorization': f'token {self.github_token}',
            'Accept': 'application/vnd.github.v3+json'
        }
        
        # Get workflow runs from the last N days
        since = (datetime.now() - timedelta(days=days)).isoformat()
        
        url = f'https://api.github.com/repos/{self.repo}/actions/runs'
        params = {
            'per_page': 100,
            'created': f'>={since}'
        }
        
        response = requests.get(url, headers=headers, params=params)
        response.raise_for_status()
        
        runs = response.json()['workflow_runs']
        
        metrics = []
        for run in runs:
            if run['conclusion'] in ['success', 'failure']:
                # Calculate duration
                created_at = datetime.fromisoformat(run['created_at'].replace('Z', '+00:00'))
                updated_at = datetime.fromisoformat(run['updated_at'].replace('Z', '+00:00'))
                duration = (updated_at - created_at).total_seconds()
                
                metrics.append({
                    'id': run['id'],
                    'name': run['name'],
                    'status': run['conclusion'],
                    'duration': duration,
                    'created_at': created_at.isoformat(),
                    'branch': run['head_branch'],
                    'commit_sha': run['head_sha'][:8]
                })
        
        return metrics
    
    def calculate_performance_trends(self, metrics: List[Dict[str, Any]]) -> Dict[str, Any]:
        """Calculate performance trends and statistics"""
        if not metrics:
            return {}
        
        # Group by workflow name
        workflows = {}
        for metric in metrics:
            name = metric['name']
            if name not in workflows:
                workflows[name] = []
            workflows[name].append(metric)
        
        trends = {}
        
        for workflow_name, runs in workflows.items():
            # Sort by creation time
            runs.sort(key=lambda x: x['created_at'])
            
            # Calculate statistics
            durations = [run['duration'] for run in runs]
            success_rate = len([r for r in runs if r['status'] == 'success']) / len(runs) * 100
            
            # Calculate trend (comparing first half vs second half)
            mid_point = len(durations) // 2
            if mid_point > 0:
                first_half_avg = sum(durations[:mid_point]) / mid_point
                second_half_avg = sum(durations[mid_point:]) / (len(durations) - mid_point)
                trend = ((second_half_avg - first_half_avg) / first_half_avg) * 100
            else:
                trend = 0
            
            trends[workflow_name] = {
                'avg_duration': sum(durations) / len(durations),
                'min_duration': min(durations),
                'max_duration': max(durations),
                'success_rate': success_rate,
                'total_runs': len(runs),
                'trend_percentage': trend,
                'recent_runs': runs[-10:]  # Last 10 runs
            }
        
        return trends
    
    def create_dashboard_data(self, trends: Dict[str, Any]) -> Dict[str, Any]:
        """Create dashboard data structure"""
        dashboard = {
            'dashboard': {
                'id': None,
                'title': 'CI/CD Performance Dashboard',
                'tags': ['cicd', 'performance'],
                'timezone': 'browser',
                'panels': [],
                'time': {
                    'from': 'now-30d',
                    'to': 'now'
                },
                'timepicker': {},
                'templating': {'list': []},
                'annotations': {'list': []},
                'refresh': '5m',
                'schemaVersion': 30,
                'version': 0,
                'links': []
            }
        }
        
        panel_id = 1
        
        # Overview panel
        overview_panel = {
            'id': panel_id,
            'title': 'Performance Overview',
            'type': 'stat',
            'targets': [],
            'gridPos': {'h': 8, 'w': 24, 'x': 0, 'y': 0},
            'options': {
                'reduceOptions': {
                    'values': False,
                    'calcs': ['lastNotNull'],
                    'fields': ''
                },
                'orientation': 'auto',
                'textMode': 'auto',
                'colorMode': 'value',
                'graphMode': 'area',
                'justifyMode': 'auto'
            },
            'pluginVersion': '8.0.0'
        }
        
        dashboard['dashboard']['panels'].append(overview_panel)
        panel_id += 1
        
        # Create panels for each workflow
        y_pos = 8
        for workflow_name, data in trends.items():
            # Duration trend panel
            duration_panel = {
                'id': panel_id,
                'title': f'{workflow_name} - Duration Trend',
                'type': 'timeseries',
                'targets': [],
                'gridPos': {'h': 8, 'w': 12, 'x': 0, 'y': y_pos},
                'options': {
                    'tooltip': {'mode': 'single'},
                    'legend': {'displayMode': 'list', 'placement': 'bottom'}
                },
                'fieldConfig': {
                    'defaults': {
                        'custom': {
                            'drawStyle': 'line',
                            'lineInterpolation': 'linear',
                            'barAlignment': 0,
                            'lineWidth': 1,
                            'fillOpacity': 0.1,
                            'gradientMode': 'none',
                            'spanNulls': False,
                            'insertNulls': False,
                            'showPoints': 'auto',
                            'pointSize': 5
                        },
                        'color': {'mode': 'palette-classic'},
                        'unit': 's'
                    }
                }
            }
            
            # Success rate panel
            success_panel = {
                'id': panel_id + 1,
                'title': f'{workflow_name} - Success Rate',
                'type': 'gauge',
                'targets': [],
                'gridPos': {'h': 8, 'w': 12, 'x': 12, 'y': y_pos},
                'options': {
                    'reduceOptions': {
                        'values': False,
                        'calcs': ['lastNotNull'],
                        'fields': ''
                    },
                    'orientation': 'auto',
                    'showThresholdLabels': False,
                    'showThresholdMarkers': True
                },
                'fieldConfig': {
                    'defaults': {
                        'color': {
                            'mode': 'thresholds'
                        },
                        'mappings': [],
                        'thresholds': {
                            'mode': 'absolute',
                            'steps': [
                                {'color': 'red', 'value': None},
                                {'color': 'yellow', 'value': 80},
                                {'color': 'green', 'value': 95}
                            ]
                        },
                        'unit': 'percent',
                        'min': 0,
                        'max': 100
                    }
                }
            }
            
            dashboard['dashboard']['panels'].extend([duration_panel, success_panel])
            panel_id += 2
            y_pos += 8
        
        return dashboard
    
    def update_grafana_dashboard(self, dashboard_data: Dict[str, Any]):
        """Update Grafana dashboard"""
        if not self.grafana_url or not self.grafana_api_key:
            logger.warning("Grafana credentials not provided, skipping dashboard update")
            return
        
        headers = {
            'Authorization': f'Bearer {self.grafana_api_key}',
            'Content-Type': 'application/json'
        }
        
        # Create or update dashboard
        url = f'{self.grafana_url}/api/dashboards/db'
        
        response = requests.post(url, headers=headers, json=dashboard_data)
        
        if response.status_code == 200:
            logger.info("Dashboard updated successfully")
        else:
            logger.error(f"Failed to update dashboard: {response.status_code} - {response.text}")
    
    def generate_performance_report(self, trends: Dict[str, Any]) -> str:
        """Generate performance report"""
        report = []
        report.append("# CI/CD Performance Report")
        report.append(f"Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        report.append("")
        
        # Overall statistics
        total_runs = sum(data['total_runs'] for data in trends.values())
        avg_success_rate = sum(data['success_rate'] for data in trends.values()) / len(trends) if trends else 0
        
        report.append("## Overall Statistics")
        report.append(f"- Total workflow runs: {total_runs}")
        report.append(f"- Average success rate: {avg_success_rate:.1f}%")
        report.append(f"- Workflows monitored: {len(trends)}")
        report.append("")
        
        # Per-workflow analysis
        report.append("## Workflow Analysis")
        
        for workflow_name, data in trends.items():
            report.append(f"### {workflow_name}")
            report.append(f"- Average duration: {data['avg_duration']:.1f}s")
            report.append(f"- Success rate: {data['success_rate']:.1f}%")
            report.append(f"- Performance trend: {data['trend_percentage']:+.1f}%")
            report.append(f"- Total runs: {data['total_runs']}")
            
            # Performance assessment
            if data['trend_percentage'] > 10:
                report.append("  - ⚠️ Performance degrading")
            elif data['trend_percentage'] < -10:
                report.append("  - ✅ Performance improving")
            else:
                report.append("  - ➡️ Performance stable")
            
            if data['success_rate'] < 90:
                report.append("  - ❌ Low success rate - needs attention")
            
            report.append("")
        
        # Recommendations
        report.append("## Recommendations")
        
        slow_workflows = [name for name, data in trends.items() if data['avg_duration'] > 600]
        if slow_workflows:
            report.append(f"- Optimize slow workflows: {', '.join(slow_workflows)}")
        
        unreliable_workflows = [name for name, data in trends.items() if data['success_rate'] < 90]
        if unreliable_workflows:
            report.append(f"- Improve reliability of: {', '.join(unreliable_workflows)}")
        
        degrading_workflows = [name for name, data in trends.items() if data['trend_percentage'] > 15]
        if degrading_workflows:
            report.append(f"- Investigate performance degradation in: {', '.join(degrading_workflows)}")
        
        return "\n".join(report)
    
    def run(self):
        """Run dashboard update process"""
        logger.info("Starting performance dashboard update...")
        
        try:
            # Get metrics
            metrics = self.get_workflow_metrics()
            logger.info(f"Retrieved {len(metrics)} workflow runs")
            
            # Calculate trends
            trends = self.calculate_performance_trends(metrics)
            logger.info(f"Analyzed {len(trends)} workflows")
            
            # Create dashboard
            dashboard_data = self.create_dashboard_data(trends)
            
            # Update Grafana
            self.update_grafana_dashboard(dashboard_data)
            
            # Generate report
            report = self.generate_performance_report(trends)
            
            # Save report
            with open('performance-report.md', 'w') as f:
                f.write(report)
            
            logger.info("Performance dashboard update completed")
            
        except Exception as e:
            logger.error(f"Dashboard update failed: {e}")
            raise

def main():
    dashboard = PerformanceDashboard()
    dashboard.run()

if __name__ == "__main__":
    main()
```

---

## 7. Hands-on Exercises

### Exercise 1: Pipeline Performance Analysis

**Objective**: Analyze and optimize a slow CI/CD pipeline

**Tasks**:
1. Set up performance monitoring for your pipeline
2. Identify bottlenecks using the provided analysis tools
3. Implement caching strategies
4. Measure improvement

**Deliverables**:
- Performance analysis report
- Optimized pipeline configuration
- Before/after metrics comparison

### Exercise 2: Advanced Caching Implementation

**Objective**: Implement multi-level caching strategy

**Tasks**:
1. Implement dependency caching
2. Set up build artifact caching
3. Configure Docker layer caching
4. Create cache invalidation strategy

**Deliverables**:
- Caching configuration files
- Cache hit rate analysis
- Performance improvement documentation

### Exercise 3: Parallel Execution Optimization

**Objective**: Optimize test execution through parallelization

**Tasks**:
1. Analyze test suite for parallelization opportunities
2. Implement dynamic test sharding
3. Set up parallel job execution
4. Monitor resource utilization

**Deliverables**:
- Parallel execution configuration
- Resource utilization report
- Test execution time comparison

---

## 8. Troubleshooting Guide

### Common Performance Issues

#### Slow Build Times

**Symptoms**:
- Build takes longer than 10 minutes
- High CPU/memory usage during build
- Frequent timeouts

**Solutions**:
1. **Enable build caching**:
   ```yaml
   - name: Cache dependencies
     uses: actions/cache@v3
     with:
       path: ~/.npm
       key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
   ```

2. **Optimize Docker builds**:
   ```dockerfile
   # Use multi-stage builds
   FROM node:18-alpine AS deps
   COPY package*.json ./
   RUN npm ci --only=production
   ```

3. **Parallelize build steps**:
   ```yaml
   strategy:
     matrix:
       component: [frontend, backend, api]
   ```

#### Cache Miss Issues

**Symptoms**:
- Low cache hit rates
- Inconsistent build times
- Frequent cache invalidation

**Solutions**:
1. **Review cache keys**:
   ```yaml
   key: ${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
   restore-keys: |
     ${{ runner.os }}-
   ```

2. **Implement cache warming**:
   ```yaml
   - name: Warm cache
     run: npm ci --prefer-offline
   ```

#### Resource Constraints

**Symptoms**:
- Out of memory errors
- CPU throttling
- Disk space issues

**Solutions**:
1. **Optimize resource allocation**:
   ```yaml
   runs-on: ubuntu-latest-4-cores
   ```

2. **Clean up artifacts**:
   ```yaml
   - name: Clean up
     run: |
       docker system prune -f
       npm cache clean --force
   ```

#### Test Performance Issues

**Symptoms**:
- Long test execution times
- Flaky tests
- Resource contention

**Solutions**:
1. **Implement test sharding**:
   ```yaml
   strategy:
     matrix:
       shard: [1, 2, 3, 4]
   ```

2. **Optimize test configuration**:
   ```javascript
   // jest.config.js
   module.exports = {
     maxWorkers: '50%',
     testTimeout: 30000,
     setupFilesAfterEnv: ['<rootDir>/test-setup.js']
   };
   ```

---

## 9. Summary

### Key Takeaways

1. **Performance Monitoring**: Continuous monitoring is essential for maintaining optimal pipeline performance

2. **Caching Strategies**: Multi-level caching can significantly reduce build times and resource usage

3. **Parallel Execution**: Intelligent parallelization can dramatically improve overall pipeline speed

4. **Resource Optimization**: Proper resource allocation and management prevents bottlenecks

5. **Build Optimization**: Optimized Docker builds and dependency management reduce overhead

6. **Data-Driven Decisions**: Use metrics and analysis tools to make informed optimization decisions

### Performance Optimization Checklist

- [ ] Implement comprehensive performance monitoring
- [ ] Set up multi-level caching strategy
- [ ] Configure parallel job execution
- [ ] Optimize Docker builds with multi-stage approach
- [ ] Implement intelligent test sharding
- [ ] Monitor resource utilization
- [ ] Set up performance alerting
- [ ] Regular performance analysis and optimization
- [ ] Document performance baselines
- [ ] Continuous improvement process

### Next Steps

1. **Advanced Monitoring**: Implement distributed tracing for complex pipelines
2. **AI-Powered Optimization**: Explore machine learning for predictive optimization
3. **Cost Optimization**: Balance performance with infrastructure costs
4. **Enterprise Patterns**: Learn advanced enterprise CI/CD patterns

---

**Module 19 Complete!** 🎉

You've mastered advanced performance optimization techniques for CI/CD pipelines. These skills will help you build faster, more efficient, and more reliable deployment processes.

**Next**: [Module 20: Enterprise Patterns](./04-enterprise-patterns.md)
# .github/workflows/performance-monitoring.yml
name: Performance Monitoring

on:
  schedule:
    - cron: '0 6 * * *'  # Daily at 6 AM
  workflow_dispatch:

jobs:
  analyze-performance:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install requests boto3 statistics
      
      - name: Analyze Pipeline Performance
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          python scripts/pipeline-analyzer.py \
            --github-token $GITHUB_TOKEN \
            --owner ${{ github.repository_owner }} \
            --repo ${{ github.event.repository.name }} \
            --days 7 \
            --export-cloudwatch
      
      - name: Upload Performance Report
        uses: actions/upload-artifact@v3
        with:
          name: performance-report
          path: performance-report.json
      
      - name: Create Performance Issue
        if: failure()
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            
            try {
              const report = JSON.parse(fs.readFileSync('performance-report.json', 'utf8'));
              
              if (report.summary.success_rate < 90 || report.summary.p95_duration > 1800) {
                await github.rest.issues.create({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  title: '🚨 Pipeline Performance Alert',
                  body: `## Performance Alert\n\n` +
                        `**Success Rate**: ${report.summary.success_rate.toFixed(1)}%\n` +
                        `**P95 Duration**: ${report.summary.p95_duration.toFixed(1)}s\n\n` +
                        `**Bottlenecks**:\n` +
                        report.bottlenecks.map(b => `- ${b.type}: ${b.stage || 'N/A'}`).join('\n') +
                        `\n\n**Recommendations**:\n` +
                        report.recommendations.slice(0, 3).map((r, i) => `${i+1}. ${r}`).join('\n'),
                  labels: ['performance', 'ci-cd']
                });
              }
            } catch (error) {
              console.log('No performance report found or failed to parse');
            }
```

---

## 2. Advanced Caching Strategies

### Multi-Level Caching Implementation

```yaml
# .github/workflows/optimized-build.yml
name: Optimized Build with Advanced Caching

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  CACHE_VERSION: v1
  NODE_VERSION: '18'
  PYTHON_VERSION: '3.11'

jobs:
  cache-dependencies:
    runs-on: ubuntu-latest
    outputs:
      node-cache-hit: ${{ steps.node-cache.outputs.cache-hit }}
      python-cache-hit: ${{ steps.python-cache.outputs.cache-hit }}
      docker-cache-hit: ${{ steps.docker-cache.outputs.cache-hit }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      # Node.js dependency caching with fallback
      - name: Cache Node.js dependencies
        id: node-cache
        uses: actions/cache@v3
        with:
          path: |
            ~/.npm
            node_modules
            .next/cache
          key: ${{ env.CACHE_VERSION }}-node-${{ env.NODE_VERSION }}-${{ hashFiles('**/package-lock.json', '**/yarn.lock') }}
          restore-keys: |
            ${{ env.CACHE_VERSION }}-node-${{ env.NODE_VERSION }}-
            ${{ env.CACHE_VERSION }}-node-
      
      # Python dependency caching
      - name: Cache Python dependencies
        id: python-cache
        uses: actions/cache@v3
        with:
          path: |
            ~/.cache/pip
            .venv
          key: ${{ env.CACHE_VERSION }}-python-${{ env.PYTHON_VERSION }}-${{ hashFiles('**/requirements.txt', '**/Pipfile.lock', '**/poetry.lock') }}
          restore-keys: |
            ${{ env.CACHE_VERSION }}-python-${{ env.PYTHON_VERSION }}-
            ${{ env.CACHE_VERSION }}-python-
      
      # Docker layer caching
      - name: Cache Docker layers
        id: docker-cache
        uses: actions/cache@v3
        with:
          path: /tmp/.buildx-cache
          key: ${{ env.CACHE_VERSION }}-docker-${{ github.sha }}
          restore-keys: |
            ${{ env.CACHE_VERSION }}-docker-
      
      # Build cache for compiled assets
      - name: Cache build artifacts
        uses: actions/cache@v3
        with:
          path: |
            dist
            build
            .next
            target
          key: ${{ env.CACHE_VERSION }}-build-${{ github.sha }}
          restore-keys: |
            ${{ env.CACHE_VERSION }}-build-${{ github.ref_name }}-
            ${{ env.CACHE_VERSION }}-build-
  
  install-dependencies:
    runs-on: ubuntu-latest
    needs: cache-dependencies
    if: needs.cache-dependencies.outputs.node-cache-hit != 'true' || needs.cache-dependencies.outputs.python-cache-hit != 'true'
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        if: needs.cache-dependencies.outputs.node-cache-hit != 'true'
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install Node.js dependencies
        if: needs.cache-dependencies.outputs.node-cache-hit != 'true'
        run: |
          npm ci --prefer-offline --no-audit
      
      - name: Setup Python
        if: needs.cache-dependencies.outputs.python-cache-hit != 'true'
        uses: actions/setup-python@v4
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: 'pip'
      
      - name: Install Python dependencies
        if: needs.cache-dependencies.outputs.python-cache-hit != 'true'
        run: |
          python -m venv .venv
          source .venv/bin/activate
          pip install --upgrade pip
          pip install -r requirements.txt
  
  build:
    runs-on: ubuntu-latest
    needs: [cache-dependencies, install-dependencies]
    if: always() && (needs.cache-dependencies.result == 'success')
    strategy:
      matrix:
        component: [frontend, backend, api]
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Restore Node.js cache
        uses: actions/cache@v3
        with:
          path: |
            ~/.npm
            node_modules
            .next/cache
          key: ${{ env.CACHE_VERSION }}-node-${{ env.NODE_VERSION }}-${{ hashFiles('**/package-lock.json', '**/yarn.lock') }}
      
      - name: Restore build cache
        uses: actions/cache@v3
        with:
          path: |
            dist
            build
            .next
            target
          key: ${{ env.CACHE_VERSION }}-build-${{ github.sha }}
          restore-keys: |
            ${{ env.CACHE_VERSION }}-build-${{ github.ref_name }}-
            ${{ env.CACHE_VERSION }}-build-
      
      - name: Build ${{ matrix.component }}
        run: |
          echo "Building ${{ matrix.component }}..."
          case "${{ matrix.component }}" in
            frontend)
              npm run build:frontend
              ;;
            backend)
              npm run build:backend
              ;;
            api)
              npm run build:api
              ;;
          esac
      
      - name: Cache component build
        uses: actions/cache@v3
        with:
          path: dist/${{ matrix.component }}
          key: ${{ env.CACHE_VERSION }}-${{ matrix.component }}-${{ github.sha }}
  
  test:
    runs-on: ubuntu-latest
    needs: [cache-dependencies, build]
    strategy:
      matrix:
        test-type: [unit, integration, e2e]
        shard: [1, 2, 3, 4]
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Restore dependencies
        uses: actions/cache@v3
        with:
          path: |
            ~/.npm
            node_modules
            .venv
          key: ${{ env.CACHE_VERSION }}-deps-${{ hashFiles('**/package-lock.json', '**/requirements.txt') }}
      
      - name: Cache test results
        uses: actions/cache@v3
        with:
          path: |
            coverage
            test-results
            .nyc_output
          key: ${{ env.CACHE_VERSION }}-tests-${{ matrix.test-type }}-${{ matrix.shard }}-${{ github.sha }}
          restore-keys: |
            ${{ env.CACHE_VERSION }}-tests-${{ matrix.test-type }}-${{ matrix.shard }}-
      
      - name: Run ${{ matrix.test-type }} tests (shard ${{ matrix.shard }})
        run: |
          npm run test:${{ matrix.test-type }} -- --shard=${{ matrix.shard }}/4
      
      - name: Upload test results
        uses: actions/upload-artifact@v3
        with:
          name: test-results-${{ matrix.test-type }}-${{ matrix.shard }}
          path: test-results/
```

### Intelligent Cache Management

```python
# scripts/cache-manager.py
import os
import json
import hashlib
import time
from datetime import datetime, timedelta
from typing import Dict, List, Any, Optional
import requests
import boto3
from pathlib import Path
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class CacheManager:
    def __init__(self, github_token: str, aws_region: str = 'us-west-2'):
        self.github_token = github_token
        self.github_headers = {
            'Authorization': f'token {github_token}',
            'Accept': 'application/vnd.github.v3+json'
        }
        self.s3 = boto3.client('s3', region_name=aws_region)
        self.cloudfront = boto3.client('cloudfront', region_name=aws_region)
        
    def analyze_cache_usage(self, owner: str, repo: str, days: int = 30) -> Dict[str, Any]:
        """Analyze cache usage patterns"""
        logger.info(f"Analyzing cache usage for {owner}/{repo}")
        
        # Get workflow runs
        since = (datetime.now() - timedelta(days=days)).isoformat()
        url = f'https://api.github.com/repos/{owner}/{repo}/actions/runs'
        params = {
            'created': f'>={since}',
            'per_page': 100
        }
        
        response = requests.get(url, headers=self.github_headers, params=params)
        response.raise_for_status()
        
        workflow_runs = response.json()['workflow_runs']
        
        cache_stats = {
            'total_runs': len(workflow_runs),
            'cache_hits': 0,
            'cache_misses': 0,
            'cache_types': {},
            'cache_keys': {},
            'time_saved': 0,
            'storage_used': 0
        }
        
        for run in workflow_runs:
            run_cache_data = self._analyze_run_cache_usage(owner, repo, run['id'])
            
            cache_stats['cache_hits'] += run_cache_data['hits']
            cache_stats['cache_misses'] += run_cache_data['misses']
            cache_stats['time_saved'] += run_cache_data['time_saved']
            
            for cache_type, count in run_cache_data['types'].items():
                cache_stats['cache_types'][cache_type] = cache_stats['cache_types'].get(cache_type, 0) + count
            
            for key, data in run_cache_data['keys'].items():
                if key not in cache_stats['cache_keys']:
                    cache_stats['cache_keys'][key] = {
                        'hits': 0,
                        'misses': 0,
                        'last_used': data['last_used'],
                        'size': data['size']
                    }
                cache_stats['cache_keys'][key]['hits'] += data['hits']
                cache_stats['cache_keys'][key]['misses'] += data['misses']
                if data['last_used'] > cache_stats['cache_keys'][key]['last_used']:
                    cache_stats['cache_keys'][key]['last_used'] = data['last_used']
        
        # Calculate hit rate
        total_requests = cache_stats['cache_hits'] + cache_stats['cache_misses']
        cache_stats['hit_rate'] = (cache_stats['cache_hits'] / total_requests * 100) if total_requests > 0 else 0
        
        return cache_stats
    
    def _analyze_run_cache_usage(self, owner: str, repo: str, run_id: int) -> Dict[str, Any]:
        """Analyze cache usage for a specific run"""
        # Get job logs (simplified - in practice, would parse actual logs)
        jobs_url = f'https://api.github.com/repos/{owner}/{repo}/actions/runs/{run_id}/jobs'
        jobs_response = requests.get(jobs_url, headers=self.github_headers)
        
        if jobs_response.status_code != 200:
            return {'hits': 0, 'misses': 0, 'types': {}, 'keys': {}, 'time_saved': 0}
        
        jobs = jobs_response.json()['jobs']
        
        run_data = {
            'hits': 0,
            'misses': 0,
            'types': {},
            'keys': {},
            'time_saved': 0
        }
        
        for job in jobs:
            for step in job.get('steps', []):
                step_name = step['name'].lower()
                
                # Detect cache operations
                if 'cache' in step_name:
                    if step['conclusion'] == 'success' and 'restore' in step_name:
                        run_data['hits'] += 1
                        run_data['time_saved'] += 30  # Estimated time saved
                        
                        # Determine cache type
                        cache_type = self._determine_cache_type(step_name)
                        run_data['types'][cache_type] = run_data['types'].get(cache_type, 0) + 1
                        
                        # Generate cache key (simplified)
                        cache_key = self._generate_cache_key(step_name, job['name'])
                        run_data['keys'][cache_key] = {
                            'hits': 1,
                            'misses': 0,
                            'last_used': datetime.now().isoformat(),
                            'size': 100  # Estimated size in MB
                        }
                    elif 'cache' in step_name and step['conclusion'] != 'success':
                        run_data['misses'] += 1
        
        return run_data
    
    def _determine_cache_type(self, step_name: str) -> str:
        """Determine cache type from step name"""
        if 'node' in step_name or 'npm' in step_name:
            return 'node_modules'
        elif 'python' in step_name or 'pip' in step_name:
            return 'python_packages'
        elif 'docker' in step_name:
            return 'docker_layers'
        elif 'build' in step_name:
            return 'build_artifacts'
        else:
            return 'other'
    
    def _generate_cache_key(self, step_name: str, job_name: str) -> str:
        """Generate cache key identifier"""
        return hashlib.md5(f"{step_name}-{job_name}".encode()).hexdigest()[:8]
    
    def optimize_cache_strategy(self, cache_stats: Dict[str, Any]) -> Dict[str, Any]:
        """Generate cache optimization recommendations"""
        recommendations = {
            'current_performance': {
                'hit_rate': cache_stats['hit_rate'],
                'time_saved_hours': cache_stats['time_saved'] / 3600,
                'total_requests': cache_stats['cache_hits'] + cache_stats['cache_misses']
            },
            'optimizations': [],
            'cache_cleanup': [],
            'new_cache_opportunities': []
        }
        
        # Analyze hit rates by type
        for cache_type, count in cache_stats['cache_types'].items():
            if count > 10:  # Significant usage
                type_hit_rate = 80  # Simplified calculation
                if type_hit_rate < 70:
                    recommendations['optimizations'].append({
                        'type': cache_type,
                        'current_hit_rate': type_hit_rate,
                        'recommendation': f'Improve {cache_type} cache strategy - consider more specific cache keys'
                    })
        
        # Identify unused caches
        cutoff_date = datetime.now() - timedelta(days=7)
        for key, data in cache_stats['cache_keys'].items():
            last_used = datetime.fromisoformat(data['last_used'])
            if last_used < cutoff_date and data['hits'] < 5:
                recommendations['cache_cleanup'].append({
                    'key': key,
                    'last_used': data['last_used'],
                    'hits': data['hits'],
                    'size_mb': data['size']
                })
        
        # Suggest new cache opportunities
        if cache_stats['hit_rate'] < 80:
            recommendations['new_cache_opportunities'].extend([
                {
                    'type': 'test_results',
                    'description': 'Cache test results for unchanged files',
                    'estimated_improvement': '15-25% faster test execution'
                },
                {
                    'type': 'compiled_assets',
                    'description': 'Cache compiled CSS/JS assets',
                    'estimated_improvement': '30-50% faster build times'
                },
                {
                    'type': 'database_fixtures',
                    'description': 'Cache database fixtures for integration tests',
                    'estimated_improvement': '20-40% faster integration tests'
                }
            ])
        
        return recommendations
    
    def implement_distributed_cache(self, bucket_name: str, distribution_id: Optional[str] = None) -> Dict[str, Any]:
        """Implement distributed cache using S3 and CloudFront"""
        logger.info(f"Setting up distributed cache with S3 bucket: {bucket_name}")
        
        # Create S3 bucket for cache storage
        try:
            self.s3.create_bucket(
                Bucket=bucket_name,
                CreateBucketConfiguration={'LocationConstraint': 'us-west-2'}
            )
        except self.s3.exceptions.BucketAlreadyExists:
            logger.info(f"Bucket {bucket_name} already exists")
        
        # Configure bucket for cache optimization
        self.s3.put_bucket_lifecycle_configuration(
            Bucket=bucket_name,
            LifecycleConfiguration={
                'Rules': [
                    {
                        'ID': 'cache-cleanup',
                        'Status': 'Enabled',
                        'Filter': {'Prefix': 'cache/'},
                        'Expiration': {'Days': 30},
                        'AbortIncompleteMultipartUpload': {'DaysAfterInitiation': 7}
                    },
                    {
                        'ID': 'old-cache-cleanup',
                        'Status': 'Enabled',
                        'Filter': {'Prefix': 'cache/'},
                        'Transitions': [
                            {
                                'Days': 7,
                                'StorageClass': 'STANDARD_IA'
                            },
                            {
                                'Days': 30,
                                'StorageClass': 'GLACIER'
                            }
                        ]
                    }
                ]
            }
        )
        
        # Enable versioning for cache consistency
        self.s3.put_bucket_versioning(
            Bucket=bucket_name,
            VersioningConfiguration={'Status': 'Enabled'}
        )
        
        # Configure CORS for cross-origin access
        self.s3.put_bucket_cors(
            Bucket=bucket_name,
            CORSConfiguration={
                'CORSRules': [
                    {
                        'AllowedHeaders': ['*'],
                        'AllowedMethods': ['GET', 'PUT', 'POST', 'DELETE'],
                        'AllowedOrigins': ['*'],
                        'ExposeHeaders': ['ETag'],
                        'MaxAgeSeconds': 3600
                    }
                ]
            }
        )
        
        cache_config = {
            'bucket_name': bucket_name,
            'cache_prefix': 'cache/',
            'max_cache_size_gb': 100,
            'default_ttl_hours': 24,
            'cleanup_schedule': 'daily'
        }
        
        # Setup CloudFront distribution if requested
        if distribution_id:
            cache_config['cloudfront_distribution'] = distribution_id
            cache_config['cdn_enabled'] = True
        
        return cache_config
    
    def generate_cache_config(self, optimization_data: Dict[str, Any]) -> str:
        """Generate optimized cache configuration"""
        config = {
            'cache_strategy': {
                'version': 'v2',
                'default_ttl': '24h',
                'max_size': '10GB',
                'compression': True
            },
            'cache_keys': {
                'dependencies': {
                    'node_modules': {
                        'key': 'deps-node-${{ hashFiles(\'**/package-lock.json\') }}',
                        'paths': ['node_modules', '~/.npm'],
                        'restore_keys': ['deps-node-']
                    },
                    'python_packages': {
                        'key': 'deps-python-${{ hashFiles(\'**/requirements.txt\') }}',
                        'paths': ['.venv', '~/.cache/pip'],
                        'restore_keys': ['deps-python-']
                    }
                },
                'build_artifacts': {
                    'compiled_assets': {
                        'key': 'build-${{ github.sha }}',
                        'paths': ['dist', 'build', '.next'],
                        'restore_keys': ['build-${{ github.ref_name }}-', 'build-']
                    }
                },
                'test_results': {
                    'coverage': {
                        'key': 'coverage-${{ hashFiles(\'**/*.js\', \'**/*.ts\') }}',
                        'paths': ['coverage', '.nyc_output'],
                        'restore_keys': ['coverage-']
                    }
                }
            },
            'optimization_rules': {
                'parallel_cache_operations': True,
                'cache_compression': True,
                'incremental_caching': True,
                'cache_warming': {
                    'enabled': True,
                    'schedule': 'nightly',
                    'popular_keys_only': True
                }
            }
        }
        
        return json.dumps(config, indent=2)

def main():
    import argparse
    
    parser = argparse.ArgumentParser(description='Cache Manager and Optimizer')
    parser.add_argument('--github-token', required=True, help='GitHub token')
    parser.add_argument('--owner', required=True, help='Repository owner')
    parser.add_argument('--repo', required=True, help='Repository name')
    parser.add_argument('--action', choices=['analyze', 'optimize', 'setup-distributed'], required=True)
    parser.add_argument('--days', type=int, default=30, help='Days to analyze')
    parser.add_argument('--s3-bucket', help='S3 bucket for distributed cache')
    parser.add_argument('--output-file', default='cache-analysis.json', help='Output file')
    
    args = parser.parse_args()
    
    cache_manager = CacheManager(args.github_token)
    
    if args.action == 'analyze':
        print(f"Analyzing cache usage for {args.owner}/{args.repo}...")
        cache_stats = cache_manager.analyze_cache_usage(args.owner, args.repo, args.days)
        
        with open(args.output_file, 'w') as f:
            json.dump(cache_stats, f, indent=2)
        
        print(f"\n{'='*50}")
        print("CACHE ANALYSIS SUMMARY")
        print(f"{'='*50}")
        print(f"Hit Rate: {cache_stats['hit_rate']:.1f}%")
        print(f"Time Saved: {cache_stats['time_saved'] / 3600:.1f} hours")
        print(f"Cache Types: {list(cache_stats['cache_types'].keys())}")
        
    elif args.action == 'optimize':
        print("Analyzing and generating optimization recommendations...")
        cache_stats = cache_manager.analyze_cache_usage(args.owner, args.repo, args.days)
        recommendations = cache_manager.optimize_cache_strategy(cache_stats)
        
        with open('cache-optimization.json', 'w') as f:
            json.dump(recommendations, f, indent=2)
        
        print(f"\nOptimization recommendations:")
        for opt in recommendations['optimizations']:
            print(f"- {opt['type']}: {opt['recommendation']}")
        
        # Generate optimized config
        config = cache_manager.generate_cache_config(recommendations)
        with open('cache-config.json', 'w') as f:
            f.write(config)
        
        print(f"\nOptimized cache configuration saved to cache-config.json")
        
    elif args.action == 'setup-distributed':
        if not args.s3_bucket:
            print("Error: --s3-bucket required for distributed cache setup")
            return
        
        print(f"Setting up distributed cache with S3 bucket: {args.s3_bucket}")
        config = cache_manager.implement_distributed_cache(args.s3_bucket)
        
        with open('distributed-cache-config.json', 'w') as f:
            json.dump(config, f, indent=2)
        
        print("Distributed cache setup complete!")
        print(f"Configuration saved to distributed-cache-config.json")

if __name__ == "__main__":
    main()
```

---

## 3. Parallel Execution Optimization

### Dynamic Job Matrix Generation

```yaml
# .github/workflows/dynamic-parallel-execution.yml
name: Dynamic Parallel Execution

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      frontend-changed: ${{ steps.changes.outputs.frontend }}
      backend-changed: ${{ steps.changes.outputs.backend }}
      api-changed: ${{ steps.changes.outputs.api }}
      docs-changed: ${{ steps.changes.outputs.docs }}
      test-matrix: ${{ steps.test-matrix.outputs.matrix }}
      build-matrix: ${{ steps.build-matrix.outputs.matrix }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Detect changes
        uses: dorny/paths-filter@v2
        id: changes
        with:
          filters: |
            frontend:
              - 'frontend/**'
              - 'packages/ui/**'
            backend:
              - 'backend/**'
              - 'packages/core/**'
            api:
              - 'api/**'
              - 'packages/api/**'
            docs:
              - 'docs/**'
              - '*.md'
      
      - name: Generate test matrix
        id: test-matrix
        run: |
          matrix="[]"
          
          if [[ "${{ steps.changes.outputs.frontend }}" == "true" ]]; then
            matrix=$(echo $matrix | jq '. + [{"component": "frontend", "test-types": ["unit", "integration", "e2e"]}]')
          fi
          
          if [[ "${{ steps.changes.outputs.backend }}" == "true" ]]; then
            matrix=$(echo $matrix | jq '. + [{"component": "backend", "test-types": ["unit", "integration"]}]')
          fi
          
          if [[ "${{ steps.changes.outputs.api }}" == "true" ]]; then
            matrix=$(echo $matrix | jq '. + [{"component": "api", "test-types": ["unit", "integration", "contract"]}]')
          fi
          
          echo "matrix=$matrix" >> $GITHUB_OUTPUT
      
      - name: Generate build matrix
        id: build-matrix
        run: |
          matrix="[]"
          
          if [[ "${{ steps.changes.outputs.frontend }}" == "true" ]]; then
            matrix=$(echo $matrix | jq '. + [{"component": "frontend", "platforms": ["linux", "windows", "macos"]}]')
          fi
          
          if [[ "${{ steps.changes.outputs.backend }}" == "true" ]]; then
            matrix=$(echo $matrix | jq '. + [{"component": "backend", "platforms": ["linux"]}]')
          fi
          
          if [[ "${{ steps.changes.outputs.api }}" == "true" ]]; then
            matrix=$(echo $matrix | jq '. + [{"component": "api", "platforms": ["linux"]}]')
          fi
          
          echo "matrix=$matrix" >> $GITHUB_OUTPUT
  
  parallel-tests:
    runs-on: ubuntu-latest
    needs: detect-changes
    if: needs.detect-changes.outputs.test-matrix != '[]'
    strategy:
      fail-fast: false
      matrix:
        include: ${{ fromJson(needs.detect-changes.outputs.test-matrix) }}
        test-type: ${{ fromJson('["unit", "integration", "e2e", "contract"]') }}
        shard: [1, 2, 3, 4]
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup test environment
        run: |
          echo "Setting up ${{ matrix.component }} for ${{ matrix.test-type }} tests (shard ${{ matrix.shard }})"
      
      - name: Run tests
        if: contains(matrix.test-types, matrix.test-type)
        run: |
          echo "Running ${{ matrix.test-type }} tests for ${{ matrix.component }} (shard ${{ matrix.shard }}/4)"
          # Actual test command would go here
          npm run test:${{ matrix.test-type }} -- --component=${{ matrix.component }} --shard=${{ matrix.shard }}/4
  
  parallel-builds:
    runs-on: ${{ matrix.platform == 'linux' && 'ubuntu-latest' || matrix.platform == 'windows' && 'windows-latest' || 'macos-latest' }}
    needs: detect-changes
    if: needs.detect-changes.outputs.build-matrix != '[]'
    strategy:
      fail-fast: false
      matrix:
        include: ${{ fromJson(needs.detect-changes.outputs.build-matrix) }}
        platform: ${{ fromJson('["linux", "windows", "macos"]') }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Build ${{ matrix.component }} on ${{ matrix.platform }}
        if: contains(matrix.platforms, matrix.platform)
        run: |
          echo "Building ${{ matrix.component }} on ${{ matrix.platform }}"
          # Actual build command would go here
          npm run build:${{ matrix.component }}
  
  resource-optimization:
    runs-on: ubuntu-latest
    needs: [parallel-tests, parallel-builds]
    if: always()
    steps:
      - name: Optimize resource allocation
        run: |
          echo "Analyzing resource usage and optimizing allocation"
          # Resource optimization logic would go here
```

### Intelligent Test Parallelization

```python
# scripts/test-optimizer.py
import json
import time
import statistics
from typing import Dict, List, Any, Optional
from pathlib import Path
import subprocess
import logging
from dataclasses import dataclass
from concurrent.futures import ThreadPoolExecutor, as_completed

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@dataclass
class TestFile:
    path: str
    estimated_duration: float
    dependencies: List[str]
    test_count: int
    last_failure_rate: float
    complexity_score: float

@dataclass
class TestShard:
    id: int
    files: List[TestFile]
    estimated_duration: float
    actual_duration: Optional[float] = None

class TestOptimizer:
    def __init__(self, project_root: str):
        self.project_root = Path(project_root)
        self.test_history_file = self.project_root / '.test-history.json'
        self.test_history = self._load_test_history()
        
    def _load_test_history(self) -> Dict[str, Any]:
        """Load historical test performance data"""
        if self.test_history_file.exists():
            with open(self.test_history_file, 'r') as f:
                return json.load(f)
        return {'test_files': {}, 'shard_history': []}
    
    def _save_test_history(self):
        """Save test performance data"""
        with open(self.test_history_file, 'w') as f:
            json.dump(self.test_history, f, indent=2)
    
    def discover_test_files(self, test_patterns: List[str] = None) -> List[TestFile]:
        """Discover and analyze test files"""
        if test_patterns is None:
            test_patterns = ['**/*test*.js', '**/*spec*.js', '**/*test*.py', '**/*spec*.py']
        
        test_files = []
        
        for pattern in test_patterns:
            for file_path in self.project_root.glob(pattern):
                if file_path.is_file():
                    test_file = self._analyze_test_file(file_path)
                    if test_file:
                        test_files.append(test_file)
        
        logger.info(f"Discovered {len(test_files)} test files")
        return test_files
    
    def _analyze_test_file(self, file_path: Path) -> Optional[TestFile]:
        """Analyze individual test file"""
        relative_path = str(file_path.relative_to(self.project_root))
        
        # Get historical data
        historical_data = self.test_history['test_files'].get(relative_path, {})
        
        try:
            with open(file_path, 'r', encoding='utf-8') as f:
                content = f.read()
            
            # Count tests
            test_count = self._count_tests(content, file_path.suffix)
            
            # Estimate complexity
            complexity_score = self._calculate_complexity(content)
            
            # Get dependencies
            dependencies = self._extract_dependencies(content, file_path.suffix)
            
            # Estimate duration based on historical data and complexity
            estimated_duration = self._estimate_duration(
                historical_data, test_count, complexity_score
            )
            
            # Get failure rate
            last_failure_rate = historical_data.get('failure_rate', 0.1)
            
            return TestFile(
                path=relative_path,
                estimated_duration=estimated_duration,
                dependencies=dependencies,
                test_count=test_count,
                last_failure_rate=last_failure_rate,
                complexity_score=complexity_score
            )
        
        except Exception as e:
            logger.warning(f"Failed to analyze {file_path}: {e}")
            return None
    
    def _count_tests(self, content: str, file_extension: str) -> int:
        """Count number of tests in file"""
        if file_extension in ['.js', '.ts']:
            # Count describe, it, test functions
            import re
            patterns = [r'\bit\s*\(', r'\btest\s*\(', r'\bdescribe\s*\(']
            count = 0
            for pattern in patterns:
                count += len(re.findall(pattern, content))
            return count
        elif file_extension in ['.py']:
            # Count test methods and functions
            import re
            patterns = [r'def\s+test_\w+', r'def\s+test\w+']
            count = 0
            for pattern in patterns:
                count += len(re.findall(pattern, content))
            return count
        return 1  # Default
    
    def _calculate_complexity(self, content: str) -> float:
        """Calculate complexity score based on content analysis"""
        # Simple complexity metrics
        lines = content.split('\n')
        non_empty_lines = [line for line in lines if line.strip()]
        
        complexity_indicators = {
            'async': content.count('async'),
            'await': content.count('await'),
            'setTimeout': content.count('setTimeout'),
            'setInterval': content.count('setInterval'),
            'fetch': content.count('fetch'),
            'axios': content.count('axios'),
            'database': content.count('database') + content.count('db.'),
            'mock': content.count('mock') + content.count('stub'),
            'nested_describes': content.count('describe') - 1 if content.count('describe') > 1 else 0
        }
        
        # Calculate weighted complexity score
        weights = {
            'async': 2.0,
            'await': 1.5,
            'setTimeout': 3.0,
            'setInterval': 3.0,
            'fetch': 2.5,
            'axios': 2.5,
            'database': 4.0,
            'mock': 1.5,
            'nested_describes': 1.2
        }
        
        complexity_score = len(non_empty_lines) * 0.1  # Base complexity
        
        for indicator, count in complexity_indicators.items():
            complexity_score += count * weights.get(indicator, 1.0)
        
        return complexity_score
    
    def _extract_dependencies(self, content: str, file_extension: str) -> List[str]:
        """Extract test dependencies"""
        dependencies = []
        
        if file_extension in ['.js', '.ts']:
            import re
            # Extract imports and requires
            import_patterns = [
                r'import\s+.*?from\s+["\']([^"\']*)["\'']',
                r'require\s*\(\s*["\']([^"\']*)["\'']\s*\)'
            ]
            
            for pattern in import_patterns:
                matches = re.findall(pattern, content)
                dependencies.extend(matches)
        
        elif file_extension in ['.py']:
            import re
            # Extract imports
            import_patterns = [
                r'from\s+(\w+(?:\.\w+)*)\s+import',
                r'import\s+(\w+(?:\.\w+)*)'
            ]
            
            for pattern in import_patterns:
                matches = re.findall(pattern, content)
                dependencies.extend(matches)
        
        return list(set(dependencies))  # Remove duplicates
    
    def _estimate_duration(self, historical_data: Dict[str, Any], 
                          test_count: int, complexity_score: float) -> float:
        """Estimate test duration"""
        if 'avg_duration' in historical_data:
            # Use historical average with complexity adjustment
            base_duration = historical_data['avg_duration']
            complexity_factor = min(complexity_score / 100, 2.0)  # Cap at 2x
            return base_duration * (1 + complexity_factor)
        else:
            # Estimate based on test count and complexity
            base_time_per_test = 0.5  # 500ms per test
            complexity_multiplier = 1 + (complexity_score / 200)
            return test_count * base_time_per_test * complexity_multiplier
    
    def optimize_test_shards(self, test_files: List[TestFile], 
                           target_shard_count: int = 4) -> List[TestShard]:
        """Optimize test distribution across shards"""
        logger.info(f"Optimizing {len(test_files)} test files into {target_shard_count} shards")
        
        # Sort by estimated duration (longest first) for better load balancing
        sorted_files = sorted(test_files, key=lambda f: f.estimated_duration, reverse=True)
        
        # Initialize shards
        shards = [TestShard(id=i+1, files=[], estimated_duration=0.0) 
                 for i in range(target_shard_count)]
        
        # Distribute files using longest processing time first algorithm
        for test_file in sorted_files:
            # Find shard with minimum estimated duration
            min_shard = min(shards, key=lambda s: s.estimated_duration)
            min_shard.files.append(test_file)
            min_shard.estimated_duration += test_file.estimated_duration
        
        # Log shard distribution
        for shard in shards:
            logger.info(f"Shard {shard.id}: {len(shard.files)} files, "
                       f"estimated duration: {shard.estimated_duration:.1f}s")
        
        return shards
    
    def generate_test_commands(self, shards: List[TestShard], 
                             test_framework: str = 'jest') -> Dict[int, str]:
        """Generate test commands for each shard"""
        commands = {}
        
        for shard in shards:
            file_list = ' '.join([f'"{ file.path}"' for file in shard.files])
            
            if test_framework == 'jest':
                commands[shard.id] = f'npx jest {file_list} --maxWorkers=2 --coverage'
            elif test_framework == 'pytest':
                commands[shard.id] = f'python -m pytest {file_list} -n auto --cov'
            elif test_framework == 'mocha':
                commands[shard.id] = f'npx mocha {file_list} --parallel'
            else:
                commands[shard.id] = f'# Custom test command for {file_list}'
        
        return commands
    
    def run_parallel_tests(self, shards: List[TestShard], 
                          test_framework: str = 'jest') -> Dict[int, Dict[str, Any]]:
        """Run tests in parallel and collect results"""
        logger.info(f"Running {len(shards)} test shards in parallel")
        
        commands = self.generate_test_commands(shards, test_framework)
        results = {}
        
        def run_shard(shard: TestShard) -> Dict[str, Any]:
            start_time = time.time()
            command = commands[shard.id]
            
            try:
                result = subprocess.run(
                    command,
                    shell=True,
                    capture_output=True,
                    text=True,
                    cwd=self.project_root,
                    timeout=1800  # 30 minute timeout
                )
                
                end_time = time.time()
                duration = end_time - start_time
                
                return {
                    'shard_id': shard.id,
                    'duration': duration,
                    'exit_code': result.returncode,
                    'stdout': result.stdout,
                    'stderr': result.stderr,
                    'estimated_duration': shard.estimated_duration,
                    'accuracy': abs(duration - shard.estimated_duration) / shard.estimated_duration if shard.estimated_duration > 0 else 0
                }
            
            except subprocess.TimeoutExpired:
                return {
                    'shard_id': shard.id,
                    'duration': 1800,
                    'exit_code': -1,
                    'stdout': '',
                    'stderr': 'Test timeout',
                    'estimated_duration': shard.estimated_duration,
                    'accuracy': 0
                }
            except Exception as e:
                return {
                    'shard_id': shard.id,
                    'duration': 0,
                    'exit_code': -1,
                    'stdout': '',
                    'stderr': str(e),
                    'estimated_duration': shard.estimated_duration,
                    'accuracy': 0
                }
        
        # Run shards in parallel
        with ThreadPoolExecutor(max_workers=len(shards)) as executor:
            future_to_shard = {executor.submit(run_shard, shard): shard for shard in shards}
            
            for future in as_completed(future_to_shard):
                shard = future_to_shard[future]
                try:
                    result = future.result()
                    results[result['shard_id']] = result
                    logger.info(f"Shard {result['shard_id']} completed in {result['duration']:.1f}s")
                except Exception as e:
                    logger.error(f"Shard {shard.id} failed: {e}")
                    results[shard.id] = {
                        'shard_id': shard.id,
                        'duration': 0,
                        'exit_code': -1,
                        'stdout': '',
                        'stderr': str(e),
                        'estimated_duration': shard.estimated_duration,
                        'accuracy': 0
                    }
        
        return results
    
    def update_test_history(self, test_files: List[TestFile], 
                           shard_results: Dict[int, Dict[str, Any]]):
        """Update test history with new performance data"""
        logger.info("Updating test performance history")
        
        # Update individual file performance
        for test_file in test_files:
            file_path = test_file.path
            if file_path not in self.test_history['test_files']:
                self.test_history['test_files'][file_path] = {
                    'runs': [],
                    'avg_duration': 0,
                    'failure_rate': 0
                }
            
            # Find which shard this file was in and get results
            for shard_id, result in shard_results.items():
                if result['exit_code'] == 0:  # Success
                    # Estimate individual file duration (simplified)
                    estimated_file_duration = test_file.estimated_duration
                    
                    self.test_history['test_files'][file_path]['runs'].append({
                        'duration': estimated_file_duration,
                        'timestamp': time.time(),
                        'success': True
                    })
                    break
        
        # Update shard history
        shard_summary = {
            'timestamp': time.time(),
            'shard_count': len(shard_results),
            'total_duration': max([r['duration'] for r in shard_results.values()]),
            'avg_accuracy': statistics.mean([r['accuracy'] for r in shard_results.values()]),
            'success_rate': len([r for r in shard_results.values() if r['exit_code'] == 0]) / len(shard_results)
        }
        
        self.test_history['shard_history'].append(shard_summary)
        
        # Keep only last 100 runs per file
        for file_data in self.test_history['test_files'].values():
            file_data['runs'] = file_data['runs'][-100:]
            
            # Recalculate averages
            if file_data['runs']:
                durations = [run['duration'] for run in file_data['runs']]
                file_data['avg_duration'] = statistics.mean(durations)
                
                failures = [run for run in file_data['runs'] if not run['success']]
                file_data['failure_rate'] = len(failures) / len(file_data['runs'])
        
        # Keep only last 50 shard runs
        self.test_history['shard_history'] = self.test_history['shard_history'][-50:]
        
        self._save_test_history()
    
    def generate_optimization_report(self, shard_results: Dict[int, Dict[str, Any]]) -> Dict[str, Any]:
        """Generate test optimization report"""
        total_duration = max([r['duration'] for r in shard_results.values()])
        estimated_total = max([r['estimated_duration'] for r in shard_results.values()])
        
        accuracy_scores = [r['accuracy'] for r in shard_results.values()]
        avg_accuracy = statistics.mean(accuracy_scores) if accuracy_scores else 0
        
        successful_shards = [r for r in shard_results.values() if r['exit_code'] == 0]
        success_rate = len(successful_shards) / len(shard_results) if shard_results else 0
        
        # Load balancing analysis
        durations = [r['duration'] for r in shard_results.values()]
        load_balance_score = 1 - (statistics.stdev(durations) / statistics.mean(durations)) if len(durations) > 1 and statistics.mean(durations) > 0 else 1
        
        report = {
            'execution_summary': {
                'total_duration': total_duration,
                'estimated_duration': estimated_total,
                'estimation_accuracy': avg_accuracy,
                'success_rate': success_rate * 100,
                'load_balance_score': load_balance_score * 100
            },
            'shard_performance': {
                shard_id: {
                    'duration': result['duration'],
                    'estimated_duration': result['estimated_duration'],
                    'accuracy': result['accuracy'],
                    'status': 'success' if result['exit_code'] == 0 else 'failed'
                }
                for shard_id, result in shard_results.items()
            },
            'optimization_recommendations': self._generate_optimization_recommendations(shard_results),
            'performance_trends': self._analyze_performance_trends()
        }
        
        return report
    
    def _generate_optimization_recommendations(self, shard_results: Dict[int, Dict[str, Any]]) -> List[str]:
        """Generate optimization recommendations"""
        recommendations = []
        
        durations = [r['duration'] for r in shard_results.values()]
        if len(durations) > 1:
            duration_variance = statistics.stdev(durations) / statistics.mean(durations)
            
            if duration_variance > 0.3:  # High variance
                recommendations.append(
                    "High variance in shard durations detected. Consider redistributing tests for better load balancing."
                )
        
        accuracy_scores = [r['accuracy'] for r in shard_results.values()]
        avg_accuracy = statistics.mean(accuracy_scores) if accuracy_scores else 0
        
        if avg_accuracy < 0.8:  # Low estimation accuracy
            recommendations.append(
                "Low estimation accuracy detected. Consider running more test cycles to improve duration predictions."
            )
        
        failed_shards = [r for r in shard_results.values() if r['exit_code'] != 0]
        if failed_shards:
            recommendations.append(
                f"{len(failed_shards)} shard(s) failed. Consider isolating flaky tests or increasing timeout values."
            )
        
        max_duration = max(durations) if durations else 0
        if max_duration > 600:  # More than 10 minutes
            recommendations.append(
                "Long test execution detected. Consider increasing shard count or optimizing slow tests."
            )
        
        return recommendations
    
    def _analyze_performance_trends(self) -> Dict[str, Any]:
        """Analyze performance trends from history"""
        shard_history = self.test_history.get('shard_history', [])
        
        if len(shard_history) < 2:
            return {'trend': 'insufficient_data'}
        
        recent_runs = shard_history[-10:]  # Last 10 runs
        durations = [run['total_duration'] for run in recent_runs]
        
        if len(durations) >= 2:
            trend_slope = (durations[-1] - durations[0]) / len(durations)
            
            if trend_slope > 10:  # Getting slower
                trend = 'degrading'
            elif trend_slope < -10:  # Getting faster
                trend = 'improving'
            else:
                trend = 'stable'
        else:
            trend = 'stable'
        
        return {
            'trend': trend,
            'avg_duration': statistics.mean(durations),
            'duration_change': durations[-1] - durations[0] if len(durations) >= 2 else 0,
            'runs_analyzed': len(durations)
        }

def main():
    import argparse
    
    parser = argparse.ArgumentParser(description='Test Optimizer')
    parser.add_argument('--project-root', default='.', help='Project root directory')
    parser.add_argument('--shard-count', type=int, default=4, help='Number of test shards')
    parser.add_argument('--test-framework', default='jest', choices=['jest', 'pytest', 'mocha'], help='Test framework')
    parser.add_argument('--dry-run', action='store_true', help='Generate commands without running tests')
    parser.add_argument('--output-file', default='test-optimization-report.json', help='Output file')
    
    args = parser.parse_args()
    
    optimizer = TestOptimizer(args.project_root)
    
    # Discover test files
    print("Discovering test files...")
    test_files = optimizer.discover_test_files()
    
    if not test_files:
        print("No test files found")
        return
    
    # Optimize shards
    print(f"Optimizing test distribution into {args.shard_count} shards...")
    shards = optimizer.optimize_test_shards(test_files, args.shard_count)
    
    # Generate commands
    commands = optimizer.generate_test_commands(shards, args.test_framework)
    
    print(f"\n{'='*60}")
    print("TEST SHARD COMMANDS")
    print(f"{'='*60}")
    for shard_id, command in commands.items():
        print(f"Shard {shard_id}: {command}")
    
    if args.dry_run:
        print("\nDry run complete. Commands generated but not executed.")
        return
    
    # Run tests
    print("\nRunning parallel tests...")
    results = optimizer.run_parallel_tests(shards, args.test_framework)
    
    # Update history
    optimizer.update_test_history(test_files, results)
    
    # Generate report
    report = optimizer.generate_optimization_report(results)
    
    with open(args.output_file, 'w') as f:
        json.dump(report, f, indent=2)
    
    # Print summary
    print(f"\n{'='*60}")
    print("TEST OPTIMIZATION SUMMARY")
    print(f"{'='*60}")
    print(f"Total Duration: {report['execution_summary']['total_duration']:.1f}s")
    print(f"Success Rate: {report['execution_summary']['success_rate']:.1f}%")
    print(f"Load Balance Score: {report['execution_summary']['load_balance_score']:.1f}%")
    print(f"Estimation Accuracy: {report['execution_summary']['estimation_accuracy']:.1f}%")
    
    print(f"\nRecommendations:")
    for i, rec in enumerate(report['optimization_recommendations'], 1):
        print(f"{i}. {rec}")
    
    print(f"\nDetailed report saved to: {args.output_file}")

if __name__ == "__main__":
    main()
```