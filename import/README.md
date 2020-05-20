# what happens when we `import foo`?

## upon init, fetch a `Finder`
the interpreter searches `sys.meta_path`, a list of `Finder` objects. these objects are eventually invoked by the interpreter when it sees `import` statements.

a `Finder`'s only job is to return a `Loader` if a specific condition is met. it is the `Loader` that does the heavy lifting.

here are `Finder` objects:
```python
# from sys.meta_path

[_frozen_importlib.BuiltinImporter,
 _frozen_importlib.FrozenImporter,
 _frozen_importlib_external.PathFinder,
 <six._SixMetaPathImporter at 0x10d16b9d0>]
```

## import foo
`from piehub.kennethreitz import requests` is converted to `piehub.kennethreitz.requests`. the interpreter calls `__import__("piehub.kennethreitz.requests")`. this invokes a `Finder` by calling the `Finder`'s module_name function, or `Finder.module_name(module_name)`. all `Finder` objects are expected to have this method.

because "piehub.kennethreitz.requests" meets the `Finder`'s condition, the `Finder` returns a `Loader` object.
```python
if module_name.startswith('piehub.'):
    return Loader()
```

## does the package exist?
the interpreter then calls `Loader.load_module("piehub.kennethreitz.requests")`.

the `Loader` checks if "piehub.kennethreitz.requests" exists by calling `Loader._is_installed("piehub.kennethreitz.requests")`:
```python
def _is_installed(self, fullname):
    try:
        self._import_module(fullname)
        return True
    except ImportError:
        return False
```
this, in turn, calls `_import_module("piehub.kennethreitz.requests")`:
```python
def _import_module(self, fullname):
    """exists already so import.
    """
    actual_name = '.'.join(fullname.split('.')[2:])
    return __import__(actual_name)
```
this, in turn, calls `__import__("requests")`, which calls `Finder.find_module("requests")`. 

`__import__("requests")` does not trigger our custom `Finder` because it does not start with "piehub".
```python
class Finder:
    def find_module(self, module_name, package_path):
        print("module_name is ", module_name)
        if module_name.startswith('piehub'):
            return Loader()
```
it triggers another `Finder` which returns a `Loader`, which tells searches for the package in the following paths in `sys.paths`:
```python
# sys.path

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
## install package from github
if the package doesnt exist, that means `Loader._install_module("piehub.kennethreitz.requests")`:
```python
def _install_module(self, fullname):
    if not self._is_installed(fullname):
        url = fullname.replace('.', '/').replace('piehub', 'git+https://github.com', 1)
        command = "pip install " + url
        os.system(command)  # pip install foo
```
through some string manipulation, the interpreter finally calls `pip install git+https://github.com/kennethreitz/requests`.

## loading newly-installed package as a module
once installed, `requests` is loaded as a `module` object with `Loader._import_module("requests")` and assigned to `sys.path`:
```python
class Loader:
    def load_module(self, fullname):
        ...
        else:
            module = self._import_module(fullname)
```
this in turn triggers:
```python
class Loader:
    ...
    def _import_module(self, fullname):
        actual_name = '.'.join(fullname.split('.')[2:])
        print("checking", actual_name)
        return __import__(actual_name)  # the real Finder returns the package as a module
```

## assigning module to sys.modules
the `Loader` binds the `requests` module object to `sys.modules`:
```python
sys.modules[fullname] = module
```
`sys.modules` is different than `globals()`. `requests` might import other modules (eg `simplejson`), which will be placed in `sys.modules` but not `globals()`. 

## test
if you do `__import__("requests")`, it should return a `requests` module object.