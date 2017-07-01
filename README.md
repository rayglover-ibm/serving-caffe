# TensorFlow Serving + Caffe

__This is a fork of Tensorflow Serving (TFS), extended with support for the
[Caffe](http://caffe.berkeleyvision.org/) deep learning framework.
For more information about Tensorflow Serving, switch to the `master` branch,
or visit the Tensorflow Serving [website](https://tensorflow.github.io/serving/).__

---

## Summary

TensorFlow Serving is an open-source software library for serving
machine learning models. It deals with the *inference* aspect of machine
learning, taking models after *training* and managing their lifetimes, providing
clients with versioned access via a high-performance, reference-counted lookup
table.

## Setup, Build & test

First, clone the repository and its submodules:

    > git clone --recurse-submodules https://github.com/rayglover-ibm/serving-caffe
    > cd serving-caffe

Caffe has been integrated in to TFS build, and as such you should follow the
[TFS installation guide](https://github.com/tensorflow/serving/blob/master/tensorflow_serving/g3doc/setup.md)
first. At a minimum you need to install bazel and configure Tensorflow manually:

    > cd tensorflow; ./configure
    > cd ..

Next, install the Caffe prerequisites on your system. For a comprehensive guide, see the [Caffe Installation guide](http://caffe.berkeleyvision.org/installation.html#prerequisites). At a minimum, you
will need the following packages (Ubuntu):

- `g++ binutils cmake`
- `libboost-thread-dev libboost-system-dev libboost-filesystem-dev`
- `libgflags-dev libgoogle-glog-dev libhdf5-dev`

To validate the Caffe build, run the following bazel command. This will retrieve Caffe
from Github and build Caffe in cpu mode:

    > bazel build -c opt @caffe//:lib

### Python Layers (optional)

Caffe [Python layers](https://github.com/NVIDIA/DIGITS/tree/master/examples/python-layer) (the `Python` layer type) can be used to execute parts of a model within Python. To work correctly, you should have Python installed on your system. In addition you'll need to install the following packages (Ubuntu):

- `libpython-dev libboost-python-dev`

The python layer is enabled at build-time with the `--define=caffe_python_layer=ON` option. For example, to run tests which demonstrate the use the python layers:

    > bazel test --define=caffe_python_layer=ON \
        //tensorflow_serving/servables/caffe:caffe_session_bundle_factory_test

### GPU Support (optional, linux only)

The Caffe build adopts the CUDA configuration from Tensorflow, and as such will use the version (and location) of cudnn, and the standard cuda libraries you specified when you configured Tensorflow. You can validate this configuration by building Caffe with CUDA:

    > bazel build -c opt --config=cuda @caffe//:lib

For more information on installing the CUDA libraries and configuring Tensorflow, read
the Tensorflow setup guide [here](https://www.tensorflow.org/versions/r0.8/get_started/os_setup.html#optional-install-cuda-gpus-on-linux).

### Tests

To run tests related to the Caffe servable implementation, run:

    > bazel test tensorflow_serving/servables/caffe/...

---

## *Example*: MNIST Handwriting Recognition

The Tensorflow model server implementation has been altered in this fork to run with either Caffe or/and Tensorflow. The steps in this example are based on the original TensorFlow-only tutorial [here](tensorflow_serving/g3doc/serving_basic.md).

#### 1. Download the Pre-trained Caffe Model:

    > bazel run //tensorflow_serving/servables/caffe/test_data:mnist_caffe_fetch -- \
        --version 1 /tmp/mnist_export_caffe

There's nothing special about this pretrained model, and it can be re-generated by following Caffe's LeNet MNIST Tutorial [here](https://github.com/BVLC/caffe/tree/master/examples/mnist).

The contents of any pretrained model must include `deploy.prototxt` `weights.caffemodel` files, which as you could imagine, contain the deployable model definition and a single training snapshot. Additionally, you can include a `classlabels.txt` file containing line delimited class labels for the output of the model.

#### 2. Build the Model Server

```
> bazel build -c opt //tensorflow_serving/model_servers:model_server
```

The `model_server` in this fork has learned a `--platform_name=<servable name>` option which supports `tensorflow` or `caffe` values. So, to begin serving the Caffe model(s),

```
>  bazel-bin/tensorflow_serving/model_servers/model_server --port=9000 \
    --model_name=mnist --platform_name=caffe --model_base_path=/tmp/mnist_export_caffe
```

*Sample output:*

```
I Using servable 'caffe'
...
I Attempting to load a SessionBundle from: /tmp/mnist_export_caffe/00000001/
I Caffe execution mode: CPU
I Loaded Network:
    name: LeNet
    inputs: 1
    outputs: 1
    initial batch-size: 1
    output classes: Tensor<type: string shape: [10] values: Zero One Two...>
I Running restore op for CaffeSessionBundle
I Done loading SessionBundle
I Wrapping SessionBundle session to perform batch processing
I Running...
```

#### 3. Build and run the client

This client is servable-agnostic; it supports both Caffe and TensorFlow servers without modification.

```
> bazel build -c opt //tensorflow_serving/example:mnist_client
> bazel-bin/tensorflow_serving/example/mnist_client \
    --num_tests=1000 --server=localhost:9000 --concurrency=10
```

*Sample output:*

```
Inference error rate: 1.2%
Request error rate: 0.0%
Avg. Throughput: 197.192047438 reqs/s
Request Latency (percentiles):
  50th ....... 46ms
  90th ....... 62ms
  99th ....... 83ms
```

---

## *Example:* Object detection

This example exposes a single service implementation supporting both _[Faster R-CNN](https://github.com/rbgirshick/py-faster-rcnn)_ and _[SSD: Single Multibox Detector](https://github.com/weiliu89/caffe/tree/ssd)_ object detection architectures; demonstrating use of multiple Caffe forks, Python interpreter integration, and a basic client/server configuration handshake.

#### 1. Fetch, build and setup demo models:

```
> bazel run -s //tensorflow_serving/example:obj_detector_fetch -- --export-path=/tmp/obj_detector \
    --version=1 --type=[ssd/rcnn] /tmp/obj_detector_data
```

#### 2. Build the detector:

Build the detector service. Note since each backend requires different version of Caffe, this must be specified using `caffe_flavour` option. The Faster-RCNN detector also requires an extra build step to compile the cython/cuda python modules (shown below.)

- ##### 2.a. SSD

    ```
    > bazel build [--config=cuda] -c opt --define=detector=ssd --define=caffe_flavour=ssd \
        //tensorflow_serving/example:obj_detector
    ```

- ##### 2.b. Faster R-CNN

    ```
    > pushd /tmp/obj_detector_data/rcnn/lib && python setup.py build_ext --inplace && popd

    > bazel build [--config=cuda] -c opt --define=detector=rcnn --define=caffe_flavour=rcnn \
        --define=caffe_python_layer=ON //tensorflow_serving/example:obj_detector
    ```

#### 3. Run the server

    > ./bazel-bin/tensorflow_serving/example/obj_detector [--resolution=<H>x<W>] --port=9000 \
        /tmp/obj_detector

The `--resolution` option specifies what size images the service will accept. The loaded model will be reshaped to accept images of the given dimensions, although you may find that some resolutions produce better results than others. If unspecified, the service will try to select sensible defaults.

#### 4. Build & run the client (python)

    > bazel build -c opt //tensorflow_serving/example:obj_detector_client
    > bazel-bin/tensorflow_serving/example/obj_detector_client --server=localhost:9000

The client has a number of options:

    --concurrency K        maximum number of concurrent inference requests
    --num_tests NUM_TESTS  Number of test images
    --server SERVER        obj_detector service host:port
    --img IMG              url or path of an image to classify
    --imgdir IMGDIR        path to a gallery of images
    --verbose              print detections to stdout
    --gui                  show detections in a gui

For example, to perform a basic load test using a gallery of images located at `/tmp/images`:

    > bazel-bin/tensorflow_serving/example/obj_detector_client --server=localhost:9000 \
        --imgdir=/tmp/images --num_tests=250 --concurrency=8

Which will produce latency and throughput statistics over 250 requests.

---


## FAQ

### How do I use my own Fork of Caffe?

If you intend to use a fork of Caffe which contains (for example) custom layers, you can add one in `tensorflow_serving/workspace.bzl` which points to the file/git location of your fork.

## Misc. Development notes

- The Caffe Servable is implemented in `serving/servables/caffe` and is based on the Tensorflow servable.

- To be able to reuse as much of the TFS infastructure as possible (batching, model versioning etc.), and to be able to create server frontends which can be switched to/from Caffe and Tensorflow with minimum effort, the core Caffe servable, `CaffeServingSession`, derives from the `tensorflow::serving::ServingSession` base class. This essentially encapsulates the Caffe model as though it were a Tensorflow one.

---

__(C) Copyright IBM Corp. / Google Inc. 2016. All Rights Reserved.__
