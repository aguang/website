---
title: Travis-CI for Python and R with conda
classes: wide
toc: true
---

I have been using [Travis-CI](travis-ci.org) for about a year or two now, and now that I've got it working pretty smoothly and know how to debug it I quite like it. Combined with [codecov](https://codecov.io/) it gives some of that RPG level-up satisfaction every time I see my code coverage badge increase and my build pass.

{% include base_path %}

{% capture fig_img %}
![Baby pumps fist up]({{ basepath }}/assets/img/pushed-to-repo-travis-ci-build-passed.jpg)
{% endcapture %}

<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption>The literal face and hand motions I've made when my Travis-CI builds finally passed.</figcaption>
</figure>

It's also more importantly a good indicator for others that your code is probably not be a hot mess and a half. However, it was the bane of my existence for a good 6 months while I was in the last stretch of my PhD. I spent more time debugging Travis than actually working on my code. ([transmissim](https://github.com/aguang/transmissim), simulations for HIV epidemics that include intra-host evolution) This is probably because I jumped into the deep end with code that relied on python, R, *and* C++ packages and pre-compiled binaries. Based on my lab's experience with Travis though, I also do think that as your project scales up, and as Travis continues to make changes that it can cause quite a lot of errors that are a pain to debug.

There's a lot of good guides on setting up Travis for a single language, and in my experience it isn't too rough. So here I'll just add to the mix some of my experiences with trying to set up Travis for multiple languages, in particular python and R, and subsequent debugging in Docker. Multiple language support for Travis probably isn't going to happen any time soon according to [this open issue](https://github.com/travis-ci/travis-ci/issues/4090) but it's obviously still possible, just a little more involved.

## How to set up Travis
The key and kind of only component needed to set up Travis is the `.travis.yml` file. Travis-CI has a really good [Getting Started guide](https://docs.travis-ci.com/user/getting-started/), but essentially copy one of their `.travis.yml` templates in the language of choice, place it in your Github repository, link your repository to Travis, and you're good to go. Your first commit post Travis-linking will trigger a build.

And that's it!

Well, ok there's a lot more, but that's all you need to get started. For a good guide on getting started in R, check out Julia Silge's [post on it]((https://juliasilge.com/blog/beginners-guide-to-travis/)).

### Some additional notes

The rest of this post is on setting up Travis to work with both R and python in Ubuntu on container-based infrastructure. Why so specific? Because of constraints with regards to Travis.

Why Ubuntu? Because Travis only has [support for Ubuntu Linux and OSX](https://docs.travis-ci.com/user/multi-os/), but at this time it only has support for Ubuntu for [python](https://docs.travis-ci.com/user/languages/python/). Why container-based infrastructure? Because that's the default [Build Environment](https://docs.travis-ci.com/user/reference/overview/) for Travis, and the `.travis.yml` file for it looks quite a bit different than when you have sudo-enabled infrastructure.

I find this kind of annoying, as [apparently](https://stackoverflow.com/questions/32535195/how-to-run-tests-on-centos-7-with-travis-ci) [have](https://github.com/travis-ci/travis-ci/issues/2312) [others](https://eddelbuettel.github.io/r-travis/) but I live with it since messing with setting up two languages in a single `.travis.yml` file is enough for me.

## Travis on multiple languages

OK, let's say you're me and want to write some code that takes advantage of both R packages and python packages. These packages are specialized enough (say, for molecular evolution and epidemiological transmission) that you haven't been able to find equivalents in either language, and you also don't want to spend time retooling packages when it probably won't contribute towards your publications or PhD. You know how it is. So you decide to throw them together with the help of [rpy2](https://rpy2.bitbucket.io/), an interface to R from python.

Since we are primarily calling R from python, we set up our `.travis.yml` file with the python language and a python3-only [build matrix](https://docs.travis-ci.com/user/customizing-the-build/#Build-Matrix):

~~~
language: python
python:
  - "3.4"
  - "3.5"
  - "3.6"
~~~

You can of course add additional python versions if you'd like as well as a nightly build. In this particular case Travis will create 3 containers with python3.4, python3.5, and python3.6 respectively for us and build and test our code on each. Since the builds are all in Ubuntu, in order to also get R on the system if you were sudo-enabled you would just run `sudo apt-get`. In a container-based environment, however, you need to follow Travis-CI's guide on [Adding APT Sources and Packages](https://docs.travis-ci.com/user/installing-dependencies/#Installing-Packages-on-Container-Based-Infrastructure).

There is however yet another option, and that is to use a conda environment.

### conda and Travis

[Conda](https://conda.io/docs/) is a great package manager for several languages, including Python, R and C++ that runs on Windows, macOS and Linux. Since it sets up a virtual environment for your code, you can set it up within Travis containers to provide an environment where you can easily install whatever you need, including `pytest`. To do so, just copy and paste the following code into your `.travis.yml` file under the `install:` field.

~~~
install:
  - wget https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh -O miniconda.sh;
  - bash miniconda.sh -b -p $HOME/miniconda
  - export PATH="$HOME/miniconda/bin:$PATH"
  - hash -r
  - conda config --set always_yes yes --set changeps1 no
  - conda update -q conda
  - conda info -a
  - conda create -q -n test-env python=$TRAVIS_PYTHON_VERSION
  - source activate test-env
 ~~~

After `test-env` is activated, to install packages just run `conda install -c $channel $package` or `pip install $package` as usual. To learn more about that, take a look [here](https://conda.io/docs/user-guide/tasks/manage-pkgs.html). Best practices are to install everything from conda if possible once you've set up a conda environment, but sometimes packages don't have conda recipes, or sometimes they conflict, so what can you do.

To take a look at my `.travis.yml` with a conda environment and subsequent build log for a passing commit, you can look [here](https://github.com/aguang/transmissim/blob/5a7d9b8c98025029b2af279f6380609b37ced5ca/.travis.yml) and [here](https://travis-ci.org/aguang/transmissim/jobs/309341101). It includes some additional stuff in the `before_script` for running a pre-compiled executable, `pytest` and `codecov` but otherwise is basically the same. Ironically my current build is no longer passing however due to what appears to be issues with the `.travis.yml` file, which leads us to...

## Debugging Travis with Docker

Often times you'll run into problems with your Travis build that have nothing to do with your code and everything to do with configurations in the `.travis.yml` file, such as forgotten package dependencies, package dependency conflicts or typos in your `before_script` that still break the Travis build although your project itself may not be broken. These have all happened to me. However, the more difficult to resolve problems for me have actually had to do with Travis itself making changes to how it works or specifics of different runtime environments. For example, Travis switching from sudo-enabled environments to container-based environments as a default led to some serious frustration in my sudo-based build. Or as another example, it so happens that Python builds are not available in the macOS environment, leading to a lot of confusion as I tried to understand why the pre-compiled macOS binaries I downloaded would not run.

Since pushing minute versions of commits to Travis and waiting to see if they work takes forever and is extremely frustrating, especially if you have multiple errors in your `yaml` file, so what you'll want to do instead is to debug Travis with Docker. Travis provides a nice guide to that [here](https://docs.travis-ci.com/user/common-build-problems/#Troubleshooting-Locally-in-a-Docker-Image), and it's what I've been using.

Note that when it says in steps 8 and 9 to "Manually install dependencies, if any" and "Manually run your Travis CI build command," what I do is literally run the commands from the job log. The `.travis.yml` file supplies a list of parameters to Travis, but it isn't actually a script indicating all of the exact commands to run. To find those you need to go the job log. So for example, in my package I have a build matrix consisting of python3.4, 3.5, and 3.6. The default python shipped with the Docker image for debugging is python2.7.

~~~
travis@57726524d0db:~/transmissim$ python --version
Python 2.7.6
~~~

In order to get 3.4, I follow the commands in the job log:

~~~
3.4 is not installed; attempting download
Downloading archive: https://s3.amazonaws.com/travis-python-archives/binaries/ubuntu/14.04/x86_64/python-3.4.tar.bz2
$ sudo tar xjf python-3.4.tar.bz2 --directory /
$ git clone --depth=50 --branch=master https://github.com/aguang/transmissim.git 
Adding APT Sources (BETA)
$ source ~/virtualenv/python3.4/bin/activate
$ python --version
Python 3.4.6
~~~

This translates on Docker to:

~~~
travis@57726524d0db:~/transmissim$ wget https://s3.amazonaws.com/travis-python-archives/binaries/ubuntu/14.04/x86_64/python-3.4.tar.bz2
--2018-02-28 20:51:35--  https://s3.amazonaws.com/travis-python-archives/binaries/ubuntu/14.04/x86_64/python-3.4.tar.bz2
Resolving s3.amazonaws.com (s3.amazonaws.com)... 52.216.131.181
Connecting to s3.amazonaws.com (s3.amazonaws.com)|52.216.131.181|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 51621316 (49M) [application/octet-stream]
Saving to: ‘python-3.4.tar.bz2’

100%[=======================================================================================>] 51,621,316  13.4MB/s   in 4.6s   

2018-02-28 20:51:39 (10.6 MB/s) - ‘python-3.4.tar.bz2’ saved [51621316/51621316]

travis@57726524d0db:~/transmissim$ sudo tar xjf python-3.4.tar.bz2 --directory /
travis@57726524d0db:~/transmissim$ source ~/virtualenv/python3.4/bin/activate
(python3.4.6) travis@57726524d0db:~/transmissim$ python --version
Python 3.4.6
(python3.4.6) travis@57726524d0db:~/transmissim$ 
~~~

So, now run all of the commands in the job log, figure out where your error is in your `.travis.yml` file, and fix it. Easy peasy, theoretically. :) In reality I'm still trying to figure out why I have so many dependency conflicts. :)))