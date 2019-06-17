# QlinkSolver_Chromium

This project  is not intended  for production purposes, but  as  a demonstration
for TPS developers  of  the  QlinkSolver's basic functionality. It consist  in a
TypeScript application bundled with Chromium and Node.js.

## Project structure

+ QlinkSolver_Chromium_(x86|x64) -------------------------------- Project Folder
  + chrome-win -------------------------------------------- Chromium Web Browser
  + ui ---------------------------------------------------------- User Interface
  + node_modules ------------------------------------------ Node.js dependencies
  + node.exe ------------------------------------------------ Node.js executable
  + app.ts ------------------------------- Application (Entry Point) source code
  + app.js ------------------------------------------- Application (Entry Point)
  + custom-utils.ts ------------------------------- Custom Utilities source code
  + custom-utils.js ------------------------------------------- Custom Utilities
  + package.json -------------------------------------- Node.js package settings
  + package-lock.json ---------------------------- Node.js package lock settings
  + tsconfig.json --------------------------------- TypeScript compiler settings
  + README.md ----------------------------------- (this one) Project Readme file

The User Interface consist in an html/javascript single page site generated from
an  Angular/Bootstrap project. The UI implements  the "Qlink v2 Public HTTP API"
which is the core of the QlinkSolver functionality.

The Application (Entry Point) implements  an (optional) Express server to manage
the UI locally, that way  the Qlink API  request/responses  are  intercepted and
handled by the Express server too. The remote UI (recomended for production) can
be accessed by removing all the server features and changing the application url
from http:/localhost:8080/client to http://api.qlinkgroup.com/client.

The Application resembles  the  QlinkSolver basic functionality  by managing the
Chromium Web Browser with the Puppeteer-core API

### Run QlinkSolver_Chromium

Open a Powershell or CMD terminal on the project folder. Then execute:
```bash
node app.js
```

### Development Environment

- Install Node.js
- Install TypeScript compiler (npm install typescript)
- Build project from sources (tsc app.ts)

For Development process, Visual Studio Code IDE is recomended.

## Qlink Public HTTP API

### Overview

The Qlink API is  a mechanism for operators  or other parties  to have access to
process of acquiring and solving captchas, along with retrieving various related
stats  and perform  some additional operations, like registering  an operator or
issuing a kick-out.

The Qlink APIv1 can  be requested  with  base url http://api.qlinkgroup.com/api.
That way the parameters  are passed unencrypted (not recomended for production).

The Qlink APIv2 can be requested with base url http://api.qlinkgroup.com/api/v2.
For  any  command  other  than  "get_server_public_key"  the  Server  looks  for
"payload" field  in the request. The payload is  a base64-encoded encrypted json
dict  of parameters  that correspond  to  the usual Qlink APIv1 ones. The Server
base64-decodes  that  field  content, decrypts  it, and  treats it's  content as
normal  request  fields  the  same  way  as  usual  Qlink APIv1. The payload  is
encrypted  with  the  RSA  method (without  padding).  The  Public  Key  can  be
requested with  the "get_server_public_key" command. The User Interface includes
a (customized) JSEncrypt library for encryption.

### API keys

API Keys  are  special pre-generated tokens  that  are published along with  the
client. For different  Operator origins  different keys  are  provided. Operator
cannot  be  logged  in,  if  he supplies  wrong, incomplete, missing or disabled
API key.

```js
API_KEY='hgq2nfbzm9earpunb1hzdlmq75wa51i5yvwko'
```

### CType

The ctype field, is a list of captchas type names separated by comma. This value
notify Server  that  for the current image request, the  operator  will like  to
receive  an image  that match some  of  the provided types. Set  a ctype will no
guarantee  that only  the explicit declared types  of images will arrive  in the
request response. The API client software should be prepared to receive any type
of supported images.

Captchas types names that can be used in ctype:
- normal
  This are the normal textual images. Normal type is inferred if omitted.

- recaptcha_image_group
  New Google CAPTCHAs, that  display  sets of boxes. This images  usually  don't
  define explicit the grid (row, columns), and server returned field grid should
  be used  to display  a  guide grid  and determine  what positions contains the
  captcha solutions. The accepted solutions  for this images are the position of
  selected boxes, surrounded by []. Like for example [1,5,9] or [3,15,16].

- recaptcha_coordinates
  New  Google  CAPTCHAs,  that  display  sets  of  boxes. This  captchas  accept
  solutions containing  an array of [x,y] coordinates in pixels, referencing the
  selected boxes. Like for example [[30,25],[120,30]].

- token_image_group
  New Google reCAPTCHA v2, that  display  a challenge  like  "Select all squares
  with  traffic lights". The solved challenge  solution contains  a Token string
  rendered by the Google reCAPTCHA v2 API. Checkbox and Invisible reCAPTCHAs are
  treated the same way.

To allow Operators solve any type of CAPTCHA set ctype as:
```js
ctype=recaptcha_coordinates,recaptcha_image_group,token_image_group
```

## Command list

HTTP API is stateless. Each command that requires client operator authentication
should provide necessary client credentials. Client operator login procedure  is
performed  at the start  of every command execution. No persistent sessions  are
created or tracked for each operator, and  every HTTP requests  is essentially a
complete command.

### Commands not requiring prior authorization

**ping**
HTTP URL postfix: /ping (as GET)
Parameters: None
Ping command, for debugging and testing connectivity.
Response:
{ "pong": true }

**get_client_version**
HTTP URL postfix: /version (as GET)
Paramethers: None
Returns current official qlink client version.
Response:
{ "update_before": DATE, "version": VERSION_NUMBER, "hash": HASH_OF_BINARY }

**get_server_time**
HTTP URL postfix: /get_server_time (as GET)
Paramethers: None
Returns server time in UTC format.
Response:
{ "time": CURRENT_TIME }

**get_server_public_key**
HTTP URL postfix: /get_server_public_key (as GET)
Paramethers: None
Returns server public key, so client could encrypt their commands.
Response:
{ "public_key": PUBLIC_KEY }

**login**
HTTP URL postfix: /login (as POST)
Parameters: username, password, api_key
Performs login operation: checks operator credentials and API key.
Response:
{ "operator": OPERATOR_ID, "is_banned"=IS_BANNED[, "banned_for"=BANNED_FOR,
  "banned_until"=BANNED_UNTIL] }

### Commands requiring prior autorization

**get_free_captcha**
HTTP URL postfix: /captcha  (as GET)
Parameters: username, password, api_key
Acquires unsolves captcha and returns captcha information for operator to solve.
Response:
{ "captcha": None }
or:
{ "captcha": captcha_id, "hash": captcha_hash,
  "content": base64_encoded_image_data, "is_premium": if_captcha_is_premium,
  "width": desired_display_width, "height": desired_display_height,
  "timeout": solving_timeout }

Note: For **token_image_group** the response json also includes those fields:
- "googlekey": Can be optained by base64-decoding the "content" field.
- "pageurl": Can be optained by base64-decoding the "content" field.
Those fields are used by the Solver  to generate a Google reCAPTCHA v2 challenge
template (as shown in the Application example).

**save_captcha**
HTTP URL postfix: /captcha/CAPTCHA_ID  (as POST)
Parameters: username, password, api_key, text. The captcha_id parameter is
  fetched from URL.
Saves captcha text to server (solves captcha).
Response:
{ "solved": is_solved }

**kick_out**
HTTP URL postfix: /captcha/CAPTCHA_ID/kick-out  (as POST)
Parameters: username, password, api_key. The captcha_id parameter is fetched
  from URL.
Registers kick-out for specified captcha id.
Response:
{ "kicked_out": is_kicked_out }

**get_notifications**
HTTP URL postfix: /notification (as GET)
Parameters: username, password, api_key, is_read (retrieve all recent
  notifications, or only unread ones).
Returns notifications for operator.
Response:
{ "notifications": [ (subject, body, added_at, is_from_admin), ...,] }

**payrate**
HTTP URL postfix: /payrate (as GET)
Parameters: username, password, api_key
Returns payment rate for each of the hours in a day for operator.
Response:
{ 0L: RATE_FOR_HOUR_0,
  1L: RATE_FOR_HOUR_1,
  ...
  23L: RATE_FOR_HOUR_23 }

Note: Undocumented  parameters  and features  may  been  disabled, deprecated or
accessibles only at qlinkgroup.com. Do not use them without prior autorization
or your Account/Application could be banned permanently.