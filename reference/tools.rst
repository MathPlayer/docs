.. _tools:

Tools
=====

Under the tools module there are several functions and utilities that can be used in conan package
recipes:

.. code-block:: python
   :emphasize-lines: 2

    from conans import ConanFile
    from conans import tools

    class ExampleConan(ConanFile):
        ...

.. _cpu_count:

tools.cpu_count()
-----------------

.. code-block:: python

    def tools.cpu_count()

Returns the number of CPUs available, for parallel builds. If processor detection is not enabled, it will safely return 1.
Can be overwritten with the environment variable ``CONAN_CPU_COUNT`` and configured in the :ref:`conan.conf file<conan_conf>`.

tools.vcvars_command()
----------------------

.. code-block:: python

    def vcvars_command(settings, arch=None, compiler_version=None, force=False, vcvars_ver=None,
                       winsdk_version=None)

Returns, for given settings, the command that should be called to load the Visual
Studio environment variables for a certain Visual Studio version. It wraps thefunctionality of
`vcvarsall <https://docs.microsoft.com/en-us/cpp/build/building-on-the-command-line>`_ but
does not execute the command, as that typically have to be done in the same command as the compilation,
so the variables are loaded for the same subprocess. It will be typically used in the ``build()``
method, like this:

.. code-block:: python

    from conans import tools

    def build(self):
        if self.settings.build_os == "Windows":
            vcvars = tools.vcvars_command(self.settings)
            build_command = ...
            self.run("%s && configure %s" % (vcvars, " ".join(args)))
            self.run("%s && %s %s" % (vcvars, build_command, " ".join(build_args)))

The ``vcvars_command`` string will contain something like ``call "%vsXX0comntools%../../VC/vcvarsall.bat"`` for the
corresponding Visual Studio version for the current settings.

This is typically not needed if using ``CMake``, as the cmake generator will handle the correct
Visual Studio version.

If **arch** or **compiler_version** is specified, it will ignore the settings and return the command
to set the Visual Studio environment for these parameters.

Parameters:
    - **settings** (Required): Conanfile settings. Use ``self.settings``.
    - **arch** (Optional, Defaulted to ``None``): Will use ``settings.arch``.
    - **compiler_version** (Optional, Defaulted to ``None``): Will use ``settings.compiler.version``.
    - **force** (Optional, Defaulted to ``False``): Will ignore if the environment is already set for a different Visual Studio version.
    - **winsdk_version** (Optional, Defaulted to ``None``): Specifies the version of the Windows SDK to use.
    - **vcvars_ver** (Optional, Defaulted to ``None``): Specifies the Visual Studio compiler toolset to use.

tools.vcvars_dict()
-------------------

.. code-block:: python

    vcvars_dict(settings, arch=None, compiler_version=None, force=False, filter_known_paths=False,
                vcvars_ver=None, winsdk_version=None, only_diff=True)

Returns a dictionary with the variables set by the **tools.vcvars_command**.

.. code-block:: python

    from conans import tools

    def build(self):
        env_vars = tools.vcvars_dict(self.settings):
        with tools.environment_append(env_vars):
            # Do something


Parameters:
    - Same as ``vcvars_command``.
    - **filter_known_paths** (Optional, Defaulted to ``False``): When True, the function will only keep the PATH
      entries that follows some known patterns, filtering all the non-Visual Studio ones. When False,
      it will keep the PATH will all the system entries.
    - **only_diff** (Optional, Defaulted to ``True``): Returns only the variables set by
      ``vcvarsall`` and not the whole environment.


tools.vcvars()
--------------

.. code-block:: python

    vcvars(settings, arch=None, compiler_version=None, force=False, filter_known_paths=False)

.. note::

    This context manager tool has no effect if used in a platform different from Windows.

This is a context manager that allows to append to the environment all the variables set by the **tools.vcvars_dict()**.
You can replace **tools.vcvars_command()** and use this context manager to get a cleaner way to activate the Visual Studio
environment:

.. code-block:: python

    from conans import tools

    def build(self):
        with tools.vcvars(self.settings):
            do_something()

.. _build_sln_commmand:


tools.build_sln_command() (DEPRECATED)
--------------------------------------

.. warning::

    This tool is deprecated and will be removed in Conan 2.0.
    Use :ref:`MSBuild()<msbuild>` build helper instead.

.. code-block:: python

    def build_sln_command(settings, sln_path, targets=None, upgrade_project=True, build_type=None,
                          arch=None, parallel=True, toolset=None, platforms=None)

Returns the command to call `devenv` and `msbuild` to build a Visual Studio project.
It's recommended to use it along with ``vcvars_command()``, so that the Visual Studio tools will be in path.

.. code-block:: python

    from conans import tools

    def build(self):
        build_command = build_sln_command(self.settings, "myfile.sln", targets=["SDL2_image"])
        command = "%s && %s" % (tools.vcvars_command(self.settings), build_command)
        self.run(command)

Parameters:
    - **settings** (Required): Conanfile settings. Use "self.settings".
    - **sln_path** (Required):  Visual Studio project file path.
    - **targets** (Optional, Defaulted to ``None``):  List of targets to build.
    - **upgrade_project** (Optional, Defaulted to ``True``): If ``True``, the project file will be upgraded if the project's VS version is
      older than current. When :ref:`CONAN_SKIP_VS_PROJECTS_UPGRADE<env_var_conan_skip_vs_project_upgrade>` environment variable is set to
      ``True``/``1``, this parameter will be ignored and the project won't be upgraded.
    - **build_type** (Optional, Defaulted to ``None``): Override the build type defined in the settings (``settings.build_type``).
    - **arch** (Optional, Defaulted to ``None``): Override the architecture defined in the settings (``settings.arch``).
    - **parallel** (Optional, Defaulted to ``True``): Enables VS parallel build with ``/m:X`` argument, where X is defined by CONAN_CPU_COUNT environment variable
      or by the number of cores in the processor by default.
    - **toolset** (Optional, Defaulted to ``None``): Specify a toolset. Will append a ``/p:PlatformToolset`` option.
    - **platforms** (Optional, Defaulted to ``None``): Dictionary with the mapping of archs/platforms from Conan naming to another one. It
      is useful for Visual Studio solutions that have a different naming in architectures. Example: ``platforms={"x86":"Win32"}`` (Visual
      solution uses "Win32" instead of "x86"). This dictionary will update the default one:

      .. code-block:: python

          msvc_arch = {'x86': 'x86',
                       'x86_64': 'x64',
                       'armv7': 'ARM',
                       'armv8': 'ARM64'}

.. _msvc_build_command:


tools.msvc_build_command() (DEPRECATED)
---------------------------------------

.. warning::

    This tool is deprecated and will be removed in Conan 2.0.
    Use :ref:`MSBuild()<msbuild>`.get_command() instead.


.. code-block:: python

    def msvc_build_command(settings, sln_path, targets=None, upgrade_project=True, build_type=None,
                           arch=None, parallel=True, force_vcvars=False, toolset=None, platforms=None)

Returns a string with a joint command consisting in setting the environment variables via ``vcvars.bat`` with the above
``tools.vcvars_command()`` function, and building a Visual Studio project with the ``tools.build_sln_command()`` function.

Parameters:
    - Same parameters as the above :ref:`tools.build_sln_command()<build_sln_commmand>`.
    - **force_vcvars**: Optional. Defaulted to False. Will set ``vcvars_command(force=force_vcvars)``.

tools.unzip()
-------------

.. code-block:: python

    def unzip(filename, destination=".", keep_permissions=False)

Function mainly used in ``source()``, but could be used in ``build()`` in special cases, as
when retrieving pre-built binaries from the Internet.

This function accepts ``.tar.gz``, ``.tar``, ``.tzb2``, ``.tar.bz2``, ``.tgz`` and ``.zip`` files, 
and decompress them into the given destination folder (the current one by default).

.. code-block:: python

    from conans import tools

    tools.unzip("myfile.zip")
    # or to extract in "myfolder" sub-folder
    tools.unzip("myfile.zip", "myfolder")

You can keep the permissions of the files using the ``keep_permissions=True`` parameter.

.. code-block:: python

    from conans import tools

    tools.unzip("myfile.zip", "myfolder", keep_permissions=True)

Parameters:
    - **filename** (Required): File to be unzipped.
    - **destination** (Optional, Defaulted to ``"."``): Destination folder for unzipped files.
    - **keep_permissions** (Optional, Defaulted to ``False``): Keep permissions of files. **WARNING:** Can be dangerous if the zip
      was not created in a NIX system, the bits could produce undefined permission schema. Use only this option if you are sure that
      the zip was created correctly.

tools.untargz()
---------------

.. code-block:: python

    def untargz(filename, destination=".")

Extract tar gz files (or in the family). This is the function called by the previous ``unzip()``
for the matching extensions, so generally not needed to be called directly, call ``unzip()`` instead
unless the file had a different extension.

.. code-block:: python

    from conans import tools
    
    tools.untargz("myfile.tar.gz")
    # or to extract in "myfolder" sub-folder
    tools.untargz("myfile.tar.gz", "myfolder")

Parameters:
    - **filename** (Required): File to be unzipped.
    - **destination** (Optional, Defaulted to ``"."``): Destination folder for *untargzed* files.

tools.get()
-----------

.. code-block:: python

    def get(url, md5="", sha1="", sha256="")

Just a high level wrapper for download, unzip, and remove the temporary zip file once unzipped.
You can pass hash checking parameters: ``md5``, ``sha1``, ``sha256``. All the specified algorithms
will be checked, if any of them doesn't match, it will raise a ``ConanException``.

.. code-block:: python

    from conans import tools

    tools.get("http://url/file", md5='d2da0cd0756cd9da6560b9a56016a0cb')
    # also, specify a destination folder
    tools.get("http://url/file", destination="subfolder")

Parameters:
    - **url** (Required): URL to download
    - **md5** (Optional, Defaulted to ``""``): MD5 hash code to check the downloaded file.
    - **sha1** (Optional, Defaulted to ``""``): SHA1 hash code to check the downloaded file.
    - **sha256** (Optional, Defaulted to ``""``): SHA256 hash code to check the downloaded file.

.. _tools_get_env:

tools.get_env()
---------------

.. code-block:: python

   def get_env(env_key, default=None, environment=None)

Parses an environment and cast its value against the **default** type passed as an argument.

Following python conventions, returns **default** if **env_key** is not defined.

See an usage example with an environment variable defined while executing conan

.. code-block:: bash

   $ TEST_ENV="1" conan <command> ...

.. code-block:: python

   from conans import tools

   tools.get_env("TEST_ENV") # returns "1", returns current value
   tools.get_env("TEST_ENV_NOT_DEFINED") # returns None, TEST_ENV_NOT_DEFINED not declared
   tools.get_env("TEST_ENV_NOT_DEFINED", []) # returns [], TEST_ENV_NOT_DEFINED not declared
   tools.get_env("TEST_ENV", "2") # returns "1"
   tools.get_env("TEST_ENV", False) # returns True (default value is boolean)
   tools.get_env("TEST_ENV", 2) # returns 1
   tools.get_env("TEST_ENV", 2.0) # returns 1.0
   tools.get_env("TEST_ENV", []) # returns ["1"]

Parameters:
   - **env_key** (Required): environment variable name.
   - **default** (Optional, Defaulted to ``None``): default value to return if not defined or cast value against.
   - **environment** (Optional, Defaulted to ``None``): ``os.environ`` if ``None`` or environment dictionary to look for.

tools.download()
----------------

.. code-block:: python

    def download(url, filename, verify=True, out=None, retry=2, retry_wait=5, overwrite=False,
                 auth=None, headers=None)

Retrieves a file from a given URL into a file with a given filename. It uses certificates from a
list of known verifiers for https downloads, but this can be optionally disabled.

.. code-block:: python

    from conans import tools
    
    tools.download("http://someurl/somefile.zip", "myfilename.zip")

    # to disable verification:
    tools.download("http://someurl/somefile.zip", "myfilename.zip", verify=False)

    # to retry the download 2 times waiting 5 seconds between them
    tools.download("http://someurl/somefile.zip", "myfilename.zip", retry=2, retry_wait=5)

    # Use https basic authentication
    tools.download("http://someurl/somefile.zip", "myfilename.zip", auth=("user", "password"))

    # Pass some header
    tools.download("http://someurl/somefile.zip", "myfilename.zip", headers={"Myheader": "My value"})

Parameters:
    - **url** (Required): URL to download
    - **filename** (Required): Name of the file to be created in the local storage
    - **verify** (Optional, Defaulted to ``True``): When False, disables https certificate validation.
    - **out**: (Optional, Defaulted to ``None``): An object with a write() method can be passed to get the output, stdout will use if not specified.
    - **retry** (Optional, Defaulted to ``2``): Number of retries in case of failure.
    - **retry_wait** (Optional, Defaulted to ``5``): Seconds to wait between download attempts.
    - **overwrite**: (Optional, Defaulted to ``False``): When `True` Conan will overwrite the destination file if exists, if False it will raise.
    - **auth** (Optional, Defaulted to ``None``): A tuple of user, password can be passed to use HTTPBasic authentication. This is passed directly to the
      requests python library, check here other uses of the **auth** parameter: http://docs.python-requests.org/en/master/user/authentication
    - **headers** (Optional, Defaulted to ``None``): A dict with additional headers.

tools.ftp_download()
--------------------

.. code-block:: python

    def ftp_download(ip, filename, login="", password="")

Retrieves a file from an FTP server. Right now it doesn't support SSL, but you might implement it yourself using the standard python FTP library, and also if
you need some special functionality.

.. code-block:: python

    from conans import tools

    def source(self):
        tools.ftp_download('ftp.debian.org', "debian/README")
        self.output.info(load("README"))

Parameters:
    - **ip** (Required): The IP or address of the ftp server.
    - **filename** (Required): The filename, including the path/folder where it is located.
    - **login** (Optional, Defaulted to ``""``): Login credentials for the ftp server.
    - **password** (Optional, Defaulted to ``""``): Password credentials for the ftp server.

tools.replace_in_file()
-----------------------

.. code-block:: python

    def replace_in_file(file_path, search, replace, strict=True)

This function is useful for a simple "patch" or modification of source files. A typical use would
be to augment some library existing ``CMakeLists.txt`` in the ``source()`` method, so it uses
conan dependencies without forking or modifying the original project:

.. code-block:: python

    from conans import tools
    
    def source(self):
        # get the sources from somewhere
        tools.replace_in_file("hello/CMakeLists.txt", "PROJECT(MyHello)",
            '''PROJECT(MyHello)
               include(${CMAKE_BINARY_DIR}/conanbuildinfo.cmake)
               conan_basic_setup()''')

Parameters:
    - **file_path** (Required): File path of the file to perform the replace in.
    - **search** (Required): String you want to be replaced.
    - **replace** (Required): String to replace the searched string.
    - **strict** (Optional, Defaulted to ``True``): If ``True``, it raises an error if the searched string
      is not found, so nothing is actually replaced.

.. _tools_check_with_algorithm_sum:

tools.check_with_algorithm_sum()
--------------------------------

.. code-block:: python

    def check_with_algorithm_sum(algorithm_name, file_path, signature)

Useful to check that some downloaded file or resource has a predefined hash, so integrity and
security are guaranteed. Something that could be typically done in ``source()`` method after
retrieving some file from the internet.

Parameters:
    - **algorithm_name** (Required): Name of the algorithm to be checked.
    - **file_path** (Required): File path of the file to be checked.
    - **signature** (Required): Hash code that the file should have.

There are specific functions for common algorithms:

.. code-block:: python

    def check_sha1(file_path, signature)
    def check_md5(file_path, signature)
    def check_sha256(file_path, signature)

For example:

.. code-block:: python

    from conans import tools
    
    tools.check_sha1("myfile.zip", "eb599ec83d383f0f25691c184f656d40384f9435")

Other algorithms are also possible, as long as are recognized by python ``hashlib`` implementation,
via ``hashlib.new(algorithm_name)``. The previous is equivalent to:

.. code-block:: python

    from conans import tools

    tools.check_with_algorithm_sum("sha1", "myfile.zip",
                                    "eb599ec83d383f0f25691c184f656d40384f9435")

tools.patch()
-------------

.. code-block:: python

    def patch(base_path=None, patch_file=None, patch_string=None, strip=0, output=None)

Applies a patch from a file or from a string into the given path. The patch should be in diff (unified diff)
format. To be used mainly in the ``source()`` method.

.. code-block:: python

    from conans import tools

    tools.patch(patch_file="file.patch")
    # from a string:
    patch_content = " real patch content ..."
    tools.patch(patch_string=patch_content)
    # to apply in subfolder
    tools.patch(base_path=mysubfolder, patch_string=patch_content)
    
If the patch to be applied uses alternate paths that have to be stripped, like:

.. code-block:: diff

    --- old_path/text.txt\t2016-01-25 17:57:11.452848309 +0100
    +++ new_path/text_new.txt\t2016-01-25 17:57:28.839869950 +0100
    @@ -1 +1 @@
    - old content
    + new content

Then it can be done specifying the number of folders to be stripped from the path:

.. code-block:: python

    from conans import tools

    tools.patch(patch_file="file.patch", strip=1)

Parameters:
    - **base_path** (Optional, Defaulted to ``None``): Base path where the patch should be applied.
    - **patch_file** (Optional, Defaulted to ``None``): Patch file that should be applied.
    - **patch_string** (Optional, Defaulted to ``None``): Patch string that should be applied.
    - **strip** (Optional, Defaulted to ``0``): Number of folders to be stripped from the path.
    - **output** (Optional, Defaulted to ``None``): Stream object.

.. _environment_append_tool:

tools.environment_append()
--------------------------

.. code-block:: python

    def environment_append(env_vars)

This is a context manager that allows to temporary use environment variables for a specific piece of code
in your conanfile:

.. code-block:: python

    from conans import tools
    
    def build(self):
        with tools.environment_append({"MY_VAR": "3", "CXX": "/path/to/cxx"}):
            do_something()

The environment variables will be overridden if the value is a string, while it will be prepended if the value is a list. When the context
manager block ends, the environment variables will be unset.

Parameters:
    - **env_vars** (Required): Dictionary object with environment variable name and its value.

tools.chdir()
-------------

.. code-block:: python

    def chdir(newdir)

This is a context manager that allows to temporary change the current directory in your conanfile:

.. code-block:: python

    from conans import tools

    def build(self):
        with tools.chdir("./subdir"):
            do_something()

Parameters:
    - **newdir** (Required): Directory path name to change the current directory.

tools.pythonpath()
------------------

This tool is automatically applied in the conanfile methods unless :ref:`apply_env<apply_env>` is deactivated, so
any PYTHONPATH inherited from the requirements will be automatically available.

.. code-block:: python

    def pythonpath(conanfile)

This is a context manager that allows to load the PYTHONPATH for dependent packages, create packages
with python code, and reuse that code into your own recipes.

It is automatically applied

.. code-block:: python

    from conans import tools
    
    def build(self):
        with tools.pythonpath(self):
            from module_name import whatever
            whatever.do_something()


When the :ref:`apply_env<apply_env>` is activated (default) the above code could be simplified as:


.. code-block:: python

    from conans import tools

    def build(self):
        from module_name import whatever
        whatever.do_something()


For that to work, one of the dependencies of the current recipe, must have a ``module_name``
file or folder with a ``whatever`` file or object inside, and should have declared in its
``package_info()``:

.. code-block:: python

    from conans import tools
    
    def package_info(self):
        self.env_info.PYTHONPATH.append(self.package_folder)

Parameters:
    - **conanfile** (Required): Current ``ConanFile`` object.


tools.no_op()
-------------

.. code-block:: python

    def no_op()

Context manager that performs nothing. Useful to condition any other context manager to get a cleaner code:

.. code-block:: python

    from conans import tools

    def build(self):
        with tools.chdir("some_dir") if self.options.myoption else tools.no_op():
            # if not self.options.myoption, we are not in the "some_dir"
            pass

tools.human_size()
------------------

.. code-block:: python

    def human_size(size_bytes)

Will return a string from a given number of bytes, rounding it to the most appropriate unit: GB, MB, KB, etc.
It is mostly used by the conan downloads and unzip progress, but you can use it if you want too.

.. code-block:: python

    from conans import tools
    
    tools.human_size(1024)
    >> 1.0KB

Parameters:
    - **size_bytes** (Required): Number of bytes.

.. _osinfo_reference:

tools.OSInfo and tools.SystemPackageTool
----------------------------------------

These are helpers to install system packages. Check :ref:`method_system_requirements`.

.. _cross_building_reference:

tools.cross_building()
----------------------

.. code-block:: python

    def cross_building(settings, self_os=None, self_arch=None)

Reading the settings and the current host machine it returns ``True`` if we are cross building a conan package:

.. code-block:: python

    from conans import tools

    if tools.cross_building(self.settings):
        # Some special action

Parameters:
    - **settings** (Required): Conanfile settings. Use ``self.settings``.
    - **self_os** (Optional, Defaulted to ``None``): Current operating system where the build is being done.
    - **self_arch** (Optional, Defaulted to ``None``): Current architecture where the build is being done.

tools.get_gnu_triplet()
-----------------------

.. code-block:: python

    def get_gnu_triplet(os, arch, compiler=None)

Returns string with GNU like ``<machine>-<vendor>-<op_system>`` triplet.

Parameters:
    - **os** (Required): Operating system to be used to create the triplet.
    - **arch** (Required): Architecture to be used to create the triplet.
    - **compiler** (Optional, Defaulted to ``None``): Compiler used to create the triplet (only needed for Windows).

.. _run_in_windows_bash_tool:

tools.run_in_windows_bash()
---------------------------

.. code-block:: python

    def run_in_windows_bash(conanfile, bashcmd, cwd=None, subsystem=None, msys_mingw=True, env=None)

Runs an unix command inside a bash shell. It requires to have "bash" in the path.
Useful to build libraries using ``configure`` and ``make`` in Windows. Check :ref:`Windows subsytems <windows_subsystems>` section.

You can customize the path of the bash executable using the environment variable ``CONAN_BASH_PATH`` or the :ref:`conan.conf<conan_conf>` ``bash_path``
variable to change the default bash location.

.. code-block:: python

    from conans import tools

    command = "pwd"
    tools.run_in_windows_bash(self, command) # self is a conanfile instance

Parameters:
    - **conanfile** (Required): Current ``ConanFile`` object.
    - **bashcmd** (Required): String with the command to be run.
    - **cwd** (Optional, Defaulted to ``None``): Path to directory where to apply the command from.
    - **subsystem** (Optional, Defaulted to ``None`` will autodetect the subsystem). Used to escape the command according to the specified subsystem.
    - **msys_mingw** (Optional, Defaulted to ``True``) If the specified subsystem is MSYS2, will start it in MinGW mode (native windows development).
    - **env** (Optional, Defaulted to ``None``) You can pass a dict with environment variable to be applied **at first place** so they will have more priority than others.


tools.get_cased_path()
----------------------

.. code-block:: python

    get_cased_path(abs_path)


For Windows, for any ``abs_path`` parameter containing a case-insensitive absolute path, returns it case-sensitive, that is, with the real cased characters.
Useful when using Windows subsystems where the file system is case-sensitive.


tools.remove_from_path()
------------------------

.. code-block:: python

    remove_from_path(command)

This is a context manager that allows you to remove a tool from the PATH. Conan will locate the executable
(using ``tools.which()``) and will remove from the PATH the directory entry that contains it.
It's not necessary to specify the extension.

.. code-block:: python

    from conans import tools

    with tools.remove_from_path("make"):
        self.run("some command")


tools.unix_path()
-----------------

.. code-block:: python

    def unix_path(path, path_flavor=None)

Used to translate Windows paths to MSYS/CYGWIN unix paths like ``c/users/path/to/file``.

Parameters:
    - **path** (Required): Path to be converted.
    - **path_flavor** (Optional, Defaulted to ``None``, will try to autodetect the subsystem): Type of unix path to be returned. Options are ``MSYS``, ``MSYS2``, ``CYGWIN``, ``WSL`` and ``SFU``.

tools.escape_windows_cmd()
--------------------------

.. code-block:: python

    def escape_windows_cmd(command)

Useful to escape commands to be executed in a windows bash (msys2, cygwin etc).

- Adds escapes so the argument can be unpacked by ``CommandLineToArgvW()``.
- Adds escapes for cmmd.exe so the argument survives cmmd.exe's substitutions.

Parameters:
    - **command** (Required): Command to execute.

tools.sha1sum(), sha256sum(), md5sum()
--------------------------------------

.. code-block:: python

    def def md5sum(file_path)
    def sha1sum(file_path)
    def sha256sum(file_path)

Return the respective hash or checksum for a file:

.. code-block:: python

    from conans import tools

    md5 = tools.md5sum("myfilepath.txt")
    sha1 = tools.sha1sum("myfilepath.txt")

Parameters:
    - **file_path** (Required): Path to the file.

tools.md5()
-----------

.. code-block:: python

    def md5(content)

Returns the MD5 hash for a string or byte object:

.. code-block:: python

    from conans import tools

    md5 = tools.md5("some string, not a file path")

Parameters:
    - **content** (Required): String or bytes to calculate its md5.

tools.save()
------------

.. code-block:: python

    def save(path, content, append=False)

Utility function to save files in one line.
It will manage the open and close of the file and creating directories if necessary.

.. code-block:: python

    from conans import tools

    tools.save("otherfile.txt", "contents of the file")

Parameters:
    - **path** (Required): Path to the file.
    - **content** (Required): Content that should be saved into the file.
    - **append** (Optional, Defaulted to ``False``): If ``True``, it will append the content.

tools.load()
------------

.. code-block:: python

    def load(path, binary=False)

Utility function to load files in one line.
It will manage the open and close of the file, and load binary encodings.
Returns the content of the file.

.. code-block:: python

    from conans import tools

    content = tools.load("myfile.txt")

Parameters:
    - **path** (Required): Path to the file.
    - **binary** (Optional, Defaulted to ``False``): If ``True``, it reads the the file as binary code.

tools.mkdir(), tools.rmdir()
----------------------------

.. code-block:: python

    def mkdir(path)
    def rmdir(path)

Utility functions to create/delete a directory.
The existance of the specified directory is checked, so ``mkdir()`` will do nothing if the directory
already exists and ``rmdir()`` will do nothing if the directory does not exists.

This makes it safe to use these functions in the ``package()`` method of a ``conanfile.py``
when ``no_copy_source=True``.

.. code-block:: python

    from conans import tools
    
    tools.mkdir("mydir") # Creates mydir if it does not already exist
    tools.mkdir("mydir") # Does nothing
    
    tools.rmdir("mydir") # Deletes mydir
    tools.rmdir("mydir") # Does nothing

Parameters:
    - **path** (Required): Path to the directory.


tools.which()
-------------

.. code-block:: python

    def which(filename)

Returns the path to a specified executable searching in the ``PATH`` environment variable. If not found, it returns ``None``.

This tool also looks for filenames with following extensions if no extension provided:

- ``.com``, ``.exe``, ``.bat`` ``.cmd`` for Windows.
- ``.sh`` if not Windows.

.. code-block:: python

    from conans import tools

    abs_path_make = tools.which("make")

Parameters:
    - **filename** (Required): Name of the executable file. It doesn't require the extension of the executable.

tools.touch()
-------------

.. code-block:: python

    def touch(fname, times=None)

Updates the timestamp (last access and last modificatiion times) of a file.
This is similar to Unix' ``touch`` command, except the command fails if the file does not exist.

Optionally, a tuple of two numbers can be specified, which denotes the new values for the
'last access' and 'last modified' times respectively.

.. code-block:: python

    from conans import tools
    import time
   
    tools.touch("myfile")                            # Sets atime and mtime to the current time
    tools.touch("myfile", (time.time(), time.time()) # Similar to above
    tools.touch("myfile", (time.time(), 1))          # Modified long, long ago

Parameters:
    - **fname** (Required): File name of the file to be touched.
    - **times** (Optional, Defaulted to ``None``: Tuple with 'last access' and 'last modified' times.

tools.relative_dirs()
---------------------

.. code-block:: python

    def relative_dirs(path)

Recursively walks a given directory (using ``os.walk()``) and returns a list of all contained file paths
relative to the given directory.

.. code-block:: python

    from conans import tools

    tools.relative_dirs("mydir")

Parameters:
    - **path** (Required): Path of the directory.

tools.vswhere()
---------------

.. code-block:: python

    def vswhere(all_=False, prerelease=False, products=None, requires=None, version="",
                latest=False, legacy=False, property_="", nologo=True)

Wrapper of ``vswhere`` tool to look for details of Visual Studio installations. Its output is always
a list with a dictionary for each installation found.

.. code-block:: python

    from conans import tools

    vs_legacy_installations = tool.vswhere(legacy=True)

Parameters:
    - **all_** (Optional, Defaulted to ``False``): Finds all instances even if they are incomplete and may not launch.
    - **prerelease** (Optional, Defaulted to ``False``): Also searches prereleases. By default, only releases are searched.
    - **products** (Optional, Defaulted to ``None``): List of one or more product IDs to find. Defaults to Community, Professional, and
      Enterprise. Specify ``["*"]`` by itself to search all product instances installed.
    - **requires** (Optional, Defaulted to ``None``): List of one or more workload or component IDs required when finding instances. See
      https://docs.microsoft.com/en-us/visualstudio/install/workload-and-component-ids for a list of workload and component IDs.
    - **version** (Optional, Defaulted to ``""``): A version range for instances to find. Example: ``"[15.0,16.0)"`` will find versions 15.*.
    - **latest** (Optional, Defaulted to ``False``): Return only the newest version and last installed.
    - **legacy** (Optional, Defaulted to ``False``): Also searches Visual Studio 2015 and older products. Information is limited. This
      option cannot be used with either ``products`` or ``requires`` parameters.
    - **property_** (Optional, Defaulted to ``""``): The name of a property to return. Use delimiters ``.``, ``/``, or ``_`` to separate
      object and property names. Example: ``"properties.nickname"`` will return the "nickname" property under "properties".
    - **nologo** (Optional, Defaulted to ``True``): Do not show logo information.

tools.vs_comntools()
--------------------

.. code-block:: python

    def vs_comntools(compiler_version)

Returns the value of the environment variable ``VS<compiler_version>.0COMNTOOLS`` for the compiler version indicated.

.. code-block:: python

    from conans import tools

    vs_path = tools.vs_comntools("14")

Parameters:
    - **compiler_version** (Required): String with the version number: ``"14"``, ``"12"``...

tools.vs_installation_path()
----------------------------

.. code-block:: python

    def vs_installation_path(version, preference=None)

Returns the Visual Studio installation path for the given version. It uses ``tools.vswhere()`` and
``tool.vs_comntools()``. It will also look for the installation paths following
``CONAN_VS_INSTALLATION_PREFERENCE`` environment variable or the preference parameter itself. If the
tool is not able to return the path it returns ``None``.

.. code-block:: python

    from conans import tools

    vs_path_2017 = tools.vs_installation_path("15", preference=["Community", "BuildTools", "Professional", "Enterprise"])

Parameters:
    - **version** (Required): Visual Studio version to locate. Valid version numbers
      are strings: ``"10"``, ``"11"``, ``"12"``, ``"13"``, ``"14"``, ``"15"``...
    - **preference** (Optional, Defaulted to ``None``): Set to value of
      ``CONAN_VS_INSTALLATION_PREFERENCE`` or defaulted to
      ``["Enterprise", "Professional", "Community", "BuildTools"]``. If only set to one type of
      preference, it will return the installation path only for that Visual type and version,
      otherwise ``None``.

tools.replace_prefix_in_pc_file()
----------------------------------

.. code-block:: python

    def replace_prefix_in_pc_file(pc_file, new_prefix)

Replaces the ``prefix`` variable in a package config file ``.pc`` with the specified value.

.. code-block:: python

    from conans import tools

    lib_b_path = self.deps_cpp_info["libB"].rootpath
    tools.replace_prefix_in_pc_file("libB.pc", lib_b_path)

**Parameters:**
    - **pc_file** (Required): Path to the pc file
    - **new_prefix** (Required): New prefix variable value (Usually a path pointing to a package).

.. seealso::

    Check section integrations/:ref:`pkg-config and pc files<pc_files>` to know more.

tools.collect_libs()
---------------------

.. code-block:: python

    def collect_libs(conanfile, folder="lib")

Fetches a list of all libraries in the package folder. Useful to collect not inter-dependent
libraries or with complex names like ``libmylib-x86-debug-en.lib``.

.. code-block:: python

    from conans import tools

    def package_info(self):
        self.cpp_info.libs = tools.collect_libs(self)

**Parameters:**
    - **conanfile** (Required): A `ConanFile` object from which to get the `package_folder`.
    - **folder** (Optional, Defaulted to ``"lib"``): The subfolder where the library files are.

.. warning::

    This tool collects the libraries searching directly inside the package folder and returns them
    in no specific order. If libraries are inter-dependent, then package_info() method should order
    them to achieve correct linking order.

.. _pkgconfigtool:

tools.PkgConfig()
-----------------

.. code-block:: python

    class PkgConfig(object):

        def __init__(self, library, pkg_config_executable="pkg-config", static=False, msvc_syntax=False, variables=None)

Wrapper of the ``pkg-config`` tool.

.. code-block:: python

    from conans import tools

    with environment_append({'PKG_CONFIG_PATH': tmp_dir}):
        pkg_config = PkgConfig("libastral")
        print(pkg_config.cflags)
        print(pkg_config.cflags_only_I)
        print(pkg_config.variables)

Parameters of the constructor:
    - **library** (Required): Library (package) name, such as ``libastral``.
    - **pkg_config_executable** (Optional, Defaulted to ``"pkg-config"``): Specify custom pkg-config executable (e.g. for cross-compilation).
    - **static** (Optional, Defaulted to ``False``): Output libraries suitable for static linking (adds ``--static`` to ``pkg-config`` command line).
    - **msvc_syntax** (Optional, Defaulted to ``False``): MSVC compatibility (adds ``--msvc-syntax`` to ``pkg-config`` command line).
    - **variables** (Optional, Defaulted to ``None``): Dictionary of pkg-config variables (passed as ``--define-variable=VARIABLENAME=VARIABLEVALUE``).

**Properties:**

+-----------------------------+---------------------------------------------------------------------+
| PROPERTY                    | DESCRIPTION                                                         |
+=============================+=====================================================================+
| .cflags                     | get all pre-processor and compiler flags                            |
+-----------------------------+---------------------------------------------------------------------+
| .cflags_only_I              | get -I flags                                                        |
+-----------------------------+---------------------------------------------------------------------+
| .cflags_only_other          | get cflags not covered by the cflags-only-I option                  |
+-----------------------------+---------------------------------------------------------------------+
| .libs                       | get all linker flags                                                |
+-----------------------------+---------------------------------------------------------------------+
| .libs_only_L                | get -L flags                                                        |
+-----------------------------+---------------------------------------------------------------------+
| .libs_only_l                | get -l flags                                                        |
+-----------------------------+---------------------------------------------------------------------+
| .libs_only_other            | get other libs (e.g. -pthread)                                      |
+-----------------------------+---------------------------------------------------------------------+
| .provides                   | get which packages the package provides                             |
+-----------------------------+---------------------------------------------------------------------+
| .requires                   | get which packages the package requires                             |
+-----------------------------+---------------------------------------------------------------------+
| .requires_private           | get packages the package requires for static linking                |
+-----------------------------+---------------------------------------------------------------------+
| .variables                  | get list of variables defined by the module                         |
+-----------------------------+---------------------------------------------------------------------+

.. _tools_git:

tools.Git()
-----------

.. code-block:: python

    class Git(object):

        def __init__(self, folder=None, verify_ssl=True, username=None, password=None, force_english=True, runner=None):

Wrapper of the ``git`` tool.

Parameters of the constructor:

    - **folder** (Optional, Defaulted to ``None``): Specify a subfolder where the code will be cloned. If not specified it will clone in the current directory.
    - **verify_ssl** (Optional, Defaulted to ``True``): Verify SSL certificate of the specified **url**.
    - **username** (Optional, Defauted to ``None``): When present, it will be used as the login to authenticate with the remote.
    - **password** (Optional, Defauted to ``None``): When present, it will be used as the password to authenticate with the remote.
    - **force_english** (Optional, Defaulted to ``True``): The encoding of the tool will be forced to use ``en_US.UTF-8`` to ease the output parsing.
    - **runner** (Optional, Defaulted to ``None``): By default ``subprocess.check_output`` will be used to invoke the ``git`` tool.

Methods:

- **run(command)**:
    Run any "git" command. ``e.j run("status")``
- **get_url_with_credentials(url)**:
    Returns the passed url but containing the ``username`` and ``password`` in the URL to authenticate (only if ``username`` and ``password`` is specified)
- **clone(url, branch=None)**:
    Clone a repository. Optionally you can specify a branch. Note: If you want to clone a repository and the specified **folder** already exist you have to specify a ``branch``.
- **checkout(element)**:
    Checkout a branch, commit or tag.
- **get_remote_url(remote_name=None)**:
    Returns the remote url of the specified remote. If not ``remote_name`` is specified ``origin`` will be used.
- **get_revision()**:
    Gets the current commit hash.


.. _tools_apple:


tools.is_apple_os()
-------------------

.. code-block:: python

    def is_apple_os(os_)

Returns ``True`` if OS is an Apple one: Macos, iOS, watchOS or tvOS.

Parameters:
    - **os_** (Required): OS to perform the check. Usually this would be ``self.settings.os``.


tools.to_apple_arch()
---------------------

.. code-block:: python

    def to_apple_arch(arch)

Converts conan-style architecture into Apple-style architecture.

Parameters:
    - **arch** (Required): arch to perform the conversion. Usually this would be ``self.settings.arch``.

tools.apple_sdk_name()
----------------------

.. code-block:: python

    def apple_sdk_name(settings)

Returns proper SDK name suitable for OS and architecture you are building for (considering simulators).

Parameters:
    - **settings** (Required): Conanfile settings.


tools.apple_deployment_target_env()
-----------------------------------

.. code-block:: python

    def apple_deployment_target_env(os_, os_version)

Environment variable name which controls deployment target: ``MACOSX_DEPLOYMENT_TARGET``, ``IOS_DEPLOYMENT_TARGET``,
``WATCHOS_DEPLOYMENT_TARGET`` or ``TVOS_DEPLOYMENT_TARGET``.

Parameters:
    - **os_** (Required): OS of the settings. Usually ``self.settings.os``.
    - **os_version** (Required): OS version.

tools.apple_deployment_target_flag()
------------------------------------

.. code-block:: python

    def apple_deployment_target_flag(os_, os_version)

Compiler flag name which controls deployment target. For example: ``-mappletvos-version-min=9.0``

Parameters:
    - **os_** (Required): OS of the settings. Usually ``self.settings.os``.
    - **os_version** (Required): OS version.

tools.XCRun()
-------------

.. code-block:: python

    class XCRun(object):

        def __init__(self, settings, sdk=None):

XCRun wrapper used to get information for building.

Properties:
    - **sdk_path**: Obtain SDK path (a.k.a. Apple sysroot or -isysroot).
    - **sdk_version**: Obtain SDK version.
    - **sdk_platform_path**: Obtain SDK platform path.
    - **sdk_platform_version**: Obtain SDK platform version.
    - **cc**: Path to C compiler (CC).
    - **cxx**: Path to C++ compiler (CXX).
    - **ar**: Path to archiver (AR).
    - **ranlib**: Path to archive indexer (RANLIB).
    - **strip**: Path to symbol removal utility (STRIP).
