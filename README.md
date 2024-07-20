# Hugo Blog

## 01 Submodule Manage

First, add submodule to project:

```sh
$ git submodule add http://github.com/username/project_name.git path/to/submodule
$ git add path/to/submodule
$ git commit -m "add new module"
```

Second, update submodule:
- `--init`: init submodule.
- `--recursive`: update all submodule recursively.

```sh
# Clone submodule.
$ git submodule update --init --recursive
# Update submodule.
$ git submodule update --recursive
```

Third, modify submodule:

```sh
$ cd path/to/submodule
$ git add .
$ git commit -m "update sth."
$
```

The last but not least, use `git submodule xx` to manage submodule just like normal git commands:

```sh
$ git submodule checkout xxx
$ git submodule branch xxx
```