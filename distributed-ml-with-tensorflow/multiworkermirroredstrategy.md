# MultiWorkerMirroredStrategy

**Loading files that can't fit into memory all at once using Python Generators**

If we have a dataset that is too large to load into memory all at once, we'll need to use a _generator_. Generators allow you to return only one value of a loop at a time. So in the code below, the training step will call the generator at each step, but when it hits 'yield', it will return the data the training code. Once the training code calls the generator again, it will pick up at the same place it left off and continue with the for loop, returning the next batch of data. This way, we only ever load one batch data into memory at a time.

```python
def get_train():
    """ Training data generator """
    for file in train_files:
        img = ( np.array(nc.Dataset(file, "r")["ims"][0:1])[0], np.array(nc.Dataset(file, "r")["migrants"][0:1]) )
        yield img


def get_val():
    """ Validation data generator """
    for file in val_files:
        img = ( np.array(nc.Dataset(file, "r")["ims"][0:1])[0], np.array(nc.Dataset(file, "r")["migrants"][0:1]) )
        yield img
        
```

**Loading generators as TF Datasets**

To be compatible with TF's `model.fit()` method, we need to convert our generators into `tf.data.Dataset()` objects. We do this by calling the constructing on our generator function name and specifying the output types of our batch data.

```python
IMAGERY_DIR = "/sciclone/scr-mlt/hmbaier/cropped/"
files = os.listdir(IMAGERY_DIR)
files = [IMAGERY_DIR + i for i in files if "484" in i]

print("Number of nCDF files: ", len(files))

train_files, val_files = train_test_split(files, .75)
train_dataset = tf.data.Dataset.from_generator(generator = get_train, output_types=(tf.float32, tf.float32)).batch(int(args.world_size))
val_dataset = tf.data.Dataset.from_generator(generator = get_val, output_types=(tf.float32, tf.float32)).batch(int(args.world_size))

print("Number of training files: ", len(train_files))
print("Number of validation files: ", len(val_files)) 
```

```python
>>> Number of nCDF files: 
>>> Number of training files: 
>>> Number of validation files: 
```

**Initializing the `MultiWorkerMirroredStrategy()`**

```python
strategy = tf.distribute.experimental.MultiWorkerMirroredStrategy()

with strategy.scope():

    train_dataset = strategy.experimental_distribute_dataset(train_dataset)
    val_dataset = strategy.experimental_distribute_dataset(val_dataset)

    multi_worker_model = resnet.resnet56(img_input = tf.keras.layers.Input((None, None, 3)), classes = 1)

    multi_worker_model.compile(
        optimizer = tf.keras.optimizers.Adam(learning_rate = 0.001),
        loss = tf.keras.losses.MeanAbsoluteError(),
        metrics=[tf.keras.losses.MeanAbsoluteError()]
    )
```

**Train the `MultiWorkerMirroredStrategy()` model**

```python
multi_worker_model.fit(train_dataset,
                    steps_per_epoch = int(len(files) // int(args.world_size)),
                    validation_data = val_dataset)
                        
```

**Launching a Tensorflow `MultiWorkerMirroredStrategy()` script**

Launching a multi-worker training job is a little more intricate than simply calling `python train.py`. In order to set up our cluster, we need to know the IP addresses of each node in our job, then we need to launch the training scipt on each node individually. We'll use a set of bash scripts to do this.

**run.sh**

The rm commands inthis script delete files from any previous runs. The purpose of this script is to qsub the `tflow_job.sh` job on the number of nodes that you want to include in training. This is the only scipt you'll run to launch your training. You can use it by running: `./run.sh <NUM_NODES>`

```bash
rm -R /sciclone/home20/hmbaier/tflow/ips/
mkdir /sciclone/home20/hmbaier/tflow/ips/

rm -R /sciclone/home20/hmbaier/tflow/logs/
mkdir /sciclone/home20/hmbaier/tflow/logs/

for ((i = 1; i <= $1; i++))
do
    qsub /sciclone/home20/hmbaier/tflow/tflow_job.sh -l nodes=1:vortex:ppn=1 -v NODE_NUM=$i,WORLD_SIZE=$1
done
```

```
#!/bin/tcsh
#PBS -N head_node
#PBS -l walltime=10:00:00
#PBS -j oe


hostname -i > /sciclone/home20/hmbaier/tflow/ips/$NODE_NUM.txt

set size=`ls /sciclone/home20/hmbaier/tflow/ips/ | wc -l`

while ( $size != $WORLD_SIZE )
    set size=`ls /sciclone/home20/hmbaier/tflow/ips/ | wc -l`
    sleep 1
end

echo "$size"


# init conda within new shell for job
source "/usr/local/anaconda3-2021.05/etc/profile.d/conda.csh"
module load anaconda3/2021.05
unsetenv PYTHONPATH
conda activate tflow

python3 /sciclone/home20/hmbaier/tflow/worker_v5.py $NODE_NUM $WORLD_SIZE > "/sciclone/home20/hmbaier/tflow/logs/log${NODE_NUM}.txt"
```

**Connecting to the Python script: Setting up the training cluster using TF\_CONFIG**

The tflow\_job you submitted through `run.sh` will call a file called `ips.sh` that will save a text file with thte IP address of each node that will participate in training. At the beginning of our python training script, we will use this make\_config function to format those IP address into the environment variable Tensorflow needs to set up it's training cluster.

```python
ips_direc = "/sciclone/home20/hmbaier/tflow/ips/"
os.environ["TF_CONFIG"] = make_config(int(args.rank) - 1, ips_direc, 45962)


def make_config(rank,  direc, port):
    ips = []
    for file in os.listdir(direc):
        with open(direc + file, "r") as f:
            ips.append(f"{f.read().splitlines()[0]}:{port}")
    config = {"cluster": {"worker": ips}, "task": {"index": int(rank), "type": "worker"}}
    return json.dumps(config)
```

**Replication Code**

Replication code is in the folder: [https://github.com/heatherbaier/distributed-wm.github.io/tree/gh-pages/examples/tf\_mwms](https://github.com/heatherbaier/distributed-wm.github.io/tree/gh-pages/examples/tf\_mwms)

To launch the training script, open up a terminal and ssh into Vortex.

If you've never done the tutorial before, copy the below into your terminal and hit enter.

```
mkdir tf_mwms
cd tf_mwms

git init .
git remote add -f origin https://github.com/heatherbaier/distributed-wm.github.io.git
git config core.sparseCheckout true

echo "examples/tf_mwms/" >> .git/info/sparse-checkout

git pull origin master
git checkout gh-pages

cd examples/tf_mwms

chmod +x run.sh

./run.sh <NUM_NODES>
```

If you've done the tutorial before, simply cd into examples/tf\_mwms and run:

```
./run.sh <NUM_NODES>
```
