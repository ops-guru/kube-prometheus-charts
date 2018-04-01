**Show Overall CPU usage for a server**
```
100 * (1 - avg by(instance)(irate(node_cpu{mode='idle'}[5m])))
```
*Summary:* Often useful to newcomers to Prometheus looking to replicate common host CPU checks. This query ultimately provides an overall metric for CPU usage, per instance. It does this by a calculation based on the `idle` metric of the CPU, working out the overall percentage of the other states for a CPU in a 5 minute window and presenting that data per `instance`.

