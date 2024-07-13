<div align="center">
  <a href="#">
    <img src="https://github.com/user-attachments/assets/cd203167-3d1d-4114-981f-014f1044142c" alt="xtra-logger logo">
  </a>

  <h1 align="center">XTRA-LOGGER</h1>

  <p align="center">
    <h3>A lightweight logger, console text coloriser and text formatter with highest formatting options written in <a href="https://www.typescriptlang.org/docs/handbook/intro.html">Typescript </a> </h3>
  </p>
</div>

<br>

## Compatibility:

| Version | Compatibility |
|----------|----------|
| Es5 Javascript    | <div align="center">✅</div>   | 
| Es6 Javascript    | <div align="center">✅</div>   | 
| Typescript    | <div align="center">✅</div>   |

<br>

## Features:

| Feature | Availability |
|----------|----------|
| Logging    | <div align="center">✅</div>   | 
| Text Formatting    | <div align="center">✅</div>   | 
| Console Colorize    | <div align="center">✅</div>   |
| Combined Formatting    | <div align="center">✅</div>   |
| Size    | <div align="center">17 kb</div>   | 

<br>

## Installation:
Node Package Manager (NPM)
```bash
npm i xtra-logger
```

Yet Another Resource Navigator (YARN)

```bash
yarn add xtra-logger
```

<br>

## Importing:

For Es5 Js

```js
const { logger, format } = require("xtra-logger");
```

For Es6 Js / Typescript

```js
import { logger, format } from "xtra-logger";
```

<br>

## Usage Examples:

***Logging features:***
- 5 Log methods available
- Changable timezome for Docker / Remote servers
- Log format : `[ DD.MM.YY - HH:MM:SS ] - [ TYPE ] - MESSAGE`

```js
import { logger, format } from "xtra-logger";

// Changing the timezone of logger [ optional ] (By default it will use system time)
logger.timeZone("Asia/Kolkata");

// Logging options
logger.info("Starting the server...");
logger.warn("Using node v:18.6");
logger.success("Server started successfully.");
logger.debug("NPM update available.");
logger.error("An error occurd in the server.");
```

*Output:*

![image](https://github.com/user-attachments/assets/68f7fd75-ff9f-454a-8f6f-851add11255c)

<br>

***Text formatting features:***
- 50+ text formatting / colorizing options available.
- Multiple formatting can be combined together to create new style.
