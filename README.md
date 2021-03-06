# KB data lab

**Note:** This repository, API endpoint and correspondning site is under active development, is not fully (or at all) functional, and is provided as-is

## About

This repository aims to provide and demo tools for researchers in preparation of gaining access to digital archives on-premise at the National Library, or anyone else wanting to access collections without active copyright. There are two main ways to access digital objects: either by using the HTTP API directly or using the provided client written in Python. You can also create a Docker image based on the one below which will have the client installed, or use the install the client using `pip` or `conda` in your own container. The data available outside the National Library, currently on https://betalab.kb.se, does not have active copyright.

## Installation

## TLDR; - [pip](https://pypi.org/) or [conda](https://www.anaconda.com/distribution/)

To install the client module using `pip` simply run
```
pip install kblab-client
```

To install using `conda` instead use the following
```
conda install kblab-client
```

Or add it to your dependencies in `environment.yml`
```
dependencies:
    - pip:
        kblab-client
```

Then, see [examples](#examples) below.

## TLDR; - Docker version

Start environment using docker. The local directory `./data` will be mounted on `/data` in the container. Any change from within the container will be reflected in the local directory and vice versa.
```
docker container run -it repository.kb.se/lab/client /bin/bash
d8fg7sjf4i # python
```

Then, see [examples](#examples) below.

## From source

First check out the source code
```
git clone https://github.com/kungbib/kblab
cd kblab
```

Then either build and run the Docker image 
```
docker build .
docker run -it <image id> /bin/bash
```

Or install the required package and python client, optionally creating a virtual environment so as to not mess up you existing one.
```
python -m venv venv
source venv/bin/activate
pip install -r requirement.txt
(cd client && ./setup.py install)
```

Then, see [examples](#examples) below.

## API

The API is a simple REST-based API that delivers JSON(-LD-ish describing packages and/or files with the addition of a search endpoint.

### URIs

Examples
- https://betalab.kb.se/dark-29967/
- https://betalab.kb.se/dark-29967/_view
- https://betalab.kb.se/dark-29967/bib4345612_18620103_0_s_0001_alto.xml

### Finding packages

Packages may contain files of type `Structure`, `Content` or `Meta` which contain structure information, content and metadata respectively (see below for examples). The meta and content files are indexed and can be searched through the API. Content is indexed under `content` and metadata under `meta.*` and can be accesed either through the web interface or through the API. For example: 

**Example**: Get all packages tagged with `SOU` created in 1927: `tags:issue meta.created:1927`.

Content is indexed under `content` which is also the default prefix so the query `"olof palme" and "carl bildt"` will find any package where the phrase `"olof palme"` and `"carl bildt"` appear.

**Also:** see examples below.

### Data model

The National Library uses a package structure modeled on OAIS. A simplified representation in JSON-LD is provided as part of the response in addition to information about the structure of the material (e.g pages, covers), some metadata, links to physical object, etc.

Indexing is experimental at this point so verify your results. 

### Structure documents

```
{
    "@id": "#1",
    "@type": "Part",
    "derived_from": "https://.../1927_1(librisid_13483334).pdf",
    "has_part": [
        {
            "@id": "#1-1",
            "@type": "Page",
            "has_part": [
                {
                    "@id": "#1-1-1",
                    "@type": "Area"
                    "has_part": [
                        {
                            "@id": "#1-1-1-1",
                            "@type": "Text"
                        }
                    ]
                }
            ]
        }
    ]
}
```

### Content documents

```
[
    {
        "@id": "#1-1-1-1", 
        "content": "..."
    }
]
```

### Meta documents

```
{
    "created": "1923",
    "title": "An example"
    ...
}
```

## Python 3.7 client

### Initializing archive
```
from kblab import Archive

# connect to betalab. Use parameter auth=(username, password) for authentication
a = Archive('https://betalab.kb.se', auth=('username', 'hunter2'))
```

**Caveat**: if you get an error about "certificate verify failed" you may need to update the root certificates on you platform. You can also add the following lines to your code. Please not that this is **NOT ADVISED**, it is better to add the correct root certificates.
```
import kblab
kblab.VERIFY_CA=False
```

### Searching content and iterating over packages
```
for package_id in a.search({ 'content': 'test' }, max=20):
    package = a.get(package_id)

    # do something with package
    ...    
```

### Listing and getting package content
```
for file in package:
    content = package.open(f).read()
    ...
```

## Docker images

## Examples

### Word count from 25 (unordered) issues of Aftonbladet
```
from collections import Counter
from kblab import Archive
from json import load

a = Archive('https://betalab.kb.se/')
c = Counter()

# find a specific issue of Aftonbladet
for package_id in a.search({ 'label': 'AFTONBLADET' }, max=25):
    print(package_id)
    p = a.get(package_id)

    if 'content.json' in p:
        for part in load(p.get_raw(fname)):
            c.update(part.get('content', '').toupper().split())

for word,count in c:
    print(word, count, sep='\t')
```

### Parallelization
When processing large result sets parallelization can be crucial. This can be achieved either through using the `multiprocessing` module or the `map` method on the search result and parameter `multi=True`. A parallelized version in the example above could look like:

```
from collections import Counter
from kblab import Archive
from json import load
import kblab

a = Archive('https://betalab.kb.se/')
c = Counter()

def count(package_id):
    print(package_id)
    c = Counter()
    p = a.get(package_id)
        
    if 'content.json' in p:
        for part in load(p.get_raw(fname)):
            c.update(part.get('content', '').toupper().split())
    
    return c

# loop over 25 issues of Aftonbladet
for words in a.search({ 'label': 'AFTONBLADET' }, max=25).map(count, multi=True):
    c.update(words)

for word,count in c.items():
    print(word, count, sep='\t')
```

The number of processes is specified by the `processes` parameter, it defaults to the number of cores on the machine running the program. For optimal performance, and if the order of the result is not important, add parameter `ordered=False` to `map(...)`.

Parallelization using `multiprocessing.Pool` would look something like this:
```
...
from multiprocessing import Pool

def f(package_id):
    # same as above
    ...

with Pool() as pool:
    for words in pool.imap(f, a.search({ 'label': 'AFTONBLADET' }, max=25)):
        c.update(words)

...
```

## IIIF support

Images in the archive can either be downloaded and dealt with directly in full resolution or they can be cropped and scaled using the [IIIF](https://iiif.io/) protocol.

<!--
### Manifests

For same packages IIIF-[manifests](https://iiif.io/api/presentation/2.0/#manifest) can be accessed by adding `/_manifest` to a URI. See example below.

-->
