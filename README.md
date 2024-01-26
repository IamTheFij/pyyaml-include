# pyyaml-include

[![GitHub tag](https://img.shields.io/github/tag/tanbro/pyyaml-include.svg)](https://github.com/tanbro/pyyaml-include)
[![Python Package](https://github.com/tanbro/pyyaml-include/workflows/Python%20package/badge.svg)](https://github.com/tanbro/pyyaml-include/actions?query=workflow%3A%22Python+package%22)
[![Documentation Status](https://readthedocs.org/projects/pyyaml-include/badge/?version=latest)](https://pyyaml-include.readthedocs.io/en/latest/)
[![PyPI](https://img.shields.io/pypi/v/pyyaml-include.svg)](https://pypi.org/project/pyyaml-include/)
[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=tanbro_pyyaml-include&metric=alert_status)](https://sonarcloud.io/dashboard?id=tanbro_pyyaml-include)

An extending constructor of [PyYAML][]: include other [YAML][] files into [YAML][] document.

## Install

```bash
pip install --pre "pyyaml-include>=2.0"
```

Since we are using [fsspec][] to open including files from v2.0, an installation can be performed like below, if want to open remote files:

- for files on website:

  ```bash
  pip install --pre "pyyaml-include>=2.0" fsspec[http]
  ```

- for files on S3:

  ```bash
  pip install --pre "pyyaml-include>=2.0" fsspec[s3]
  ```

- see [fsspec][]'s documentation for more

> ℹ️ **Note**
>
> “pyyaml-include” itself depends on [fsspec][], so it will be installed no matter including local or remote files

## Basic usages

Consider we have such [YAML][] files:

```text
├── 0.yml
└── include.d
    ├── 1.yml
    └── 2.yml
```

- `1.yml` 's content:

  ```yaml
  name: "1"
  ```

- `2.yml` 's content:

  ```yaml
  name: "2"
  ```

To include `1.yml`, `2.yml` in `0.yml`, we shall:

1. add a `YamlInclude` to [PyYAML][]'s loader class, with `!inc` as it's tag:

   ```python
   import yaml
   from yamlinclude import YamlInclude

   # add the tag
   yaml.add_constructor(
      tag="!inc",
      constructor=YamlInclude(base_dir='/your/conf/dir'),
      Loader=yaml.Loader
   )
   ```

1. write `!inc` tags in `0.yaml`:

   ```yaml
   file1: !inc include.d/1.yml
   file2: !inc include.d/1.yml
   ```

1. load it

   ```python
   with open('0.yml') as f:
      data = yaml.load(f, Loader=yaml.Loader)
   print(data)
   ```

   we'll get:

   ```python
   {'file1':{'name':'1'},'file2':{'name':'2'}}
   ```

1. (optional) the constructor can be removed:

   ```python
   del yaml.Loader.yaml_constructors["!inc"]
   ```

### Include in Mapping

If `0.yml` was:

```yaml
file1: !inc include.d/1.yml
file2: !inc include.d/2.yml
```

We'll get:

```yaml
file1:
  name: "1"
file2:
  name: "2"
```

### Include in Sequence

If `0.yml` was:

```yaml
files:
  - !inc include.d/1.yml
  - !inc include.d/2.yml
```

We'll get:

```yaml
files:
  - name: "1"
  - name: "2"
```

## Advanced usages

### Wildcards

File name can contain shell-style wildcards. Data loaded from the file(s) found by wildcards will be set in a sequence.

That is, a list will be returned when including file name contains wildcards.
Length of the returned list equals number of matched files:

- when only 1 file matched, length of list will be 1
- when there are no files matched, an empty list will be returned

We support `**`, `?` and `[..]`. We do not support `^` for pattern negation.
The `maxdepth` option is applied on the first `**` found in the path.

> ℹ️ **Note**
>
> Using the `**` pattern in large directory trees or remote file system (S3, HTTP ...) may consume an inordinate amount of time.

If `0.yml` was:

```yaml
files: !inc include.d/*.yml
```

We'll get:

```yaml
files:
  - name: "1"
  - name: "2"
```

### Work with fsspec

In `v2.0`, we use [fsspec][] to open including files, which makes it possible to include files from many different sources, such as local file system, S3, HTTP, SFTP ...

For example, we can include a file from website in YAML:

```yaml
conf:
  logging: !inc http://domain/etc/app/conf.d/logging.yml
```

In such situations, when creating a `YamlInclude` constructor, a [fsspec][] filesystem object shall be set to `fs` argument.

For example, if want to include files from website, we shall:

1. create a YamlInclude constructor, with a [fsspec][] HTTP filesystem object as it's `fs`:

   ```python
   import yaml
   import fsspec
   from yamlinclude import YamlInclude

   http_fs = fsspec.filesystem("http", client_kwargs={"base_url": f"http://{HOST}:{PORT}"})

   ctor = YamlInclude(http_fs, base_dir="/foo/baz")
   yaml.add_constructor("!inc", ctor, yaml.Loader)
   ```

1. then, write a [YAML][] document to include files from `http://${HOST}:${PORT}`:

   ```yaml
   key1: !inc doc1.yml    # relative path to "base_dir"
   key2: !inc ./doc2.yml  # relative path to "base_dir" also
   key3: !inc /doc3.yml   # absolute path, "base_dir" does not affect
   key3: !inc ../doc4.yml # relative path one level upper to "base_dir"
   ```

1. load it with [PyYAML][]:

   ```python
   yaml.load(yaml_string, yaml.Loader)
   ```

Above [YAML][] snippet will be loaded like:

- `key1`: pared YAML of `http://${HOST}:${PORT}/foo/baz/doc1.yml`
- `key2`: pared YAML of `http://${HOST}:${PORT}/foo/baz/doc2.yml`
- `key3`: pared YAML of `http://${HOST}:${PORT}/doc3.yml`
- `key4`: pared YAML of `http://${HOST}:${PORT}/foo/doc4.yml`

> 💬 **Tips**
>
> Check [fsspec][]'s documentation for more

> ℹ️ **Note**
>
> If `fs` argument is omitted or `None`, a `"file"`/`"local"` [fsspec][] filesystem object will be used automatically. That is to say:
>
> ```yaml
> data: !inc: foo/baz.yaml
> ```
>
> is equivalent to (if no `base_dir` was set in `YamlInclude()`):
>
> ```yaml
> data: !inc: file://foo/baz.yaml
> ```
>
> and
>
> ```python
> yaml.add_constructor("!inc", YamlInclude())
> ```
>
> is equivalent to:
>
> ```python
> yaml.add_constructor("!inc", YamlInclude(fs=fsspec.filesystem("file")))
> ```

### Parameters in YAML

As a callable object, `YamlInclude` passes YAML tag parameters to [fsspec][] for more detailed operations.

The first argument is `urlpath`, it's fixed and must-required, either positional or named.
Normally, we put it as a string after the tag(eg: `!inc`), just like examples above.

However, there are more parameters.

- in a mapping way, parameters will be passed to python as positional arguments, like `*args` in python function. eg:

  ```yml
  files: !inc [include.d/**/*.yaml, {maxdepth: !!int '2'}, {encoding: utf16}]
  ```

- in a sequence way, parameters will be passed to python as named arguments, like `**kwargs` in python function. eg:

  ```yml
  files: !inc {urlpath: /foo/baz.yaml, encoding: utf16}
  ```

But the format of parameters has multiple cases, and differs variably in different [fsspec][] implementation backends.

- If a scheme/protocol(“`http://`”, “`sftp://`”, “`file://`”, etc.) is defined in `urlpath`, `YamlInclude` will invoke [`fsspece.open`](https://filesystem-spec.readthedocs.io/en/stable/api.html#fsspec.open) directly to open it. Which means `YamlInclude`'s `fs` will be ignored, and a new standalone `fs` will be created implicitly.

  In this situation, `urlpath` will be passed to `fsspece.open`'s first argument, and all other parameters will also be passed to the function.

  For example,

  - the [YAML][] snippet

    ```yml
    files: !inc [file:///foo/baz.yaml, r]
    ```

    will cause python code like

    ```python
    with fsspec.open("file:///foo/baz.yaml", "r") as f:
        yaml.load(f, Loader)
    ```

  - and the [YAML][] snippet

    ```yml
    files: !inc {urlpath: file:///foo/baz.yaml, encoding: utf16}
    ```

    will cause python code like

    ```python
    with fsspec.open("file:///foo/baz.yaml", encoding="utf16") as f:
        yaml.load(f, Loader)
    ```

  > ⁉️ **Attention**
  >
  > `urlpath` with scheme/protocol **SHOULD NOT** include wildcards character(s).
  > `urlpath` like “`file:///etc/foo/*.yml`” is illegal.

- If `urlpath` has wildcards in it, `YamlInclude` will:

  1. invoke corresponding [fsspec][] implementation backend's [`glob`](https://filesystem-spec.readthedocs.io/en/stable/api.html#fsspec.spec.AbstractFileSystem.glob) method to search files,
  1. then call [`open`](https://filesystem-spec.readthedocs.io/en/stable/api.html#fsspec.spec.AbstractFileSystem.open) method to open the found file(s).

  `urlpath` will be passed as the first argument to both `glob` and `open` method of the corresponding [fsspec][] implementation backend, and other parameters will also be passed to `glob` and `open` method as their following arguments.

  In the case of wildcards, what need to pay special attention to is that there are **two separated parameters** after `urlpath`, the first is for `glob` method, and the second is for `open` method. Each of them could be in either mapping or sequence form. For example:

  - If we want to include every `.yml` file in directory `etc/app` recursively with max depth at 2, and open them in utf-16 codec, we shall write the [YAML][] as below:

    ```yml
    files: !inc ["etc/app/**/*.yml", {maxdepth: !!int "2"}, {encoding: utf16}]
    ```

    it will cause python code like:

    ```python
    for file in local_fs.glob("etc/app/**/*.yml", maxdepth=2):
        with local_fs.open(file, encoding="utf16") as f:
            yaml.load(f, Loader)
    ```

  - Since `maxdepth` is following `path` in `glob` method, we can write the [YAML][] like this:

    ```yml
    files: !inc ["etc/app/**/*.yml", [!!int "2"]]
    ```

    The parameters for `open` is omitted, means no more arguments except `urlpath` is passed.

  - Also we can set the two parameters in a mapping form, and name of the keys are `"glob"` and `"open"`. for example:

    ```yml
    files: !inc {urlpath: "etc/app/**/*.yml", glob: [!!int "2"], open: {encoding: utf16}}
    ```

    The parameters for `open` is omitted, means no more arguments beside `urlpath` is passed.

  > ⁉️ **Attention**
  >
  > [PyYAML][] takes any scalar parameter of custom constructor as string, if it is defined in a sequence. So we must use ‘Standard YAML tag’ to ensure non-string data type in the situation.
  >
  >
  > For example, follow [YAML][] snippet will cause an error:
  >
  > ```yml
  > files: !inc ["etc/app/**/*.yml", [1]]
  > ```
  >
  > because [PyYAML][] takes the `[1]` as `["1"]`, which makes  python code like `glob(path, maxdepth="1")`. To solve the problem, we shall write the [YAML][] like:
  >
  > ```yml
  > files: !inc ["etc/app/**/*.yml", [!!int 1]]
  > ```
  >
  > where `!!int` is the ‘Standard YAML tag’ to force integer type of `maxdepth` argument.
  >
  > > ⚠️ **Warning**
  > >
  > > `BaseLoader`, `SafeLoader`, `CBaseLoader`, `CSafeLoader` do **NOT** support ‘Standard YAML tag’.

- Else, `YamlInclude` will invoke corresponding [fsspec][] implementation backend's [`open`](https://filesystem-spec.readthedocs.io/en/stable/api.html#fsspec.spec.AbstractFileSystem.open) method to open the file, parameters beside `urlpath` will be passed to the method.

### Absolute and Relative URL/Path

### Base Dir

- If `base_dir` is omitted or `None`, the actually including file path is the path in defined in [YAML][] without a change, and different [fsspec][] filesystem will treat them differently. For a local filesystem, it will be `$CWD`.

[YAML]: http://yaml.org/ "YAML: YAML Ain't Markup Language™"
[PyYaml]: https://pypi.org/project/PyYAML/ "PyYAML is a full-featured YAML framework for the Python programming language."
[fsspec]: https://github.com/fsspec/filesystem_spec/ "Filesystem Spec (fsspec) is a project to provide a unified pythonic interface to local, remote and embedded file systems and bytes storage."
