# Developer Guide

## Steps to run Distributed Tensorflow

All the docker image related artifacts are located under - `oci_dist_training_artifacts/tensorflow/v1/`


### 1. Prepare Docker Image
The instruction assumes that you are running this within the folder where you ran `ads opctl distributed-training init --framework tensorflow`

All files in the current directory is copied over to `/code` folder inside docker image.

For example, you can have the following training Tensorflow script saved as `mnist.py`:

```
# Script adapted from tensorflow tutorial: https://www.tensorflow.org/tutorials/distribute/multi_worker_with_keras

# ==============================================================================

import tensorflow as tf
import tensorflow_datasets as tfds
import os
import sys
import time
import ads
from ocifs import OCIFileSystem
from tensorflow.data.experimental import AutoShardPolicy

BUFFER_SIZE = 10000
BATCH_SIZE_PER_REPLICA = 64

if '.' not in sys.path:
    sys.path.insert(0, '.')


def create_dir(dir):
    if not os.path.exists(dir):
        os.makedirs(dir)


def create_dirs(task_type="worker", task_id=0):
    artifacts_dir = os.environ.get("OCI__SYNC_DIR", "/opt/ml")
    model_dir = artifacts_dir + "/model"
    print("creating dirs for Model: ", model_dir)
    create_dir(model_dir)
    checkpoint_dir = write_filepath(artifacts_dir, task_type, task_id)
    return artifacts_dir, checkpoint_dir, model_dir

def write_filepath(artifacts_dir, task_type, task_id):
    if task_type == None:
        task_type = "worker"
    checkpoint_dir = artifacts_dir + "/checkpoints/" + task_type + "/" + str(task_id)
    print("creating dirs for Checkpoints: ", checkpoint_dir)
    create_dir(checkpoint_dir)
    return checkpoint_dir


def scale(image, label):
    image = tf.cast(image, tf.float32)
    image /= 255
    return image, label


def get_data(data_bckt=None, data_dir="/code/data", num_replicas=1, num_workers=1):
    if data_bckt is not None and not os.path.exists(data_dir + '/mnist'):
        print(f"downloading data from {data_bckt}")
        ads.set_auth(os.environ.get("OCI_IAM_TYPE", "resource_principal"))
        authinfo = ads.common.auth.default_signer()
        oci_filesystem = OCIFileSystem(**authinfo)
        lck_file = os.path.join(data_dir, '.lck')
        if not os.path.exists(lck_file):
            os.makedirs(os.path.dirname(lck_file), exist_ok=True)
            open(lck_file, 'w').close()
            oci_filesystem.download(data_bckt, data_dir, recursive=True)
        else:
            print(f"data downloaded by a different process. waiting")
            time.sleep(30)

    BATCH_SIZE = BATCH_SIZE_PER_REPLICA * num_replicas * num_workers
    print("Now printing data_dir:", data_dir)
    datasets, info = tfds.load(name='mnist', with_info=True, as_supervised=True, data_dir=data_dir)
    mnist_train, mnist_test = datasets['train'], datasets['test']
    print("num_train_examples :", info.splits['train'].num_examples, " num_test_examples: ",
          info.splits['test'].num_examples)

    train_dataset = mnist_train.map(scale).cache().shuffle(BUFFER_SIZE).batch(BATCH_SIZE)
    test_dataset = mnist_test.map(scale).batch(BATCH_SIZE)
    train = shard(train_dataset)
    test = shard(test_dataset)
    return train, test, info


def shard(dataset):
    options = tf.data.Options()
    options.experimental_distribute.auto_shard_policy = AutoShardPolicy.DATA
    return dataset.with_options(options)


def decay(epoch):
    if epoch < 3:
        return 1e-3
    elif epoch >= 3 and epoch < 7:
        return 1e-4
    else:
        return 1e-5


def get_callbacks(model, checkpoint_dir="/opt/ml/checkpoints"):
    checkpoint_prefix = os.path.join(checkpoint_dir, "ckpt_{epoch}")

    class PrintLR(tf.keras.callbacks.Callback):
        def on_epoch_end(self, epoch, logs=None):
            print('\nLearning rate for epoch {} is {}'.format(epoch + 1, model.optimizer.lr.numpy()), flush=True)

    callbacks = [
        tf.keras.callbacks.TensorBoard(log_dir='./logs'),
        tf.keras.callbacks.ModelCheckpoint(filepath=checkpoint_prefix,
                                           # save_weights_only=True
                                           ),
        tf.keras.callbacks.LearningRateScheduler(decay),
        PrintLR()
    ]
    return callbacks


def build_and_compile_cnn_model():
    print("TF_CONFIG in model:", os.environ.get("TF_CONFIG"))
    model = tf.keras.Sequential([
        tf.keras.layers.Conv2D(32, 3, activation='relu', input_shape=(28, 28, 1)),
        tf.keras.layers.MaxPooling2D(),
        tf.keras.layers.Flatten(),
        tf.keras.layers.Dense(64, activation='relu'),
        tf.keras.layers.Dense(10)
    ])

    model.compile(loss=tf.keras.losses.SparseCategoricalCrossentropy(from_logits=True),
                  optimizer=tf.keras.optimizers.Adam(),
                  metrics=['accuracy'])
    return model
```

And, save the following script as `train.py`

```
import tensorflow as tf
import argparse
import mnist


print(tf.__version__)

parser = argparse.ArgumentParser(description='Tensorflow Native MNIST Example')
parser.add_argument('--data-dir',
                    help='location of the training dataset in the local filesystem (will be downloaded if needed)',
                    default='/code/data')
parser.add_argument('--data-bckt',
                    help='location of the training dataset in an object storage bucket',
                    default=None)

args = parser.parse_args()

artifacts_dir, checkpoint_dir, model_dir = mnist.create_dirs()

strategy = tf.distribute.MirroredStrategy()
print('Number of devices: {}'.format(strategy.num_replicas_in_sync))

train_dataset, test_dataset, info = mnist.get_data(data_bckt=args.data_bckt, data_dir=args.data_dir,
                                                   num_replicas=strategy.num_replicas_in_sync)
with strategy.scope():
    model = mnist.build_and_compile_cnn_model()

model.fit(train_dataset, epochs=2, callbacks=mnist.get_callbacks(model, checkpoint_dir))

model.save(model_dir, save_format='tf')
```

**Note**: Whenever you change the code, you have to build, tag and push the image to repo. If you change the tag, it needs to be updated inside the cluster definition yaml.

The required python dependencies are provided inside the conda environment file `oci_dist_training_artifacts/tensorflow/v1/environments.yaml`.  If your code requires additional dependency, update this file. 

**Note**: While updating `environments.yaml` do not remove the existing libraries. You can append to the list.

Building docker image - 

Update the TAG and the IMAGE_NAME as per your needs - 

```
export IMAGE_NAME=<region.ocir.io/my-tenancy/image-name>
export TAG=latest
```

```
docker build -t $IMAGE_NAME:$TAG \
    -f oci_dist_training_artifacts/tensorflow/v1/Dockerfile .

```

If you are behind proxy, use this command - 

```
docker build  --build-arg no_proxy=$no_proxy \
              --build-arg http_proxy=$http_proxy \
              --build-arg https_proxy=$http_proxy \
              -t $IMAGE_NAME:$TAG \
              -f oci_dist_training_artifacts/tensorflow/v1/Dockerfile .
```

Push the docker image - 

```
docker push $IMAGE_NAME:$TAG
```

#### 2a. Test locally with stand-alone docker run.

Before triggering the job run, you can test the docker image and verify the training code,
 dependencies etc. You can do this using a local stand-alone docker run or via a docker-compose setup(section 2b)

For a stand-alone docker run, start a docker container with ```bash``` entrypoint.

```
docker run --rm -it --privileged --entrypoint bash $IMAGE_NAME:$TAG

```
Optionally, you can choose to mount oci keys and code directory to the docker container.  

```
docker run --rm -it --privileged -v $HOME/.oci:/home/oci_dist_training/.oci -v $PWD:/code --entrypoint bash $IMAGE_NAME:$TAG

```

You have now a docker container and you are logged into it. Now test your training script with the following command.

```
OCI_IAM_TYPE=api_key TF_CONFIG='''{"cluster": {"worker": ["localhost:12345"]}, "task": {"type": "worker", "index": 0}}''' python /code/examples/tensorflow/ctl/train.py 
```
**Note** Pass on any args that your training script requires using the ```--``` flag. For example:

```
OCI_IAM_TYPE=api_key TF_CONFIG='''{"cluster": {"worker": ["localhost:12345"]}, "task": {"type": "worker", "index": 0}}''' python /code/examples/tensorflow/ctl/train.py
```

Once done, exit the container with the exit command.

```
exit
```

You can also test in a clustered manner using docker-compose. Next section.

#### 2b. Test locally with help of `docker-compose`

Create `docker-compose.yaml` file and copy the following content.

Once the docker-compose.yaml is created, you can start the containers by running - 

```
docker compose up --remove-orphans
```
You can learn more about docker compose [here](https://docs.docker.com/compose/)

```
#docker-compose.yaml for distributed tensorflow testing

services:
  chief:
    environment:
      WORKER_PORT: 12345
      OCI_IAM_TYPE: api_key
      OCI__CLUSTER_TYPE: TENSORFLOW
      OCI__ENTRY_SCRIPT: /code/train.py
      OCI__ENTRY_SCRIPT_ARGS: --data-dir /code/data/
      OCI__MODE: MAIN
      OCI__WORKER_COUNT: '1'
      OCI__WORK_DIR: /work_dir
    image: <region>.ocir.io/<tenancy_id>/<repo_name>/<image_name>:<image_tag>
    volumes:
      - ~/.oci:/home/oci_dist_training/.oci
      - ./work_dir:/work_dir
      - ./artifacts:/opt/ml
      - ./:/code
  worker-0:
    environment:
      WORKER_PORT: 23456
      OCI_IAM_TYPE: api_key
      OCI__CLUSTER_TYPE: TENSORFLOW
      OCI__ENTRY_SCRIPT: /code/train.py
      OCI__ENTRY_SCRIPT_ARGS: --data-dir /code/data/
      OCI__MODE: WORKER
      OCI__WORKER_COUNT: '1'
      OCI__WORK_DIR: /work_dir
    image: <region>.ocir.io/<tenancy_id>/<repo_name>/<image_name>:<image_tag>
    volumes:
      - ~/.oci:/home/oci_dist_training/.oci
      - ./work_dir:/work_dir
      - ./artifacts:/opt/ml
      - ./:/code
```

Things to keep in mind:
1. You can mount your training script directory to /code like the above docker-compose (`./:/code`). This way there's no need to build docker image every time you change training script during local development
2. Pass in any parameters that your training script requires in 'OCI__ENTRY_SCRIPT_ARGS'.
3. During multiple runs, delete your all mounted folders as appropriate; especially `OCI__WORK_DIR`
4. `OCI__WORK_DIR` can be a location in object storage or a local folder like `OCI__WORK_DIR: /work_dir`.
5. In case you want to use a config_profile other than DEFAULT, please change it in `OCI_CONFIG_PROFILE` env variable.

### 3. Create yaml file to define your cluster. Here is an example.

In this example, we bring up 1 worker node and 1 chief-worker node. 
The training code to run is `train.py`. All code is assumed to be present inside `/code` directory within the container.
Additionaly you can also put any data files inside the same directory (and pass on the location ex '/code/data/**' as an argument to your training script).

```
kind: distributed
apiVersion: v1.0
spec:
  infrastructure:
    kind: infrastructure
    type: dataScienceJob
    apiVersion: v1.0
    spec:
      projectId: oci.xxxx.<project_ocid>
      compartmentId: oci.xxxx.<compartment_ocid>
      displayName: HVD-Distributed-TF
      logGroupId: oci.xxxx.<log_group_ocid>
      subnetId: oci.xxxx.<subnet-ocid>
      shapeName: VM.Standard2.4
      blockStorageSize: 50
  cluster:
    kind: TENSORFLOW
    apiVersion: v1.0
    spec:
      image: "<region>.ocir.io/<tenancy_id>/<repo_name>/<image_name>:<image_tag>"
      workDir:  "oci://<bucket_name>@<bucket_namespace>/<bucket_prefix>"
      name: "tf_multiworker"
      config:
        env:
          - name: WORKER_PORT #Optional. Defaults to 12345
            value: 12345
          - name: SYNC_ARTIFACTS #Mandatory: Switched on by Default.
            value: 1
          - name: WORKSPACE #Mandatory if SYNC_ARTIFACTS==1: Destination object bucket to sync generated artifacts to.
            value: "<bucket_name>"
          - name: WORKSPACE_PREFIX #Mandatory if SYNC_ARTIFACTS==1: Destination object bucket folder to sync generated artifacts to.
            value: "<bucket_prefix>"
      main:
        name: "chief"
        replicas: 1 #this will be always 1.
      worker:
        name: "worker"
        replicas: 1 #number of workers. This is in addition to the 'chief' worker. Could be more than 1
  runtime:
    kind: python
    apiVersion: v1.0
    spec:
    spec:
      entryPoint: "/code/train.py" #location of user's training script in docker image.
      args:  #any arguments that the training script requires.
          - --data-dir    # assuming data folder has been bundled in the docker image.
          - /code/data/
      env:
```

### 4. Dry Run to validate the Yaml definition 

```
ads opctl run -f train.yaml --dry-run
```

This will print the Job and Job run configuration without launching the actual job.

### 5. Start Distributed Job

This will run the command and also save the output to `info.yaml`. You could use this yaml for checking the runtime details of the cluster.

```
ads opctl run -f train.yaml | tee info.yaml
```

### 6. Tail the logs

This command will stream the log from logging infrastructure that you provided while defining the cluster inside `train.yaml` in the example above.

```
ads opctl watch <job runid>
```

### 7. Check runtime configuration, run - 

```
ads opctl distributed-training show-config -f info.yaml
```

### 8. Saving Artifacts to Object Storage Buckets
In case you want to save the artifacts generated by the training process (model checkpoints, TensorBoard logs, etc.) to an object bucket
you can use the 'sync' feature. The environment variable ``OCI__SYNC_DIR`` exposes the directory location that will be automatically synchronized
to the configured object storage bucket location. Use this directory in your training script to save the artifacts.

To configure the destination object storage bucket location, use the following settings in the workload yaml file(train.yaml).

```
    - name: SYNC_ARTIFACTS
      value: 1
    - name: WORKSPACE
      value: "<bucket_name>"
    - name: WORKSPACE_PREFIX
      value: "<bucket_prefix>"
```
**Note**: Change ``SYNC_ARTIFACTS`` to ``0`` to disable this feature.
Use ``OCI__SYNC_DIR`` env variable in your code to save the artifacts. Example:

```
tf.keras.callbacks.ModelCheckpoint(os.path.join(os.environ.get("OCI__SYNC_DIR"),"ckpts",'checkpoint-{epoch}.h5'))
```
