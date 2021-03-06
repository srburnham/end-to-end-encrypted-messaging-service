# End to End Encrypted Messager w. MQTT
A fun (but educational!) way to learn about asymmetric encryption and MQTT

## Prerequisites

To follow this workshop, you will need:

- A modern web browser
- Your favourite IDE
- An IBM Cloud account (which you can create [here](https://ibm.biz/Bdq8Yb))

## Introduction

In this workshop, we'll cover the basics of creating an end to end encrypted messaging application which uses MQTT to pass messages between one or more users.

To this end, we'll be creating a Node.js application which will use 

- [MQTT](https://en.wikipedia.org/wiki/MQTT) to pass messages between users through a public MQTT broker
- [Public-key cryptography](https://en.wikipedia.org/wiki/Public-key_cryptography) to encrypt messages sent to others, and decrypt messages sent to us.
- [Express.js](https://en.wikipedia.org/wiki/Express.js) to deliver an application in a web browser that can send/receive messages

We won't be covering the minutiae of public-key cryptography, but by the end of this workshop we'll have an application that can send messages that should be secure up until [at least 2030](https://www.jscape.com/blog/should-i-start-using-4096-bit-rsa-keys).

## Getting started

First things first, to follow this workshop and make deploying our application to the cloud a little easier, fork this repo by clicking the 'Fork' button at the top of this repo to create your own copy that you'll be able to work from.

Once the forking process have completed, clone the repository to your local system for working on. You can do this by clicking on the "Code" button found at the top of your newly forked repo. A new dialog will appear with a URL to download the contents of your repo (it should look something like `git@github.com:<YOUR_GH_USERNAME>/end-to-end-encrypted-messaging-service.git`).

Copy that address and open the terminal application on your system and run the following command:

```
git clone <URL YOU JUST COPIED>
```

You should now have a local copy of your repository on your system that you can work on 🎉

Enter that folder with:
```
cd end-to-end-encrypted-messaging-service
```

...and then create a new file called `.env` however you prefer to create new files.

I like `touch .env`.

Finally, now's a good time to install our application dependecies. Run `npm install` to get all of our dependencies ready when we want to run our app.

Next, open up `index.js` in your favourite IDE, this is where we'll be writing most of the code for this workshop.

## Generating Key pair

When you see the contents of `index.js` in your favourite IDE, you should see a number of dependencies and variables that we'll be using throughout the application we're about to create.

Let's take a quick look at those.

[dotenv](https://www.npmjs.com/package/dotenv) is a lovely little module which allows us to store environment variables for our application in a .env file when we're developing locally, but don't include to production when we deploy our application to the cloud.
```javascript
require('dotenv').config({silent : process.env.NODE_ENV === "production"});
```

Next up, we have `fs` and `crypto`. We'll be using these modules to create our pair of encryption keys and store them in our system.

```javascript
const fs = require("fs");
const crypto = require("crypto");
```

Afterwards, we bring in the Express framework, HTTP, and some Express middleware modules that will let us stand up a very simple HTTP server that will deliver the web app that we'll send messages from.

```javascript
const express = require('express');
const http = require('http');
const bodyParser = require('body-parser');
```

We use MQTT in this application to pass messages between users using a publicly accessible MQTT broker. Using a public broker to send messages between users may seem counterintuitive, but remember, we're using asymmetric encryption, nothing unencrypted is ever sent to the broker, so even though it's publicly accessible to anybody, nobody but the intended recipient will be able to read your messages!

All of the variables that we're assigning here and populated with values from our environment variables. Using environment variables is good practice for building applications that are scalable and transferrable. On our local system, these values will be populated with the values of a matching name from the `.env` file, but on a production system, they'll be populated by the values of the actual environment variables.

If you'd like to know more about building highly scalable/available/transferrable applications [The 12 factory app methodology](https://12factor.net/) is a must-read.

We also connect to the public MQTT broker with `mqtt.connect`. We'll be configuring the address in the next few steps.

```javascript
// Communication Libraries
const mqtt = require('mqtt');

// Configuration Variables
const privateKeyPassphrase = process.env.PRIVATE_KEY_PASSPHRASE || "test";
const MQTT_BROKER_ADDR = process.env.MQTT_BROKER_ADDR;
const USERNAME = process.env.USER_NAME;
const MSGTOPIC = process.env.MESSAGE_TOPIC;

const MQTTClient = mqtt.connect(MQTT_BROKER_ADDR);
```

Finally, we set up the variables that we'll use to spin up our HTTP server, and keep track of messages and users we'll want to communicate with.

```javascript
const app = express();
const server = http.createServer(app);

const RECEIVED_MESSAGES = [];
const PUBLIC_USER_KEYS = {};

let publicKey;
let privateKey;
```

## Setting Local Enviroment Variables

As noted above, our application is configured with environment variables. On our local system these are stored in that lovely little `.env` file we created a little earlier.

Open up `.env` in your favourite IDE and copy and paste the following, editing where instructed:

```
USER_NAME=<REPLACE WITH THE NAME YOU WISH TO SHOW WHEN COMMUNICATING WITH OTHERS>
MESSAGE_TOPIC=ibm_developer_uk
MQTT_BROKER_ADDR=mqtt://mqtt.eclipse.org
PRIVATE_KEY_PASSPHRASE=<REPLACW WITH A REASONABLY SECURE PASSPHRASE FOR ENCRYPTING YOUR PRIVATE KEY>
```

The `USER_NAME` and `PRIVATE_KEY_PASSPHRASE` are self explanatory. 

`MESSAGE_TOPIC` is the [MQTT topic](https://en.wikipedia.org/wiki/MQTT#Overview) that we'll use to make sure everyone can find messages being sent to/from them.

`MQTT_BROKER_ADDR` is the URL that we can access the MQTT-powered message broker that will relay messages to and from each user. We'll be using the publily accessible Eclipse Foundation MQTT broker for this workshop

Once you've added that data to the `.env` file save and close it.

## Encryption keys
### Checking if we have existing keys

When our application spins up, the first thing we want it to do is check if there is already a public and private encryption key that we can use, and if so, load it up into our application. If there's no key-pair we will then generate and store those keys for use in the future.

On the line after `let privateKey` add the following chunk of code.

```javascript
if(!fs.existsSync(`${__dirname}/public.pem`) || !fs.existsSync(`${__dirname}/private.pem`)){
	console.log('Valid key pair not found. Generating new pair...');

    // Code Block 1
    

} else {
    
    // Code Block 2

    
}

// Code Block 3

```

Here, we're checking to see if there is both a `private.pem` and `public.pem` in our applications working directory. If either one is not found, we'll generate a new pair and save them to disk.

Now, I know what some of you looking at ```!fs.existsSync(`${__dirname}/public.pem`)``` might be saying:

_"You shouldn't be using synchronous operations in your application, that slows things down!"_

And you're right, that's completely true 99% of the time - but this is one of the 1% instances where I think it's totally fine:

1. Our application is starting up at this point, there's nothing it needs to be doing right now other than checking it has everything it needs to get going
2. Everything after this point in the application depends upon these operations being completed - managing the asynchronously is messy and unnecessary complex at this point in the application
3. Yes, it is slower - but we're talking **milliseconds** here, I think we can sacrifice a little bit of startup time for the savings we get in not spending 5 minutes writing pleasing, but ultimately uneeded promise chains or callbacks.

### Generating our key pair

Just below the line that reads `// Code Block 1`, copy and paste the following code:

```javascript
const keys = crypto.generateKeyPairSync('rsa', {
    modulusLength: 4096,
    namedCurve: 'secp256k1',
    publicKeyEncoding: {
        type: 'spki',
        format: 'pem'
    },
    privateKeyEncoding: {
        type: 'pkcs8',
        format: 'pem',
        cipher: 'aes-256-cbc',
        passphrase: privateKeyPassphrase
    }
});

publicKey = keys.publicKey;
privateKey = keys.privateKey;

console.log('Key pair successfully generated. Writing to disk.');

fs.writeFileSync(`${__dirname}/public.pem`, publicKey, 'utf8');
fs.writeFileSync(`${__dirname}/private.pem`, privateKey, 'utf8');

console.log('Files successfully written.');
```

The first little bit of code generates a public-key pair using the [RSA cryptosystem](https://en.wikipedia.org/wiki/RSA_(cryptosystem)) with a [key size](https://en.wikipedia.org/wiki/Key_size) of 4096 (the largest  key that can be generated with RSA) passed through in `modulusLength`.

Long story short, bigger `modulusLength`, more secure keys we have - at least, that's the theory 😅

For our public key we're using SPKI (pronounced _spooky_) to encode our key. For our private key we're using PKSC-8 and a 256 bit AES cipher. The main difference between these two encodings is that our PKCS-8 key can be encrypted with a passphrase.

Both keys are returned as PEM format (a base-64 encoded) certificates and assigned to the `publicKey` and `privateKey` properties of the `keys` object.

Once our keys have been generated, we're passing them to our global `publicKey` and `privateKey` variables for use in the rest of our application.

Finally, we write our certificates to disk so that the next time we run our applications we can just use those instead of generating a new pair.

### Loading our keys from disk

To that end, if we already have keys we can load them and assign them to the `publicKey` and `privateKey` variables. Copy the following code and paste it just after the line that reads `// Code Block 2`:

```javascript
console.log('Existing keys found. Using those.');
publicKey = fs.readFileSync(`${__dirname}/public.pem`, 'utf8');
privateKey = fs.readFileSync(`${__dirname}/private.pem`, 'utf8');
```

## Encrypting + Decrypting Messages

### Encrypting Messages
Now that we have a public-key pair we're able to encrypt and then decrypt our own messages. Because we're going to be doing that a lot in our application, we're going to write some helpful functions that will do the job for us `encrypt` and `decrypt`.

First, let's put together the code for the `encrypt` function.

On the line immediately after `// Code Block 3` copy and paste the following:

```javascript
function encrypt(data, key){

	const buffer = Buffer.from(data);
	const encrypted = crypto.publicEncrypt(key, buffer);
	return encrypted.toString("base64");

}

// Code Block 4
```

As its name may suggest, this is our "encrypt" function. It takes two arguments `data` and `key`. The data argument is the thing that we want to encrypt, the key argument is the key that we want to use to encrypt that data. When we use this function, we'll be passing our messages that we want to send to other people as the `data` parameter, and the recipient's public key as the `key` parameter.

We first convert our string to a buffer. Any of data can be encrypted with public-key encryption, so it makes little sense for the `crypto.publicEncrypt` function to expect a string as an argument.

Once `crypto.publicEncrypt` has done it's job, it returns a buffer of encrypted data. We'll want to send this to other people through an MQTT broker in a little bit, so we'll convert it to a BASE64 string to save us having to deal with any headaches that may arise from the data being in binary form while it's being transmitted.

### Decrypting Messages

Now that we've put together a way to encrypt messages, we'll need to be able to decrypt some too.

Copy and paste the following block of code after the line that reads `// Code Block 4`.

```javascript
function decrypt(data, key){

	var buffer = Buffer.from(data, "base64");
	const decrypted = crypto.privateDecrypt({ key: key, passphrase: privateKeyPassphrase }, buffer);
	return decrypted.toString("utf8");

}

// Code Block 5
```

As you may expect, our `decrypt` function is like our `encrypt` but in reverse. Like the `encrypt` function our `decrypt` function has a `data` and `key` paramater - except this time our data is the base64 encoded string we wish to decode, and the key is our private key.

**_It's important to note that our private key can only be used to decrypt messages that have been encrypted with its matching public key._**

This time, we reconvert our base64 encoded string back to a buffer and then pass this through to `crypto.privateDecrypt` with our private key *and* the passphrase we will have set to encrypt our private key (we wouldn't want just anybody to be able to read it now, would we?).

`crypto.privateDecrypt` will take our buffer of data, decrypt it and then return a new buffer with the decrypted information. We then convert this back to a user readable string and return it as the result of the function with `return decrypted.toString('utf8');`.

If you want to test this out, you can copy and paste the following after `// Code Block 5` and then run the code with `node index.js`

```javascript
const originalString = "Hello, world.";
console.log('Original:', originalString);

const encryptedString = encrypt(originalString, publicKey);
console.log('Encrypted:', encryptedString);

const decryptedString = decrypt(encryptedString, privateKey);
console.log('Decrypted:', decryptedString);
```

And you should see something like this:

```
Original: Hello, world.
Encrypted: lBoO0thivi/kl8dBJ2UOaNQQn3KwuI..... // Truncated
Decrypted: Hello, world.
```

Remember, the only reason we can decrypt this message with our private key is _because we encrypted it with our public key_. If we were using _someone else's public key_ to encrypt the message, **we wouldn't be able to decrypt it with our private key**.

## Connecting to the Message Broker

Now that we've gotten all of our encryption/decryption logic set up, it's time to start putting together the code that will enable users to communicate with each other by an MQTT-powered messaging broker.

At the start of our `index.js` file, we have the line `const MQTTClient = mqtt.connect(MQTT_BROKER_ADDR);` which will connect our application to the MQTT broker. Once the connection has been established, we'll want to do something with that connection.

We're going to do two things

1. Subscribe to a some topics on the broker
2. Publish a message to the broker on a topic.

Copy and 

```javascript
MQTTClient.on('connect', function () {

    // Subscription 1
	MQTTClient.subscribe(`${MSGTOPIC}/message/${USERNAME}/#`, function (err) {
		if (err) {
			console.log('Failed to subscribe to:', `${MSGTOPIC}/message/${USERNAME}`);
		}
	});

    // Subscription 2
    MQTTClient.subscribe(`${MSGTOPIC}/announce/#`, function (err) {
        if (err) {
            console.log('Failed to subscribe to:', `${MSGTOPIC}/announce/#`);
		}
	});

    // C'est Moi.
	MQTTClient.publish(`${MSGTOPIC}/announce/${USERNAME}`, publicKey );

});
```

When we subscribe to a topic on an MQTT broker, we're essentially saying "I would like to know about any message sent on this topic". 

The `${MSGTOPIC}` variable is one that we set in the environment variables at the start. With it, we're essentially creating a namespace that we can call our own to listen for messages on - exactly analogous to a chat room, but with MQTT topics rather than having to manually manage connections + users.

For `Subscription 1` we're asking to have any message sent for **us specifically** to be sent on to us. The `#` is the MQTT wildcard - in our application it will be populated by the name of the person publishing the message to us on that topic.

For `Subscription 2` we're asking to have any message sent to the `/announce/` topic forwarded to us. When we spin up our application we'll publish our public key on the `/announce/<YOUR USERNAME>` topic. This will allow anybody already connected to the broker to encrypt messages to send to the person that just arrived.

Finally, with `MQTTClient.publish` we transmit our public key on the topic `/announce/<YOUR_USERNAME>` topic. This way, people will be able to encrypt messages with our public that only we'll be able to read.

## Handling messages

Now that we're subscribed to the two different topics, we want to be able to handle messages that arrive for those topics in our application.

Remember, there are two topics that we're looking for messages on:

1. `/announce` - Fired when someone new joins the broker with this application
2. `/message` - triggered whenever a message is sent for us.

Copy and paste the following code after 
```javascript
MQTTClient.on('message', function (topic, message) {

	const topicParts = topic.split('/');
	const type = topicParts[1];

    console.log('Message type:', type);
    
    // Code Block 6

});
```

Every time a message is received on one of the topics that we've subsribed to, this function will be triggered.

Now, we'll want to know how best to handle each message based on what "type" it is (that's not an MQTT concept, it's just something that we're creating for this application).

Each of the topics weve subscribed to starts with `<FAUX_NAMESPACE>/<TYPE>/<USER>`, so we can split the topic on the `/` character and deduce how best to handle the message payload based on that information.

### Exchanging public keys

Let's handle when the message arrives on the `"announce"` topic. Copy and paste the following code after `// Code Block 6`:

```javascript
if(type === 'announce'){

    const user = topicParts[2];

    if(user !== USERNAME && !PUBLIC_USER_KEYS[user]){

        PUBLIC_USER_KEYS[user] = {
            key : message.toString(),
            name : user
        };

        MQTTClient.publish(`${MSGTOPIC}/announce/${USERNAME}`, publicKey );
            
    }

} else if(type === 'message') {

    // Code Block 7

}

// Code Block 8

```

In this block of code, if we deem the message to be one that's announcing someone new is passing messages around, we first assign the variable `user` to be the value that's passed along in the (now split) topic.

Next, we want to check that the announcement of a key isn't one that we sent ourselves. Remember, when we first connect to the broker we announce our own presence too - there's no point in processing a message that we ourselves sent.

Second, we check whether or not we have a public key for this user already. If not, we'll add them to our `PUBLIC_USER_KEYS` object with their name (as derived from the topic) and their public key (recieved in the payload of the message) so that we can send encrypted messages to them.

Finally, we then also announce ourselves. For everyone who's already aware of us this will have no effect, but for the new person connected to the broker, it'll give them a chance to receive our public key so that they can send messages to us.

### Handling received messages

Now that we have the code for handling users announcing their presence, we're ready to write a little bit of code to handle receiving messages intended for us.

Copy and paste the following code after `// Code Block 7`:

```javascript
const to = topicParts[2];

if(to === USERNAME){

    const from = topic.split('/')[3];
    RECEIVED_MESSAGES.push({
        from : from,
        msg : decrypt( message.toString(), privateKey ),
        received : Number(Date.now())
    });

    console.log('Received messages:', RECEIVED_MESSAGES);

}
```

The first thing we're doing here is creating a handy variable `to` for checking who the message was intended for.

We subscribed to recieve every single message that matched our topic after `/message` so we're going to recieve other peoples messages too - but that doesn't matter - any message that has been sent by our application will only be decryptable by people with the right private key. So, while we're able to receive the encrypted messages meant for other people, we don't have the right private key to decrypt them, so all we'll get is gobbledygook.

To save us wasting time trying to decrypt messages that aren't intended for us, we'll simply check if the `to` variable is equivalent to `USERNAME` - the username we set for ourselves in the environment variables at the start of this workshop.

If the message is intended for us, then we'll split up the message topic again and get the username of the person who sent it to us. Then we'll store an object in the `RECEIVED_MESSAGES` (one of the variables that we started this application with) with the senders name, the decrypted message, and the time receive so that they can be retrieved by a viewer later.

## Sending Messages

With that, we have all of the code we need to receive public keys and messages from other users - Great! 🎉

But now we need some way to send messages...

Well, we could create a command-line interface, but that wouldn't be much fun - things are better when they're visual right?

Yes. They are 👍

In this last short section, we'll use the Express.js dependency that we included earlier to enable our application to recieve HTTP requests containing messages from a web app which will then be encrypted by our application and sent over MQTT to whomever we so desire.

### Delivering our web app

After the line that reads `// Code Block 8` copy and paste the following code:

```javascript
app.get('/', (req, res, next) => {
	res.sendFile(`${__dirname}/index.html`);
});

// Code Block 9
```

This registers an HTTP route in our application server that will deliver the `index.html` page (it contains all of the logic we need to send and display messages we receive) to any web browser that requests it.

### Handling messages to be sent

Next up, we need a route that our HTTP app can `POST` data to for encryption and forwarding to the MQTT broker.

Copy and paste the following block of code beneath the line that reads `// Code Block 9`:

```javascript
app.post('/send', [ bodyParser.json() ], (req, res) => {

	Object.keys(PUBLIC_USER_KEYS).forEach(key => {
		const user = PUBLIC_USER_KEYS[key];
		MQTTClient.publish(`${MSGTOPIC}/message/${user.name}/${USERNAME}`, encrypt( req.body.msg, user.key ) );
	});

	res.end();
});

// Code Block 10
```

In this code, we're sending the messages that we want to send to everyone that we have recieved a public key for - so this is more like a chat room than a one-to-one messenger application, but there's nothing stop us from adjusting our code to achieve 1-2-1 communication between people.

First, we get the users information from `PUBLIC_USER_KEYS` and create a `user` variable that we can use to neatly access that users' name and public key. We then pass that information to `MQTTClient.publish` telling it to publish the information on a topic specifically for that user. 

Rather than sending the message unencrypted we pass that as an argument to `encrypt` with the message we wish to send, and that users public encryption key.

Finally, we end the HTTP request with `res.end()` so the our web application know that the message has been successfully received.

### Getting the messages for display

Lastly, we need one final HTTP route that our web app can use to check for messages that have been recieved and decrypted by our application.

After the line that reads `// Code Block 10` copy and paste the following bit of code:

```javascript
app.get('/check', (req, res, next) => {

	res.json({
		messages : RECEIVED_MESSAGES
	});

	RECEIVED_MESSAGES.length = 0;

});
```

Every 500ms, our web application will hit the `/check` endpoint to see if there's any messages waiting for our user to read. Every time we send the messages back to the client we empty them out from our server. We don't need to hang on to them anymore.

### Starting our HTTP server

That's all of our Asymmetric Encryption/MQTT/HTTP logic done 🚀

All that needs to be done now is right one final bit of code which tells our application to listen on a designated port for connections and requests. We can do that by adding the following right at the end of our `index.js` file.

```javascript
server.listen(process.env.PORT || 8080, () => {
    console.log(`Server started on port ${server.address().port} :)`);
});
```

And we're good to go!

If you want to test this out locally, you can enter `npm run start` in the working directory of this project with your terminal and (fingers crossed) it should spin up and happily report that there's now a server started. Then you can head over to [localhost](http://localhost:8080) to see our lovely web app that will let us send and recieve encrypted messages.

### Committing to GitHub

So, now that we have all of our code, we want to send it somewhere right?

At the very start, we forked this repo so we had our own copy of the code, and now that we've worked on it, it's ready to go back to GitHub.

In your teminal run the following commands one after the other

`git add .`
`git commit -m 'First Verion'`
`git push origin master`

That will send our application up to GitHub for distrubtion somewhere else on the internet - And that's cool - but do you know what's really cool? ~~A Billion Dollars~~ deploying our application to the cloud!

## Deploying to IBM Cloud
### Accessing the IBM Cloud Shell
Deploying our application to IBM Cloud is super-simple and should take less than 5 minutes to fire up.

We're going to use the IBM Cloud Shell which will give us a free virtual environment to run an interactive shell from which we can deploy our application.

Go to https://cloud.ibm.com/shell and you should see this:

![An image of the IBM Cloud Shell](images/1.png)

The IBM Cloud shell is pretty neat - it has all of the tools we'll need to deploy our app to IBM Cloud, so we can get things stood up even quicker than we normally would!

### Cloning our code
The first thing we're going to do it clone our GitHub repo to this system so we can deploy it as an application.

Just like we did when we first cloned our newly forked repo to our local system, run the following command to clone it to your Shell environment.

`git clone https://github.com/<YOUR_GITHUB_USERNAME>/end-to-end-encrypted-messaging-service.git`

Note that this command retrieves the code over HTTPS, and that we will need to replace `<YOUR_GITHUB_USERNAME>` with your GitHub username.

### Configuring the IBM Cloud CLI tool

Once your code has been downloaded to the shell environment, we want to enter that directory to execute commands. Do that with:

`cd end-to-end-encrypted-messaging-service/`

Next, you'll need to login to the IBM Cloud with the CLI tool. Run the following commands and follow the on-screen instructions.

`ibmcloud login`

Once that's done, we're logged in and able to execute commands to deploy our application to the IBM Cloud, but first we need to do a little configuration to make sure it goes to the right place.

### Configuring the IBM Cloud CLI

#### Targeting the correct region
This first thing we need to do is target the correct region. As our shell environment is based in Frankfurt, the CLI tool defaults to the `eu-de` region, but we'll want to target the `eu-gb` region instead.

Run the following command to target the `eu-gb` region

`ibmcloud target -r eu-gb`

#### Target the right Cloud Foundry Space

The application we've built is going to be deployed as a Cloud Foundry application (the PaaS platform for running apps on the IBM Cloud). To do this, we need to target an org and a space. Fortunately, when you IBM Cloud account is created you'll have some defaults, so run the following command to interactively target the correct org and space and select all of the defaults

`ibmcloud target --cf`

#### Deploying our application

It's time! Let's ship it 🚢

Doing so is really easy - simply type in

`ibmcloud cf push`

And watch as our application is deployed before our eyes! This should take 2-3 minutes.

Once that's done, you should see something like this

![An image of a successfully deployed CF app](images/2.png)

Take note of the `route` property, that's where our application is going to be accessible. Your's won't be the same as the one in the image, but if you head to the URL, you should see an app not unlike this

![An image of our web app front end](images/3.png)

It's not much, but it's honest work - but we're not _quite_ ready to send messages yet.

#### Setting environment variables - one. last. time 👍

Remember those environment variables we had stored in our `.env` file a little while ago? Well, they aren't included in our repo because that's where our secrets are kept 😅

This is the prime time now, and we don't want all of our secrets out on the open web, so we're going to add our environment variables to our production environment with the following commands.

`ibmcloud cf set-env e2e-workshop USER_NAME <YOUR_DESIRED_USERNAME>`
`ibmcloud cf set-env e2e-workshop MESSAGE_TOPIC ibm_developer_uk`
`ibmcloud cf set-env e2e-workshop MQTT_BROKER_ADDR mqtt://mqtt.eclipse.org`
`ibmcloud cf set-env e2e-workshop PRIVATE_KEY_PASSPHRASE <YOUR PRIVATE KEY PASSPHRASE>`

Once that's done (just as the CLI tool suggests) run `ibmcloud cf restage e2e-workshop` to make sure all of our newly assigned environment variables take effect.

This will rebuild our app, but then we're ready to go!

Congrats, you should now be able to use the web app to talk to other people who've completed this workshop 🎉
