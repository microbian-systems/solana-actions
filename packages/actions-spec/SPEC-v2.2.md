# Solana Actions and Blockchain Links (Blinks)

[Read the docs to get started](https://solana.com/docs/advanced/actions)

Solana Actions are specification-compliant APIs that return transactions on the
Solana blockchain to be previewed, signed, and sent across a number of various
contexts, including QR codes, buttons + widgets, and websites across the
internet. Actions make it simple for developers to integrate the things you can
do throughout the Solana ecosystem right into your environment, allowing you to
perform blockchain transactions without needing to navigate away to a different
app or webpage.

Blockchain links – or blinks – turn any Solana Action into a shareable,
metadata-rich link. Blinks allow Action-aware clients (browser extension
wallets, bots) to display additional capabilities for the user. On a website, a
blink might immediately trigger a transaction preview in a wallet without going
to a decentralized app; in Discord, a bot might expand the blink into an
interactive set of buttons. This pushes the ability to interact on-chain to any
web surface capable of displaying a URL.

## Simplified Type Definitions

The types and interfaces declared within this readme files are often the
simplified version of the types to aid in readability.

For better type safety and improved developer experience, the
`@solana/actions-spec` package contains more complex type definitions. You can
find the
[source code for them here](https://github.com/solana-developers/solana-actions/blob/main/packages/actions-spec/index.d.ts).

## Contributing

If you would like to propose an update the Solana Actions specification, please
publish a proposal on the official Solana forum under the Solana Request for
Comments (sRFC) section: https://forum.solana.com/c/srfc/6

## Specification

The Solana Actions specification consists of key sections that are part of a
request/response interaction flow:

- Solana Action [URL scheme](#url-scheme) providing an Action URL
- [OPTIONS response](#options-response) to an Action URL to pass CORS
  requirements
- [GET request](#get-request) to an Action URL
- [GET response](#get-response) from the server
- [POST request](#post-request) to an Action URL
- [POST response](#post-response) from the server

Each of these requests are made by the _Action client_ (e.g. wallet app, browser
extension, dApp, website, etc) to gather specific metadata for rich user
interfaces and to facilitate user input to the Actions API.

Each of the responses are crafted by an application (e.g. website, server
backend, etc) and returned to the _Action client_. Ultimately, providing a
signable transaction or message for a wallet to prompt the user to approve,
sign, and send to the blockchain.

### URL Scheme

A Solana Action URL describes an interactive request for a signable Solana
transaction or message using the `solana-action` protocol.

The request is interactive because the parameters in the URL are used by a
client to make a series of standardized HTTP requests to compose a signable
transaction or message for the user to sign with their wallet.

```text
solana-action:<link>
```

- A single `link` field is required as the pathname. The value must be a
  conditionally
  [URL-encoded](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/encodeURIComponent)
  absolute HTTPS URL.

- If the URL contains query parameters, it must be URL-encoded. URL-encoding the
  value prevents conflicting with any Actions protocol parameters, which may be
  added via the protocol specification.

- If the URL does not contain query parameters, it should not be URL-encoded.
  This produces a shorter URL and a less dense QR code.

In either case, clients must
[URL-decode](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/decodeURIComponent)
the value. This has no effect if the value isn't URL-encoded. If the decoded
value is not an absolute HTTPS URL, the wallet must reject it as **malformed**.

### OPTIONS response

In order to allow Cross-Origin Resource Sharing
([CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)) within Actions
clients (including blinks), all Action endpoints should respond to HTTP requests
for the `OPTIONS` method with valid headers that will allow clients to pass CORS
checks for all subsequent requests from their same origin domain.

An Actions client may perform
"[preflight](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#preflighted_requests)"
requests to the Action URL endpoint in order check if the subsequent GET request
to the Action URL will pass all CORS checks. These CORS preflight checks are
made using the `OPTIONS` HTTP method and should respond with all required HTTP
headers that will allow Action clients (like blinks) to properly make all
subsequent requests from their origin domain.

At a minimum, the required HTTP headers include:

- `Access-Control-Allow-Origin` with a value of `*`
  - this ensures all Action clients can safely pass CORS checks in order to make
    all required requests
- `Access-Control-Allow-Methods` with a value of `GET,POST,PUT,OPTIONS`
  - ensures all required HTTP request methods are supported for Actions
- `Access-Control-Allow-Headers` with a minimum value of
  `Content-Type, Authorization, Content-Encoding, Accept-Encoding`

For simplicity, developers should consider returning the same response and
headers to `OPTIONS` requests as their [`GET` response](#get-response).

> The `actions.json` file response must also return valid Cross-Origin headers
> for `GET` and `OPTIONS` requests, specifically the
> `Access-Control-Allow-Origin` header value of `*`.
>
> See [actions.json](#actionsjson) below for more details.

### GET Request

The Action client (e.g. wallet, browser extension, etc) should make an HTTP
`GET` JSON request to the Action's URL endpoint.

- The request should not identify the wallet or the user.
- The client should make the request with an
  [`Accept-Encoding` header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Encoding).
- The client should display the domain of the URL as the request is being made.

### GET Response

The Action's URL endpoint (e.g. application or server backend) should respond
with an HTTP `OK` JSON response (with a valid payload in the body) or an
appropriate HTTP error.

- The client must handle HTTP
  [client errors](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status#client_error_responses),
  [server errors](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status#server_error_responses),
  and
  [redirect responses](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status#redirection_messages).
- The endpoint should respond with a
  [`Content-Encoding` header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Encoding)
  for HTTP compression.
- The endpoint should respond with a
  [`Content-Type` header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Type)
  of `application/json`.

- The client should not cache the response except as instructed by
  [HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching#controlling_caching)
  response headers.
- The client should display the `title` and render the `icon` image to user.

#### GET Response Body

A `GET` response with an HTTP `OK` JSON response should include a body payload
that follows the interface specification:

```ts filename="ActionGetResponse"
export type ActionType = "action" | "completed";

export type ActionGetResponse = Action<"action">;

export interface Action<T extends ActionType = "action"> {
  /** type of Action to present to the user */
  type: T;
  /** image url that represents the source of the action request */
  icon: string;
  /** describes the source of the action request */
  title: string;
  /** brief summary of the action to be performed */
  description: string;
  /** button text rendered to the user */
  label: string;
  /** UI state for the button being rendered to the user */
  disabled?: boolean;
  links?: {
    /** list of related Actions a user could perform */
    actions: LinkedAction[];
  };
  /** non-fatal error message to be displayed to the user */
  error?: ActionError;
}
```

- `type` - The type of action being given to the user. Defaults to `action`. The
  initial `ActionGetResponse` is required to have a type of `action`.

  - `action` - Standard action that will allow the user to interact with any of
    the `LinkedActions`
  - `completed` - Used to declare the "completed" state within action chaining.
    After the

- `icon` - The value must be an absolute HTTP or HTTPS URL of an icon image. The
  file must be an SVG, PNG, or WebP image, or the client/wallet must reject it
  as **malformed**.

- `title` - The value must be a UTF-8 string that represents the source of the
  action request. For example, this might be the name of a brand, store,
  application, or person making the request.

- `description` - The value must be a UTF-8 string that provides information on
  the action. The description should be displayed to the user.

- `label` - The value must be a UTF-8 string that will be rendered on a button
  for the user to click. All labels should not exceed 5 word phrases and should
  start with a verb to solidify the action you want the user to take. For
  example, "Mint NFT", "Vote Yes", or "Stake 1 SOL".

- `disabled` - The value must be boolean to represent the disabled state of the
  rendered button (which displays the `label` string). If no value is provided,
  `disabled` should default to `false` (i.e. enabled by default). For example,
  if the action endpoint is for a governance vote that has closed, set
  `disabled=true` and the `label` could be "Vote Closed".

- `error` - An optional error indication for non-fatal errors. If present, the
  client should display it to the user. If set, it should not prevent the client
  from interpreting the action or displaying it to the user. For example, the
  error can be used together with `disabled` to display a reason like business
  constraints, authorization, the state, or an error of external resource.

```ts filename="ActionError"
export interface ActionError {
  /** simple error message to be displayed to the user */
  message: string;
}
```

- `links.actions` - An optional array of related actions for the endpoint. Users
  should be displayed UI for each of the listed actions and expected to only
  perform one. For example, a governance vote action endpoint may return three
  options for the user: "Vote Yes", "Vote No", and "Abstain from Vote".

  - If no `links.actions` is provided, the client should render a single button
    using the root `label` string and make the POST request to the same action
    URL endpoint as the initial GET request.

  - If any `links.actions` are provided, the client should only render buttons
    and input fields based on the items listed in the `links.actions` field. The
    client should not render a button for the contents of the root `label`.

```ts filename="LinkedAction"
export interface LinkedAction {
  /** URL endpoint for an action */
  href: string;
  /** button text rendered to the user */
  label: string;
  /**
   * Parameters to accept user input within an action
   * @see {ActionParameter}
   * @see {ActionParameterSelectable}
   */
  parameters?: Array<TypedActionParameter>;
}
```

The `ActionParameter` allows declaring what input the Action API is requesting
from the user:

```ts filename="ActionParameter"
/**
 * Parameter to accept user input within an action
 * note: for ease of reading, this is a simplified type of the actual
 */
export interface ActionParameter {
  /** input field type */
  type?: ActionParameterType;
  /** parameter name in url */
  name: string;
  /** placeholder text for the user input field */
  label?: string;
  /** declare if this field is required (defaults to `false`) */
  required?: boolean;
  /** regular expression pattern to validate user input client side */
  pattern?: string;
  /** human-readable description of the `type` and/or `pattern`, represents a caption and error, if value doesn't match */
  patternDescription?: string;
  /** the minimum value allowed based on the `type` */
  min?: string | number;
  /** the maximum value allowed based on the `type` */
  max?: string | number;
}
```

The `pattern` should be a string equivalent of a valid regular expression. This
regular expression pattern should by used by blink-clients to validate user
input before before making the POST request. If the `pattern` is not a valid
regular expression, it should be ignored by clients.

The `patternDescription` is a human readable description of the expected input
requests from the user. If `pattern` is provided, the `patternDescription` is
required to be provided.

The `min` and `max` values allows the input to set a lower and/or upper bounds
of the input requested from the user (i.e. min/max number and or min/max
character length), and should be used for client side validation. For input
`type`s of `date` or `datetime-local`, these values should be a string dates.
For other string based input `type`s, the values should be numbers representing
their min/max character length.

If the user input value is not considered valid per the `pattern`, the user
should receive a client side error message indicating the input field is not
valid and displayed the `patternDescription` string.

The `type` field allows the Action API to declare more specific user input
fields, providing better client side validation and improving the user
experience. In many cases, this type will resemble the standard
[HTML input element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input).

The `ActionParameterType` can be simplified to the following type:

```ts filename="ActionParameterType"
/**
 * Input field type to present to the user
 * @default `text`
 */
export type ActionParameterType =
  | "text"
  | "email"
  | "url"
  | "number"
  | "date"
  | "datetime-local"
  | "checkbox"
  | "radio"
  | "textarea"
  | "select";
```

Each of the `type` values should normally result in a user input field that
resembles a standard HTML `input` element of the corresponding `type` (i.e.
`<input type="email" />`) to provide better client side validation and user
experience:

- `text` - equivalent of HTML
  [“text” input](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/text)
  element
- `email` - equivalent of HTML
  [“email” input](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/email)
  element
- `url` - equivalent of HTML
  [“url” input](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/url)
  element
- `number` - equivalent of HTML
  [“number” input](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/number)
  element
- `date` - equivalent of HTML
  [“date” input](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/date)
  element
- `datetime-local` - equivalent of HTML
  [“datetime-local” input](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/datetime-local)
  element
- `checkbox` - equivalent to a grouping of standard HTML
  [“checkbox” input](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/checkbox)
  elements. The Action API should return `options` as detailed below. The user
  should be able to select multiple of the provided checkbox options.
- `radio` - equivalent to a grouping of standard HTML
  [“radio” input](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/radio)
  elements. The Action API should return `options` as detailed below. The user
  should be able to select only one of the provided radio options.
- Other HTML input type equivalents not specified above (`hidden`, `button`,
  `submit`, `file`, etc) are not supported at this time.

In addition to the elements resembling HTML input types above, the following
user input elements are also supported:

- `textarea` - equivalent of HTML
  [textarea element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/textarea).
  Allowing the user provide multi-line input.
- `select` - equivalent of HTML
  [select element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/select),
  allowing the user to experience a “dropdown” style field. The Action API
  should return `options` as detailed below.

When `type` is set as `select`, `checkbox`, or `radio` then the Action API
should include an array of `options` that each provide a `label` and `value` at
a minimum. Each option may also have a `selected` value to inform the
blink-client which of the options should be selected by default for the user
(see `checkbox` and `radio` for differences).

This `ActionParameterSelectable` can be simplified to the following type
definition:

```ts filename="ActionParameterSelectable"
/**
 * note: for ease of reading, this is a simplified type of the actual
 */
interface ActionParameterSelectable extends ActionParameter {
  options: Array<{
    /** displayed UI label of this selectable option */
    label: string;
    /** value of this selectable option */
    value: string;
    /** whether or not this option should be selected by default */
    selected?: boolean;
  }>;
}
```

If no `type` is set or an unknown/unsupported value is set, blink-client should
default to `text` and render a simple text input.

The Action API is still responsible to validate and sanitize all data from the
user input parameters, enforcing any “required” user input as necessary.

For platforms other that HTML/web based ones (like native mobile), the
equivalent native user input component should be used to achieve the equivalent
experience and client side validation as the HTML/web input types described
above.

### POST Request

The client must make an HTTP `POST` JSON request to the action URL with a body
payload of:

```json
{
  "account": "<account>"
}
```

- `account` - The value must be the base58-encoded public key of an account that
  may sign the transaction.

The client should make the request with an
[Accept-Encoding header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Encoding)
and the application may respond with a
[Content-Encoding header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Encoding)
for HTTP compression.

The client should display the domain of the action URL as the request is being
made. If a `GET` request was made, the client should also display the `title`
and render the `icon` image from that GET response.

### POST Response

The Action's `POST` endpoint should respond with an HTTP `OK` JSON response
(with a valid payload in the body) or an appropriate HTTP error.

- The client must handle HTTP
  [client errors](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status#client_error_responses),
  [server errors](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status#server_error_responses),
  and
  [redirect responses](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status#redirection_messages).
- The endpoint should respond with a
  [`Content-Type` header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Type)
  of `application/json`.

#### POST Response Body

A `POST` response with an HTTP `OK` JSON response should include a body payload
of:

```ts filename="ActionPostResponse"
/**
 * Response body payload returned from the Action POST Request
 */
export interface ActionPostResponse<T extends ActionType = ActionType> {
  /** base64 encoded serialized transaction */
  transaction: string;
  /** describes the nature of the transaction */
  message?: string;
  links?: {
    /**
     * The next action in a successive chain of actions to be obtained after
     * the previous was successful.
     */
    next: NextActionLink;
  };
}
```

- `transaction` - The value must be a base64-encoded
  [serialized transaction](https://solana-labs.github.io/solana-web3.js/v1.x/classes/Transaction.html#serialize).
  The client must base64-decode the transaction and
  [deserialize it](https://solana-labs.github.io/solana-web3.js/v1.x/classes/Transaction.html#from).

- `message` - The value must be a UTF-8 string that describes the nature of the
  transaction included in the response. The client should display this value to
  the user. For example, this might be the name of an item being purchased, a
  discount applied to a purchase, or a thank you note.

- `links.next` - An optional value use to "chain" multiple Actions together in
  series. After the included `transaction` has been confirmed on-chain, the
  client can fetch and render the next action. See
  [Action Chaining](#action-chaining) for more details.

- The client and application should allow additional fields in the request body
  and response body, which may be added by future specification updates.

> The application may respond with a partially or fully signed transaction. The
> client and wallet must validate the transaction as **untrusted**.

#### POST Response - Transaction

If the transaction
[`signatures`](https://solana-labs.github.io/solana-web3.js/v1.x/classes/Transaction.html#signatures)
are empty or the transaction has NOT been partially signed:

- The client must ignore the
  [`feePayer`](https://solana-labs.github.io/solana-web3.js/v1.x/classes/Transaction.html#feePayer)
  in the transaction and set the `feePayer` to the `account` in the request.
- The client must ignore the
  [`recentBlockhash`](https://solana-labs.github.io/solana-web3.js/v1.x/classes/Transaction.html#recentBlockhash)
  in the transaction and set the `recentBlockhash` to the
  [latest blockhash](https://solana-labs.github.io/solana-web3.js/v1.x/classes/Connection.html#getLatestBlockhash).
- The client must serialize and deserialize the transaction before signing it.
  This ensures consistent ordering of the account keys, as a workaround for
  [this issue](https://github.com/solana-labs/solana/issues/21722).

If the transaction has been partially signed:

- The client must NOT alter the
  [`feePayer`](https://solana-labs.github.io/solana-web3.js/v1.x/classes/Transaction.html#feePayer)
  or
  [`recentBlockhash`](https://solana-labs.github.io/solana-web3.js/v1.x/classes/Transaction.html#recentBlockhash)
  as this would invalidate any existing signatures.
- The client must verify existing signatures, and if any are invalid, the client
  must reject the transaction as **malformed**.

The client must only sign the transaction with the `account` in the request, and
must do so only if a signature for the `account` in the request is expected.

If any signature except a signature for the `account` in the request is
expected, the client must reject the transaction as **malicious**.

#### Action Chaining

Solana Actions can be "chained" together in a successive series. After an
Action's transaction is confirmed on-chain, the next action can be obtained and
presented to the user.

Action chaining allows developers to build more complex and dynamic experiences
within blinks, including:

- providing multiple transactions (and eventually sign message) to a user
- customized action metadata based on the user's wallet address
- refreshing the blink metadata after a successful transaction
- receive an API callback with the transaction signature for additional
  validation and logic on the Action API server
- customized "success" messages by updating the displayed metadata (e.g. a new
  image and description)

To chain multiple actions together, in any `ActionPostResponse` include a
`links.next` of either:

- `PostNextActionLink` - POST request link with a same origin callback url to
  receive the `signature` and user's `account` in the body. This callback url
  should respond with a `NextAction`.
- `InlineNextActionLink` - Inline metadata for the next action to be presented
  to the user immediately after the transaction has confirmed. No callback will
  be made.

```ts filename="NextActionLink"
export type NextActionLink = PostNextActionLink | InlineNextActionLink;

/** @see {NextActionPostRequest} */
export interface PostNextActionLink {
  /** Indicates the type of the link. */
  type: "post";
  /** Relative or same origin URL to which the POST request should be made. */
  href: string;
}

/**
 * Represents an inline next action embedded within the current context.
 */
export interface InlineNextActionLink {
  /** Indicates the type of the link. */
  type: "inline";
  /** The next action to be performed */
  action: NextAction;
}
```

##### NextAction

After the `ActionPostResponse` included `transaction` is signed by the user and
confirmed on-chain, the blink client should either:

- execute the callback request to fetch and display the `NextAction`, or
- if a `NextAction` is already provided via `links.next`, the blink client
  should update the displayed metadata and make no callback request

If the callback url is not the same origin as the initial POST request, no
callback request should be made. Blink clients should display an error notifying
the user.

```ts filename="NextAction"
/** The next action to be performed */
export type NextAction = Action<"action"> | CompletedAction;

/** The completed action, used to declare the "completed" state within action chaining. */
export type CompletedAction = Omit<Action<"completed">, "links">;
```

Based on the `type`, the next action should be presented to the user via blink
clients in one of the following ways:

- `action` - (default) A standard action that will allow the user to see the
  included Action metadata, interact with the provided `LinkedActions`, and
  continue to chain any following actions.

- `completed` - The terminal state of an action chain that can update the blink
  UI with the included Action metadata, but will not allow the user to execute
  further actions.

If no `links.next` is not provided, blink clients should assume the current
action is final action in the chain, presenting their "completed" UI state after
the transaction is confirmed.

### actions.json

The purpose of the [`actions.json` file](#actionsjson) allows an application to
instruct clients on what website URLs support Solana Actions and provide a
mapping that can be used to perform [GET requests](#get-request) to an Actions
API server.

> The `actions.json` file response must also return valid Cross-Origin headers
> for `GET` and `OPTIONS` requests, specifically the
> `Access-Control-Allow-Origin` header value of `*`.
>
> See [OPTIONS response](#options-response) above for more details.

The `actions.json` file should be stored and universally accessible at the root
of the domain.

For example, if your web application is deployed to `my-site.com` then the
`actions.json` file should be accessible at `https://my-site.com/actions.json`.
This file should also be Cross-Origin accessible via any browser by having a
`Access-Control-Allow-Origin` header value of `*`.

### Rules

The `rules` field allows the application to map a set of a website's relative
route paths to a set of other paths.

**Type:** `Array` of `ActionRuleObject`.

```ts filename="ActionRuleObject"
interface ActionRuleObject {
  /** relative (preferred) or absolute path to perform the rule mapping from */
  pathPattern: string;
  /** relative (preferred) or absolute path that supports Action requests */
  apiPath: string;
}
```

- [`pathPattern`](#rules-pathpattern) - A pattern that matches each incoming
  pathname.

- [`apiPath`](#rules-apipath) - A location destination defined as an absolute
  pathname or external URL.

#### Rules - pathPattern

A pattern that matches each incoming pathname. It can be an absolute or relative
path and supports the following formats:

- **Exact Match**: Matches the exact URL path.

  - Example: `/exact-path`
  - Example: `https://website.com/exact-path`

- **Wildcard Match**: Uses wildcards to match any sequence of characters in the
  URL path. This can match single (using `*`) or multiple segments (using `**`).
  (see [Path Matching](#rules-path-matching) below).

  - Example: `/trade/*` will match `/trade/123` and `/trade/abc`, capturing only
    the first segment after `/trade/`.
  - Example: `/category/*/item/**` will match `/category/123/item/456` and
    `/category/abc/item/def`.
  - Example: `/api/actions/trade/*/confirm` will match
    `/api/actions/trade/123/confirm`.

#### Rules - apiPath

The destination path for the action request. It can be defined as an absolute
pathname or an external URL.

- Example: `/api/exact-path`
- Example: `https://api.example.com/v1/donate/*`
- Example: `/api/category/*/item/*`
- Example: `/api/swap/**`

#### Rules - Query Parameters

Query parameters from the original URL are always preserved and appended to the
mapped URL.

#### Rules - Path Matching

The following table outlines the syntax for path matching patterns:

| Operator | Matches                                                                                                                                                                                  |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `*`      | A single path segment, not including the surrounding path separator / characters.                                                                                                        |
| `**`     | Matches zero or more characters, including any path separator / characters between multiple path segments. If other operators are included, the `**` operator must be the last operator. |
| `?`      | Unsupported pattern.                                                                                                                                                                     |

## License

The Solana Actions JavaScript SDK is open source and available under the Apache
License, Version 2.0. See the [LICENSE](./LICENSE) file for more info.
