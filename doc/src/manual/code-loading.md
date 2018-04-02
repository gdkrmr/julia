# Code Loading

Julia has two mechanisms for loading code:

1. **Code inclusion:** e.g. `include("source.jl")`. Code inclusion allows you to split a single program across multiple source files. The expression `include("source.jl")` causes the contents of the file `source.jl` to be evaluated inside of the module where the `include` call occurs, much as if the text of `source.jl` were pasted into that file in place of the `include` call. If `include("source.jl")` is called multiple times, `source.jl` will be evaluated multiple times. The included path, `source.jl`, is interpreted relative to the file where the `include` call occurs. This makes it simple to relocate program split across files. In the REPL, included paths are interpreted relative to the current working directory, `pwd()`.
2. **Package loading:** e.g. `import X` or `using X`. The import mechanism allows you to load a package—i.e. an independent, reusable collection of Julia code—and makes the resulting module available by the name `X` inside of the importing module. If the same `X` package is imported multiple times in the same Julia session, it's only loaded the first time: on subsequent imports, the importing module gets a reference to the same already-loaded package module. It should be noted, however, that `import X` can load different packages in different contexts: `X` can refer to one package named `X` in the main project but potentially different package named `X` in each dependency. More on this below.

Code inclusion is quite straightforward: it merely consists of parsing a source file and evaluating the expressions in it one after another. Package loading is a more complex mechanism, built on top of code inclusion. Accordingly, the rest of this chapter focuses on package loading.

## Package Loading

A *package* in Julia is a source tree with a standard layout providing functionality which can be reused by other Julia projects. A package is loaded by `import X` and  `using X` statements. These statements also make the package's module `X` available within the module where the import statement occurs. Evaluating `import X` or `using X` depends on the answers to two questions:

1. **What** is `X` in this context?
2. **Where** can `X` be found?

Understanding how Julia answers these questions is key to understanding package loading.

### Federation of packages

Julia supports federated management of packages. This means that multiple independent parties can maintain public and/or private registries of packages, and that projects can depend on mixes of packages from all visible registries. Packages from both public and private registries are installed and managed using the same package management tools and workflows. One consequence of federation is that there cannot be a central authority for package naming. Different entities may use the same name to refer to unrelated packages. This possibility is unavoidable since these entities do not coordinate and may not even know about each other. Beacuse of the lack of a central naming authority, a single project may end up depending on different packages with the same name. Julia's package loading mechanism handles this by not requiring package names to be globally unique; instead packages are identified by [universally unique identifiers](https://en.wikipedia.org/wiki/Universally_unique_identifier) (UUIDs) which are assigned to packages before they are registered.

Since the decentralized naming problem is somewhat abstract, it may help to walk through a concrete scenario to understand the issue. Suppose you're developing an application called `App`, which uses two packages: `Pub` and  `Priv`. `Priv` is a private package that you created, whereas `Pub` is a public package that you use but don't control. When you created `Priv`, there was no public package by that name. Subsequently, however, an unrelated package also named `Priv` has been published and become popular. In fact, the `Pub` package has started to use it. Therefore, when you next upgrade `Pub` to get the latest bug fixes and features, `App` will end up—through no action of yours other than upgrading—depending on two different packages named `Priv`. `App` has a direct dependency on your private `Priv` package, and an indirect dependency, through `Pub`, on the new, public `Priv` package. Since these two `Priv` packages are different but both required for `App` to continue working correctly, the expression `import Priv` must refer to different `Priv` packages depending on whether it occurs in the code of `App` or `Pub`. Julia's package loading mechanism allows precisely this by distinguishing the two `Priv` packages by UUID and context.

### The identity of `X`

The above package name collision scenario brings us to the first question that Julia must answer to evaluate `import X`: *What is `X` in this context?* The identity of `X` is determined by a combination of three things:

1. The **module** in which the expression is evaluated;
2. The **load path** as determined by the contents of the `LOAD_PATH` global array;
3. The **contents** of each directory in the load path.






------

If the dependencies of a project needed share a single namespace, this would be a big problem for you if you ever tried to upgrade `Public` since you would then have two different packages named `Private` as dependencies of your project. With a shared package namespace, the the only real recourse is to rename your private package to something else. This situation is not hypthetical—it's been encountered repeatedly in the wild. 

To handle this kind of situation without  is why `import Private` can mean different things in 



here may not have been any public package named `X`, but at some point you want to upgrade 

Julia packages are identified by [universally unique identifiers](https://en.wikipedia.org/wiki/Universally_unique_identifier) (UUIDs), unique 128-bit values that can be generated independently with astronomically probability of collision. When a Julia package is created, a UUID is generated which identifies it persistently across various events, including:

- name changes,
- changes of repository location,
- changes of ownership,
- package forks.

In any situation where the question *"are these the same package?"* needs to be asked, the answer is determined by comparing UUIDs—two Julia source trees are the same package if and only if they have the same UUID. Accordingly, the question *"what is `X` in this context?"* when evaluating `import X` is answered by determining which UUID the name `X` corresponds to in the context in which the import occurs.

Julia package UUIDs serve a similar purpose to Java's reversed domain package names. For example, in Java you might write `import com.sun.net.httpserver.*;` to import all classes from the `com.sun.net.httpserver` package. This package name suggests that the `httpserver` package is maintained by Sun Microsystems, owner of the `sun.com` domain. Of course, Sun Microsystems no longer exists today, having been acquired by Oracle in 2010. The `com.sun` prefix for these packages, however, is permanent and cannot be updated to `com.oracle`. Even if Oracle were to sell the `sun.com` domain, all Java code using the `httpserver` package would still use the `com.sun` prefix. Since UUIDs have no inherent meaning, they avoid this problem—there's no reason to change a UUID since it entails no meaning. However, UUIDs are are hard for humans to remember and tedious to type. Accordingly, Julia's [package manager](https://julialang.org/Pkg3.jl/latest/) keeps UUIDs mostly out of sight. They don't appear in your Julia code, only in project files specifically designed to record the mapping from names to UUIDs in your projects.

Once the name `X` has been mapped to a UUID, answering the question *"has `X` already been loaded?"* is a simple matter of looking up `X`'s UUID in a table of loaded packages. If `X`'s UUID is in the table, then the loaded module associated with the UUID is bound to `X` in the module where the `import X` or `using X` expression is evaluated. If the `X`'s UUID is not in the global package table, then it must be loaded, which requires answering the next qeustion: *"where can `X` be loaded from?"* but with 

for example, but since UUIDs have no meaning, they can handle package renames and even changing the controlling entitity of a package. In the case, for example, Sun Microsystems no longer exists, and `sun.com` redirects to `oracle.com`. 



The load path is a stack of "environments", each of which provides three things:

1. `roots`: a map from top-level names to UUIDs
2. `graph`: a graph of dependencies from 

searched for in the load path, which is controlled by the `LOAD_PATH` global array 