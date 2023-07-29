# Configuration Guide
This guidance describes the environmental variables and hyperparameters needed to launch each optimization technique in our GeoMX system.

## Synchronization Algorithms
GeoMX currently supports two fundamental synchronization algorithms, i.e., the fully-synchronous algorithm, the mixed-synchronous algorithm, and an advanced algorithm, i.e., hierarchical frequency aggregation.

### Fully-Synchronous Algorithm
Fully-Synchronous Algorithm (FSA) is the default strategy for model synchronization. In this synchronous algorithm, training nodes synchronize their model data (can be parameters or gradients) each round, and both parameter server systems within and between data centers run in a synchronous parallel mode. All training nodes are synchronized to ensure a consistent model. However, this comes at the expense of training speed, as it requires waiting for all computations and communications to complete at every iteration.

To use FSA, all that's required is to set `dist_sync` as a hyperparameter during the initialization of `kvstore`. For example:

```python
import mxnet as mx

# Initialize distributed kvstore in synchronous mode.
kvstore_dist = mx.kv.create("dist_sync")

# Master worker sets the optimizer of the global parameter server to Adam.
if kvstore_dist.is_master_worker:
    kvstore_dist.set_optimizer(mx.optimizer.Adam(learning_rate=lr))

for epoch in range(num_epochs):
    for _, batch in enumerate(train_iter):
        # Perform forward and backward propagation to calculate gradients.
        ...
        # Synchronize gradients to obtain updated parameters.
        for idx, param in enumerate(net_params):
            if param.grad_req == "null": continue
            kvstore_dist.push(idx, param.grad(), priority=-idx)
            kvstore_dist.pull(idx, param.data(), priority=-idx)
```

The demo code can be found in [`examples/cnn.py`](https://github.com/INET-RC/GeoMX/blob/main/examples/cnn.py). You can run this demo by simply `bash scripts/xpu/run_vanilla_hips.sh`, where `xpu` should be `cpu` or `gpu`.

### Mixed-Synchronous Algorithm
Mixed-Synchronous Algorithm (MixedSync) is an asynchronous version of FSA, where the difference is that the parameter server system between data centers runs in an asynchronous parallel mode. This setup is particularly suitable for scenarios where intra-data center training nodes display high homogeneity, yet there is significant resource heterogeneity between different data centers. This asynchronous method resolves the problem of straggling data centers, thereby accelerating distributed training across WANs.

To use MixedSync, all that's required is to set `dist_async` instead of `dist_sync` when initializing `kvstore`. The rest of the setup remains the same:

```python
import mxnet as mx

kvstore_dist = mx.kv.create("dist_async")
```

Alternatively, you can enable MixedSync by using the `--mixed-sync` option in our provided python script:

```shell
python examples/cnn.py --mixed-sync
```

You can also run this demo by executing `bash scripts/xpu/run_mixed_sync.sh`, where `xpu` should be `cpu` or `gpu`.

### Use DCASGD Optimizer in MixedSync
To alleviate the issue of stale gradients in asynchronous parallel operations, the global parameter server can be configured to use the [DCASGD](http://proceedings.mlr.press/v70/zheng17b/zheng17b.pdf) optimizer. This adjustment aids in improving training convergence while preserving model accuracy.

The way to enable DCASGD in MixedSync is the same as in MXNET: simply replace the `Adam` optimizer with the `DCASGD` optimizer:

```python
import mxnet as mx

kvstore_dist = mx.kv.create("dist_async")
if kvstore_dist.is_master_worker:
    kvstore_dist.set_optimizer(mx.optimizer.DCASGD(learning_rate=lr))
```

We can use the following command to enable the DCASGD optimizer in MixedSync:

```shell
python examples/cnn.py --mixed-sync --dcasgd
```

Just modify `scripts/xpu/run_mixed_sync.sh` and try it!

### Hierarchical Frequency Aggregation
Inspired by [this paper](https://ieeexplore.ieee.org/abstract/document/9148862), our Hierarchical Frequency Aggregation (HFA) algorithm first performs $K_1$ steps of local updates at the training nodes, followed by $K_2$ steps of synchronizations at the local parameter server. Finally, a global synchronization is performed at the global parameter server. This approach effectively reduces the frequency of model synchronization across data centers, thereby boosting distributed training.

To enable HFA, we initialize `kvstore` in `dist_sync` mode and make a simple modification to the training loop:

```python
import mxnet as mx

# Initialize distributed kvstore in synchronous mode.
kvstore_dist = mx.kv.create("dist_sync")

# Obtain K1 from environmental variables.
period_k1 = int(os.getenv('MXNET_KVSTORE_HFA_K1'))

# Obtain the number of training nodes in each data center.
num_local_workers = kvstore_dist.num_workers

# Define local trainer to use Adam optimizer.
optimizer = mx.optimizer.Adam(learning_rate=lr)
trainer = Trainer(net.collect_params(), optimizer=optimizer)

global_iters = 1
for epoch in range(num_epochs):
    for _, batch in enumerate(train_iter):
        # Perform forward and backward propagation to calculate gradients.
        ...
        # Update local model parameters.
        trainer.step(num_samples)
        # Synchronize model parameters every K1 round.
        if global_iters % period_k1 == 0:
            for idx, param in enumerate(net_params):
                kvstore_dist.push(idx, param.data() / num_local_workers, priority=-idx)
                kvstore_dist.pull(idx, param.data(), priority=-idx)
        # Update the iteration counter
        global_iters += 1
```

Then, let's set three environmental variables:

```shell
MXNET_KVSTORE_USE_HFA = 1  # whether HFA is enabled
MXNET_KVSTORE_HFA_K1 = 20  # number of loops before a local synchronization
MXNET_KVSTORE_HFA_K2 = 10  # number of loops before a global synchronization
```

The demo code can be found in [`examples/cnn_hfa.py`](https://github.com/INET-RC/GeoMX/blob/main/examples/cnn_hfa.py). You can run this demo by simply `bash scripts/xpu/run_hfa_sync.sh`, where `xpu` should be `cpu` or `gpu`.

## Communication Optimization Techniques
Given the often limited and varied network conditions in WANs, distributed training across data centers can potentially create communication bottlenecks. To mitigate these issues, GeoMX employs a variety of optimization techniques. These include gradient sparsification, mixed-precision quantization, advanced transmission protocols, synchronization algorithms, flow scheduling, and priority scheduling, among others (e.g., overlay scheduling, currently in development). These techniques comprehensively tackle communication issues, further enhancing the efficiency and robustness of distributed machine learning training in GeoMX.

> NOTE: Please try these optimization techniques in the default fully-synchronous mode.

### Bidirectional Gradient Sparsification
Traditional approaches such as [Deep Gradient Compression](https://arxiv.org/pdf/1712.01887.pdf) sparsify the pushed gradient tensors. For further compression, we also sparsify the pulled (aggregated) gradient tensors rather than pulling full parameters. This technique is enabled between the global parameter server and the intra-domain parameter servers of different data centers. (Refer to [this paper](https://www.zte.com.cn/content/dam/zte-site/res-www-zte-com-cn/mediares/magazine/publication/com_cn/article/202005/cn202005004.pdf) for more details.)

To enable bidirectional gradient sparsification, define it in `kvstore_dist.set_gradient_compression` and set the compression ratio:

```python
import mxnet as mx

# Initialize distributed kvstore in synchronous mode.
kvstore_dist = mx.kv.create("dist_sync")

# Obtain the total number of training nodes.
num_all_workers = kvstore_dist.num_all_workers

# Master worker enables bidirectional gradient sparsification on the global parameter server.
if kvstore_dist.is_master_worker:
    kvstore_dist.set_gradient_compression({"type": "bsc", "threshold": 0.01})

# Define local trainer to use Adam optimizer.
optimizer = mx.optimizer.Adam(learning_rate=lr)
trainer = Trainer(net.collect_params(), optimizer=optimizer)

for epoch in range(num_epochs):
    for _, batch in enumerate(train_iter):
        # Perform forward and backward propagation to calculate gradients.
        ...
        # Synchronize gradients for gradient aggregation.
        for idx, param in enumerate(net_params):
            if param.grad_req == "null": continue
            kvstore_dist.push(idx, param.grad(), priority=-idx)
            kvstore_dist.pull(idx, param.grad(), priority=-idx)
        # Use aggregated gradients to update local model parameters.
        trainer.step(num_all_workers * batch_size)
        # Put gradients to zero manually.
        for param in net_params:
            param.zero_grad()
```

Note that gradient tensors are classified into large and tiny tensors based on their size, and only the large tensors will be sparsified for transmission. The threshold for classifying large and tiny tensors can be set through the environmental variable `MXNET_KVSTORE_SIZE_LOWER_BOUND`. For example:

```shell
MXNET_KVSTORE_SIZE_LOWER_BOUND = 200000
```

The demo code can be found in [`examples/cnn_bsc.py`](https://github.com/INET-RC/GeoMX/blob/main/examples/cnn_bsc.py). You can run this demo by simply `bash scripts/xpu/run_bisparse_compression.sh`, where `xpu` should be `cpu` or `gpu`.

### Mixed-Precision Quantization
This technique quantifies the parameter and gradient tensors set for transmission into FP16 format, which effectively halves the data traffic volume over both LANs and WANs. However, if bidirectional gradient sparsification is enabled, the communication between the intra-domain parameter servers and the global parameter server remains in FP32 format. This precaution is taken to minimize the loss of crucial information and avoid significant degradation to model performance.

### Differential Gradient Transmission
This advanced transmission protocol is optimized for distributed machine learning tasks. Leveraging the tolerance of gradient descent algorithms towards partial parameter loss, this protocol transfers gradients across multiple channels, each with distinct levels of reliability and priority, contingent on their respective contributions to model convergence. Through these prioritized channels, critical gradients receive precedence in transmission, while other non-important gradients are transmitted with lower priority and reliability. This helps to reduce tail latency and thus reduce the end-to-end transmission delay of parameter synchronization. (Refer to [this paper](https://drive.google.com/file/d/1IbmpFybX_qXZM2g_8BrcD9IF080qci94/view) for more details and [this repo](https://github.com/zhouhuaman/dgt) for individual use.)

Only a few Environment Variables are needed here.

```shell
ENABLE_DGT=0 # whether to enable DGT. Attention, 2 is for enabled, not 1
DMLC_UDP_CHANNEL_NUM=3 # Channels
DMLC_K=0.8 # compression ratio
ADAPTIVE_K_FLAG=1 # set K adaptively
```

### TSEngine

Only a few Environment Variables are needed here.

```shell
ENABLE_INTER_TS=0 # whether to enable locally
ENABLE_INTRA_TS=0 # whether to enable globally
MAX_GREED_RATE_TS=0.9
```

### P3

Only an Environment Variables are needed here.

```shell
ENABLE_P3=0 # whether to enable P3
```