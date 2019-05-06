---
title: "INS'HACK :: 2019 :: Neurovision Misc 206 pts Writeup"
author: "CHERIEF Yassine (omega_coder)"
date: 2019-05-06T14:19:30+02:00
subtitle: "Neural Network weights visualization"
image: "https://res.cloudinary.com/https-omega-coder-github-io/image/upload/b_rgb:000000,o_33/v1557149191/1_0FlvitTZnPKh8qkJ7UPLeQ.png"
bigimg: [{"src": "https://res.cloudinary.com/https-omega-coder-github-io/image/upload/b_rgb:000000,o_20/v1557149191/1_0FlvitTZnPKh8qkJ7UPLeQ.png", "desc": "Neural Nets"}]
tags: ["neural-network", "image", "hdf5", "INS'HACK"]
categories: ["security", "writeups", "INShAck", "misc"]
---



# Challenge details.

| **Category**  | **Points** | **Solves** | **Difficulty** |
|-----------|--------|--------|--------|
| Misc | 206    |  20    | **Hard** |


# 1. Challenge description

> We found this strange file from an AI Startup. Maybe it contains sensitive information...<br><br>[Neurovision files](https://static.ctf.insecurity-insa.fr/dc1c30c75ea98475b324890d81da1f92b5ab04e8.tar.gz)


# 2. SOLUTION #1

We re given an `HDF5` filename named `neurovision-2d327377b559adb7fc04e0c3ee5c950c`, executing `file` command will tell us its an `HDF5` file

```bash
file neurovision-2d327377b559adb7fc04e0c3ee5c950c
# outputs 
neurovision-2d327377b559adb7fc04e0c3ee5c950c: Hierarchical Data Format (version 5) data
```

> For More on HDF5 structure  [HERE](https://en.wikipedia.org/wiki/Hierarchical_Data_Format)


executing `strings` on the model file (I believe its a model), we get the following, 

![strings_command_results](https://res.cloudinary.com/https-omega-coder-github-io/image/upload/v1557152375/Screenshot_2019-05-06_15-42-46.png)

some interesting infos.

```json
{
    "class_name": "Sequential",
    "config": {
        "name": "sequential_1",
        "layers": [{
            "class_name": "Flatten",
            "config": {
                "name": "flatten_1",
                "trainable": true,
                "batch_input_shape": [null, 68, 218],
                "dtype": "float32",
                "data_format": "channels_last"
            }
        }, {
            "class_name": "Dense",
            "config": {
                "name": "dense_1",
                "trainable": true,
                "units": 1,
                "activation": "sigmoid",
                "use_bias": true,
                "kernel_initializer": {
                    "class_name": "VarianceScaling",
                    "config": {
                        "scale": 1.0,
                        "mode": "fan_avg",
                        "distribution": "uniform",
                        "seed": null
                    }
                },
                "bias_initializer": {
                    "class_name": "Zeros",
                    "config": {}
                },
                "kernel_regularizer": null,
                "bias_regularizer": null,
                "activity_regularizer": null,
                "kernel_constraint": null,
                "bias_constraint": null
            }
        }]
    }
}
```
We have the input size which seems to be taking a 2-dimentional array (68x218) and outputs a single number between `0` and `1`, so we can guess that the model takes a 68x218 image and outputs some kind of `probability maybe.`, one more thing, we are dealing with a single layer Neural Network. (`No Hidden Layers`).

this is clearly a [`Keras`](https://keras.io/) Model. Let's load the model using Keras library and see wat we can do.

If anyone does not have keras Library installed on PC, you can work in [Colab](https://colab.research.google.com/), just open a Python3 notebook and start using Keras and many other libraries

`NOTE: Pillow and numpy are also required if you are using your own computer`


### 2.1 Load the Model and check layers and weights

```python
from keras import load_model

model = load_model("neurovision-2d327377b559adb7fc04e0c3ee5c950c")
```

Let's check layers;

```python
>>> model.layers
[<keras.layers.core.Flatten object at 0x7f2b069de860>, <keras.layers.core.Dense object at 0x7f2b069de9b0>]
>>> model.layers[0].input_shape
(None, 68, 218)
>>> model.layers[1].output_shape
(None, 1)
```

What about the weights,
```python
>>> model.get_weights()
[array([[-4.2567684e-05],
       [-4.2567684e-05],
       [-4.2567684e-05],
       ...,
       [-4.2567684e-05],
       [-4.2567684e-05],
       [-4.2567684e-05]], dtype=float32), array([0.], dtype=float32)]

```

We can notice that all weights have the same `absolute value`, some are `positive` and some are `negative` which is kinda weird!!!


Let's try `reshaping` the weights array to a 2D array of size 68x218 and create an image by turning all negative weights to black pixels and all positive weights to white pixels.

Here is how you can do it using Pillow Imaging Library
```python
import keras
from keras.models import load_model
import numpy as np
from PIL import Image

# Loading Model
model = load_model("neurovision-2d327377b559adb7fc04e0c3ee5c950c")

# getting weights and transforming them to values of 0s and 255
pixel_values_ = (model.get_weights()[0] + 1).astype(np.uint8) * 255

# reshape flattened array
reshaped_pixels = pixel_values_.reshape((68, 218))

# Create an image from a numpy 2d array
flag = Image.fromarray(reshaped_pixels)

# save the image, if you are on Colab, you can either export image bytes or save it
# you will find it in /content folder
flag.save('flag.png')
```

Here is an example from `Colab`.

![colab_solution_keras](https://res.cloudinary.com/https-omega-coder-github-io/image/upload/v1557155707/Screenshot_2019-05-06_16-37-10.png)

`eXecute` and get your flag :D

Greetz to [@youben11](https://github.com/youben11) for his `great` ideas and contribution.

`INSA{0v3erfitt3d_th3_Fl4g!}`


## SOLUTION #2 WILL BE POSTED LATER







