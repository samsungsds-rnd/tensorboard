# Bring TensorBoard to MXNet from scratch
With proper visualization, we could have a better understanding of the mechanism of Deep Learning. 

 * Monitoring training/testing metrics through the learning curve. You know how well your model learning from the data.
 * Visualizing the dynamics under the gradients of a layer with histogram, you know your network is live or dead(gradient).
 * By interpreting the embedding features from a layer, using t-SNE for high-dimensional data visualization, you get intuitions from its representation power.

There are way more techniques in visualizing neural networks than the above. So that’s why we want to build a handy tool for our MXNet users.

Thanks to the community, we already have [TensorBoard](https://www.tensorflow.org/versions/master/how_tos/graph_viz/index.html)  and it’s easy-to-use and meets most daily use cases. However, TensorBoard is built together with TensorFlow and we have to come up a way to make a stand-alone version for general visualization purpose.

## Why write this?
The main purpose, of writing this note, is to give you a basic idea of *how does it feel to get involved in an open-sourced project* and provides something for those potential/future contributors. As one could learn a lot from this community! 

Also, it’s my first time:

 * Learn what’s continuous integration.
 * How to write a proper setup.py
 * …

## Before we start
It's my first time to get involved in an open-sourced project like MXNet, and it has some visualization solutions there.  I found there’re several similar issues requests for TensorBoard-liked tool, some people want to build the tool from scratch, while [@piiswrong](https://github.com/piiswrong) asked whether is possible to strip TensorBoard from TensorFlow. I like the latter one, so I created an issue [dmlc/mxnet#4003](https://github.com/dmlc/mxnet/issues/4003) for discussion,  proposed my solution and roadmap towards this direction. 

## The Logging Part
Technically, TensorBoard contains two parts: logging and rendering. In TensorBoard, it supports these types of data:

* Scalar. 
* Image.
* Video.
* Histogram.
* Graph. The TensorFlow computational graph.
* Embedding.

### Get  summary without running TensorFlow

In TensorFlow, `summary` object could be generated by running a `session` or by running an operation, here's an example in [TensorBoard Document](https://www.tensorflow.org/how_tos/summaries_and_tensorboard/)

```python
with tf.name_scope('cross_entropy'):
  # The raw formulation of cross-entropy,
  #
  # tf.reduce_mean(-tf.reduce_sum(y_ * tf.log(tf.softmax(y)),
  #                               reduction_indices=[1]))
  #
  # can be numerically unstable.
  #
  # So here we use tf.nn.softmax_cross_entropy_with_logits on the
  # raw outputs of the nn_layer above, and then average across
  # the batch.
  diff = tf.nn.softmax_cross_entropy_with_logits(y, y_)
  with tf.name_scope('total'):
    cross_entropy = tf.reduce_mean(diff)
tf.summary.scalar('cross_entropy', cross_entropy)
```

Luckily, those summaries are [Protocol Buffers](https://developers.google.com/protocol-buffers/) and it makes things easy, which means it could be changed into pure Python. Thanks to the community, I found [@mufeili](https://github.com/mufeili) has been working on this feature and TensorBoard has provided a relatively clean API for this purpose, which allows us to get these without running TensorFlow’s operation. 

The logging part code is placed in [tensorboard/python](https://github.com/dmlc/tensorboard/tree/master/python).

### Logging events in pure Python

TensorBoard relies on the `Event` file, in which `summary` information is included in this file.

```bash
$ tensorboard --logdir=path-to-event-files
```

See [tensorboard/record_writer.py](https://github.com/dmlc/tensorboard/blob/master/python/tensorboard/record_writer.py) for details.

## The Rendering Part
The next step is to build TensorBoard's rendering part. The goal is to provide an easy-to-maintain solution, so we didn’t try to strip the relevant codes and build it from scratch. Rather, we pull the TensorFlow codebase and use bazel to build TensorBoard:

```bash
$ git clone https://github.com/tensorflow/tensorflow
$ cd tensorflow
$ ./configure
$ bazel build tensorflow/tensorboard:tensorboard
```

Then all dependencies could be found in `/bazel-bin/tensorflow/tensorboard`, a Python binary `tensorboard` to launch the app and its dependencies `tensorboard.runfiles`.

Here’s the file structure before we build TensorBoard(and move files):

```
├── LICENSE
├── Makefile
├── README.md
├── installer.sh
├── python
│   ├── README.md
│   ├── setup.py
│   └── tensorboard
│       ├── __init__.py
│       ├── crc32c.py
│       ├── event_file_writer.py
│       ├── record_writer.py
│       ├── src
│       │   └── __init__.py
│       ├── summary.py
│       └── writer.py
├── tensorboard
│   └── src
│       ├── event.proto
│       ├── resource_handle.proto
│       ├── summary.proto
│       ├── tensor.proto
│       ├── tensor_shape.proto
│       └── types.proto
└── tools
    └── pip_package
        ├── MANIFEST.in
        ├── README
        └── build_pip_package.sh
```

Then we get `tensorboard` and `tensorboard.runfiles/`:

```
├── LICENSE
├── Makefile
├── README.md
├── installer.sh
├── python
│   ├── MANIFEST.in
│   ├── README
│   ├── README.md
│   ├── setup.py
│   └── tensorboard
│       ├── __init__.py
│       ├── crc32c.py
│       ├── event_file_writer.py
│       ├── record_writer.py
│       ├── src
│       ├── summary.py
│       ├── tensorboard           <--- python binary
│       ├── tensorboard.runfiles  <--- directory/dependencies
│       └── writer.py
├── tensorboard
│   └── src
│       ├── event.proto
│       ├── resource_handle.proto
│       ├── summary.proto
│       ├── tensor.proto
│       ├── tensor_shape.proto
│       └── types.proto
└── tools
    └── pip_package
        ├── MANIFEST.in
        ├── README
        └── build_pip_package.sh
```

The most important, the `tensorboard` binary is directly usable, as it searches and imports relevant packages:

```python
# Find the runfiles tree
def FindModuleSpace():
  # Follow symlinks, looking for my module space
  stub_filename = os.path.abspath(sys.argv[0])
  while True:
    # Found it?
    module_space = stub_filename + '.runfiles'
    if os.path.isdir(module_space):
      break

    runfiles_pattern = "(.*\.runfiles)/.*"
    if IsWindows():
      runfiles_pattern = "(.*\.runfiles)\\.*"
    matchobj = re.match(runfiles_pattern, os.path.abspath(sys.argv[0]))
    if matchobj:
      module_space = matchobj.group(1)
      break

    raise AssertionError('Cannot find .runfiles directory for %s' %
                         sys.argv[0])
  return module_space
```

However, this package search logic may have some problems if we install from the Python wheel. And we would discuss the solution later.

## Build pip package for both logging and rendering
Now we’ve finished all preparations for building a Python wheel. As we have a binary file and want to launch TensorBoard in the following way:

```bash
$ tensorboard --logdir=path-to-event-file/
```

It could be done by assigning binary script entry in `setup.py`, see [tensorboard/setup.py](https://github.com/dmlc/tensorboard/blob/master/python/setup.py) for more. The basic idea of a setup function, in this scenario, is to pack all necessary dependencies and setup the binary file correctly.

Ops, back to the `FindModuleSpace` issue mentioned above. The `tensorboard` binary file would be copied to `/Users/zihao.zzh/anaconda2/envs/tensorboard/bin/tensorboard`, for example. But when you launch TensorBoard, it searches the `.runfiles` in the same directory, so we make a [tensorboard/tensorboard-binary.patch](https://github.com/dmlc/tensorboard/blob/master/tensorboard-binary.patch) and apply this patch in the building process.

It provides an `installer.sh` to automate this process for you, and you might want to check it out here: [tensorboard/installer.sh](https://github.com/dmlc/tensorboard/blob/master/installer.sh)

## Future works
* Supports `image`, `video`, `embedding` and even `graph` for MXNet.
* Make package installation more easier by providing a pre-built wheel, then users could install this package in one line. 

## About the author
[Zihao Zheng](https://github.com/zihaolucky) is an algorithm engineer at AI Lab, Alibaba Group. Before joined the industry, he studied Mathematics at South China Normal University and learned Machine Learning and Computer Science from MOOCs.

Many thanks to our community, TensorBoard authors and contributors!!


