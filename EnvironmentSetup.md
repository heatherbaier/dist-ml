# Environment Setup with pyenv (Optional)

## MacOS

This guide covers how to make pyenv, conda, Jupyter, and VS Code all work together on Mac OSX. Windows users will likely have to find a better solution 

### Part 1: Homebrew and pyenv Installation 

For this part, the order of tasks will like this once done: Homebrew install —> pyenv Install —> pyenv install Python 3 —> pyenv global Python3 —> brew install packages

1. Homebrew Install: https://brew.sh

The guide on the site is quite comprehensive and if you run into a pinch, google probably has a solution to your issue. 

2. Pyenv Install: https://github.com/pyenv/pyenv#homebrew-in-macos 

This guide will take you through using "brew install" to install pyenv. This step is **CRITICAL** as it's what allows pyenv to run under homebrew and properly access homebrew packages, instead of homebrew being installed under pyenv and running into pathing issues. 

3. Install Python 3 under pyenv

This step will install the latest version of python 3 and uses the command:&#x20;

```
pyenv install python3 
```
Other versions of python can then be installed using the same format `pyenv install VERSION.NUM`. 

4. Set the default environment to the newest version of Python

Run the following `pyenv global VERSION.NUMBER'

The version number will be the number for the recently downloaded version of python 3 and will have the format `3.10.6` or some other number. On MacOS, the default is python 2.7. Many newer things don't play nice with this as some things have changed from python 2 to python 3, thus necessitating this step

5. Package Installation

Now that pyenv is up and running with a version of python, you can install homebrew packages using `brew install PACKAGE.NAME`

### Part 2: Conda Installation and Deconfliction 

Normally, conda's package management will conflict with pyenv. However, there is a way around it. Much credit goes to https://www.soudegesu.com/en/python/pyenv/anaconda/ and https://ducfilan.wordpress.com/2017/11/13/manage-versions-of-python-with-pyenv-and-zsh-in-macos/ for helping me to figure this out. 

1. Environment Preparation 

Spin up pyenv and install the python version that you want Anaconda to run on. I used python 3.9.13, as this is what the class requires. 

2. Conda Installation

Brew install Anaconda **UNDER** your desired running pyenv environment. This can be set by running `pyenv local PYTHON.VERSION` with the version number for what python version anaconda will be running on.

Once installed, anytime you want to run anaconda, run `pyenv local ANACONDA.VERSION`. If you forget the version number, just run `pyenv versions` and anaconda will show up as a distinct version. 

3. Modify Shell Profiles

On MacOS versions above Catalina, we will need to modify the files .zprofile and .zshrc. For versions below Catalina, we will need to modify .bash_profile and .bashrc. For me, I used the first two. 

To open my profiles, I ran `open ~/.zprofile` and `open ~/.zshrc`. The same syntax will also apply to .bash_profile and .bashrc.


For your .something_profile, paste:

```
export PYENV_ROOT="$HOME/.pyenv"
command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"

```

Then paste into .zshrc or .bashrc:

```
# setting for pyenv

export PYENV_ROOT="${HOME}/.pyenv"

if [ -d "${PYENV_ROOT}" ]; then

    export PATH=${PYENV_ROOT}/bin:${PYENV_ROOT}/shims:${PATH}

    eval "$(pyenv init --path)"
    eval "$(pyenv init -)"
	

fi

# >>> conda initialize >>>
# !! Contents within this block are managed by 'conda init' !!
__conda_setup="$('/Users/aidanlucas/.pyenv/versions/anaconda3-2021.05/bin/conda' 'shell.zsh' 'hook' 2> /dev/null)"
if [ $? -eq 0 ]; then
    eval "$__conda_setup"
else
    if [ -f "/Users/aidanlucas/.pyenv/versions/anaconda3-2021.05/etc/profile.d/conda.sh" ]; then
        . "/Users/aidanlucas/.pyenv/versions/anaconda3-2021.05/etc/profile.d/conda.sh"
    else
        export PATH="/Users/aidanlucas/.pyenv/versions/anaconda3-2021.05/bin:$PATH"
    fi
fi
unset __conda_setup
# <<< conda initialize <<<

```

Reboot your shell. These steps will then force any subsequent shell to first run pyenv and then anaconda. IDEs like VS Code will also read these files and use them to help initalize their command line shells. 

Congratulations! You may now manage new python versions with pyenv and create new conda virtual environments as normal! Best of all, it should now be considerably easier to keep your python environments clean. 

### Part 3: Jupyter Notebook Management

This section is on how to install jupyter notebook and get the kernel to play nice with the system. Information on these steps is taken from https://medium.com/@nrk25693/how-to-add-your-conda-environment-to-your-jupyter-notebook-in-just-4-steps-abeab8b8d084

1. Install Jupyter

Run `conda install jupyternotebook` or `conda install jupyterlab`. This should give all conda environments access to jupyter. 

2. Create Virtual Environment

Use `conda create --name PROJECT_NAME` to create a virtual environment for a project. Then use `conda activate PROJECT_NAME` to start the environment. Then install any packages you need before proceeding to the next step.

3. Add environment to the Jupyter kernel 

To use environments with specific versions of python or packages on Jupyter, we need to add them to the Jupyter kernel. 

To do this, run `conda install -c anaconda ipykernel` and `python -m ipykernel install --user --name=firstEnv`. The first command adds the Jupyter kernel manager to your environment and the second adds your environment to the kernel.

### Part 4: VS Code Management 

NEED TO FINISH