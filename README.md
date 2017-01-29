# Behavioral Cloning for Self Driving Car
The scope of project is to teach car about human driving behavior using deep learning so that the car can predict steering angle by itself. This is the 3rd project in Udacity Self Driving Car nanodegree. Data collection, driving and testing are performed on Udacity car simulation.
## Installation & Resources
1. Anaconda Python 3.5
2. Udacity [Carnd-term1 starter kit](https://github.com/udacity/CarND-Term1-Starter-Kit) with miniconda installation 
3. Udacity Car Simulation on [Window x64](https://d17h27t6h515a5.cloudfront.net/topher/2016/November/5831f3a4_simulator-windows-64/simulator-windows-64.zip)
4. Udacity [sample data](https://d17h27t6h515a5.cloudfront.net/topher/2016/December/584f6edd_data/data.zip)

### Quickstart
**1. Control of the car is by using button on PC keyboard or joystick or game controller.**

:arrow_up: accelerate :arrow_down: brake :arrow_left: steer left :arrow_right: steer right

**2. Two driving modes:**
- Training: For user to take control over the car
- Autonomous: For car to drive by itself

**3. Collecting data:**
User drives on track 1 and collects data by recording the driving experience by toggle ON/OFF the recorder. Data is saved as frame images and a driving log which shows the location of the images, steering angle, throttle, speed, etc. 
Another option is trying on Udacity data sample.

## Test Drive
Drive around the tracks several time to feel familiar with the roads and observe the environment around the track.

Track 1: *flat road, mostly straight driving, occasionally sharp turns, bright day light.*
![track1](https://cloud.githubusercontent.com/assets/23693651/22400792/a8927a68-e58c-11e6-8a66-839869832cce.png)

Track 2: *hilly road, many light turns and sharp turns, dark environment*
![track2](https://cloud.githubusercontent.com/assets/23693651/22400796/be938938-e58c-11e6-9938-6ba32ef3d554.png)

## Project Requirement
Deep Learning training is on **Track 1** data. To pass the project, the car has to successfully drive by itself without getting off the road on **Track 1**. 
For self evaluation, the model can successfully drive the entire **Track 2** without getting off the road.

## Approach
To have any idea to start this project, [End to End Learning for Self-Driving Cars](http://images.nvidia.com/content/tegra/automotive/images/2016/solutions/pdf/end-to-end-dl-using-px.pdf) by Nvidia is a great place to read.
From the paper, data collection is the first important part. Per project requirement, data collection can only performed on **Track 1**. User drives about 4 laps around **Track 1** by keyboard control to collect data. The experimence wasn't extrememly smooth as actual driving. So user decides to use Udacity sample data as starting point.

### Understanding Data
There are 3 cameras on the car which shows left, center and right images for each steering angle. 

![views_plot](https://cloud.githubusercontent.com/assets/23693651/22402134/546e68ec-e5ba-11e6-9266-ff9d7fdf3431.png)

After recording and save data, the simulator saves all the frame images in `IMG` folder and produces a driving_log.csv file which containts all the information needed for data preparation such as path to images folder, steering angle at each frame, throttle, brake and speed values.

![driving_log](https://cloud.githubusercontent.com/assets/23693651/22401702/65c154a6-e5ab-11e6-966f-c39d0f6aaa9c.png)

In this project, we only need to predict steering angle. So we will ignore throttle, brake and speed information.

### Argumentation
From the observation in both tracks, there are many factor of road condition and environment to account for. Below are argumentation methods:

1. Dark and bright environement: user chooses to generate random brightness by adjusting V channel of HSV colorspace in each image.
`new_V_channel = current_V_channel*np.random.uniform(0.3, 1)`. This adjustment will apply darker transformation on the images.

2. Inconsistent turns in each track: user choose to randomly flipping images (bernoulli trial) on vertical axis.
`flipped_image = cv2.flip(image,1)`, `flipped_angle = angle*(-1)`.

### Recovery
In general sense, driving behavior can be trained using the central images because we always look ahead when driving. Driving is mostly straight driving as well or small adjustment to turn the car unless there is a sharp turn. Below is the plot of steering angle on track 1 from Udacity data.

![scatter](https://cloud.githubusercontent.com/assets/23693651/22402161/495cf170-e5bb-11e6-85cb-33bca2ba276f.png)
![distribution](https://cloud.githubusercontent.com/assets/23693651/22402177/905481e2-e5bb-11e6-9c61-82df41884ba0.png)

But by inutition, if our car goes off lane (for example, distraction during text and drive), we can adjust it back on track right away. The machine doesn't have this intuition, so once it goes off road it would be hard to recover. To teach the machine this kind of intuion, we have to show it the scenarios. Hence, we use left and right camera images. Udacity gives out a great hint how to apply this method.
>In the simulator, you can weave all over the road and turn recording on and off. In a real car, however, that’s not really possible. At least not legally.
>So in a real car, we’ll have multiple cameras on the vehicle, and we’ll map recovery paths from each camera. **For example, if you train the model to associate a given image from the center camera with a left turn, then you could also train the model to associate the corresponding image from the left camera with a somewhat softer left turn. And you could train the model to associate the corresponding image from the right camera with an even harder left turn.

So the job is to determine when the car is turning left or right, pick out a set of its left or right images and add/subject with adjustment angle. The logic here is:
  1. Left turn: + adjustment_angle on left image, - adjustment_angle on right image
  2. Right turn: +adjustment_angle on right image, -adjustment_angle on left image

### Preprocessing
1. To help the system avoid learning other part of the image but only the track, user crops out the sky and car deck parts in the image. Original image size (160x320), after cropping 60px on top and 20px on the bottom, new image size is (80x320).
2. To help running a smaller training model, images are scaled to (64x64) size from after cropping size (80x320).

### Generators
The model is trained using Keras with Tensorflow backend. User goal is to not generate extra data from what has been collected. To help always getting new training samples by apply random argumentation, fit_generator() is used to fit the training model.

There are two generators in this project. **Training generator** is to generate samples per batches to feed into fit_generator(). At each batch, random samples are picked, applied argumentation and preprocessing . So training samples feeding into model is always different. **Validation generator** is also to feed random samples in batches for validation, unlike training generator, only central images are used here and only proprocessing is applied.

### Training
Training model is inspired by Nvidia model. Some changes are less 1 less convolution layers, smaller dense() parameters and many layers of Relu, Dropout, Regularization are applied to prevent overfitting.
Saved training trained with 20 epoch. To run training: `python model.py --epoch 20`

![architecture](https://cloud.githubusercontent.com/assets/23693651/22402330/ac793d4a-e5c0-11e6-9c41-a014fe3dd1a7.png)
![training2](https://cloud.githubusercontent.com/assets/23693651/22402343/f892ac92-e5c1-11e6-82da-ce39e51a96be.png)

### Testing
Use the training model that was saved in `model.json`, and weights in `model.h5`. Make sure the feeding images from test track is preprocessed as well to match with final training images shape in the training model.

To run test: `python drive.py model.json`





