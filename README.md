[![Build Status](https://travis-ci.org/webrtc/apprtc.svg?branch=master)](https://travis-ci.org/webrtc/apprtc)

# AppRTC Demo Code (XirSys - ICE)

## Note

This version of the web AppRTC application has been modified to use the XirSys ICE endpoint.  A version of this application using both XirSys ICE and XirSys Signalling will be released with the upcoming v3 of the XirSys platform.

## Development

Detailed information on devloping in the [webrtc](https://github.com/webrtc) github repo can be found in the [WebRTC GitHub repo developer's guide](https://docs.google.com/document/d/1tn1t6LW2ffzGuYTK3366w1fhTkkzsSvHsBnOHoDfRzY/edit?pli=1#heading=h.e3366rrgmkdk).

The development AppRTC server can be accessed by visiting [http://localhost:8080](http://localhost:8080).

Running AppRTC locally requires the [Google App Engine SDK for Python](https://cloud.google.com/appengine/downloads#Google_App_Engine_SDK_for_Python) and [Grunt](http://gruntjs.com/).

Detailed instructions for running on Ubuntu Linux are provided below.

### Running on Ubuntu Linux

Install grunt by first installing [npm](https://www.npmjs.com/). npm is
distributed as part of nodejs.

```
sudo apt-get install nodejs
sudo npm install -g npm
```

On Ubuntu 14.04 the default packages installs `/usr/bin/nodejs` but the `/usr/bin/node` executable is required for grunt. This is installed on some Ubuntu package sets; if it is missing, you can add this by installing the `nodejs-legacy` package,

```
sudo apt-get install nodejs-legacy
```

It is easiest to install a shared version of `grunt-cli` from `npm` using the `-g` flag. This will allow you access the `grunt` command from `/usr/local/bin`. More information can be found on [`gruntjs` Getting Started](http://gruntjs.com/getting-started).

```
sudo npm -g install grunt-cli
```

*Omitting the `-g` flag will install `grunt-cli` to the current directory under the `node_modules` directory.*

Finally, you will want to install grunt and required grunt dependencies. *This can be done from any directory under your checkout of the [webrtc/apprtc](https://github.com/webrtc/apprtc) repository.*

```
npm install
```

On Ubuntu, you will also need to install the webtest package:
```
sudo apt-get install python-webtest
```


Before you start the AppRTC dev server and *everytime you update the source code you need to recompile the App Engine package by running,

```
grunt build
```

Start the AppRTC dev server from the `out/app_engine` directory by running the Google App Engine SDK dev server,

```
<path to sdk>/dev_appserver.py ./out/app_engine
```
Then navigate to http://localhost:8080 in your browser (given it's on the same machine).

### Testing

All tests by running `grunt`.

To run only the Python tests you can call,

```
grunt runPythonTests
```

### Enabling Local Logging

*Note that logging is automatically enabled when running on Google App Engine using an implicit service account.*

By default, logging to a BigQuery from the development server is disabled. Log information is presented on the console. Unless you are modifying the analytics API you will not need to enable remote logging.

Logging to BigQuery when running LOCALLY requires a `secrets.json` containing Service Account credentials to a Google Developer project where BigQuery is enabled. DO NOT COMMIT `secrets.json` TO THE REPOSITORY.

To generate a `secrets.json` file in the Google Developers Console for your project:
1. Go to the project page.
1. Under *APIs & auth* select *Credentials*.
1. Confirm a *Service Account* already exists or create it by selecting *Create new Client ID*.
1. Select *Generate new JSON key* from the *Service Account* area to create and download JSON credentials.
1. Rename the downloaded file to `secrets.json` and place in the directory containing `analytics.py`.

When the `Analytics` class detects that AppRTC is running locally, all data is logged to `analytics` table in the `dev` dataset. You can bootstrap the `dev` dataset by following the instructions in the [Bootstrapping/Updating BigQuery](#bootstrappingupdating-bigquery).

## BigQuery

When running on App Engine the `Analytics` class will log to `analytics` table in the `prod` dataset for whatever project is defined in `app.yaml`.

### Schema

`bigquery/analytics_schema.json` contains the fields used in the BigQuery table. New fields can be added to the schema and the table updated. However, fields *cannot* be renamed or removed. *Caution should be taken when updating the production table as reverting schema updates is difficult.*

Update the BigQuery table from the schema by running,

```
bq update -t prod.analytics bigquery/analytics_schema.json
```

### Bootstrapping

Initialize the required BigQuery datasets and tables with the following,

```
bq mk prod
bq mk -t prod.analytics bigquery/analytics_schema.json
```

### Deployment
Note this assumes you are setting up AppRTC (Web client and backend), Collider (signalling server) and the Coturn TURN server on the same machine and that it is running Linux, if you are not just perform the steps of each application on the machine you want to run them on. Instructions were performed on Ubuntu 14.04 using Python 2.7.9 and Go 1.6.3.

1. Clone the AppRTC repository on the machine you want to host it on (git clone <this repo URL>)
2. Do all the steps in the [Collider instructions](https://github.com/webrtc/apprtc/blob/master/src/collider/README.md).
3. In the AppRTC git checkout navigate to `src/app_engine/constants.py` and change the `WSS_INSTANCE_HOST_KEY:` to the address and port Collider is listening too, in this case `localhost:8089`.
4. If you want just want to setup AppRTC with a TURN server directly then continue to step 5 otherwise jump to 7.
5. Install and start the Coturn TURN server according to their [instructions](https://github.com/coturn/coturn/wiki/CoturnConfig)
6. Now navigate to `src/web_app/js/utils.js` and change the `requestIceServers` function to the following: 
```javascript
function requestIceServers(iceServerRequestUrl, iceTransports) {
  return new Promise(function(resolve, reject) {
    var servers = [{
        credential: "turnPassword",
        username: "turnUser",
        urls: [
          "turn:localhost:3478?transport=udp",
          "turn:localhost:3478?transport=tcp"
        ]
    }];
    resolve(servers);
  });
}
```
7\. (Only do this if you skipped step 5 and 6) AppRTC by default uses an ICE server provider to get TURN servers, it's basically just a web server with authentication that returns a [JSON response](https://github.com/webrtc/apprtc/blob/master/src/web_app/js/util.js#L77) containing TURN servers with credentials, note that before it provides a response, it checks where the user is connecting from, checks if there are any TURN servers in that area, if not it spins up an instance and gets it's reachable address and credentials. If you have such a service then change the ICE server constants in [constants.py](https://github.com/webrtc/apprtc/blob/master/src/app_engine/constants.py#L19) to point to that.

8\. Now build AppRTC using `grunt build` and then start it using dev appserver provided by GAE
`pathToGAESDK/dev_appserver.py  out/app_engine/`.

9\. Open a WebRTC enabled browser and navigate to `http://localhost:8080` (`http://localhost:8080?wstls=false` if you have TLS disabled on Collider)

