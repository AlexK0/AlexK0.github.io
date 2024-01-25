---
title:  "Colorizing a black and white photo"
date:   2024-01-24 12:00:00 +0100
categories: [Blogging, ML]
tags: [ML]
---

We all have a treasure trove of old photos passed down from our parents, grandparents, and so on. Often, these photos are in black and white,
and sometimes we wish to bring them to life in color. There are numerous websites online where you can achieve this with just a few clicks right in your browser.
However, the problem with these sites lies in their many restrictions on quantity and quality.
Additionally, these services often require registration and subscriptions, which can be quite frustrating.
In this post, I'd like to share how you can colorize any number of photos using the fantastic tool,
[DeOldify](https://github.com/jantic/DeOldify), without the hassle of limitations, registrations, or subscriptions.

DeOldify has an [amazing introduction](https://github.com/jantic/DeOldify?tab=readme-ov-file#getting-started-yourself), and you may use it,
but for non-prepared people, it may seem a bit difficult. Here, I'd like to demonstrate a simpler way of using DeOldify.

## Preparing the environment

To get started, you'll need a computer running Linux or macOS. If you're using Windows, you can still proceed by utilizing the [Windows Subsystem for Linux (WSL)](https://learn.microsoft.com/en-us/windows/wsl/install).

There is a list of required packages for **Ubuntu**:
```bash
sudo apt update
sudo apt install python3-pip wget git ffmpeg libsm6 libxext6
```

You may use the list as a reference if you have another Linux distribution.

For **macOS** (unfortunately, this is an approximate list as I don't have a clear macOS for testing):
```bash
brew install python wget git ffmpeg libsm libxext
```

Create a separate folder named **_Sandbox_** to do everything in. You can choose any other name or even choose not to create anything and skip this step altogether.
```bash
# Optional
mkdir Sandbox
cd Sandbox
```

Then clone [DeOldify](https://github.com/jantic/DeOldify):
```bash
git clone https://github.com/jantic/DeOldify.git
```

DeOldify doesn't have any releases or tags, which means that we work with the [master](https://github.com/jantic/DeOldify/tree/master) branch.
I believe that everything should work with the latest master at any time.
But just in case, here is a revision number with which I tested everything: [be725ca6c5f411c47550df951546537d7202c9bc](https://github.com/jantic/DeOldify/commit/be725ca6c5f411c47550df951546537d7202c9bc). If you want, you may checkout to it:
```bash
# Optional
cd DeOldify
git checkout be725ca6c5f411c47550df951546537d7202c9bc
cd ..
```

After that install all python dependencies:
```bash
pip3 install --user -r DeOldify/requirements.txt
pip3 install --user -r DeOldify/requirements-colab.txt
pip3 install --user -r DeOldify/requirements-dev.txt
```

Next, create a directory for the models and download the model:
```bash
mkdir -p DeOldify/models
wget https://data.deepai.org/deoldify/ColorizeArtistic_gen.pth -O DeOldify/models/ColorizeArtistic_gen.pth
```

That's it! The environment is ready, the hardest part is behind us. Now, let's write a bit of Python code.

## Colorize everything

The most interesting and useful function for us is [ModelImageVisualizer.plot_transformed_image()](https://github.com/jantic/DeOldify/blob/be725ca6c5f411c47550df951546537d7202c9bc/deoldify/visualize.py#L91).
All we need to do is write code around it.

Here is my implementation:

```python
#!/usr/bin/python3

from deoldify import device
from deoldify.device_id import DeviceId

# choices:  CPU, GPU0...GPU7
device.set(device=DeviceId.GPU0)

from deoldify.visualize import *
import warnings

import sys
import os

warnings.filterwarnings("ignore", category=UserWarning, message=".*?Your .*? set is empty.*?")

# Play with this constant!
render_factor = 35

input_dir = sys.argv[1]
output_dir = sys.argv[2]

if not os.path.isdir(input_dir):
    print("input directory is not a directory or not exist")
    sys.exit(1)

if os.path.exists(output_dir):
    print("out directory is already exist")
    sys.exit(1)

os.makedirs(output_dir)

root_dir = Path(os.environ['PYTHONPATH'])
colorizer = get_image_colorizer(root_folder=root_dir, artistic=True)
for filename in os.listdir(input_dir):
    f = os.path.join(input_dir, filename)
    if os.path.isfile(f) and (f.endswith(".png") or f.endswith(".jpg") or f.endswith(".jpeg")):
        image_path = colorizer.plot_transformed_image(
            path=f,
            results_dir=Path(output_dir),
            render_factor=render_factor,
            compare=True,
            watermarked=False
        )
        print("{} ready".format(image_path))
```

Feel free to copy, paste, and modify it however you want.

The script reads all `.png`, `.jpg`, and `.jpeg` images from the directory passed as the first script argument and colorizes them,
saving the results into the directory passed as the second script argument.
Note that the output directory must not exist; the script creates it itself.
I saved the script in the `Sandbox/runner.py` file, but you may use a different location and name if you prefer.

Let's try it! I placed a black and white photo into `Sandbox/photos/in`, and the output directory is `Sandbox/photos/out`:
```bash
# from Sandbox directory
PYTHONPATH=./DeOldify/ python3 runner.py photos/in/ photos/out/
```

> The environment variable `PYTHONPATH` must refer to the [DeOldify](https://github.com/jantic/DeOldify) directory.
{: .prompt-info }

Here is the result:

![Black and white photo](/assets/posts-images/colorizing-photos/bw.jpg)
_Black and white photo_

![Colorized photo](/assets/posts-images/colorizing-photos/bw-color.jpg)
_Colorized photo_

![Original photo](/assets/posts-images/colorizing-photos/original.jpg)
_Original photo_
