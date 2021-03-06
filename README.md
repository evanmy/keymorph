# KeyMorph: Robust Multi-modal Affine Registration via Unsupervised Keypoint Detection

Implementation of KeyMorph, an unsupervised end-to-end learning-based image registration framework that relies on automatically detecting corresponding keypoints. Our core insight is straightforward: matching keypoints between images can be used to obtain the optimal transformation via a differentiable closed-form expression. We use this observation to drive the unsupervised learning of anatomically-consistent keypoints from images. This not only leads to substantially more robust registration but also yields better interpretability, since the keypoints reveal which parts of the image are driving the final alignment. Moreover, KeyMorph can be designed to be equivariant under image translations and/or symmetric with respect to the input image ordering. We demonstrate the proposed framework in solving 3D affine registration of multi-modal brain MRI scans. Remarkably, we show that this strategy leads to consistent keypoints, even across modalities.

## Requirements
We tested our algorithm with ***Python 3.8*** and ***PyTorch 1.10*** and ***Torchvision 0.11.1***. Install the packages with `pip3 install -r requirement.txt`

## Decrompressing Trained Weights
The self-supervised pretraining, brain extractor and trained model weights are found in the `./data/` folder. Combine and decompress the files using:

`cat ./data/weights* | tar xzpvf -`

## TLDR
Keypoint registration using close-form solution (equation 2) in the paper can be done as follows:

```python
from functions import registration_tools as rt

# Predict keypoints
# model ouputs coordinate for each keypoint 
# this is a tensor [n_batch, 3, n_keypoints] values ranging -1 to 1 (pytorch grid convention)

moving_kp = model(x_moving)
target_kp = model(x_target)

# Close form
affine_matrix = rt.close_form_affine(moving_kp, target_kp)
inv_matrix = torch.zeros(x_moving.size(0), 4, 4)
inv_matrix[:, :3, :4] = affine_matrix
inv_matrix[:, 3, 3] = 1
inv_matrix = torch.inverse(inv_matrix)[:, :3, :]
grid = F.affine_grid(inv_matrix,
                     x.size(),
                     align_corners=False)

# Align
x_aligned = F.grid_sample(x_moving,
                          grid=grid,
                          mode='bilinear',
                          padding_mode='border',
                          align_corners=False)

```
## Step-by-Step Guide

### Dataset 
[A] Scripts in `./notebooks/[A] Download Data` will download the IXI data and perform some basic preprocessing

[B] Once the data is downloaded `./notebooks/[B] Brain extraction` can be used to extract remove non-brain tissue. 

[C] Once the brain has been extracted, we center the brain using `./notebooks/[C] Centering`. During training, we randomly introduce affine augmentation to the dataset. This ensure that the brain stays within the volume given the affine augmentation we introduce. It also helps during the pretraining step of our algorithm.

### Pretraining KeyMorph

This step greatly helps with the convergence of our model. As explained in section 3.3, we picked 1 subject and random points within the brain of that subject. We then introduce affine transformation to the subject brain and same transformation to the keypoints. In other words, this is a self-supervised task in where the network learns to predict the keypoints on a brain under random affine transformation. We found that initializing our model with these weights helps with the training.

 To pretrain run:`python pretraining.py`

### Training KeyMorph
We use the weights from the pretraining step to initialized our model. The pretraining weights we used in `./data/weights/pretrained_model.pth.tar`.
To train KeyMorph run. 

To train run: `python train.py`

### Evaluating KeyMorph
Once trained, this script goes through the data folder and randomly pick two images as moving and fixed pairs. It then introduces random affine transformation to the moving image and register the image to the fixed image. It outputs a dictionary containing the moving, fixed and aligned image. We provided the trained version of our model in  `./data/weights/trained_model.pth.tar`.

Examples of how to run the evaluation script:

`python eval.py 0 0` register T1 to T1 images

`python eval.py 1 0` register T1 to T2 images

`python eval.py 1 2` register T1 to PD images

... and so on

### Automatic Deliniation/Segmentation of the Brain
For evaluation, we use [SynthSeg](https://github.com/BBillot/SynthSeg) to automatically segment different brain regions. Follow their repository for detailed intruction on how to use the model. 

## Contact
Feel free to open an issue in github for any problems or questions.

## References
Evan, M. Yu, et al. "KeyMorph: Robust Multi-modal Affine Registration via Unsupervised Keypoint Detection." (2021).

Kleesiek and Urban et al. "Deep MRI brain extraction: A 3D convolutional neural network for skull stripping." NeuroImage (2016).

Billot, Benjamin, et al. "A learning strategy for contrast-agnostic MRI segmentation." MIDL (2020).



