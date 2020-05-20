# how does import work?

## finder object
the interpreter searches `sys.meta_path`, which is a list of `finder` objects. a `finder` tells the interpreter how to handle `import` statements.

here are some `finder` objects:
```python
[_frozen_importlib.BuiltinImporter,
 _frozen_importlib.FrozenImporter,
 _frozen_importlib_external.PathFinder,
 <six._SixMetaPathImporter at 0x10d16b9d0>]
```

## where do packages reside?
when the interpreter searches for a package, it will search the following paths in `sys.paths`, which is a python list of directory names.

examples include:
```python
['/Users/vincenttang/Dropbox/Temp',
 '/Users/vincenttang/anaconda3/lib/python37.zip',
 '/Users/vincenttang/anaconda3/lib/python3.7',
 '/Users/vincenttang/anaconda3/lib/python3.7/lib-dynload',
 '',
 '/Users/vincenttang/.local/lib/python3.7/site-packages',
 '/Users/vincenttang/anaconda3/lib/python3.7/site-packages',
 '/Users/vincenttang/anaconda3/lib/python3.7/site-packages/aeosa',
 '/Users/vincenttang/anaconda3/lib/python3.7/site-packages/IPython/extensions',
 '/Users/vincenttang/.ipython']
```

## what happens when you do `import foo`?
the interpreter calls `__import__(fname)`. `__import__(fname)` will operate according to the `finder` object in `sys.meta_path`.

`from piehub.kennethreitz import requests` is converted to `piehub.kennethreitz.requests`. the interpreter calls `__import__("piehub.kennethreitz.requests")`

first, the interpreter does `__import__("piehub.kennethreitz.requests")`, which calls the `Finder` object's module_name method, or `Finder.module_name(module_name)`.

"piehub.kennethreitz.requests is the module_name. since it meets the condition
```python
if module_name.startswith('piehub.'):
    return GithubComLoader()
```
this`Finder` object returns a `Loader` object to the interpreter.

the interpreter then calls `Loader.load_module(fullname)`, where fullname is 'piehub.kennethreitz.requests'.

the `Loader` checks if `piehub.kennethreitz.requests` is in the repository by calling `_is_installed("piehub.kennethreitz.requests")`. 
```python
def _is_installed(self, fullname):
    try:
        self._import_module(fullname)
        return True
    except ImportError:
        return False
```
this, in turn, calls `_import_module("piehub.kennethreitz.requests")`, or:
```python
def _import_module(self, fullname):
    """exists already so import.
    """
    actual_name = '.'.join(fullname.split('.')[2:])
    return __import__(actual_name)
```
this, in turn, calls `__import__("requests")`, which calls `Finder.find_module("requests")`. 

since `__import__("requests")` will not trigger our custom `Finder` because it does not start with "piehub".
```python
class GithubComFinder:
    def find_module(self, module_name, package_path):
        print("module_name is ", module_name)
        if module_name.startswith('piehub'):
            return GithubComLoader()
```
that means `Loader._install_module("piehub.kennethreitz.requests")`. 
```python
def _install_module(self, fullname):
    if not self._is_installed(fullname):
        url = fullname.replace('.', '/').replace('piehub', 'git+https://github.com', 1)
        command = "pip install " + url
        print("command is", command)
        os.system(command)
```
through some string manipulation, the interpreter finally calls `pip install git+https://github.com/kennethreitz/requests`.

now that the interpreter has pip installed requests, it needs to load it into the environment. it executes this:
```python
class Loader:
    def load_module(self, fullname):
        ...
        else:
            print('assigning', fullname, 'to path 3')
            module = self._import_module(fullname)
```
this in turn triggers
```python
class Loader:
    ...
    def _import_module(self, fullname):
        actual_name = '.'.join(fullname.split('.')[2:])
        print("checking", actual_name)
        return __import__(actual_name)
```
which calls `__import__("requests")`. this uses another `Finder` object (not ours), which in turn returns the `requests` object.

the `Loader` then binds this `requests` object to `sys.modules` with `sys.modules[fullname] = module`