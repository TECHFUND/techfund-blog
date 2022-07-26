+++
author = "Piyush"
title = "Using Rust for writing NodeJS modules"
date = "2022-07-25"
description = "Writing native NodeJS modules in rust for performance optimization."
tags = [
    "rust",
    "nodejs"
]
categories = [
    "technical"
]
series = ["Rust"]
aliases = ["rust"]
image = "bg.jpg"
+++

## Using Rust for writing NodeJS modules

Around 2009 I watched some presentations by Ryan introducing NodeJS and was really impressed by the possibilities. That was one of the reasons, around 2011-2012, I started working on a project that used NodeJS to expose Clutter C++ library ( https://wiki.gnome.org/Projects/Clutter ). 
The idea was to integrate Clutter, which allowed us to write code in JS to create near-native UI for Linux. Here is a video ( by CEO of the company I was working for then ) during a JS conference 

{{< youtube bd95kIJK_D0 >}}

After years of development in C++ and JS, Rust has become my new go-to language. This article acts as a guide/reference for writing Rust modules for NodeJS in order to achieve better performance where required.

For the sake of article let us assume we need an API that needs to return size of directory on server. 
That article assumes basic understanding of NodeJS and Rust. 

## Setting up Express API
```js
    const express = require('express')
    const { readdir, stat } = require('fs/promises');
    const { join } = require('path');
    const app = express()
    const port = 3000
    
    const dirSize = async dir => {
      const files = await readdir( dir, { withFileTypes: true } );
      const paths = files.map( async file => {
        const path = join( dir, file.name );
        if ( file.isDirectory() ) return await dirSize( path );
        if ( file.isFile() ) {
          const { size } = await stat( path );
          return size;
        }
        return 0;
      } );
      return (await Promise.all(paths))
        .flat(Infinity)
        .reduce((i, size) => i + size, 0);
    }

    app.get('/', async (req, res) => {
      console.time('dir');
      let size = await dirSize('./assets/');
      console.timeEnd('dir');
      res.send(size)
    })
    
    app.listen(port, () => {
      console.log(`App listening on port ${port}`)
    })
```

The test was performed on the assets folder having 40 directories with 1000 files each. On Macbook Pro 2017 ( i7, 16 GB Ram ) it takes about `591ms` for above API to give response. The dirSize function reads the directory and uses `stat` to compute the size of the file in recursive manner. We will try to replicate the same in Rust.

To use Rust in our API we have two options. 
1) Compile a function as dylib and use `ffi` to load the dynamic library. 
  `node-ffi is a Node.js addon for loading and calling dynamic libraries using pure JavaScript. It can be used to create bindings to native libraries without writing any C++ code.` : From [Github Readme](https://github.com/node-ffi/node-ffi) of Node-FFI
2) Integrate with NodeJS bindings directly.

We will use the second option since using ffi might be a little slow and we already have support of awesome libraries like node-bindgen ( which automatically generates a lot of binding related code for us )

## Rust Bindings

Let us first create our Cargo.toml file

```toml
    [package]
    name = "techfund"
    version = "1.0.0"
    authors = [ "Piyush <piyush_gururani@techfund.jp>" ]
    edition = "2021"
    
    [dependencies]
    node-bindgen = { version = "4.0" }
    
    [build-dependencies]
    node-bindgen = { version = "4.0", features = ["build"] }
    
    [lib]
    crate-type = ["cdylib"]
```

and our build.rs file 

```rust
    fn main() {
        node_bindgen::build::configure();
    }
```

Finally in the src/lib.rs we can create our binding code 

```rust
    use node_bindgen::derive::node_bindgen;
    use std::{fs, io, path::PathBuf, string};
    
    fn get_size(path: impl Into<PathBuf>) -> io::Result<u64> {
        fn dir_size(mut dir: fs::ReadDir) -> io::Result<u64> {
            dir.try_fold(0, |acc, file| {
                let file = file?;
                let size = match file.metadata()? {
                    data if data.is_dir() => dir_size(fs::read_dir(file.path())?)?,
                    data => data.len(),
                };
                Ok(acc + size)
            })
        }
    
        dir_size(fs::read_dir(path.into())?)
    }
    
    #[node_bindgen]
    async fn size(path: String) -> u64 {
        if let Ok(dir_size) = get_size(path) {
            return dir_size;
        } else {
            return 0;
        }
    }
```

The final structure should look something like this : 

{{< figure src="./dir.png" title="Directory structure" >}}

This allows us to generate a node module that we can directly import inside node console.

To build the project, simply run 
> cargo install nj-cli
to install the nj-cli and 
> nj-cli build
to build the module ( dist/index.node ) which can be imported inside Node directly.

{{< figure src="./node.png" title="Node Module" >}}

Finally we will modify our Express API to check performance of both versions

```js
    const express = require("express");
    const { readdir, stat } = require("fs/promises");
    const { join } = require("path");
    const extramodule = require("./dist")
    const app = express();
    const port = 3000;
    
    const dirSize = async (dir) => {
      const files = await readdir(dir, { withFileTypes: true });
      const paths = files.map(async (file) => {
        const path = join(dir, file.name);
        if (file.isDirectory()) return await dirSize(path);
        if (file.isFile()) {
          const { size } = await stat(path);
          return size;
        }
        return 0;
      });
      return (await Promise.all(paths))
        .flat(Infinity)
        .reduce((i, size) => i + size, 0);
    };
    
    app.get("/v1", async (req, res) => {
      console.time("dir");
      let size = await dirSize("./assets/");
      console.timeEnd("dir");
      res.json(size);
    });
    
    app.get("/v2", async (req, res) => {
      console.time("dir");
      let size = await extramodule.size("./assets/");
      console.timeEnd("dir");
      res.json({size: Number(size)});
    });
    
    app.listen(port, () => {
      console.log(`App listening on port ${port}`);
    });
```

API Version | Time in ms
--------|------
version 1 | 591.127
version 2 | 191.134

Thats almost 3x savings !