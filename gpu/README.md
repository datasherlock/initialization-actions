# GPU driver installation

GPUs require special drivers and software which are not pre-installed on
[Dataproc](https://cloud.google.com/dataproc) clusters by default.
This initialization action installs GPU driver for NVIDIA GPUs on master and
worker nodes in a Dataproc cluster.

## Using this initialization action

**:warning: NOTICE:** See
[best practices](/README.md#how-initialization-actions-are-used) of using
initialization actions in production.

You can use this initialization action to create a new Dataproc cluster with GPU
support - it will install NVIDIA GPU drivers and CUDA on cluster nodes with
attached GPU adapters.

1.  Use the `gcloud` command to create a new cluster with NVIDIA-provided GPU
    drivers and CUDA installed by initialization action.

    ```bash
    REGION=<region>
    CLUSTER_NAME=<cluster_name>
    gcloud dataproc clusters create ${CLUSTER_NAME} \
        --region ${REGION} \
        --master-accelerator type=nvidia-tesla-v100 \
        --worker-accelerator type=nvidia-tesla-v100,count=4 \
        --initialization-actions gs://goog-dataproc-initialization-actions-${REGION}/gpu/install_gpu_driver.sh
    ```

1.  Use the `gcloud` command to create a new cluster with NVIDIA GPU drivers
    and CUDA installed by initialization action as well as the GPU
    monitoring service. The monitoring service is supported on Dataproc 2.0+ Debian
    and Ubuntu images. Please create a Github issue if support is needed for other
    Dataproc images.

    *Prerequisite:* Create GPU metrics in
    [Cloud Monitoring](https://cloud.google.com/monitoring/docs/) using Google
    Cloud Shell with the
    [create_gpu_metrics.py](https://github.com/GoogleCloudPlatform/ml-on-gcp/blob/master/dlvm/gcp-gpu-utilization-metrics/create_gpu_metrics.py)
    script.

    If you run this script locally you will need to set up a service account.

    ```bash
    export GOOGLE_CLOUD_PROJECT=<project-id>

    git clone https://github.com/GoogleCloudPlatform/ml-on-gcp.git
    cd ml-on-gcp/dlvm/gcp-gpu-utilization-metrics
    pip install -r ./requirements.txt
    python create_gpu_metrics.py
    ```

    Expected output:

    ```
    Created projects/project-sample/metricDescriptors/custom.googleapis.com/utilization_memory.
    Created projects/project-sample/metricDescriptors/custom.googleapis.com/utilization_gpu.
    Created projects/project-sample/metricDescriptors/custom.googleapis.com/memory_used
    ```

    Create cluster:

    ```bash
    REGION=<region>
    CLUSTER_NAME=<cluster_name>
    gcloud dataproc clusters create ${CLUSTER_NAME} \
        --region ${REGION} \
        --master-accelerator type=nvidia-tesla-v100 \
        --worker-accelerator type=nvidia-tesla-v100,count=4 \
        --initialization-actions gs://goog-dataproc-initialization-actions-${REGION}/gpu/install_gpu_driver.sh \
        --metadata install-gpu-agent=true \
        --scopes https://www.googleapis.com/auth/monitoring.write
    ```

1.  Use the `gcloud` command to create a new cluster using Multi-Instance GPU (MIG) feature of the
    NVIDIA Ampere architecture. This creates a cluster with the NVIDIA GPU drivers
    and CUDA installed and the Ampere based GPU configured for MIG.

    After cluster creation each MIG instance will show up like a regular GPU to YARN. For instance, if you requested
    2 workers each with 1 A100 and used the default 2 MIG instances per A100, the cluster would have a total of 4 GPUs
    that can be allocated.

    It is important to note that CUDA 11 only supports enumeration of a single MIG instance. It is recommended that you
    only request a single MIG instance per container. For instance, if running Spark only request
    1 GPU per executor (spark.executor.resource.gpu.amount=1). Please see the
    [MIG user guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/) for more information.

    First decide which Amphere based GPU you are using. In the example we use the A100.
    Decide the number of MIG instances and [instance profiles to use](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/#lgi).
    By default if the MIG profiles are not specified it will configure 2 MIG instances with profile id 9. If
    a different instance profile is required, you can specify it in the MIG_CGI metadata parameter. Either a
    profile id or the name (ie 3g.20gb) can be specified. For example:

    ```bash
        --metadata=^:^MIG_CGI='3g.20gb,9'
    ```

    Create cluster with MIG enabled:

    ```bash
    REGION=<region>
    CLUSTER_NAME=<cluster_name>
    gcloud dataproc clusters create ${CLUSTER_NAME} \
        --region ${REGION} \
        --worker-machine-type a2-highgpu-1g
        --worker-accelerator type=nvidia-tesla-a100,count=1 \
        --initialization-actions gs://goog-dataproc-initialization-actions-${REGION}/gpu/install_gpu_driver.sh \
        --metadata=startup-script-url=gs://goog-dataproc-initialization-actions-${REGION}/gpu/mig.sh
    ```

#### GPU Scheduling in YARN:

YARN is the default Resource Manager for Dataproc. To use GPU scheduling feature
in Spark, it requires YARN version >= 2.10 or >=3.1.1. If intended to use Spark
with Deep Learning use case, it recommended to use YARN >= 3.1.3 to get support
for [nvidia-docker version 2](https://github.com/NVIDIA/nvidia-docker).

In current Dataproc set up, we enable GPU resource isolation by initialization
script without NVIDIA Docker, you can find more information at
[NVIDIA Spark RAPIDS getting started guide](https://nvidia.github.io/spark-rapids/).

#### cuDNN

You can also install [cuDNN](https://developer.nvidia.com/CUDNN) on your
cluster. cuDNN is used as a backend for Deep Learning frameworks, such as
TensorFlow. To select a version, include the metadata parameter `--metadata
cudnn-version=x.x.x.x`. You can find the list of archived versions
[here](https://developer.nvidia.com/rdp/cudnn-archive) which includes all
versions except the latest. To locate the version you need, click on Download
option for the correct cuDNN + CUDA version you desire, copy the link address
for the `cuDNN Runtime Library for Ubuntu18.04 x86_64 (Deb)` file of the
matching CUDA version and find the full version from the deb file. For instance,
for `libcudnn8_8.0.4.30-1+cuda11.0_amd64.deb`, the version is `8.0.4.30`. Below
is a table for mapping some recent major.minor cuDNN versions to full versions
and compatible CUDA versions:

Major.Minor | Full Version | CUDA Versions
----------- | ------------ | --------------------------
8.1         | 8.1.1.33     | 10.2, 11.0, 11.1, 11.2
8.0         | 8.0.5.39     | 10.1, 10.2, 11.0, 11.0, 11.1
7.6         | 7.6.5.32     | 9.0, 9.2, 10.0, 10.1, 10.2
7.5         | 7.5.1.10     | 9.0, 9.2, 10.0, 10.1

To figure out which version you need, refer to the framework's documentation,
sometimes found in the "building from source" sections.
[Here](https://www.tensorflow.org/install/source#gpu) is TensorFlow's.

#### Metadata parameters:

-   `install-gpu-agent: true|false` - this is an optional parameter with
    case-sensitive value. Default is `false`.

    **Note:** This parameter will collect GPU utilization and send statistics to
    Stackdriver. Make sure you add the correct scope to access Stackdriver.

-   `gpu-driver-url: <URL>` - this is an optional parameter for customizing
    NVIDIA-provided GPU driver on Debian.

-   `cuda-url: <URL>` - this is an optional parameter for customizing
    NVIDIA-provided CUDA on Debian. This is required if not using CUDA `10.1` or
    `10.2` with a Debian image. Please find the appropriate linux-based
    runtime-file URL [here](https://developer.nvidia.com/cuda-toolkit-archive).

-   `cuda-version: 10.1|10.2|<VERSION>` - this is an optional parameter for
    customizing NVIDIA-provided CUDA version. Default is `10.2`.

-   `cudnn-version: <VERSION>` - this is an optional parameter for installing
    [NVIDIA cuDNN](https://developer.nvidia.com/CUDNN) version `x.x.x.x`.
    There is no default value.

#### Verification

1.  Once the cluster has been created, you can access the Dataproc cluster and
    verify NVIDIA drivers are install successfully.

    ```bash
    sudo nvidia-smi
    ```

2.  If you install the GPU collection service, verify installation by using the
    following command:

    ```bash
    sudo systemctl status gpu-utilization-agent.service
    ```

For more information about GPU support, take a look at
[Dataproc documentation](https://cloud.google.com/dataproc/docs/concepts/compute/gpus)

### Report metrics

The initialization action installs a
[monitoring agent](https://github.com/GoogleCloudPlatform/ml-on-gcp/tree/master/dlvm/gcp-gpu-utilization-metrics)
that monitors the GPU usage on the instance. This will auto create and send the
GPU metrics to the Cloud Monitoring service.

### Troubleshooting

Problem: Error when running `report_gpu_metrics`

```
google.api_core.exceptions.InvalidArgument: 400 One or more TimeSeries could not be written:
One or more points were written more frequently than the maximum sampling period configured for the metric.
:timeSeries[0]
```

Solution: Verify service is running in background

```bash
sudo systemctl status gpu-utilization-agent.service
```

## Important notes

*   This initialization script will install NVIDIA GPU drivers in all nodes in
    which a GPU is detected.
