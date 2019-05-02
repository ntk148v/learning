# Prometheus tips

## `avg_over_time`

Make use of the query function `avg_over_time` to discover for what percentage of time a given service was up or down for. Using `avg_over_time` in our alerting rules allows us to negate the issues described above by tracking how a particular time series appears over time rather than just the last known value of that time series. This way gaps in our data resulting from failed scrapes and flaky time series will no longer result in false positives or negatives in our alerting.

For example, rather than having the alerting rule:

```
up{job="jobname"} < 1
```

which would trigger an alert as soon as one scrape failed, we could have:

```
avg_over_time(up{job="jobname"} [5m]) < 0.9
```

which would trigger an alert if the average of `up` was below 0.9 for the last 5 minutes of scrapes.

## Alerting on approach open file limits

```
groups:
- name: example
  rules:
  - alert: ProcessNearFDLimits
    expr: process_open_fds / process_max_fds > 0.8
    for: 10m
```

## Absent alerting for Jobs

It is advised to have alerts on all the targets for a job going away, for example:

```
groups:
- name: example
  rules:
  - alert: MyJobMissing
    expr: absent(up{job="myjob"})
    for: 10m
```

## Absent alerting for Scraped metrics

It can happen that certain subsystems of a target don't always return all metrics that they should. It is possible to detect this situation by noticing that the `up` metric exists, but the metric in question does not. In addition you will want to check that `up` is 1, so that the alert doesn't spuriously fire when the target is down. If you already have down alerts for the job, there's no need to spam yourself with additional ones about missing metrics too.

```
groups:
- name: example
  rules:
  - alert: MyJobMissingMyMetric
    expr: up{job="myjob"} == 1 unless my_metric
    for: 10m
```

## Don't put the value in alert labels

Example:

```
groups:
- name: example
  rules:
  - alert: ExampleAlert
    expr: metric > 10
    for: 5m
    labels:
      severity: page
      value: "{{ $value }}"
    annotations:
      summary: "Instance {{ $labels.instance }}'s metric is too high"
```

It's likely this alert will never fire. The reason is that `labels` include a label called `value`. The effect of this is that each evaluation will see a brand new alert, and treat the previous one as no longer firing. This is the labels of an alert define its identity, and thus the `for` will never be satisfied.

If you want to have the value of the alert expression or anything else that can vary from one evaluation of an alert instance to another, you could use `annotations` instead.

```
groups:
- name: example
  rules:
  - alert: ExampleAlert
    expr: metric > 10
    for: 5m
    labels:
      severity: page
    annotations:
      summary: "Instance {{ $labels.instance }}'s metric is {{ $value }}"
```