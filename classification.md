---
jupytext:
  text_representation:
    format_name: myst
kernelspec:
  display_name: Python 3
  name: python3
repository:
  url: https://github.com/emdupre/nha2020-nilearn
---

# Classification of age groups using functional connectivity

## Extracting signal from fMRI volumes

```{figure} images/masking.jpg
---
height: 300px
name: masking
---
Masking fMRI data.
```

Essentially, we can think about overlaying a 3D grid on an image.
Then, our mask tells us which cubes or “voxels” (like 3D pixels) to sample from.
Since our Nifti images are 4D files, we can’t overlay a single grid –
instead, we use a series of 3D grids (one for each volume in the 4D file),
so we can get a measurement for each voxel at each timepoint.
These are reflected in the shape of the matrix !
You can check this by checking the number of positive voxels in our brain mask.

## An example classification problem

This example compares different kinds of functional connectivity between
regions of interest: correlation, partial correlation, and tangent space
embedding.

The resulting connectivity coefficients can be used to
discriminate children from adults. In general, the tangent space embedding
**outperforms** the standard correlations: see {cite}`Dadi_2019`
for a careful study.

## Load brain development fMRI dataset and MSDL atlas

We study only 30 subjects from the dataset, to save computation time.

```{code-cell} python3
:tags: [hide-output]
import numpy as np
from matplotlib import pyplot as plt

from sklearn.metrics import accuracy_score
from sklearn.model_selection import StratifiedShuffleSplit
from sklearn.svm import LinearSVC

from nilearn import plotting
from nilearn.connectome import ConnectivityMeasure
from nilearn import input_data
from nilearn import datasets

# for NHA env, will need to update to include `data_dir='~/data'`
development_dataset = datasets.fetch_development_fmri(n_subjects=30)
```

We use probabilistic regions of interest (ROIs) from the MSDL atlas.

```{code-cell} python3
# for NHA env, will need to update to include `data_dir='~/data'`
msdl_data = datasets.fetch_atlas_msdl()
msdl_coords = msdl_data.region_coords
n_regions = len(msdl_coords)
print('MSDL has {0} ROIs, part of the following networks :\n{1}.'.format(
    n_regions, msdl_data.networks))
```

## Region signals extraction

To extract regions time series, we instantiate a
:class:`nilearn.input_data.NiftiMapsMasker` object and pass the atlas the
file name to it, as well as filtering band-width and detrending option.

```{code-cell} python3
masker = input_data.NiftiMapsMasker(
    msdl_data.maps, resampling_target="data", t_r=2, detrend=True,
    low_pass=.1, high_pass=.01, memory='nilearn_cache', memory_level=1).fit()
```

Then we compute region signals and extract useful phenotypic informations.

```{code-cell} python3
children = []
pooled_subjects = []
groups = []  # child or adult
for func_file, confound_file, phenotypic in zip(
        development_dataset.func,
        development_dataset.confounds,
        development_dataset.phenotypic):
    time_series = masker.transform(func_file, confounds=confound_file)
    pooled_subjects.append(time_series)
    if phenotypic['Child_Adult'] == 'child':
        children.append(time_series)
    groups.append(phenotypic['Child_Adult'])

print('Data has {0} children.'.format(len(children)))
```

## ROI-to-ROI correlations of children

The simpler and most commonly used kind of connectivity is correlation. It
models the full (marginal) connectivity between pairwise ROIs. We can
estimate it using :class:`nilearn.connectome.ConnectivityMeasure`.

```{code-cell} python3
correlation_measure = ConnectivityMeasure(kind='correlation')
```

From the list of ROIs time-series for children, the
`correlation_measure` computes individual correlation matrices.

```{code-cell} python3
correlation_matrices = correlation_measure.fit_transform(children)
```

All individual coefficients are stacked in a unique 2D matrix.

```{code-cell} python3
print('Correlations of children are stacked in an array of shape {0}'
      .format(correlation_matrices.shape))
```

as well as the average correlation across all fitted subjects.

```{code-cell} python3
mean_correlation_matrix = correlation_measure.mean_
print('Mean correlation has shape {0}.'.format(mean_correlation_matrix.shape))
```

We display the connectome matrices of the first 3 children

```{code-cell} python3
_, axes = plt.subplots(1, 3, figsize=(15, 5))
for i, (matrix, ax) in enumerate(zip(correlation_matrices, axes)):
    plotting.plot_matrix(matrix, tri='lower', colorbar=False, axes=ax,
                         title='correlation, child {}'.format(i))
```

The blocks structure that reflect functional networks are visible.

Now we display as a connectome the mean correlation matrix over all children.

```{code-cell} python3
plotting.plot_connectome(mean_correlation_matrix, msdl_coords,
                         title='mean correlation over all children')
```

## Studying partial correlations

We can also study **direct connections**, revealed by partial correlation
coefficients. We just change the `ConnectivityMeasure` kind

```{code-cell} python3
partial_correlation_measure = ConnectivityMeasure(kind='partial correlation')
partial_correlation_matrices = partial_correlation_measure.fit_transform(
    children)
```

Most of direct connections are weaker than full connections.

```{code-cell} python3
_, axes = plt.subplots(1, 3, figsize=(15, 5))
for i, (matrix, ax) in enumerate(zip(partial_correlation_matrices, axes)):
    plotting.plot_matrix(matrix, tri='lower', colorbar=False, axes=ax,
                         title='partial correlation, child {}'.format(i))
```

```{code-cell} python3
plotting.plot_connectome(
    partial_correlation_measure.mean_, msdl_coords,
    title='mean partial correlation over all children')
```

## Extract subjects variabilities around a group connectivity

We can use **both** correlations and partial correlations to capture
reproducible connectivity patterns at the group-level.
This is done by the tangent space embedding.

```{code-cell} python3
tangent_measure = ConnectivityMeasure(kind='tangent')
```

We fit our children group and get the group connectivity matrix stored as
in `tangent_measure.mean_`, and individual deviation matrices of each subject
from it.

```{code-cell} python3
tangent_matrices = tangent_measure.fit_transform(children)
```

`tangent_matrices` model individual connectivities as
**perturbations** of the group connectivity matrix `tangent_measure.mean_`.
Keep in mind that these subjects-to-group variability matrices do not
directly reflect individual brain connections. For instance negative
coefficients can not be interpreted as anticorrelated regions.

```{code-cell} python3
_, axes = plt.subplots(1, 3, figsize=(15, 5))
for i, (matrix, ax) in enumerate(zip(tangent_matrices, axes)):
    plotting.plot_matrix(matrix, tri='lower', colorbar=False, axes=ax,
                         title='tangent offset, child {}'.format(i))
```

The average tangent matrix cannot be interpreted, as individual matrices
represent deviations from the mean, which is set to 0.

## What kind of connectivity is most powerful for classification?

We will use connectivity matrices as features to distinguish children from
adults. We use cross-validation and measure classification accuracy to
compare the different kinds of connectivity matrices.
We use random splits of the subjects into training/testing sets.
StratifiedShuffleSplit allows preserving the proportion of children in the
test set.

```{code-cell} python3
kinds = ['correlation', 'partial correlation', 'tangent']
_, classes = np.unique(groups, return_inverse=True)
cv = StratifiedShuffleSplit(n_splits=15, random_state=0, test_size=5)
pooled_subjects = np.asarray(pooled_subjects)

scores = {}
for kind in kinds:
    scores[kind] = []
    for train, test in cv.split(pooled_subjects, classes):
        # *ConnectivityMeasure* can output the estimated subjects coefficients
        # as a 1D arrays through the parameter *vectorize*.
        connectivity = ConnectivityMeasure(kind=kind, vectorize=True)
        # build vectorized connectomes for subjects in the train set
        connectomes = connectivity.fit_transform(pooled_subjects[train])
        # fit the classifier
        classifier = LinearSVC().fit(connectomes, classes[train])
        # make predictions for the left-out test subjects
        predictions = classifier.predict(
            connectivity.transform(pooled_subjects[test]))
        # store the accuracy for this cross-validation fold
        scores[kind].append(accuracy_score(classes[test], predictions))
```

display the results

```{code-cell} python3
mean_scores = [np.mean(scores[kind]) for kind in kinds]
scores_std = [np.std(scores[kind]) for kind in kinds]

plt.figure(figsize=(6, 4))
positions = np.arange(len(kinds)) * .1 + .1
plt.barh(positions, mean_scores, align='center', height=.05, xerr=scores_std)
yticks = [k.replace(' ', '\n') for k in kinds]
plt.yticks(positions, yticks)
plt.gca().grid(True)
plt.gca().set_axisbelow(True)
plt.gca().axvline(.8, color='red', linestyle='--')
plt.xlabel('Classification accuracy\n(red line = chance level)')
plt.tight_layout()
```

This is a small example to showcase nilearn features. In practice such
comparisons need to be performed on much larger cohorts and several
datasets.
{cite}`Dadi_2019` showed that across many cohorts and clinical questions,
the tangent kind should be preferred.

```{bibliography} references.bib
:style: unsrt
```