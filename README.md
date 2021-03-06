# YOLOv3

**YOLOv3 in Tensorflow(TF-Slim)**

| python | tensorflow | opencv |
| :----: | :--------: | :----: |
| 3.6.5  |   1.8.0    | 3.3.1  |

Refactor All the Codes

- [x] **transformation Script**: Convert original [yolov3.weight](https://pjreddie.com/media/files/yolov3.weights) to Tensorflow style.
- [x] **forward**:  Inference is available.(Tested)
- [ ] **Training operation**: Losses have be implemented.(Not Tested)
- [ ] ...

---

## Transformation

Download yolov3.weight and yolov3.cfg from the [Homepage](https://pjreddie.com/darknet/yolo/).

Run our script(you need Tensorflow, Numpy and Python3 only).

```python
from yoloparser import YoloParser

weights_path = "./yolov3/yolov3.weights"
cfg_path = "./yolov3/yolov3.cfg"
out_path = "./yolov3/yolov3.ckpt"

parser = YoloParser(cfg_path, weights_path, out_path)
parser.run()
```

```shell
Reading .cfg file ...
Converting ...
From ./yolov3/yolov3.weights
To   ./yolov3/yolov3.ckpt
Encode weights...
Success!
Model Parameters:
<tf.Variable 'Conv_0/weights:0' shape=(3, 3, 3, 32) dtype=float32_ref>
<tf.Variable 'Conv_0/BatchNorm/gamma:0' shape=(32,) dtype=float32_ref>
<tf.Variable 'Conv_0/BatchNorm/beta:0' shape=(32,) dtype=float32_ref>
<tf.Variable 'Conv_0/BatchNorm/moving_mean:0' shape=(32,) dtype=float32_ref>
<tf.Variable 'Conv_0/BatchNorm/moving_variance:0' shape=(32,) dtype=float32_ref>
<tf.Variable 'Conv_1/weights:0' shape=(3, 3, 32, 64) dtype=float32_ref>
<tf.Variable 'Conv_1/BatchNorm/gamma:0' shape=(64,) dtype=float32_ref>
<tf.Variable 'Conv_1/BatchNorm/beta:0' shape=(64,) dtype=float32_ref>
<tf.Variable 'Conv_1/BatchNorm/moving_mean:0' shape=(64,) dtype=float32_ref>
<tf.Variable 'Conv_1/BatchNorm/moving_variance:0' shape=(64,) dtype=float32_ref>
...
Finish !
```

You can check all variables by Tensorflow API. Of course,  renaming all variables is possible by tf.train.Saver.

```python
import tensorflow as tf

reader = tf.train.NewCheckpointReader(out_path)
var_to_shape_map = reader.get_variable_to_shape_map()
for key in var_to_shape_map:
  print( key)
```

```shell
Conv_15/weights
Conv_0/BatchNorm/beta
Conv_0/BatchNorm/gamma
Conv_27/BatchNorm/gamma
Conv_11/weights
Conv_57/BatchNorm/moving_mean
Conv_1/BatchNorm/gamma
Conv_56/BatchNorm/gamma
Conv_0/BatchNorm/moving_mean
Conv_7/BatchNorm/gamma
Conv_44/BatchNorm/moving_variance
Conv_0/weights
Conv_15/BatchNorm/beta
Conv_0/BatchNorm/moving_variance
...
```

---

## **Backbone**

```python
import tensorflow as tf
import cv2
import numpy as np

import vis
from model.yolo import YOLOv3

# you can find all of them in ./src
ANCHORS = [(10, 13), (16, 30), (33, 23),
          (30, 61), (62, 45), (59, 119),
          (116, 90), (156, 198), (373, 326)]

NUM_CLASS = 80

ckpt_path = "./src/yolov3.ckpt"
coco_name_path = "./src/coco.name"
dog_jpg_path = "./src/dog.jpg"
bird_jpg_path = "./src/birds.jpg"
```

We test the program with the minibatch(batch size=1, 2), so the input is generated firstly(A 4-D tensor, [Batch_size, Height, Width, Channel] ). Normalizing the pixel value is also necessary.

```python
img = cv2.imread(dog_jpg_path)
img_resized = cv2.resize(img, (416, 416))
img_rgb = cv2.cvtColor(img_resized, cv2.COLOR_BGR2RGB)
dog = np.expand_dims(img_rgb/255., axis=0) #[1, 416, 416, 3]

img = cv2.imread(bird_jpg_path)
img_resized = cv2.resize(img, (416, 416))
img_rgb = cv2.cvtColor(img_resized, cv2.COLOR_BGR2RGB)
bird = np.expand_dims(img_rgb/255., axis=0) #[1, 416, 416, 3]

imgs = np.concatenate([dog, bird], axis=0) #[2, 416, 416, 3]
```

The process of detection is simple.

```python
yolov3 = YOLOv3(ANCHORS, NUM_CLASS)
x = tf.placeholder(tf.float32, [None, 416, 416, 3])
feature = yolov3.forward(x)
bboxes, scores, labels = yolov3.decode_feature(feature)

saver = YOLOv3.load_weight()
with tf.Session() as sess:
  saver.restore(sess, ckpt_path) # restore yolov3 parameters
  bboxes, scores, labels = sess.run([bboxes, scores, labels], feed_dict={x:imgs})
```

A simple visualization API is also provided.

```python
coco_name = YOLOv3.load_coco_name(coco_name_path)
vis.plot(imgs, bboxes, scores, labels, coco_name, output_path)
```

![dog](./src/detection1.jpg)



![birds](./src/detection2.jpg)

