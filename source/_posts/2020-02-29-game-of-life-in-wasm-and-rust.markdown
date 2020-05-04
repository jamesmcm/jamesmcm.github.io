---
layout: post
title: "(Test) Game of Life in WASM and Rust"
titleen: "(Test) Game of Life in WASM and Rust"
titlesp: "(Test) Game of Life in WASM and Rust"
date: 2020-02-29 18:38:23 +0100
comments: true
categories: 
---
<!-- more -->

This is a test for embedding WebAssembly on Github pages.

<noscript>This page contains webassembly and javascript content, please enable javascript in your browser.</noscript>
<script type="module">
      // Use ES module import syntax to import functionality from the module
      // that we have compiled.
      //
      // Note that the `default` import is an initialization function which
      // will "boot" the module and make it ready to use. Currently browsers
      // don't support natively imported WebAssembly as an ES module, but
      // eventually the manual initialization won't be required!
      import init, { run } from '/assets/pkg/wasm_game_of_life_jamesmcm.js';

            async function runit() {
        // First up we need to actually load the wasm file, so we use the
        // default export to inform it where the wasm file is located on the
        // server, and then we wait on the returned promise to wait for the
        // wasm to be loaded.
        // It may look like this: `await init('./pkg/without_a_bundler_bg.wasm');`,
        // but there is also a handy default inside `init` function, which uses
        // `import.meta` to locate the wasm file relatively to js file
        //
        // Note that instead of a string here you can also pass in an instance
        // of `WebAssembly.Module` which allows you to compile your own module.
        // Also note that the promise, when resolved, yields the wasm module's
        // exports which is the same as importing the `*_bg` module in other
        // modes
        await init();

        // And afterwards we can use all the functionality defined in wasm.
        //run();
        }

      runit();
 </script>
<div id="placehere"></div>

