To locally compile and view the reference guide, run these commands:

$ npm install gitbook-cli

(npm will complain there is no package.json file, that can be ignored)

$ ./node_modules/gitbook-cli/bin/gitbook.js install

This installs the necessary plugins and the latest gitbook version

$ ./node_modules/gitbook-cli/bin/gitbook.js build

This builds the book into the _book folder

Alternatively, you can use:

$ ./node_modules/gitbook-cli/bin/gitbook.js serve

This starts a webserver at port 4000;
It also monitors your .md files for changes, rebuilds the book and refreshes your browser tab.

Additionally, if you want to create a eBook from the reference guide (.pdf, .epbu or .mobi), you can do the following.

Ensure you have Calibre (https://calibre-ebook.com/download) installed locally, as that's what's used under the covers.

Added to that, ensure to install `ebook-convert` through `npm install` (like so `npm install ebook-convert`).

After that, you can run the following from the reference guide root directory:

$ gitbook pdf ./ ./ref-guide.pdf

In doing so you will receive a .pdf of the reference guide.

See https://toolchain.gitbook.com/ebook.html for a short explanation around the GitBook to eBook conversion.