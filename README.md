PocketSphinx.js
---------------
### Speech Recognition in JavaScript

PocketSphinx.js is a speech recognizer that runs entirely in the web browser. It is built on:

* a speech recognizer written in C ([PocketSphinx](http://cmusphinx.sourceforge.net/)) converted into JavaScript using [Emscripten](https://github.com/kripken/emscripten),
* an audio recorder using the web audio API. The audio recorder can be used independently to build other kinds of audio-related web applications. There is a more detailed documentation in `doc/AudioRecorder/README.md`.

You can try it on the project page: <http://syl22-00.github.io/pocketsphinx.js> and have a look at our [FAQ](https://github.com/syl22-00/pocketsphinx.js/wiki/FAQ).

Table of contents:

1. Overview
2. Compilation of `pocketsphinx.js`
3. API of `pocketsphinx.js`
4. Using `pocketsphinx.js` inside a Web Worker with `recognizer.js`
5. Wiring `recognizer.js` to the audio recorder
6. Live demo
7. Test suite
8. Notes about speech recognition and performance
9. License

# 1. Overview

This project includes several components that can be used independently:

* `pocketsphinx.js`, a JavaScript library generated by emscripten which is basically PocketSphinx wrapped to provide a simpler API, and compiled into JavaScript.
* `recognizer.js`, a wrapper around `pocketsphinx.js` inside a Web Worker to unload the UI thread from downloading and running the large JavaScript file and running the costly speech recognition process.
* `audioRecorder.js`, an audio recording library, based on [Recorderjs](https://github.com/mattdiamond/Recorderjs). It converts the recorded samples to the proper sample rate and passes them to the recognizer. There is a more detailed documentation in `doc/AudioRecorder/README.md`.
* `callbackManager.js`, a small utility to interact with Web Workers with calls and callbacks rather than message passing.

The file `webapp/live.html` illustrates how these work together in a real application, that is a good starting point. Make sure you load it through a web server or start Chrome with `--disable-web-security`. For instance, you can start a small web server with `python -m SimpleHTTPServer` in the base directory and open `http://localhost:8000/webapp/live.html` in your browser.

There is also a live demo for Chinese. To try it, open `http://localhost:8000/webapp/live_zh.html` in your browser.

In addition to speech recognition, there is also a keyword spotting functionality that detects a specific word or phrase in the audio input. There is also a live demo in `webapp/live_kws.html`.

# 2. Compilation of `pocketsphinx.js`

A prebuilt version of `pocketsphinx.js` is available in `webapp/js`, or you can build it yourself. Below is the procedure on Linux (and Mac OS X). On Windows, refer to the emscripten manual.

## 2.a Compilation with the default acoustic model

You will need:

* [emscripten](https://github.com/kripken/emscripten) (which implies also node.js and LLVM), 
* [CMake](http://www.cmake.org/).

The build is a classic CMake cross-compilation, using the toolchain provided by emscripten:

    $ cd .../pocketsphinx.js # This folder
    $ mkdir build
    $ cd build
    $ cmake -DEMSCRIPTEN=1 -DCMAKE_TOOLCHAIN_FILE=path_to_emscripten/cmake/Platform/Emscripten.cmake ..
    $ make

This generates `pocketsphinx.js`. At this point, optimization level is hard-coded, so modify `CMakeLists.txt` directly if you would like to change it.

## 2.b Compilation with custom models and dictionary

The compilation process packages the acoustic models inside the resulting JavaScript file and also, possibly, language models and dictionary files. If you would like to package your own models, you should specify where they are when running `cmake`. For that, place all models you want to package inside a base folder and specify the files or sub-folders you want to include.

For instance, to package acoustic models, place them inside a `HMM_BASE` folder. Each model being in its own folder inside `HMM_BASE`:

    $ cmake -DEMSCRIPTEN=1 -DCMAKE_TOOLCHAIN_FILE=path_to_emscripten/cmake/Platform/Emscripten_unix.cmake -DHMM_BASE=/path/to/models -DHMM_FOLDERS="model1;model2;..." ..

If you only need to package one model, you can also do:

    $ cmake -DEMSCRIPTEN=1 -DCMAKE_TOOLCHAIN_FILE=path_to_emscripten/cmake/Platform/Emscripten_unix.cmake -DHMM_BASE=/path/to/models -DHMM_FOLDERS=model ..

Make sure the files of the acoustic model are directly inside the `HMM_FOLDERS`:

    $ cd /path/to/models
    $ ls *
    model1:
    feat.params  mdef  means  sendump  transition_matrices  variances
    
    model2:
    feat.params  mdef  means  sendump  transition_matrices  variances

You can do the same thing with statistical language models and dictionary files, using the following CMake parameters:

* Acoustic models: `HMM_BASE` and `HMM_FOLDERS`,
* Statistical language models: `LM_BASE` and `LM_FILES`.
* Dictionary files: `DICT_BASE` and `DICT_FILES`.

By default, we turn on `ALLOW_MEMORY_GROWTH` in emscripten as files tend to be large. However, this apparently takes out some optimization steps, so if you do not want that, you can pass `-DDISABLE_MEMORY_GROWTH=on` when invoking `cmake`.

Please note that:

* If you want to package files, you need to set both `..._BASE` and `..._FOLDERS` or `..._FILES`.
* If you do not specify an acoustic model to package, the default model will be packaged (the model is in `am/rm1_200/`).
* By default, the first provided acoustic model will be loaded if none is specified before the recognizer is initialized. The model can be selected by giving the `"-hmm"` parameter. See upcoming sections for how to specify recognizer parameters.
* Make sure you optimize the size of your acoustic models (`mdef` in binary format, `sendump` instead of `mixture_weights`, see PocketSphinx docs).
* Statistical language models and dictionary files are optional. As explained later, grammars and dictionary words can be added at runtime.
* If you want to package statistical language models, you must provide a dictionary that contains all words used in the SLMs.
* The PocketSphinx parameter for dictionary files is `"-dict"` and for language models `"-lm"`. See next sections for how to specify recognizer parameters.

# 3. API of `pocketsphinx.js`

You can interact with `pocketsphinx.js` directly if you need to, but it is probably easier to build your application against the API of `recognizer.js` described in a later section.

## 3.1 Principles

The file `pocketsphinx.js` can be directly included into an HTML file but as it is fairly large (a few MB, depending on the optimization level used during compilation and packaged files), downloading and loading it will take time and affect the UI thread. So, as explained later, you should use it inside a Web worker, for instance using `recognizer.js`.

This API is based on `embind`, you should probably have a look at that [section in emscripten's docs](https://github.com/kripken/emscripten/wiki/embind) to understand how to interact with emscripten-generated JavaScript. Earlier versions of Pocketsphinx.js used a C-style API which is now deprecated, but it is still available in the `OBSOLETE_API` branch. 


As a first example, to create a new recognizer:

```javascript
var recognizer = new Module.Recognizer();
/* ... */
recognizer.delete();
```

Calls to `pocketsphinx.js` functions are synchronous, that's also why you probably need to load it in a Web Worker, as explained in later sections.

Most calls return a ResultType object, which can be one of the following:

* SUCCESS, if the action was performed successfully.
* BAD_STATE, if the current state does not allow the action.
* BAD_ARGUMENT, if the argument provided is invalid.
* RUNTIME_ERROR, if there is a runtime error in the recognizer.

In JavaScript these values can be referred as `Module.ReturnType.SUCCESS`, `Module.ReturnType.BAD_STATE`, etc. For instance:

````javascript
var recognizer = new Module.Recognizer();
/* ... */
if (recognizer.reInit(config) != Module.ReturnType.SUCCESS)
    alert("Error while recognizer is re-initialized");
```

According to `embind`'s documentation, all objects created with the `new` operator must be deleted explicitly with a `.delete()` call.

## 3.2 Recognizer Object

The entry point of `pocketsphinx.js` is the recognizer object. You can create as many instances as you want, but you probably don't need to and want so save memory. When a new instance is created, an optional `Config` object can be given which will be used to set parameters used to initialize Pocketsphinx. Refer to Pocketsphinx documentation to learn about the possible parameters. A `Config` object is basically an array of key-value pairs:

```javascript
var config = new Module.Config();
config.push_back(["-fwdflat", "no"]);
var recognizer = new Module.recognizer(config);
config.delete();
/* ... */
recognizer.delete();
```

This will initialize a recognizer with `"-fwdflat"` set to `"no"`.

If you have included several acoustic models when compiling `pocketsphinx.js`, you can select which one should be used by setting the `"-hmm"` parameter. Say you have two models, one for English, one for French, and you have compiled the library with `-DHMM_FOLDERS="english;french"`, you can initialize the recognizer with the French model by setting the correct value in the `Config` object:

```javascript
var config = new Module.Config();
config.push_back(["-hmm", "french"]);
var recognizer = new Module.recognizer(config);
```

If you do not give the `"-hmm"` parameter, or give it an invalid value, the first model in the list will be used (here, `english`).

Similarly, you should use recognizer config parameters to load a statistical language model (`"-lm"`) or dictionary (`"-dict"`) you have previously packaged inside `pocketshinx.js`. Note that if you want to use a SLM, you must also have a dictionary file that contains the words used in the SLM.

In addition, a recognizer object can be re-initialized with new parameters after the instance was created, with a call to `reInit`, for instance:

```javascript
var config_english = new Module.Config();
config_english.push_back(["-hmm", "english"]);
var config_french = new Module.Config();
config_french.push_back(["-hmm", "french"]);
var recognizer = new Module.recognizer(config_english);
/* ... */
if (recognizer.reInit(config_french) != Module.ReturnType.SUCCESS)
    alert("Error while recognizer is re-initialized");
```

## 3.3 Adding words, grammars and key phrases

Dictionary and language model files can be packaged at compile time as explained previously. Meanwhile, dictionary words, grammars (Finite State Grammars, FSG) or key phrases (for keyword spotting) can be added at runtime.

### a. Adding words

All words used in grammars or for keyword spotting must be present in the pronunciation dictionary. Refer to the [CMU Pronunciation Dictionary site](http://www.speech.cs.cmu.edu/cgi-bin/cmudict) if you are not familiar with it. Words can be added as a vector of pairs word-pronunciation:

```javascript
var recognizer = new Module.Recognizer();
var words = new Module.VectorWords();
words.push_back(["HELLO", "HH AH L OW"]);
words.push_back(["WORLD", "W ER L D"]);
if (recognizer.addWords(words) != Module.ReturnType.SUCCESS)
    // Probably bad format used for pronunciation
    alert("Error while adding words");
words.delete()
```

Note that PocketSphinx allows you to input several pronunciation alternatives for a word, by adding suffixes to it (`(2)`, `(3)`, etc.). However, adding a word with a suffix before the word without suffix will fail when calling `addWords`:

```javascript
words.push_back(["HELLO", "HH AH L OW"], ["HELLO(2)", "HH EH L OW"]); // OK
/* ... */
words.push_back(["HELLO", "HH AH L OW"], ["HELLO", "HH EH L OW"]); // Invalid
/* ... */
words.push_back(["HELLO(2)", "HH AH L OW"], ["HELLO", "HH EH L OW"]); // Invalid
```

### b. Adding grammars

A FSG is a structure that includes an initial state, a last state as well as a set of transitions between these states. Again, make sure all words used in transitions are in the dictionary (either loaded through a packaged dictionary file or added at runtime using `addWords`). Here is an example of inputting one grammar:

```javascript
var transitions = new Module.VectorTransitions();
// log-probability is 0 (i.e. probability is 1.0):
transitions.push_back({from: 0, to: 1, logp: 0, word: "HELLO"});
transitions.push_back({from: 1, to: 2, logp: 0, word: "WORLD"});
// null-transition:
transitions.push_back({from: 1, to: 2, logp: 0, word: ""});
var ids = new Module.Integers();
if (recognizer.addGrammar(ids,
                          {start: 1,
                           end: 2,
                           numStates: 3,
                           transitions: transitions}) != Module.ReturnType.SUCCESS)
     alert("Error while adding grammar"); // Meaning that the grammar has issues
transitions.delete();
var id = ids.get(0); // This is the id assigned to the grammar
ids.delete();
```

Notice the `Integers` object that is used to return an id back to the app to refer to the grammar. This id is then used to switch the recognizer to using that specific grammar. You will note that `new Module.Integers()` actually returns a vector object that is then passed as a reference to `addGrammar`. If the call is successful, the first element of the array is the id assigned to the grammar.

### c. Adding key phrases

PocketSphinx also includes a keyword spotting search. Give the decoder a keyword or key phrase to catch and you can get, at any time, the number of times it was spotted. The key phrase is just a string with the phrase to spot. All words from the phrase must have been previously added with `addWord`.

```javascript
words.push_back(["HELLO", "HH AH L OW"], ["WORLD", "W ER L D"]);
recognizer.addWords(words);
var ids = new Module.Integers();
if (recognizer.addKeyword(ids, "HELLO WORLD") != Module.ReturnType.SUCCESS)
     alert("Error while adding key phrase"); // Meaning that the key phrase has issues
var id = ids.get(0); // This is the id assigned to the search
ids.delete();
```

Note that there is a threshold that can be set to define how sensitive the search is. Add `["-kws_threshold", "2"]` for instance to the config object. Default value is 1, higher values means more likely to be spotted and more likely to get false positives.

### d. Switching between grammars or keyword searches

A recognizer object can have any number of grammars and keyword searches but only one can be active at a time. The active search is the one used when there is a call to `start()`, described later in this document. To switch to a specific search, you must use the id that was given during the call to `addGrammar` or `addKeyword`.

```javascript
// id is the first element of the ids vector after call to addGrammar or addKeyword:
if (recognizer.switchSearch(id) != Module.ReturnType.SUCCESS)
     alert("Error while switching search"); // The id is probably wrong
```

## 3.4 Recognizing audio

To recognize audio, one must first call `start` to initialize recognition, then feed the recognizer with audio data with calls to `process` and finally call `stop` once done. During and after recognition, the recognized string can be retrieved with a call to `getHyp`.

Before calling start, one must make sure that the current language model is the correct one, mainly, whatever happened last:

* If a grammar or keyword search has just been given to the recognizer, it is automatically used as current language model.
* If a call to `switchSearch` was successful, the specified search will be used in the next call to `start`.
* If a SLM was packaged in `pocketsphinx.js` and was loaded by being added in the parameters to the `Config` object used when the recognizer was instantiated (or re-initialized), then this model is the current language model.

Calls to process must include audio buffers in the form of an `AudioBuffer` object. `AudioBuffer` objects can be re-used. They must contain audio samples, as 2-byte integers, recorded at 16kHz (unless your acoustic model uses different characteristics).

Here is an example:

```javascript
var array = ... // array that contains an audio buffer
var buffer = new Module.AudioBuffer();
for (var i = 0 ; i < array.length ; i++)
    buffer.push_back(array[i]); // Feed the array with audio data
var output = recognizer.start(); // Starts recognition on current language model
output = recognizer.process(buffer); // Processes the buffer
var hyp = recognizer.getHyp(); // Gets the current recognized string (hypothesis)
/* ... */
for (var i = 0 ; i < array.length ; i++)
    buffer.set(i, array[i]); // Feed buffer with new data
output = recognizer.process(buffer);
hyp = recognizer.getHyp();
/* ... */
output = recognizer.stop();
// Gets the final recognized string:
var final_hyp = recognizer.getHyp();
buffer.delete();
```

Remember to check the return values of the different calls and compare them to `Module.ReturnType....`.

For a keyword spotting search, use `addKeyword` instead of `addGrammar` as explained previously and use `getCount` instead of `getHyp`. `getCount` returns the number of times the key phrase was spotted since recognition started.


## 3.5 Releasing memory

In most cases you probably don't need to do that, but to free the memory used by the recognizer, you must call `recognizer.delete()`. Since you can re-initialize a recognizer with new parameters with a call to `reInit`, this should be only necessary if you're sure you don't need any recognizer object anymore.


# 4. Using `pocketsphinx.js` inside a Web Worker with `recognizer.js`

Using `recognizer.js`, `pocketsphinx.js` is downloaded and executed inside a Web worker. The file is located in `webapp/js/`, both `recognizer.js` and `pocketsphinx.js` must be in the same folder at runtime. It is intended to be loaded as a new Web worker object:

```javascript
var recognizer = new Worker("js/recognizer.js");
```

You can then interact with it using messages.

## 4.1 Incoming Messages

Messages posted to the recognizer worker might include the following attributes:

* `command`, command to be executed,
* `data`, data to be passed to the command,
* `callbackId`, id to be passed to the outgoing message, might be used to trigger a callback.

## 4.2 Outgoing Messages

The worker sends messages back to the UI thread, either to call back when actions have been performed, report errors or send periodic information such as the current recognition hypothesis.

Messages posted by the recognizer worker might include:

* `status`, which can be either `done` or `error`,
* `command`, the command that sent the message,
* `code`, an error code,
* `id`, a callback id that was given in the received incoming message,
* `data`, additional data that the callback function might make use of,
* `hyp`, the current recognition hypothesis,
* `final`, a boolean that indicates whether the hypothesis is final (sent after call to `stop`).

## 4.3 API description

### a. Error codes

The error codes returned in messages posted back from the worker can be:

* the error code returned by `pocketsphinx.js` as explained previously,
* or one of the following strings:
    * "js-data", if the provided data are invalid,
    * "js-no-recognizer", if the recognizer is not initialized.

### b. Initialization

Once the worker is created, the recognizer must be initialized:

```javascript
// This value will be given in the message received after the action completes:
var id = 0;
recognizer.postMessage({command: 'initialize', callbackId: id});
```

Once it is done, the recognizer will post a message back, for instance:

* `{status: "done", command: "initialize", id: clbId}`, if successful, where `clbId` is the callback id given in the original command message.
* `{status: "error", command: "initialize", code: initStatus}`, if there is an error, where `initStatus` is the value returned by the call to `psInitialize`, see above for possible values.

Recognizer parameters to be passed to `PocketSphinx` can be given in the call to `initialize`. For instance:

```javascript
recognizer.postMessage({command: 'initialize',
                        callbackId: id,
                        data: [["-hmm", "french"],
                               ["-fwdflat", "no"],
                               ["-dict", "french.dic"],
                               ["-lm", "french.DMP"]]
                       });
```

This will set the `pocketsphinx` command-line parameter `-fwdflat` to `no` and initialize the recognizer with the acoustic model `french`, the language model `french.DMP` and the dictionary `french.dic`, assuming `pocketsphinx.js` was compiled with such models.

Note that once it is initialized, the recognizer can be re-initialized with different parameters. That way, for instance, a web application can switch between different acoustic and language models at runtime.

### c. Adding words

Words to be recognized must be added to the recognizer before they can be used in grammars. See previous sections to know more about the format of dictionary items. Words can be added at any time after the recognizer is initialized, and several words can be added at once:

```javascript
// An array of pairs [word, pronunciation]:
var words = [["ONE", "W AH N"], ["TWO", "T UW"], ["THREE", "TH R IY"]];
recognizer.postMessage({command: 'addWords', data: words, callbackId: id});
```

The message back could be:

* `{id: clbId}`, the provided callback id, if given, as explained before, if successful.
* `{status: "error", command: "addWords", code: code}`, if error, where possible values of the error code was described above.

Note that words can have several pronunciation alternatives as explained in Section 3.3.a.

### d. Adding grammars or key phrases

As described previously, any number of grammars or keyword searches can be added. The recognizer can then switch between them. 


A grammar can be added at once using a JavaScript object that contains the number of states, the first and last states, and an array of transitions, for instance:

```javascript
var grammar = {numStates: 3,
               start: 0,
               end: 2,
               transitions: [{from: 0, to: 1, word: "HELLO"},
                             {from: 1, to: 2, logp: 0, word: "WORLD"},
                             {from: 1, to: 2}]
              };
recognizer.postMessage({command: 'addGrammar', data: grammar, callbackId: id});
```

All words must have been added previously using the `addWords` command.

Notice that `logp` is optional, it defaults to 0. `word` is also optional, it defaults to `""` which is a null-transition.

In the message back, the grammar id assigned to the grammar is given. It can be used to switch to that grammar. So the message, if successful, would be like `{id: clbId, data: id, status: "done", command: "addGrammar"}`, where `id` is the id of the newly created grammar. In case of errors, the message would be as described previously.

Similarly, keyword spotting search can be added by just providing the key phrase to spot:

```javascript
var keyphrase = "HELLO WORLD";
recognizer.postMessage({command: 'addKeyword', data: keyphrase, callbackId: id});
```

Just as like with grammars, words should already be in the recognizer, and the id of the newly added search is given in the callback. As explained previously, you might want to ajust the sensitivity threshold when initializing the recognizer, for example with providing `["-kws_threshold", "2"]`.


### e. Starting recognition

The message to start recognition should include the id of the grammar (or keyword search) to be used:

```javascript
// id is the id of a previously added grammar:
recognizer.postMessage({command: 'start', data: id});
```

### f. Processing data

Audio samples should be sent to the recognizer using the `process` command:

```javascript
// array is an array of audio samples:
recognizer.postMessage({command: 'process', data: array});
```
Audio samples should be 2-byte integers, at 16 kHz.

While data are processed, hypothesis will be sent back in a message in the form `{hyp: "RECOGNIZED STRING", count: c}`. If it is a keyword spotting search, the number of occurrences of the key phrase is given in the `count` field.

### g. Ending recognition

Recognition can be simply stopped using the `stop` command: 

```javascript
recognizer.postMessage({command: 'stop'});
```

It will then send a last message with the hypothesis, marked as final (which means that it is more accurate as it comes after a second pass that was triggered by the `stop` command). It would look like: `{hyp: "FINAL RECOGNIZED STRING", count: c, final: true}`.

## 4.4 Using `CallbackManager`

In order to facilitate the interaction with the recognizer worker, we have made a simple utility that helps associate callbacks to be executed when the worker posts a message responding to a command you sent. You can find `callbackManager.js` in `webapp/js`.

To use it, first create a new instance of CallbackManager:

```javascript
var callbackManager = new CallbackManager();
```

When you post a message to the recognizer worker and want to associate a callback function to it, you first add your callback function to the manager, which gives you a callback id in return:

```javascript
recognizer.postMessage({command: 'addWords',
                        data: words,
                        callbackId: callbackManager.add(
                           function() {alert("Words added");})
                       });
```

In the `onmessage` function of your worker, use the callback manager to check and trigger callback functions:

```javascript
recognizer.onmessage = function(e) {
    if (e.data.hasOwnProperty('id')) {
        // If the message has an id field, it
        // means that we might have a callback associated
        var clb = callbackManager.get(e.data['id']);
        var data = {};
        // As mentioned previously, additional data can be passed to the callback
        // such as the id of a newly added grammar
        if(e.data.hasOwnProperty('data')) data = e.data.data;
        if(clb) clb(data);
    }
    // Check for other message types here
};
```

Check `live.html` in `webapp` for more examples.


## 4.5 Detecting when the worker is ready

When a new worker is instantiated, it immediately returns a worker object, but the actual download of the JavaScript files might take some time, especially in our case where `pocketsphinx.js` is fairly large. One way of detecting whether the files are fully downloaded and loaded is to post a first message right after it is instantiated and wait for a message back from the worker.

````javascript
var recognizer;
function spawnWorker(workerurl, onReady) {
    recognizer = new Worker(workerurl);
    recognizer.onmessage = function(event) {
        // onReady will be called when there is a message
        // back
        onReady(recognizer);
    };
    recognizer.postMessage('');
};
```

The first message posted to the recognizer can include the name of the PocketSphinx JavaScript file to load. This is handy if you want to build an application with several different models, you can keep the same `recognizer.js` file for different parts of your application and load any PocketSphinx JavaScript file that you want. By default, it will load `pocketsphinx.js`, but if you want your application to load a file called `pocketsphinx_chinese.js`, you can just add it as parameter to the first posted message:

````javascript
var recognizer;
function spawnWorker(workerurl, onReady) {
    recognizer = new Worker(workerurl);
    recognizer.onmessage = function(event) {
        // onReady will be called when there is a message
        // back
        onReady(recognizer);
    };
    recognizer.postMessage('pocketsphinx_chinese.js');
};
```

After the first message back was received, proper listening to onmessage can be added:

```javascript
spawnWorker("js/recognizer.js", function(worker) {
    worker.onmessage = function(e) {
    // Add what you want to do with messages back from the worker
    };
    // Here is a good place to send the 'initialize' command to the recognizer
});
```

Of course, the worker must be able to respond to the first message, as we did in `recognizer.js`:

```javascript
function startup(onMessage) {
    self.onmessage = function(event) {
        self.onmessage = onMessage;
        self.postMessage({});
    }
};
// This function is called first, it triggers
// a first postmessage, then adds the proper respond to
// commands: 
startup(function(event) {
    switch(event.data.command){
        //We deal with commands properly
    }
});
```

All these are illustrated in `webapp/live.html` and `recognizer.js`.

# 5. Wiring `recognizer.js` to the audio recorder

We include an audio recording library based on the Web Audio API that accesses the microphone, gets audio samples, converts them to the proper sample rate (16kHz for our default acoustic model), and sends them to the recognizer. This library is derived from [Recorderjs](https://github.com/mattdiamond/Recorderjs). To know more about audio capture and playback on the web, you could have a look at this [overview of audio on the Web](https://github.com/syl22-00/TechDocs/blob/master/AudioInBrowser.md). A more complete documentation of the recorder can be found in `doc/AudioRecorder/README.md`.

Include `audioRecorder.js` in the HTML file and make sure `audioRecorderWorker.js` is in the same folder. To use it, create a new instance of `AudioRecorder` giving it as argument a `MediaStreamSource`. As of Today, the Google Chrome and Firefox (25+) implement it. You also need to set the recognizer attribute to a Recognizer worker, as described above.

```javascript
// Deal with prefixed APIs
window.AudioContext = window.AudioContext || window.webkitAudioContext;
navigator.getUserMedia = navigator.getUserMedia ||
                         navigator.webkitGetUserMedia ||
                         navigator.mozGetUserMedia;

// Instantiating AudioContext
try {
    var audioContext = new AudioContext();
} catch (e) {
    console.log("Error initializing Web Audio");
}

var recorder;
// Callback once the user authorizes access to the microphone:
function startUserMedia(stream) {
    var input = audioContext.createMediaStreamSource(stream);
    recorder = new AudioRecorder(input);
    // We can, for instance, add a recognizer as consumer
    if (recognizer) recorder.consumers.push(recognizer);
  };

// Actually call getUserMedia
if (navigator.getUserMedia)
    navigator.getUserMedia({audio: true},
                            startUserMedia,
                            function(e) {
                                console.log("No live audio input in this browser");
                            });
else console.log("No web audio support in this browser");
```

Once the recorder is up and running, you can start and stop recording and recognition with:

```javascript
// To start recording:
recorder.start();
// The hypothesis is periodically sent by the recognizer, as described previously
// To stop recording:
recorder.stop();  // The final hypothesis is sent
```

The constructor for AudioRecorder can take an optional config object. This config can include a callback function which is executed when there is an error during recording. As of today, the only possible error is when the input samples are silent. It can also include the output sample rate, which you might need to set if you use an acoustic model of 8kHz audio.

```javascript
var audioRecorderConfig = {
    errorCallback: function(x) {alert("Error from recorder: " + x);},
    outputSampleRate: 8000
    };
recorder = new AudioRecorder(input, audioRecorderConfig);
```

All these are illustrated in the given live demo, in the `webapp/` folder.

Note that live audio capture is only available on recent versions of Google Chrome and Firefox. Chrome, prior to version 29, only produced silent audio on many platforms. Firefox includes the necessary features starting from version 25.

# 6. Live demo

The file `webapp/live.html` is an example of live recognition using the web audio API. It works on Chrome and Firefox (25+), if the web audio API actually works. Note that we observed the recorded audio to be silent on some configurations we have tried.

To build an application, this is a good starting point as it illustrates the different components described in this document. In that demo, three different grammars are available and the app can switch between them.

There is also:

* `live_kws.html` for keyword spotting,
* `live_zh.html` for recognition of Chinese.

# 7. Test suite

There is a test suite in `tests/js` which makes use of [QUnit](http://qunitjs.com). There is a README file inside the folder.

# 8. Notes about speech recognition and performance

If you are not familiar with speech recognition, you might need to take some time to learn some of the concepts, mainly:

* acoustic models (we provide one small model for English but other Sphinx acoustic models can be used as well),
* language models (at this point, we only have the API to input grammars, as FSGs, but API to input statistical language models could be added),
* Cepstral Mean Normalization (CMN) and the different CMN strategies.

In terms of performance, you should get exactly the same result as using PocketSphinx compiled on other platforms. For instance, because of the CMN policy, the accuracy of the first utterance is usually pretty bad, especially for non-native speakers.

## 8.1 Acoustic model

The `am` folder contains an acoustic model trained with [SphinxTrain](http://cmusphinx.sourceforge.net/wiki/tutorialam). It is built using the [RM1](http://www.speech.cs.cmu.edu/databases/rm1/index.html) corpus, semi-continuous, with 200 senones.

## 8.2 PocketSphinx

PocketSphinx.js ships with PocketSphinx and Sphinxbase as they appear in the trunk of the subversion tree, and we try to keep is regularly updated. There is no modification nor fixes. However, the `model` folder of PocketSphinx (which contains large acoustic and language models) was not included.

# 9. License

PocketSphinx licensing terms are included in the `pocketsphinx` and `sphinxbase` folders. 

The files `webapp/js/audioRecorder.js` and `webapp/js/audioRecorderWorker.js` are based on [Recorder.js](https://github.com/mattdiamond/Recorderjs), which is under the MIT license (Copyright © 2013 Matt Diamond).

The remaining of this software is licensed under the MIT license:

Copyright © 2013-2014 Sylvain Chevalier

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
