# A simple logger for experiments [![Build Status](https://travis-ci.org/oval-group/logger.svg?branch=master)](https://travis-ci.org/oval-group/logger)

Full credits to the authors of [tnt](https://github.com/pytorch/tnt) for the structure with metrics.

## Installation

To install the package, run:
* `pip install -r requirements.txt` (the only requirement are `GitPython` and `numpy`, `visdom` is optional but recommended for real-time visualization)
* `python setup.py install`.

## Usage

There are two types of objects in this package: `Experiment` and `Metric`. An `Experiment` instance serves as an interface, and all `Metric` objects are reattached to it (see the example below).

Each metric has an `update` method that updates its internal state (e.g. current elapsed time for a `TimerMetric`, cumulated average for an `AvgMetric`), and a `reset` method to re-initialize it. The result of each metric can be fetched by the `Experiment` through the `log` methods. Note that calling `log` allows to store the result of a metric in the `Experiment` but does *not* reset the metric.

Each metric is identified by `Experiment` by a name and a tag. For instance, one can choose the common tag `train` for all metrics measuring statistics on the training data, and `test` for the testing set (example below).

Please see the (short) source code for details.

## Supported metric inputs

The value given in the `update` method of a metric must be one of the following:
* pytorch autograd Variable with one element
* pytorch tensor with one element
* numpy array with one element
* any type supported by the python `float()` function

It is then converted to a python `float` number.

## Example
A toy example can be found at `examples/example.py`:

```python
import logger
import numpy as np

# code to generate fake data
# ...

n_epochs = 10
use_visdom = True
# some hyperparameters we wish to save for this experiment
hyperparameters = dict(regularization=1,
                       n_epochs=n_epochs)
# options for the remote visualization backend
visdom_opts = dict(server='http://localhost',
                   port=8097)
xp = logger.Experiment("xp_name", use_visdom=use_visdom,
                       visdom_opts=visdom_opts)
# log the hyperparameters of the experiment
xp.log_config(hyperparameters)
# create parent metric for training metrics (easier interface)
train_metrics = xp.ParentWrapper(tag='train', name='parent',
                                 children=(xp.AvgMetric(name='loss'),
                                           xp.AvgMetric(name='acc1'),
                                           xp.AvgMetric(name='acck')))
# same for validation metrics (note all children inherit tag from parent)
val_metrics = xp.ParentWrapper(tag='val', name='parent',
                               children=(xp.AvgMetric(name='loss'),
                                         xp.AvgMetric(name='acc1'),
                                         xp.AvgMetric(name='acck')))

for epoch in range(n_epochs):
    # reset metrics
    train_metrics.reset()
    val_metrics.reset()
    # accumulate metrics over epoch
    for (x, y) in training_data():
        loss, acc1, acck = oracle(x, y)
        train_metrics.update(loss=loss, acc1=acc1,
                             acck=acck, n=len(x))
    # Method 1 for logging: log all metrics tagged with 'train'
    # Does *not* reset them
    xp.log_with_tag('train')

    for (x, y) in validation_data():
        loss, acc1, acck = oracle(x, y)
        val_metrics.update(loss=loss, acc1=acc1,
                           acck=acck, n=len(x))
    # Method 2 for logging: log Parent wrapper
    # (automatically logs all children)
    xp.log_metric(val_metrics)

# save to pickle file
xp.to_pickle("my_pickle_log.pkl")
# save to json file
xp.to_json("my_json_log.json")
```

This generates the following plot on `visdom`:
![alt text](examples/example.jpg)
