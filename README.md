# Zotero Translation Server

Server-side Zotero translation

Currently supports import, export, and web translation


## Installation

The recommended version of translation-server for production use is available [from Docker Hub](https://hub.docker.com/r/zotero/translation-server/):

``
docker run --rm -p 1969:1969 zotero/translation-server
``

## Development

### Docker (recommended)

1. `git clone --recursive https://github.com/zotero/translation-server`

1. `cd translation-server`

1. `docker build -t translation-server .`

1. `docker run --rm -p 1969:1969 translation-server`

You can test changes without rebuilding the image each time. First, run `./fetch_sdk` once to install the Firefox SDK. (The SDK won’t be used in your build, but it’s required by the build script.) Next, compile the client code into `./modules/zotero/build` by running `npm i` and `npm run build` from `./modules/zotero`. Then, after each change, run the translation-server build script and a modified `docker run`:

``
./build.sh && docker run -p 1969:1969 -ti --rm -v `pwd`/build/app/:/opt/translation-server/app/ translation-server
``

This will copy files from `src`, `modules/zotero/build`, and `modules/zotero-connectors` into `build` and mount that directory in the container in place of the directory created during the `docker build` step above.

If you’re changing files in the main Zotero repository, you can run `npm start` in `./modules/zotero` and leave it running as you make changes to keeps its `build` subdirectory up to date. If you already have Zotero or Zotero Connector repositories on your system, you can pass `-d ~/zotero-client/build` or `-c ~/zotero-connectors` to `./build.sh` above to use those directories instead of the submodules used by default.

To inspect the container before running the server, include `--entrypoint /bin/bash` in the `docker run` command.

To run translator tests, add `-test /tmp/results.json` to the end of the `docker run` line. You can specify individual tests to run by adding translator labels (e.g., "ACM Digital Library") to the `includeTranslators` array in `chrome/content/zotero/tools/testTranslators/translatorTester.js` in the main Zotero repo.

When you’re done, ensure your changes are applied to `modules/zotero` and `modules/zotero-connector` (either manually or by updating the submodule to point to your own fork of those repos) and rebuild the translation-server image to incorporate your changes.

### Manually (unsupported)

1. Install [required libraries](https://github.com/zotero/translation-server/blob/master/Dockerfile#L4) (e.g., by installing Firefox)

1. `git clone --recursive https://github.com/zotero/translation-server`

1. `cd translation-server/modules/zotero`

1. `npm i && npm st`

1. `cd ../..`

1. `./fetch_sdk`

1. `./build.sh`

1. Run the server:

   ```
   $ build/run_translation-server.sh 

   zotero(3)(+0000000): HTTP server listening on *:1969
   ```

## Try a query

   ```
   $ curl -d '{"url":"http://www.tandfonline.com/doi/abs/10.1080/15424060903167229","sessionid":"abc123"}' \
          --header "Content-Type: application/json" \
          127.0.0.1:1969/web
   ```

## Endpoints

Supported endpoints are: `/web`, `/import`, `/export`, and `/refresh`.

Read [server_translation.js](./src/server_translation.js) for more information.

### Web Translators

Translates a web page

* endpoint: `/web`
* request method: `POST`
* request body: JSON object containing a `url` and a random `sessionid`
* example:
```bash
curl -X POST --header 'Content-Type: application/json' -d '{
  "url": "http://papers.ssrn.com/sol3/papers.cfm?abstract_id=1664470",
  "sessionid": "abc123"
}' 127.0.0.1:1969/web
```

### Import Translators

Converts input in any format Zotero can import (RIS, BibTeX, etc.) to items in Zotero API JSON format

* endpoint: `/import`
* request method: `POST`
* request body: item data in a supported format
* example:
```bash
curl -X POST -d 'TY  - JOUR
TI  - Die Grundlage der allgemeinen Relativitätstheorie
AU  - Einstein, Albert
PY  - 1916
SP  - 769
EP  - 822
JO  - Annalen der Physik
VL  - 49
ER  -' 127.0.0.1:1969/import
```

### Export Translators

Converts items in Zotero API JSON format to a supported export format (RIS, BibTeX, etc.)

* endpoint: `/export`
* request method: `POST`
* query parameter: `format`, which must be a [supported export format](https://github.com/zotero/translation-server/blob/master/src/server_translation.js#L31-43)
* request body: An array of items in Zotero API JSON format
