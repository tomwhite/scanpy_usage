# Visualizing and Clustering 1.3M neurons, with optimizations

## Speeding up Nearest Neighbors and UMAP with CPU parallelization

We will use a machine on Google Cloud Platform, since it makes it easy
to get a GPU-enabled machine with all the software pre-configured.
(See https://cloud.google.com/deep-learning-vm/docs/images.)

You may wish to change the zone specified here:

```bash
export IMAGE_FAMILY="ubuntu-1804-lts"
export ZONE="us-central1-a"
export INSTANCE_NAME="$USER-scanpy"

gcloud compute instances create $INSTANCE_NAME \
  --zone=$ZONE \
  --image-family=$IMAGE_FAMILY \
  --image-project=ubuntu-os-cloud \
  --machine-type=n1-standard-16 \
  --subnet=default \
  --network-tier=PREMIUM \
  --maintenance-policy=TERMINATE
```

Connect to the instance with the following command. Note that it may
take a few minutes before it is ready to accept connections.

```bash
gcloud compute ssh --zone $ZONE $INSTANCE_NAME
```

Install Python and other packages:

```bash
sudo apt-get update && sudo apt-get install -y git python3-pip python3-tk
```

Install louvain:

```bash
sudo apt-get install -y libz-dev libxml2-dev
pip3 install louvain # takes a while to install from source
```

Install scanpy that uses pynndescent as a dependency (not released). Also use a umap optimization.

```bash
# TODO: installation is failing...
git clone https://github.com/tomwhite/scanpy
(cd scanpy; mkdir data; git checkout -b pynndescent-dependency-threaded origin/pynndescent-dependency-threaded; pip3 install -e .)

pip3 uninstall -y umap-learn
git clone https://github.com/tomwhite/umap
(cd umap; git checkout -b embedding_optimization_joblib origin/embedding_optimization_joblib; pip3 install -e .)
```

Checkout this repo in the instance to give access to the scripts.

```bash
git clone https://github.com/tomwhite/scanpy_usage
(cd scanpy_usage; git checkout -b gpu-optimization origin/gpu-optimization)
cd scanpy_usage/1908_umap_parallel
```

Copy test data to the instance. Note that you will have to change the user
(`-u`) to be your GCP project since the data is in a requester pays bucket.

```bash
gsutil -u hca-scale cp gs://ll-sc-data/10x/1M_neurons_filtered_gene_bc_matrices_h5.h5 1M_neurons_filtered_gene_bc_matrices_h5.h5
```

Run the analysis (optimized):

```bash
python3 cluster_130K_opt.py 1M_neurons_filtered_gene_bc_matrices_h5.h5
```

Install regular scanpy:

```bash
pip3 uninstall -y scanpy umap-learn
pip3 install scanpy==1.4.4.post1
```

Run the analysis (regular):

```bash
python3 cluster_130K.py 1M_neurons_filtered_gene_bc_matrices_h5.h5
```

_Summary: unoptimized UMAP takes 3:39 minutes, optimized takes
1:24 minutes._

To view the figures, run the following from your local machine to copy
them locally:

```bash
gcloud compute scp --zone $ZONE --recurse $INSTANCE_NAME:scanpy_usage/1908_umap_parallel/figures figures
```

## 1M cells

_Note_: this currently fails due to out of memory errors (even with `n1-highmem-16`).

Run the analysis (optimized):

```bash
python3 cluster_opt.py 1M_neurons_filtered_gene_bc_matrices_h5.h5
```

Run the analysis (regular):

```bash
python3 cluster.py 1M_neurons_filtered_gene_bc_matrices_h5.h5
```

## Finishing up

```bash
gcloud compute instances delete --zone $ZONE --quiet $INSTANCE_NAME
```