---
title: "Why I Stopped Using vCenter Alarms and Built My Own Monitoring"
date: 2026-02-18
description: "vCenter's built-in alerting is fine until it isn't. Here's how I replaced it with a lightweight stack that actually tells me what matters."
tags: ["VMware", "Monitoring", "Automation", "PowerShell"]
---

vCenter alarms are great for a small environment. You set thresholds on CPU, memory, and datastore usage, and you get emails when something crosses a line. Simple.

But at enterprise scale, vCenter alarms become noise machines. You end up with thousands of alerts, most of which are either false positives, already known issues, or symptoms of a root cause that the alarm system can't identify.

After years of tuning vCenter alarms and still drowning in noise, I built something better.

## The Problem with Threshold-Based Alerting

Traditional monitoring asks: "Is this metric above X?" That's the wrong question. The right question is: "Is this metric behaving abnormally compared to its own baseline?"

A host running at 85% CPU might be perfectly fine if it always runs at 85% CPU. A host running at 45% CPU might be a problem if it normally runs at 20%. Threshold-based alerting can't tell the difference.

## The Stack

Nothing exotic here. The goal was lightweight, maintainable, and something that could run without a dedicated monitoring team:

- **PowerShell scripts** pulling metrics from the vSphere API on a schedule
- **InfluxDB** for time-series storage (surprisingly lightweight for the amount of data)
- **Grafana** for dashboards and alerting
- **Custom anomaly detection** using rolling averages and standard deviations

The PowerShell piece runs on a simple Windows server with scheduled tasks. Every 5 minutes, it pulls performance data from every host, VM, and datastore in the environment and writes it to InfluxDB.

## Smart Alerting Rules

Instead of static thresholds, the alerting rules compare current values to rolling baselines:

- Alert if a host's CPU usage is more than 2 standard deviations above its 7-day rolling average
- Alert if a datastore's growth rate suggests it will fill in less than 30 days (not when it crosses 85%)
- Alert if vMotion events spike beyond the normal pattern (usually indicates a host issue triggering DRS)
- Alert if VM snapshot age exceeds 72 hours (one of the most common causes of datastore issues)

## The Result

Alert volume dropped by about 90%. The alerts we do get are actionable — they tell us something is genuinely different, not just that a metric crossed an arbitrary line.

The dashboards also became a conversation tool. In our weekly infrastructure review, we pull up Grafana and can immediately see trends, capacity projections, and performance patterns that would be invisible in vCenter's built-in views.

---

I'll share the PowerShell collection scripts and Grafana dashboard templates in a follow-up post. The whole setup took about two weeks to build and has been running reliably for over a year with minimal maintenance.
