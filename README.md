# Building an Integromat app for NEAR Protocol

This article describes how I built the NEAR app for Integromat, including what research I went through,
what sites I used to learn and build the app as well as some tips on what to avoid and what to not to.

## Overview

We will go over the following topics (if any of the topics seem like you already know them, feel free to skip to the part that interests you):

**Note**: For those of you who just want the quick fix of information, you can skip ahead to the summary section, but I would encourage you to also glance over the tips and tricks section.

- [Demo](#demo)
- [Integromat](#integromat)
- [NEAR](#near)
- [Integromat custom apps](#integromat-custom-apps)
  - [Base](#base)
  - [Modules](#modules)
    - [Parameters](#parameters)
  - [RPC](#rpc)
  - [IML Functions](#iml-functions)
- [Integrating with NEAR RPC API](#integrating-with-near-rpc-api)
- [Tips, tricks and issues](#tips-tricks-and-issues)
- [Summary](#summary)

## Demo

Here is the end product of the app, pretty much like any other Integromat app:
<!-- TODO: add video link -->

## Integromat

Integromat is an automation platform that

> ...lets you connect apps and automate workflows in a few clicks. Move data between apps without effort so you can focus on growing your business. [^1]

[^1]: https://integromat.com/

The platform has many integrations including AWS, Google Drive, Google Docs, Slack and lots of others.

The basic building blocks of the platform are modules.
These modules perform specific tasks like sending emails, fetching files from Google Drive, uploading files to AWS S3, etc.
You can combine these modules to create a scenario which can be triggered by a schedule or by a third party action, e.g. you recieved an email, or edited a spreadsheet.
This is very useful for automating certain workflows like sending an email to your accounting team whenever you change some numbers on a spreadsheet, or you want to send an email to yourself when NEAR Protocol configuration changes.

## NEAR

NEAR is a new layer 1 blockchain that solves some of the problems existing blockchains have. More specificaly

> NEAR is a [sharded](https://near.org/downloads/Nightshade.pdf), [proof-of-stake](https://en.wikipedia.org/wiki/Proof_of_stake), [layer-one blockchain](https://blockchain-comparison.com/blockchain-protocols) that is simple to use, secure and scalable.[^2]

[^2]: https://near.org/

Like [Polkadot](https://polkadot.network/), [Solana](https://solana.com/) and [Avalanche](https://www.avax.network/) NEAR tries to solve the three main problems of today's popular blockchains:

1. Cost - current blockchains are expensive to use and maintain
2. Low throughput - current blockchains are slow to process transactions and execute smart contracts
3. Hard to scale - current blockchains don't scale well to high traffic

If you want to learn more about NEAR, head over to their [website](https://near.org/), or directly to their [university](https://www.near.university/) to learn about the technology and how to use it.

## Integromat custom apps

Integromat has a large offering of integrations but you might want to use something that they do not offer.

This is where custom apps come into play.

You can create your own custom app to integrate with any API you want. In this case I built an app for NEAR which allows querying all public data on the NEAR blockchain, including calling functions that are free (e.g. do not require a transaction fee).

To start of with a custom app you need to create some of the following pieces of logic.

### Base

The base of the app represents the logic that is common to all other components, like the base URL, the common headers, response mapping, common body or query string parameters, etc.

The base of my app looks like this:

```json
{
    "baseUrl": "{{getNetwork(parameters)}}",
    "body": {
        "jsonrpc": "2.0",
        "id": "dontcare"
    },
    "response": {
        "output": "{{body.result}}",
        "valid": "{{isValid(body)}}",
        "error": {
            "message": "{{formatError(statusCode,body)}}"
        }
    }
}
```

These wierd looking double curly braces in json are IML (Integromat Markup Language) templateing specifiers. Within them we can call integrated and custom IML functions with parameters which are available in IML. I will cover the custom IML functions in one of the following [sections](#iml-functions).

For example the base url is a custom function called `getNetwork`, which as you guessed, returns the proper url based on the network the user selected. And this selection is available in the parameters object which I will go over a bit [later](#parameters).

Similarly the `isValid` and `formatError` functions are custom functions that are called when the response is returned from the API.

For a more in depth overview of the base IML template, see the [Integromat docs](https://docs.integromat.com/apps/app-structure/general/base)

### Modules

Modules are where the magic happens. They represent different functions that you can call on your API, like calling a view function on a NEAR smart contract:

```json
{
    "url": "/",
    "method": "POST",
    "body": {
        "method": "query",
        "params": {
            "request_type": "call_function",
            "account_id": "{{parameters.account_id}}",
            "method_name": "{{parameters.method_name}}",
            "args_base64": "{{if(length(parameters.args)=0;'';base64(createJSON(toCollection(parameters.args;'key';'value'))))}}",
            "{{...}}": "{{getFinalityOrBlockID(parameters)}}"
        }
    }
}
```

As you can see the format is pretty similar to the base configuration.
You can see that we are using the parameters object extensively to get the user input.

Also you might notice that we are using the `{{...}}` syntax to destructure the values we get from the `getFinalityOrBlockID` function. It works like the `...` syntax in JavaScript.

For more information about modules, see the [Integromat docs](https://docs.integromat.com/apps/app-structure/modules).

#### Parameters

Parameters represent the user input and we specify the name, label, type and shape of these parameters. Here is the parameters specification for the above module:

```json
[
    "rpc://accountId",
    "rpc://finalityOrBlockId",
    {
        "name": "method_name",
        "type": "text",
        "label": "Method Name",
        "required": true
    },
    {
        "name": "args",
        "type": "array",
        "label": "Arguments",
        "spec": [
            {
                "name": "key",
                "type": "text",
                "label": "Key"
            },
            {
                "name": "value",
                "type": "any",
                "label": "Value"
            }
        ]
    },
    "rpc://networkSelection"
]
```

As you can see the parameters specification is a list of objects where each object represents a parameter.

You might also notice that there are some strings in the array (in this case `rpc://accountId`, `rpc://finalityOrBlockId` and `rpc://networkSelection`). These strings are actually **R**emote **P**rocedure **C**alls (RPC) which can call your API while the user is filling out input fields to get some data. This data can be the different networks available, the shape of parameters we want to use for this function etc. I will go into further detail abou RPCs [later](#rpc).

If you want to learn more about the parameter specification, head over to the [Integromat docs](https://docs.integromat.com/apps/other/parameters)

### RPC

**R**emote **P**rocedure **C**alls, or RPCs for short, are a way for you to call your API while the user is filling out input fields to get some data.
We can customize the behaviour of these functions just like the communication templates for the modules.

In my case I have used these RPCs to get some parameter specifications like the account ID one:

```json
{
    "response": {
        "output": {
            "name": "account_id",
            "type": "text",
            "label": "Account ID",
            "multiline": false,
            "required": true,
            "validate": {
                "min": 2,
                "max": 64,
                "pattern": "^(([a-z\\d]+[\\-_])*[a-z\\d]+\\.)*([a-z\\d]+[\\-_])*[a-z\\d]+$"
            }
        }
    }
}
```

You can see here that the output of our RPC is a parameter specification.
We could have used it to say fetch the public keys of the account, and then offer a selection/dropdown of these keys to the user to choose from.

For further explanation of what RPCs can do, check out [Integromat docs](https://docs.integromat.com/apps/app-structure/rpcs).

### IML Functions

IML (Integromat Markup Language) functions are custom JavaScript functions that can be called from within the IML json templates. These functions are like normal JavaScript functions with ES6 (ES2015) syntax/features and also have the Buffer class available.
They are mostly used for formating or more complex branching logic.

In my case I have used them to make code more readable, or in some cases to make certain goals achievable at all. For example the `getFinalityOrBlockID` function is used to choose between the finality or block ID input based on the users choice, but also changing the name of the key to be used as the parameter.

```js
function getFinalityOrBlockID(parameters) {
  switch (parameters.param_type) {
    case "finality":
      return { finality: parameters.finality };
    case "height":
    case "hash":
      return { block_id: parameters.block_id };
    default:
      return { finality: "final" };
  }
}
```

If for example the user chooses to input the height of the block, the input is going to be validated as an unsined integer, and the name of the parameter is going to be `block_id`. If on the other hand the user chooses the finality parameter, than the name of the parameter is going to be `finality` while the value is going to be a choice of `final` or `optimistic`.
This would not be achievable through the included IML functions as there is no way to dynamicly change the name of the key.

Another benefit of using these custom functions is, as I mentioned above, code readability, like when formatting the error message:

```js
function formatError(statusCode, { error }) {
  const info = error.cause.info.error_message || JSON.stringify(error.cause.info);
  const type = error.name;
  const cause = error.cause.name;

  return `[${statusCode}] ${info} (error type: ${type}, error cause: ${cause})`;
}
```

This function is completely doable with the included IML functions, but that format would be far more verbose.
Here is a comparison of the two:

Regular IML functions:

```json
{
    ...
    "response": {
        ...
        "error": {
            "message": "[{{statusCode}}] {{ifempty(body.error.info.error_message;toString(createJSON(body.error.info)))}} (error type: {{body.error.name}}, error cause: {{body.error.cause.name}}"
        }
    }
}
```

Custom IML function:

```json
{
    ...
    "response": {
        ...
        "error": {
            "message": "{{formatError(statusCode,body)}}"
        }
    }
}
```

If you are interested about going in depth with IML functions, visit the [Integromat docs](https://docs.integromat.com/apps/app-structure/iml-functions).

## Integrating with NEAR RPC API

In order to work with the NEAR protocol you have some options:

- [NEAR CLI](https://docs.near.org/docs/tools/near-cli) - for a command line interface
- [near-api-js](https://docs.near.org/docs/develop/front-end/near-api-js) - for integrating with NEAR through JavaScript code
- [near-sdk-as](https://docs.near.org/docs/develop/contracts/as/intro) - for developing smart contracts in AssemblyScript
- [near-sdk-rs](https://docs.near.org/docs/develop/contracts/rust/intro) - for developing smart contracts in Rust
- [NEAR RPC API](https://docs.near.org/docs/api/rpc) - for interacting with NEAR from any other environment

When creating a custom app in Integromat, you don't have access to the NPM ecosystem, so you have to rely on REST or GraphQL calls to third party APIs to do the heavy lifting. Fortunately, ***there is nothing preventing you from performing calls to the RPC API***. You can just call the same endpoint like in GraphQL, and pass in parameters in the body, which will get serialized as JSON by Integromat, like you saw in the [module](#module) above.

What I did for this app is basically go through the RPC API [documentation](https://docs.near.org/docs/api/rpc) and create a module for each of the endpoints available.
The documentation is pretty clear, with examples and descriptions for each endpoint, as well as error explanations. And if you find yourself in a situation where you can't figure out how to handle a certain action/endpoint, don't worry I had my share of *deer in the headlights look*, and the best way to get out is by checking how things were handled in the [near-api-js](https://github.com/near/near-api-js/) library.

## Tips, tricks and issues

Here is a list of some of the tips I can tell you so you don't waste your time like I did.

### Use the **data structure generator**

If you are developing in Visual Studio Code, which you should be if developing apps for Integromat, you can save yourself a lot of time when creating parameter and interface specifications by using this feature.

You can see it in the top right corner of the editor when editing parameter or interface specifications. It allows you to copy over JSON, XML, Form Data or Query Strings to the text area and generates a parameter definition for you.
Ofcourse it's not perfect, it might not provide a proper label based on the parameter name, but it will save you a lot of typing.
So don't be like me and waste hours typing everything, and then finding out about it halfway through.

### Use RPCs, a lot

For any parameter that you seem to be using across multiple modules, create an RPC and hard-code the parameter specification as the response of the RPC.

It will save you some typing, and make it more readable, as well as make it easier to change everything in one place, instead of having to go through each file and copy paste the changes.

### Define the response structure in base

It is very common for an API to have a common structure for responses.

It is also common for you to define this logic once and don't think about it later.

The Integromat docs do not however mention that this is possible, so don't wait until you have 24 modules with the same error handling and response mapping until you look at a few more examples to find out that yes in fact you can put all that boilerplate in one place, e.g. the base template.

### The docs are fragmented

The Integromat docs are a bit fragmented and sometimes incomplete, or maybe just not up to date.

Most of the stuff you need is in the [App docs](https://docs.integromat.com/apps), but some of it is in the [help area](https://www.integromat.com/en/help/docs) on the main site. So make sure to keep both of them open to find as much as you can.

### There is a `toCollection` function

I couldn't find this in documentation anywhere except when the support team responded to a request of mine about custom IML functions, and if you hover over it when composing functions in the scenario editor.

But nevertheless, it exists an can be quite useful.

For example when creating a array of objects with key value pairs where the user can choose the name of the key as well as input the value, it is much friendlier to have the user create an array of these inputs and then create the object when passing it to the API.

The `toCollection` function is a great way to do that as it accepts an array of objects, the name of the key parameter and the name of the value parameter and maps it to an object with those key value pairs extracted from the array.

### Don't hope to create complex IML function

You cannot create any complex functions which would require third party NPM modules as there is no module importing, and if you try to package the code and copy paste the minified version of it into Integromat, you will most likely hit the size cap.

Try to offload that complexity to other services, or prefferably your API and use the functions as helpers and utilities for less advanced usage.

## Summary

In short, to build the NEAR app in Integromat you have to:

- Understand what the NEAR blockchain can provide, and how to interact with it
- Use the [docs](https://docs.near.org/docs/api/rpc) to understand how to interact with NEAR
- Stick to free, no fee transactions and calls to avoid third party services for access key management
- Build a base that all the modules/functionalities of the app have incommon, e.g. error handling, response mapping, base url, etc.
- Format errors in a way that users can react to them properly
- Utilize the power of RPCs, wether to make your code more concise and maintainable, or to make it easier for the users to input correct data
- Think of IML functions only as formatting/utility tools
- Use both the [app docs](https://docs.integromat.com/apps) and the [help area](https://www.integromat.com/en/help/docs) to find out more about Integromat
- Play with scenarios that utilize your modules to see the user experience
