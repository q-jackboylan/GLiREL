# A guide to making a release

This guide collects the steps we do in GLiREL to make a release on PyPI. They result from (variations of) hard-learned lessons and while following this guide is completely optional, it’s strongly recommended to do so. 🙂 This is a truncated version of the [SetFit](https://github.com/huggingface/setfit/blob/main/RELEASE.md) release guide, which is more exhaustive and does some additional steps.

### Preparation

```bash
pip install build twine
```

To be able to make a release for a given project, you’ll need an account on [PyPI](https://pypi.org/) and on [Test PyPI](https://test.pypi.org/). If you are making a release for an existing project, your username will need to be added to that project by one of the current maintainers on PyPI. Note that we strongly recommend enabling two-factor authentication on PyPI.

Additionally, it can be nice to familiarize yourself with [Semantic Versioning](https://semver.org/). This is a fairly strict document, but it provides a useful summary that library maintainers should follow:

> Given a version number MAJOR.MINOR.PATCH, increment the:
> 
> 1. MAJOR version when you make incompatible API changes
> 2. MINOR version when you add functionality in a backward compatible manner
> 3. PATCH version when you make backward compatible bug fixes
> 
> Additional labels for pre-release and build metadata are available as extensions to the MAJOR.MINOR.PATCH format.

The very first release should be "0.1.0".

## Releases

### Step 1: Adjust the version of your package

You should have the current version specified in [`glirel/__init__.py`](glirel/__init__.py). This version should be a dev version (e.g. `0.1.0.dev`) before you release, change it to the name of the version you are releasing:

```diff
- __version__ = "0.4.0.dev"
+ __version__ = "0.4.0"
```

Commit the changes on your release branch and push them:

```bash
git add glirel
git commit -m "Release: v{VERSION}"
git push -u origin main
```

### Step 2: (Optional) Make sure all tests pass

If you add tests, then you should also add CI, e.g. like this [`tests.yaml`](https://github.com/tomaarsen/SpanMarkerNER/blob/main/.github/workflows/tests.yaml) file. This will automatically run tests whenever you make changes, it can be very useful. Make sure all tests that you may have pass before proceeding to the next step.

### Step 3: Add a tag for your release

A tag will flag the exact commit associated to your release (and be easier to remember than the commit hash!). The tag should be `v<VERSION>` so for instance `v4.12.0`. 

Here is how you can create and push your tag:

```bash
git tag v<VERSION>
git push --tags origin main
```

### Step 4: (Optional) Prepare the release notes

You can then put your release notes in a Draft Release on GitHub, in [https://github.com/jackboyla/GLiREL/releases](https://github.com/jackboyla/GLiREL/releases) and write a small paragraph highlighting each of the new features this release is adding.

You can use the previously created tag to let GitHub auto-generate some release notes based on recent pull requests.

### Step 5: Create the wheels for your release

This is what you'll upload on PyPI and what everyone will download each time they `pip install` your package.

Clean previous builds by deleting the `build` and `dist` directories or by running:

```
rm -rf build && rm -rf dist
```

Then run:

```bash
python -m build
```

This will create two folders, `build` and a `dist` with the new versions of your package. These contain a 1) source distribution and a 2) wheel.

### Step 6: Upload your package on PyPI test

**DO NOT SKIP THIS STEP!**

This is the most important check before actually releasing your package in the wild. Upload the package on PyPI test and check you can properly install it.

To upload it:

```bash
twine upload dist/* -r pypitest --repository-url=https://test.pypi.org/legacy/
```

You will be prompted for your username and password. 

If that doesn't work, you can create an API Token for your Test PyPI account and create a `~/.pypirc` account if it doesn't already exist, with:

```
[distutils]
  index-servers =
    glirel_test

[glirel_test]
  repository = https://test.pypi.org/legacy/
  username = __token__
  password = pypi-...
```
(some more details on this [here](https://pypi.org/help/#apitoken))

And then run:
```bash
twine upload dist/* -r glirel_test
```

Once that has uploaded the package, in a fresh environment containing all dependencies you need (tip: you can use Google Colab for this!), try to install your new package from the PyPI test server. First install all dependencies, and then your package.

```bash
python -m pip install torch transformers huggingface_hub flair seqeval tqdm
python -m pip install -i https://test.pypi.org/simple/ glirel
```

If everything works, you should be able to run this code:

```python
from glirel import GLiREL
import spacy

model = GLiREL.from_pretrained("jackboyla/glirel_base")

text = "Jack Dorsey's father, Tim Dorsey, is a licensed pilot. Jack met his wife Sarah Paulson in New York in 2003. They have one son, Edward."

nlp = spacy.load('en_core_web_sm')
doc = nlp(text)

labels = ['country of origin', 'licensed to broadcast to', 'parent', 'followed by', 'located in or next to body of water', 'spouse', 'child']

tokens = [token.text for token in doc]

ner = [[ent.start, ent.end, ent.label_, ent.text] for ent in doc.ents]
print(f"Entities detected: {ner}")

relations = model.predict_relations(tokens, labels, threshold=0.01, ner=ner)

print('Number of relations:', len(relations))

sorted_data_desc = sorted(relations, key=lambda x: x['score'], reverse=True)
print("\nDescending Order by Score:")
for item in sorted_data_desc:
    print(f"{item['head_text']} --> {item['label']} --> {item['tail_text']} | socre: {item['score']}")

```

### Step 7: Publish on PyPI

This cannot be undone if you messed up, so make sure you have run Step 6!

Once you’re fully ready, upload your package on PyPI:

```bash
twine upload dist/* -r pypi
```

You will be prompted for your username and password, unless you're using the recommended [PyPI API token](https://pypi.org/help/#apitoken). 

### Step 8: (Optional) Publish your release notes

Go back to the draft you did at step 4 ([https://github.com/jackboyla/GLiREL/releases](https://github.com/jackboyla/GLiREL/releases)) and publish them.

### Step 9: Bump the dev version on the main branch

You’re almost done! Just go back to the `main` branch and change the dev version in [`glirel/__init__.py`](glirel/__init__.py) to the new version you’re developing, for instance `4.13.0.dev` if just released `4.12.0`.
