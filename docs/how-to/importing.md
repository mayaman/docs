# Importing Models into Runway

Runway models are platform-agnostic; models written in any framework/language can be used by Runway as long as the model can be made accessible via an HTTP server. This process, however, is easiest in Python where we provide an SDK for parsing the inputs to the model, serializing its outputs, and setting up the server environment.

A Runway Model consists of an HTTP server that exposes a common interface over a network and a configuration file that specifies dependencies and build steps for running that server (and your model code) inside a Docker container. Both the network interface for interacting with the model and the Docker images created as a result of the configuration file are abstracted from the developer using the [Runway Model SDK](https://sdk.runwayml.com).

Here are the steps involved in porting an ML model written in Python to Runway:

1. Create a `runway_model.py` file which implements several methods from the [Runway Model SDK](https://sdk.runwayml.com).
1. Write a `runway.yml` config file.
1. Upload your code to a GitHub repository.
1. Import your new model into the Runway app using your GitHub account.
1. Push new commits to your GitHub repo to trigger new model versions to be built and published on Runway.

Once you've imported your model into Runway using your GitHub account, each `git push` will trigger the latest version of your code to be built and optionally deployed publicly through Runway.

In this tutorial, we will demonstrate how to port the [SqueezeNet](https://arxiv.org/abs/1512.03385) computer vision model to Runway. We also provide a [Runway Model Template](https://github.com/runwayml/model-template) repository that contains boilerplate code for porting a new model.

### Importing SqueezeNet into Runway

We recommend forking the [`runwayml/model-squeezenet`](https://github.com/runwayml/model-squeezenet) GitHub repository to your own GitHub user account so that you can follow along with the tutorial.

#### 1. Create a `runway_model.py` model server file

SqueezeNet is a neural network architecture used for computer vision tasks, optimized for mobile/embedded devices. [PyTorch](https://github.com/pytorch/vision) includes an implementation of SqueezeNet pre-trained on ImageNet that can be used out-of-the-box for object recognition.

Here's the code to classify a single local image (via a hard-coded path) with pre-trained SqueezeNet:

```python
# model.py
import json
from PIL import Image
from torchvision import models, transforms
from torch.autograd import Variable

# Read the index -> label mapping from a file
labels = json.load(open('labels.json'))

# set up preprocessing transformations

normalize = transforms.Normalize(
   mean=[0.485, 0.456, 0.406],
   std=[0.229, 0.224, 0.225]
)

preprocess = transforms.Compose([
   transforms.Scale(256),
   transforms.CenterCrop(224),
   transforms.ToTensor(),
   normalize
])

# initialize the model and download pre-trained weights

model = models.squeezenet1_1(pretrained=True)

# open a local image
img = Image.open('still_life.jpg')

# preprocess the image and convert it into a tensor
img_tensor = preprocess(img)
img_tensor.unsqueeze_(0)
img_variable = Variable(img_tensor)

# do a forward pass and classify the image
fc_out = model(img_variable)
label = labels[str(fc_out.data.numpy().argmax())]
print(label)
# lemon
```

And here's the modified code for the same model wrapped in a server so that Runway can access it:


```diff
import json
from PIL import Image
from torchvision import models, transforms
from torch.autograd import Variable
+ import runway
+ from runway.data_types import image, text

labels = json.load(open('labels.json'))

normalize = transforms.Normalize(
   mean=[0.485, 0.456, 0.406],
   std=[0.229, 0.224, 0.225]
)

preprocess = transforms.Compose([
   transforms.Scale(256),
   transforms.CenterCrop(224),
   transforms.ToTensor(),
   normalize
])

+ @runway.setup
+ def setup():
+     return models.squeezenet1_1(pretrained=True)

+ @runway.command('classify', inputs={ 'photo': image() }, outputs={ 'label': text() })
+ def classify(model, input):
+     img = input['photo']
      img_tensor = preprocess(img)
      img_tensor.unsqueeze_(0)
      img_variable = Variable(img_tensor)
      fc_out = model(img_variable)
      label = labels[str(fc_out.data.numpy().argmax())]
+     return { 'label': label }

+ if __name__ == '__main__':
+     runway.run()
```
<p class='subtitle'>Contents of model.py</p>

Behind the scenes, the Runway Python SDK uses the `@runway.command()` decorator to set up a server with a `/classify` endpoint that accepts a base64-encoded image. The SDK then converts this image into a `PIL.Image` before passing it to the `classify()` function, which classifies it and returns a text label.

<p class="note"><b>NOTE:</b> You don't have to use the Runway Python SDK to set up the server. You can just use any server framework (e.g. Flask) and set up the classification endpoint manually. The SDK just makes things easier by taking care of parsing/serializing inputs/outputs and setting up the endpoints for you.</p>

##### Testing your `runway_model.py` locally

Before we configure our build environment and publish our model to Runway, we should test it locally to make sure that it works as expected.

```bash
## Optionally create and activate a Python 3 virtual environment
# virtualenv -p python3 venv && source venv/bin/activate

pip install -r requirements.txt

# Run the entrypoint script
python runway_model.py
```

You should see an output similar to this, indicating your model is running.

```
Setting up model...
Starting model server at http://0.0.0.0:8000...
```

We can test the model's `/classify` command by `POST`ing an image and having the model classify its contents for us.

```bash
# Base64 encode an image and save the encoded version as an environment variable
# to use in a POST request to the modelf
curl -o cat.jpg "https://runway.nyc3.cdn.digitaloceanspaces.com/documentation/tutorial_model_importing/cat.jpg"
BASE64_IMAGE=$(base64 -i cat.jpg | xargs echo "data:image/jpeg;base64," | sed "s/ //" )

# Make a POST request to the /classify command, receiving
curl http://0.0.0.0:8000/classify \
   -X POST \
   -H "content-type: application/json" \
   -d "{ \"photo\": \"${BASE64_IMAGE}\" }"
```

You should see the model classify the image and return a class label in JSON like so:

```
{"label": "tabby, tabby cat"}
```

Once you've confirmed your model works correctly locally, we can create a `runway.yml` config file before building the model remotely with Runway + GitHub.

#### 2. Write a `runway.yml` config file

Next we need to write a config file that defines the environment, dependencies, and build steps required to build and run our model. This file is written in [YAML](https://learnxinyminutes.com/docs/yaml/), a human-readable superset of JSON. Below is an example of the `runway.yml` file for Squeezenet:

```yaml
python: 3.6
cuda: 9.2
entrypoint: python runway_model.py
build_steps:
  - pip install -r requirements.txt
```

This file specifies the Python and CUDA versions to use, the entrypoint command which launches the Runway model and HTTP server, and a build step which installs the Python dependencies required to run our model. See the [Runway YAML reference page](https://sdk.runwayml.com/en/latest/runway_yaml_file.html) for a full list of config values supported in the `runway.yml` config file.

Peeking at the `requirements.txt` file reveals that the only dependencies for the SqueezeNet Runway model are PyTorch and PyTorch Vision, as well as the Runway Model SDK itself.

```
torch==1.0.0
torchvision==0.2.1
runway-python
```

<p class="note">
  <b>NOTE:</b> The <code>runway-python</code> module must always be installed via the <code>build_steps</code>. Failing to install this required dependency via the <code>runway.yml</code> file will cause all model builds to fail.
</p>

#### 3. Upload your code to a GitHub repository

Once your model has a `runway_model.py` and `runway.yml` config file it's time to upload your repository to GitHub. If you forked the `runway/model-squeezenet` repo at the beginning of this tutorial, you should already have a `git remote` that points to the forked repository on your GitHub user account. If you instead cloned the repo, or you started creating a model from scratch instead of using the `runwayml/model-squeezenet` repo, you should publish your code to GitHub as an open source repository now. For the remainder of this tutorial, we will assume the repository being ported is located at `https://github.com/YOUR_USERNAME/runway-model-tutorial`.

#### 4. Import the model

Once you've pushed your model to GitHub, it's time to import your model in the Runway app. Open Runway and select the _Create New Model From GitHub_ button on the lower left.

![Import Model #1](https://runway.nyc3.cdn.digitaloceanspaces.com/documentation/tutorial_model_importing/1_small.png)

Authorize Runway to access public data on your GitHub account.

![Import Model #2](https://runway.nyc3.cdn.digitaloceanspaces.com/documentation/tutorial_model_importing/2_small.png)
<img src="https://runway.nyc3.cdn.digitaloceanspaces.com/documentation/tutorial_model_importing/3_small.png" style="max-width: 50%; margin: auto; display: block;">

Back in the Runway app, select the repository that contains your Runway model; `runway-model-tutorial` in our case.

![Import Model #4](https://runway.nyc3.cdn.digitaloceanspaces.com/documentation/tutorial_model_importing/4_small.png)

Next you can edit the model name and add a category to your model. Other info settings are available to edit later.

![Import Model #5](https://runway.nyc3.cdn.digitaloceanspaces.com/documentation/tutorial_model_importing/5_small.png)

#### 5. Push a new commit to trigger a build

Once you've imported your model to Runway, you must push a new commit to GitHub to trigger a model build. When you do so, Runway will clone the code from your repository and use its `runway.yml` file to build a Docker image from your model. Each new commit pushed to the `master` branch of GitHub repository linked to a model will trigger a new model version to be built by Runway. Select the "Versions" tab in your model page to view all builds, past and present.

![Import Model #6](https://runway.nyc3.cdn.digitaloceanspaces.com/documentation/tutorial_model_importing/6_small.png)

Click on a model version to view details about the version builds. You will notice that the `default` switch is flipped automatically for the first build. This means that the model is now published by your Runway users and is available for others to use. You can set any successfully built version to be the `default` version, but you must have at least one default version. This is an intentional decision for the time being, as _Private Models_ is a feature that is still to come.

![Import Model #7](https://runway.nyc3.cdn.digitaloceanspaces.com/documentation/tutorial_model_importing/7_small.png)

You can view model logs during or after a build to debug your model build process.

![Import Model #8](https://runway.nyc3.cdn.digitaloceanspaces.com/documentation/tutorial_model_importing/8_small.png)

Once a model version has been successfully built you can add it to a workspace. You can also add any successful model versions to your personal workspace by hovering over the model version check mark in the "Versions" panel until it becomes a "+" icon, and then selecting it. This allows you to test new model versions before making them the default model version that is published to all Runway users.

### Edit Model Info

Once you've imported your model you should edit its model info to help other users understand who made it and what its used for. As the publisher of the model, you can do this from the model page using the "Edit Info" button. We recommend adding all of these fields for each model:

* **Model Name**: The name of the model. Alphanumeric values, underscores, and hyphens are allowed here. No spaces.
* **Tagline**: A short one-line description of what your model does.
* **Description**: A long description of what your model does. The description can be a paragraph or more and is interpreted as Markdown.
* **Images**: A collection of images showcasing your model. The first image you add in the "Gallery" tab will be used as the thumbnail for the image.
* **Attributions**: A list of names and respective roles for people or organizations that significantly contributed to a model. This usually includes authors or publishers of machine learning papers or frameworks. Be generous with attributions; give credit where credit is due. We intentionally differentiate between a model publisher (the Runway user that adds/publishes a model) and the original authors of the model code or paper. This provides flexibility and encourages remix, experimentation, and "forking" of models.
* **LICENSE**: The license defining how the model can be used.
* **Keywords**: A list of tags that can be used to find your model in Runway.
* **Performance Notes**: How can users expect the model to perform on GPU or CPU environments?
* **Code, Paper, and More**: A list of links to resources related to your model, like source code, ArXiv papers, blog posts, or more.

### Links

For more information about how to port a new or existing model to Runway, check out these resources:

* [Runway Model SDK Docs](https://sdk.runwayml.com): Documentation for the Python SDK used to port models
* [Runway Model Template](https://github.com/runwayml/model-template): A boilerplate model template that you can use as a starting point to port models
