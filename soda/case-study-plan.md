# Case study plan

## Configuration

### Tune database configuration in response to dynamic workload

1. Provision database cluster

2. Simulate cluster load

3. Automatically adjust and apply database configuration

    * buffers, caches, isolation levels, etc. (e.g. innodb_buffer_pool_size)

4. Notify operators

## Healing

### Live migration of cluster nodes from network-isolated compute

1. Provision database cluster with anti-affinity locality policy (each node on different compute)

2. Simulate tenant network failure on one compute

    * Cluster is partitioned

    * Node on failed compute loses connection to remaining nodes

    * Remaining nodes lose connection to node on failed compute

3. Live migrate node from failed compute to healthy compute

4. Restore cluster functioning

5. Notify operators

### Live migration of cluster nodes from suboptimal compute

1. Provision database cluster with anti-affinity locality policy (each node on different compute)

2. Simluate non-crticical failures on one compute

    * Indicating that compute may fail in a while due to critical failure

    * React proactively

3. Live migrate nodes from failing compute to healthy compute

4. Notify operators

## Optimization

### Cluster capacity scaling

1. Provision database cluster

2. Start continuously populating data into the cluster

3. Automatically detect insufficient disk space, trigger cluster rolling volume scaling

4. Notify operators

### Cluster compute scaling

1. Provision database cluster

2. Simulate heavy load on cluster

3. Automatically detect insufficient resources, trigger cluster rolling flavor scaling

4. Notify operators

## Protection

