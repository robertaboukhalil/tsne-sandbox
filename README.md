# tSNE Sandbox
Interactive, WebAssembly-powered tSNE playground

![Screenshot](https://tsne.sandbox.bio/assets/img/screenshot.gif)


## How to use tSNE Sandbox?

You can use the hosted version at [tsne.sandbox.bio](https://tsne.sandbox.bio).

Or, to run it locally:

```bash
npm install
npm run build
```

and open the `index.html` file in your browser.


## How it works

- To perform the tSNE calculations, this app runs the C tool [bhtsne](https://github.com/lh3/bhtsne) directly in the browser using WebAssembly. For details about the compilation from C to WebAssembly, see the [biowasm](https://github.com/biowasm/biowasm) project.
- tSNE sandbox uses the [aioli](https://github.com/biowasm/aioli) library to run the WebAssembly module in a WebWorker, and passes results back to the main thread after each tSNE iteration.
