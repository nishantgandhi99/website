---
author: "Nishant M Gandhi"
date: 2026-03-24
linktitle: Keras 2 to Keras 3 migration and reproducibility
title: Keras 2 to Keras 3 migration and reproducibility
image : ""
---

There are few changes in software libraries that do not show any error but still change the behavior of the application. The upgrade from Keras 2 to Keras 3 had one such problem.

Same seed, same model, same input, but different output in different runs. There was no error and no warning.

The problem was related to the way random numbers were managed.

### **1. tf.random.set_seed() was not enough**

In Keras 2, we were using the TensorFlow global seed to control the random behavior.

``` python
import tensorflow as tf
import numpy as np

tf.random.set_seed(42)
np.random.seed(42)
```

This worked for many common cases such as weight initialization, dropout, data shuffling and TensorFlow operations.

After moving to Keras 3, setting the TensorFlow seed alone was not enough. Keras has its own random state and some layers use a `SeedGenerator`.

The TensorFlow random state and Keras random state do not automatically stay synchronized.

### **2. Set the seed before creating the model**

The seed should be set before creating or building the layers. Setting it after the model is already built does not recreate the existing weights or the layer random state.

``` python
import keras
import tensorflow as tf
import numpy as np

keras.utils.set_random_seed(42)
tf.random.set_seed(42)
np.random.seed(42)

model = keras.Sequential([
    keras.layers.Dense(64, activation='relu', input_shape=(10,)),
    keras.layers.Dropout(0.5),
    keras.layers.Dense(1)
])
```

`keras.utils.set_random_seed()` is the important change here. It sets the Python, NumPy, and backend random seeds together.

### **3. Resetting the seed after the layer is built does not help**

``` python
model = keras.Sequential([
    keras.layers.Dense(64, activation='relu', input_shape=(10,))
])

model.build((None, 10))

# This does not reinitialize the model
keras.utils.set_random_seed(42)
```

The model is already built at this point. Reassigning the random state does not change the weights that were already created.

If a model needs to be recreated with a particular seed, set the seed first and create a new model. This is less confusing than trying to repair the state of an existing model.

### **4. Check reproducibility as part of the upgrade**

It is useful to run the same small experiment before and after the upgrade.

``` python
def create_model():
    return keras.Sequential([
        keras.layers.Dense(32, activation='relu', input_shape=(10,)),
        keras.layers.Dropout(0.3),
        keras.layers.Dense(1)
    ])

keras.utils.set_random_seed(42)
model1 = create_model()
output1 = model1.predict(X_test, verbose=0)

keras.utils.set_random_seed(42)
model2 = create_model()
output2 = model2.predict(X_test, verbose=0)

assert np.allclose(output1, output2)
```

This test does not prove that every operation is deterministic on every hardware platform. It does catch the common migration problem where the model is created before the seed is set or only the TensorFlow seed is set.

### **Few things to check during Keras 3 upgrade**

* Use `keras.utils.set_random_seed()` instead of relying only on `tf.random.set_seed()`.
* Set the seed before creating the model and its layers.
* Test dropout, weight initialization, and data shuffling separately.
* Check custom layers that generate random values.
* Compare model outputs and metrics before declaring the migration complete.

The difficult part of this issue was not fixing an exception. There was no exception. The difficult part was finding the assumption that was no longer true after the library upgrade.

When upgrading a machine learning library, testing that the code runs is not enough. The outputs and metrics also need to be compared.