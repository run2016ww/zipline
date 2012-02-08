# qbt - Quantopian Backtesting Services

##Development Setup

Install the necessary python libraries. Because there are dependencies between the various C packages that don't seem to be handled by ```pip install -r```. So, we provide a helper to install the libraries as listed in requirements.txt and requirements_dev.txt:

	```
	./ordered_pip.sh requirements_sci.txt #go get coffee, this will compile a heap of C/C++ code
	./ordered_pip.sh requirements.txt
	./ordered_pip.sh requirements_dev.txt
	```
	
You need to have zeromq installed - http://www.zeromq.org/intro:get-the-software. 

### Tooling hints
QBT relies heavily on scientific python components (numpy, scikit, pandas, matplotlib, ipython, etc). Tooling up can be a pain, and it often involves managing a configuration including your OS, c/c++/fortran compilers, python version, and versions of numerous modules. I've found the following tools absolutely indispensable: 

- some kind of package manager for your platform. package managers generally give you a way to search, install, uninstall, and check currently installed packages. They also do a great job of managing dependencies.
   - linux: yum/apt-get
   - mac: homebrew/macport/fink (I highly recommend homebrew: https://github.com/mxcl/homebrew) 
   - windows: probably best if you use a complete distribution, like: enthought, ActiveState, or Python(x,y)
- Python also provides good package management tools to help you manage the components you install for Python.
   - pip 
   - easy_install/setuptools. I have always used setuptools, and I've been quite happy with it. Just remember that setuptools is coupled to your python version. 
- virtualenv and virtualenvwrapper are your very best friends. They complement your python package manager by allowing you to create and quickly switch between named configurations.
    - *Install all the versions of Python you like to use, but install setuptools, virtualenv, and virtualenvwrapper with the very latest python.* Use the latest python to install the latest setuptools, and the latest setuptools to install virtualenv and virtualenvwrapper. virtualenvwrapper allows you to specify the python version you wish to use (mkvirtualenv -p <python executable> <env name>), so you can create envs of any python denomination.

### Mac OS hints

Scientific python on the Mac can be a bit confusing because of the many independent variables. You need to have several components installed, and be aware of the versions of each:

- XCode. XCode includes the gcc and g++ compilers and architecture specific assemblers. Your version of XCode will determine which compilers and assemblers are available. The most common issue I encountered with scientific python libraries is compilation errors of underlying C code. Most scientific libraries are optimized with C routines, so this is a major hurdle. In my environment (XCode 4.0.2 with iOS components installed) I ran into problems with -arch flags asking for power pc (-arch ppc passed to the compiler). Read this stackoverflow to see how to handle similar problems: http://stackoverflow.com/questions/5256397/python-easy-install-fails-with-assembler-for-architecture-ppc-not-installed-on
- gfortran 	- you need this to build numpy. With brew you can install with just: ```brew install gfortran```
- umfpack 	- you need this to build scipy. ```brew install umfpack```
- swig		- you need this to build scipy. ```brew install swig```
- hdf5	 	- you need this to build tables. ```brew install hdf5```
- zeromq 	- you need this to run qbt. ```brew install zmq``` 

### Database and Collections expected in MongoDB ###
QBT requires a running mongodb instance with a few collections:

- user collection. See handlers.BaseHandler and handlers.LoginHandler for code using this collection. Documents must have:
	- email - standard email address
	- salt - sha256 hex of: datetime.utcnow()--password 
	- encrypted_password - an sha256 hex digest of: salt--password
	- _id - standard issue mongodb primary key

## Authenticating

## Requesting a Backtest

#Data Sources
The Backtest can handle multiple concurrent data sources. QBT will start a subprocess to run each datasource, and merge all events from all sources into a single serial feed, ordered by date.

Data sources have events with very different frequencies. For example, liquid stocks will trade many times per minute, while illiquid stocks may trade just once a day. In order to serialize events from all sources into a single feed, qbt loads events from all sources into memory, then sorts. The communication happens like this:
1. QBT requests the next event from each data source, ignoring date (i.e. just next in sequence for all)
2. Using the earliest date from all the events from all sources, QBT then asks for "next after <date>" from all sources. 
3. All datasources send all events in their history from before <date>, moving their internal pointer forward to the next unsent event.
4. QBT merges all events in memory
5. goto 1!
