# Understanding Namespaces

## Structure of the Key Database

The _key database_ of Elektra is _hierarchically structured_. This means that keys are organized similar to directories in a file system.

Lets add some keys to the database. To add a key we can use this command:

```sh
kdb set <key> <value>
```

Now add the the key **/a** with the Value **Value 1** and the key **/b/c** with the Value **Value 2**:

```sh
kdb set /a 'Value 1'
#> Using name user/a
#> Create a new key user/a with string Value 1
kdb set /b/c 'Value 2'
#> Using name user/b/c
#> Create a new key user/b/c with string Value 2
```

![Hierarchical structure of key database](/doc/images/tutorial_namespaces_hierarchy.svg)

Here you see the internal structure of the database after adding the keys **/a** and **/b/c**.
For instance the key **/b/c** has the path **/** -> **b** -> **c**.

Note how the name of the key determines the path to its value.

You can use the file system analogy as a mnemonic to remember these commands (like the file system commands in your favorite operating system):

- `kdb ls <path>`
	lists keys below _path_
- `kdb rm <key>`
	removes a _key_
- `kdb cp <source> <dest>`
	copies a key to another path
- `kdb get <key>`
	gets the value of _key_

For example `kdb get /b/c` should return `Value 2` now, if you set the values before.

## Namespaces

Now we abandon the file system analogy and introduce the concept of _namespaces_.

Every key in Elektra belongs to one of these namespaces:

- **spec** for specification of other keys
- **proc** for in-memory keys (e.g. command-line)
- **dir** for dir keys in current working directory
- **user** for user keys in home directory
- **system** for system keys in `/etc` or `/`

All namespaces save their keys in a _separate hierarchical structure_ from the other namespaces.

But when we set the keys **/a** and **/b/c** before we didn't provide a namespace.
So I hear you asking, if every key has to belong to a namespace, where are the keys?
They are in the _user_ namespace, as you can verify with:

```sh
kdb ls user | grep -E '(/a|/b/c)'
#> user/a
#> user/b/c
```

When we don't provide a namespace Elektra assumes a default namespace, which should be **user** for non-root users.
So if you are a normal user the command `kdb set /b/c 'Value 2'` was synonymous to `kdb set user/b/c 'Value 2'`.

At this point the key database should have this structure:
![Elektra’s namespaces](/doc/images/tutorial_namespaces_namespaces.svg)

### Cascading Keys

Another question you may ask yourself now is, what happens if we lookup a key without providing a namespace. So let us retrieve the key **/b/c** with the -v flag in order to make _kdb_ more talkative.

```sh
kdb get -v /b/c
# STDOUT-REGEX: got \d+ keys
#>  searching spec/b/c, found: <nothing>, options: KDB_O_CALLBACK
#>  searching proc/b/c, found: <nothing>, options:
#>  searching dir/b/c, found: <nothing>, options:
#>  searching user/b/c, found: user/b/c, options:
#> The resulting key name is user/b/c
#> Value 2
```

Here you see how Elektra searches all namespaces for matching keys in this order:
**spec**, **proc**, **dir**, **user** and finally **system**

If a key is found in a namespace, it masks the key in all subsequent namespaces, which is the reason why the system namespace isn't searched. Finally the virtual key **/b/c** gets resolved to the real key **user/b/c**.
Because of the way a key without a namespace is retrieved, we call keys, that start with '**/**' **cascading keys**.
You can find out more about cascading lookups in the [cascading tutorial](cascading.md).



Having namespaces enables both admins and users to set specific parts of the application's configuration, as you will see in the following example.

## How it Works on the Command Line (kdb)

Let's say your app requires the following configuration data:

- **/sw/org/myapp/policy** - a security policy to be applied
- **/sw/org/myapp/default_dir** - a place where the application stores its data per default

We now want to enter this configuration by using the **kdb** tool.

The security policy will most probably be set by your system administrator.
So she enters

```sh
sudo kdb set "system/sw/org/myapp/policy" "super-high-secure"
#> Create a new key system/sw/org/myapp/policy with string super-high-secure
```

The key **system/sw/org/myapp/policy** will be stored in the system namespace (probably at `/etc/kdb` on a Linux/Unix system).

Then the user sets his app directory by issuing:

```sh
kdb set "user/sw/org/myapp/default_dir" "/home/user/.myapp"
#> Create a new key user/sw/org/myapp/default_dir with string /home/user/.myapp
```

This key will be stored in the user namespace (below the home directory) and thus may vary from user to user.
Elektra loads the value for the current user and passes it to the application.

You can also retrieve the two values with the command line tool **kdb**:

```sh
kdb get system/sw/org/myapp/policy
#> super-high-secure
kdb get user/sw/org/myapp/default_dir
#> /home/user/.myapp
```

The two key lookups can be done in an easier and more general way, using _cascading keys_ (i.e. key names 
without namespace and leading slash `/`).
You do not need to search every namespace by yourself.
Just make a lookup for **/sw/org/myapp**, like this:

```sh
kdb get /sw/org/myapp/policy
#> super-high-secure
kdb get /sw/org/myapp/default_dir
#> /home/user/.myapp
```

When using cascading keys Elektra will usually search all namespaces for a matching key in the following order: `spec`, `proc`, `dir`, `user`, `system`. This also means, that a key in the `user` namespace can override a key in the `system` namespace. For more information on this, read the [cascading turorial](cascading.md).

