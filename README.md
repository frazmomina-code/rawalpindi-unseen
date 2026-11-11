<!DOCTYPE html>
<html lang="en">
<head><style>
@font-face {
  font-family: "Optimistic";
  font-style: normal;
  font-weight: 400 600;
  font-display: swap;
  src: url("/fonts/OptimisticAI_VF_Optimized.woff2") format("woff2");
}
@font-face {
  font-family: "Optimistic Mono";
  font-style: normal;
  font-weight: 400;
  font-display: swap;
  src: url("/fonts/OptimisticMono_W_TextRegular.woff2") format("woff2");
}
:where(html) {
  font-family: "Optimistic", system-ui, sans-serif;
}
:where(code, pre, kbd, samp) {
  font-family: "Optimistic Mono", ui-monospace, monospace;
}
</style><style data-metaai-safe-area>:root { scroll-padding-top: var(--safe-area-scroll-padding-top, 0px); }
[data-safe-area-inset-root]::before {
  background: inherit;
  content: "";
  height: var(--safe-area-inset-top, 0px);
  inset: 0 0 auto;
  pointer-events: none;
  position: fixed;
  z-index: 2147483647;
}
</style><script data-metaai-safe-area-runtime>// (c) Meta Platforms, Inc. and affiliates. Confidential and proprietary.
(() => {
  const params = new URLSearchParams(window.location.search);
  const rawInset = params.get('top_inset_px');
  const parsedInset = rawInset !== null && /^\d+$/.test(rawInset)
    ? Number(rawInset)
    : 0;
  const topInset = params.get('has_top_banner') === '1'
    ? Math.min(parsedInset, 256)
    : 0;
  document.documentElement.style.setProperty('--safe-area-inset-top', `${topInset}px`);
  document.documentElement.style.setProperty('--safe-area-scroll-padding-top', `${topInset}px`);
  function observeNavigation() {
    const root = document.getElementById('root');
    if (root === null) return;
    const observed = new Set();
    let scheduled = false;
    function syncScrollPadding() {
      const elements = new Set(root.querySelectorAll('[data-safe-area-sticky-top]'));
      for (const element of observed) {
        if (!elements.has(element)) {
          resizeObserver.unobserve(element);
          observed.delete(element);
        }
      }
      for (const element of elements) {
        if (!observed.has(element)) {
          resizeObserver.observe(element);
          observed.add(element);
        }
      }
      let padding = topInset;
      for (const element of elements) {
        const style = window.getComputedStyle(element);
        if (style.position !== 'sticky' && style.position !== 'fixed') continue;
        const top = Number.parseFloat(style.top);
        if (Number.isFinite(top)) {
          padding = Math.max(padding, top + element.getBoundingClientRect().height);
        }
      }
      document.documentElement.style.setProperty(
        '--safe-area-scroll-padding-top', `${Math.ceil(padding)}px`,
      );
    }
    function scheduleUpdate() {
      if (scheduled) return;
      scheduled = true;
      queueMicrotask(() => {
        scheduled = false;
        syncScrollPadding();
      });
    }
    const resizeObserver = new ResizeObserver(scheduleUpdate);
    const mutationObserver = new MutationObserver(scheduleUpdate);
    mutationObserver.observe(root, {
      attributeFilter: ['class', 'data-safe-area-sticky-top', 'hidden', 'style'],
      attributes: true,
      childList: true,
      subtree: true,
    });
    window.addEventListener('resize', scheduleUpdate);
    scheduleUpdate();
  }
  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', observeNavigation, {once: true});
  } else {
    observeNavigation();
  }
})();
</script><script data-metaai-records-guard>// (c) Meta Platforms, Inc. and affiliates. Confidential and proprietary.
// When the records API rejects a write because the app is at its record
// limit, this page answers the next writes itself with the same rejection.
// The answers come after a delay, under an exponential backoff, and one probe
// write goes to the network per backoff window. A new record that the API
// accepts, or a delete that removes a record, ends the backoff. The backoff
// also ends when a write comes 60 s or more after the last window, and no probe
// from the last 60 s is in flight. The state is in memory only.
//
// The compiled-HTML contract checks (`stateful_crud_contract.rs` and
// `stateful_hosted_release.rs`) read this file as part of the app. So it must
// not contain a quoted records action, a browser storage API, the JSON request
// header or encoder, a quoted POST method, a collection argument, or a name
// bound to a quoted word. It must also not call a network function with a
// quoted URL.
(() => {
  const nativeFetch = window.fetch;
  if (typeof nativeFetch !== 'function') return;
  const NativeRequest = window.Request;
  const NativeResponse = window.Response;
  const NativeURL = window.URL;
  const NativePromise = window.Promise;
  const parseJson = JSON.parse;
  const reflectApply = Reflect.apply;
  const startTimer = window.setTimeout.bind(window);
  const stopTimer = window.clearTimeout.bind(window);
  const monotonicNow = window.performance.now.bind(window.performance);
  const CAPACITY_STATUS = 409;
  const CAPACITY_BODY = '{"error":"artifact record limit reached"}';
  const FIRST_WINDOW_MS = 1000;
  const MAX_WINDOW_MS = 60000;
  const LOCAL_ANSWER_DELAY_MS = 500;
  const MAX_TRACKED_RECORDS = 2048;
  const actionName = operation => ['records', operation].join('.');
  const CREATE = actionName('create');
  const PUT = actionName('put');
  const GET = actionName('get');
  const LIST = actionName('list');
  const DELETE = actionName('delete');
  const OBSERVED_ACTIONS = [CREATE, PUT, GET, LIST, DELETE];
  const POST_METHOD = ['PO', 'ST'].join('');
  const LOCAL_HEADERS = [
    [['content', 'type'].join('-'), ['application', 'json'].join('/') + '; charset=utf-8'],
    ['cache-control', 'no-store'],
    ['x-metaai-records-guard', 'capacity'],
  ];
  // `epoch` changes when a window opens or closes. A rejected or accepted write
  // that was sent in an earlier epoch does not change the window, so the writes
  // that were in flight when the limit was reached count as one rejection.
  let streak = 0;
  let windowEndsAt = 0;
  let epoch = 0;
  let activeProbe = null;
  const knownIds = new Set();
  const knownKeys = new Map();
  function recordKey(collection, value) {
    return `${collection}\n${value}`;
  }
  function rememberId(collection, id) {
    if (typeof id !== 'string') return;
    const key = recordKey(collection, id);
    knownIds.delete(key);
    knownIds.add(key);
    if (knownIds.size > MAX_TRACKED_RECORDS) knownIds.delete(knownIds.values().next().value);
  }
  // `savedId` is null when the outcome of the write is unknown. Then the key
  // keeps the id that an earlier answer saved.
  function rememberIdempotencyKey(collection, idempotencyKey, savedId) {
    if (typeof idempotencyKey !== 'string') return;
    const key = recordKey(collection, idempotencyKey);
    const id = savedId ?? knownKeys.get(key) ?? null;
    knownKeys.delete(key);
    knownKeys.set(key, id);
    if (knownKeys.size > MAX_TRACKED_RECORDS) knownKeys.delete(knownKeys.keys().next().value);
  }
  function forgetRecord(savedId) {
    knownIds.delete(savedId);
    for (const [idempotencyKey, keySavedId] of knownKeys) {
      if (keySavedId === savedId) knownKeys.delete(idempotencyKey);
    }
  }
  function forgetId(collection, id) {
    if (typeof id === 'string') forgetRecord(recordKey(collection, id));
  }
  function targetsActions(input, init) {
    const isRequest = input instanceof NativeRequest;
    const method = init?.method ?? (isRequest ? input.method : 'GET');
    if (String(method).toUpperCase() !== POST_METHOD) return false;
    const base = window.location.href;
    const url = isRequest ? input.url : String(input);
    return new NativeURL(url, base).href === new NativeURL('actions', base).href;
  }
  // Returns the body text, a promise of it for a Request body, or null when
  // the body cannot be JSON text.
  function requestBody(input, init) {
    const body = init?.body;
    if (body !== undefined && body !== null) return typeof body === 'string' ? body : null;
    return input instanceof NativeRequest ? input.clone().text() : null;
  }
  function requestSignal(input, init) {
    if (init !== undefined && init !== null && init.signal !== undefined) return init.signal;
    return input instanceof NativeRequest ? input.signal : null;
  }
  function parseAction(text) {
    let payload;
    try {
      payload = parseJson(text);
    } catch {
      return null;
    }
    if (payload === null || typeof payload !== 'object' || Array.isArray(payload)) return null;
    const name = payload.action;
    if (!OBSERVED_ACTIONS.includes(name)) return null;
    const args = payload.args ?? {};
    if (typeof args !== 'object' || Array.isArray(args)) return null;
    const collection = args.collection ?? 'items';
    if (typeof collection !== 'string') return null;
    return {name, args, collection};
  }
  function isWrite(action) {
    return action.name === CREATE || action.name === PUT;
  }
  // The records API accepts these writes at the limit when their record
  // exists: a create that replays a known idempotency key, and a put to a known
  // record. A key or id is known when this page saw its record saved, or sent
  // it in a write whose outcome is unknown.
  function mayStillSucceed(action) {
    if (action.name === CREATE) {
      const idempotencyKey = action.args.idempotency_key;
      return typeof idempotencyKey === 'string'
        && knownKeys.has(recordKey(action.collection, idempotencyKey));
    }
    const id = action.args.id;
    return typeof id === 'string' && knownIds.has(recordKey(action.collection, id));
  }
  function rememberUnknownOutcome(action) {
    if (action.name === CREATE) {
      rememberIdempotencyKey(action.collection, action.args.idempotency_key, null);
    } else {
      rememberId(action.collection, action.args.id);
    }
  }
  // The API rejects a known key or id for capacity only when its record does
  // not exist.
  function revokeExemption(action) {
    if (action.name === PUT) {
      forgetId(action.collection, action.args.id);
      return;
    }
    const key = recordKey(action.collection, action.args.idempotency_key);
    const savedId = knownKeys.get(key);
    knownKeys.delete(key);
    if (typeof savedId === 'string') forgetRecord(savedId);
  }
  // A probe that never settles stops holding writes back after the longest
  // window.
  function windowIsOpen(time) {
    if (time < windowEndsAt) return true;
    return activeProbe !== null && time - activeProbe.startedAt < MAX_WINDOW_MS;
  }
  function openWindow(sentEpoch) {
    if (sentEpoch !== epoch) return;
    streak += 1;
    windowEndsAt = monotonicNow() + Math.min(MAX_WINDOW_MS, FIRST_WINDOW_MS * 2 ** (streak - 1));
    epoch += 1;
  }
  function closeWindow() {
    if (streak === 0) return;
    streak = 0;
    windowEndsAt = 0;
    activeProbe = null;
    epoch += 1;
  }
  // The delay keeps an awaited retry loop from running on every microtask.
  function answerLocally(signal) {
    return new NativePromise((resolve, reject) => {
      if (signal?.aborted) {
        reject(signal.reason);
        return;
      }
      let timer = 0;
      const abort = () => {
        stopTimer(timer);
        reject(signal.reason);
      };
      timer = startTimer(() => {
        signal?.removeEventListener('abort', abort);
        resolve(new NativeResponse(CAPACITY_BODY, {status: CAPACITY_STATUS, headers: LOCAL_HEADERS}));
      }, LOCAL_ANSWER_DELAY_MS);
      signal?.addEventListener('abort', abort, {once: true});
    });
  }
  function responseData(text) {
    try {
      const payload = parseJson(text);
      return payload !== null && typeof payload === 'object' ? payload.data : undefined;
    } catch {
      return undefined;
    }
  }
  function track(action, data) {
    const collection = action.collection;
    switch (action.name) {
      case LIST:
        if (Array.isArray(data?.records)) {
          for (const record of data.records) rememberId(collection, record?.id);
        }
        return;
      case DELETE:
        if (data === true) {
          forgetId(collection, action.args.id);
          closeWindow();
        }
        return;
      case CREATE:
        if (typeof data?.id === 'string') {
          rememberIdempotencyKey(collection, action.args.idempotency_key, recordKey(collection, data.id));
        }
        rememberId(collection, data?.id);
        return;
      default:
        rememberId(collection, data?.id);
    }
  }
  async function observe(action, response, sent) {
    try {
      const status = response.status;
      const succeeded = status >= 200 && status < 300;
      // A write that the API accepts only below the limit proves free space.
      if (succeeded && isWrite(action) && !sent.mayStillSucceed && sent.epoch === epoch) {
        closeWindow();
      }
      const capacityCandidate = status === CAPACITY_STATUS && isWrite(action);
      if (!succeeded && !capacityCandidate) return;
      const text = await response.clone().text();
      if (succeeded) {
        track(action, responseData(text));
      } else if (text === CAPACITY_BODY) {
        if (sent.mayStillSucceed) revokeExemption(action);
        openWindow(sent.epoch);
      }
    } finally {
      if (sent.probe !== null && activeProbe === sent.probe) activeProbe = null;
    }
  }
  function handle(action, send, signal) {
    if (action === null) return send();
    const sent = {epoch, mayStillSucceed: false, probe: null};
    if (isWrite(action)) {
      sent.mayStillSucceed = mayStillSucceed(action);
      if (streak > 0 && !sent.mayStillSucceed) {
        const time = monotonicNow();
        if (windowIsOpen(time)) return answerLocally(signal);
        if (time - windowEndsAt < MAX_WINDOW_MS) {
          activeProbe = {startedAt: time};
          sent.probe = activeProbe;
        } else {
          // Another client or the agent can free space where this page cannot
          // see it. So after a quiet period the write goes out as a normal write.
          closeWindow();
          sent.epoch = epoch;
        }
      }
    }
    return send().then(
      response => observe(action, response, sent).then(() => response, () => response),
      error => {
        if (sent.probe !== null && activeProbe === sent.probe) activeProbe = null;
        // A write that fails or is aborted after it is sent can still save its
        // record.
        if (isWrite(action)) rememberUnknownOutcome(action);
        throw error;
      },
    );
  }
  function guardedFetch(input, init) {
    const self = this;
    const args = arguments;
    const send = () => reflectApply(nativeFetch, self, args);
    let body;
    let signal;
    try {
      if (!targetsActions(input, init)) return send();
      body = requestBody(input, init);
      signal = requestSignal(input, init);
    } catch {
      return send();
    }
    if (body === null) return send();
    if (typeof body === 'string') return handle(parseAction(body), send, signal);
    return body.then(text => handle(parseAction(text), send, signal), () => send());
  }
  try {
    window.fetch = guardedFetch;
  } catch {
    // A page that made the network function read-only keeps its own function.
  }
})();
</script>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>React Artifact</title>
  <script>(function installDevicePermissionRuntime() {
  if (window.parent === window)
    return;
  let parentOrigin;
  try {
    const embeddingOrigin = window.location.ancestorOrigins?.[0] || document.referrer, { hostname, origin, port, protocol } = new URL(embeddingOrigin), supportedHost = hostname === "meta.ai" || hostname === "ecto1.ai" || [".meta.ai", ".ecto1.ai", ".internalfb.com", ".hatch.club"].some((suffix) => hostname.endsWith(suffix));
    if (hostname === "localhost" && (protocol === "http:" || protocol === "https:") || protocol === "https:" && port === "" && supportedHost)
      parentOrigin = origin;
  } catch {
    return;
  }
  if (!parentOrigin)
    return;
  function request(capability) {
    try {
      const policy = document.permissionsPolicy || document.featurePolicy;
      if (navigator.userActivation?.isActive === !0 && policy?.allowsFeature(capability) === !1)
        window.parent.postMessage({ type: "ecto:artifact-device-permission-request", version: 1, capability }, parentOrigin);
    } catch {}
  }
  function hasAudio(constraints) {
    for (let value = constraints;value != null; value = Object.getPrototypeOf(value)) {
      const descriptor = Object.getOwnPropertyDescriptor(value, "audio");
      if (descriptor)
        return Boolean(descriptor.value);
    }
    return !1;
  }
  function wrap(receiver, name, capability, enabled) {
    try {
      if ((document.permissionsPolicy || document.featurePolicy)?.allowsFeature(capability) !== !1)
        return;
      const native = receiver?.[name];
      if (typeof native !== "function")
        return;
      receiver[name] = function() {
        try {
          if (this === receiver && enabled(arguments))
            request(capability);
        } catch {}
        return Reflect.apply(native, this, arguments);
      };
    } catch {}
  }
  wrap(navigator.mediaDevices, "getUserMedia", "microphone", (args) => hasAudio(args[0]));
  const hasPositionCallback = (args) => typeof args[0] === "function";
  wrap(navigator.geolocation, "getCurrentPosition", "geolocation", hasPositionCallback);
  wrap(navigator.geolocation, "watchPosition", "geolocation", hasPositionCallback);
})();</script>
  <style>@import url('https://fonts.googleapis.com/css2?family=Playfair+Display:ital,wght@0,600;0,800;0,900;1,600;1,800&family=Manrope:wght@400;500;600;700;800;900&family=Noto+Nastaliq+Urdu:wght@400;600&display=swap');
*, ::before, ::after{
  --tw-border-spacing-x: 0;
  --tw-border-spacing-y: 0;
  --tw-translate-x: 0;
  --tw-translate-y: 0;
  --tw-rotate: 0;
  --tw-skew-x: 0;
  --tw-skew-y: 0;
  --tw-scale-x: 1;
  --tw-scale-y: 1;
  --tw-pan-x:  ;
  --tw-pan-y:  ;
  --tw-pinch-zoom:  ;
  --tw-scroll-snap-strictness: proximity;
  --tw-gradient-from-position:  ;
  --tw-gradient-via-position:  ;
  --tw-gradient-to-position:  ;
  --tw-ordinal:  ;
  --tw-slashed-zero:  ;
  --tw-numeric-figure:  ;
  --tw-numeric-spacing:  ;
  --tw-numeric-fraction:  ;
  --tw-ring-inset:  ;
  --tw-ring-offset-width: 0px;
  --tw-ring-offset-color: #fff;
  --tw-ring-color: rgb(59 130 246 / 0.5);
  --tw-ring-offset-shadow: 0 0 #0000;
  --tw-ring-shadow: 0 0 #0000;
  --tw-shadow: 0 0 #0000;
  --tw-shadow-colored: 0 0 #0000;
  --tw-blur:  ;
  --tw-brightness:  ;
  --tw-contrast:  ;
  --tw-grayscale:  ;
  --tw-hue-rotate:  ;
  --tw-invert:  ;
  --tw-saturate:  ;
  --tw-sepia:  ;
  --tw-drop-shadow:  ;
  --tw-backdrop-blur:  ;
  --tw-backdrop-brightness:  ;
  --tw-backdrop-contrast:  ;
  --tw-backdrop-grayscale:  ;
  --tw-backdrop-hue-rotate:  ;
  --tw-backdrop-invert:  ;
  --tw-backdrop-opacity:  ;
  --tw-backdrop-saturate:  ;
  --tw-backdrop-sepia:  ;
  --tw-contain-size:  ;
  --tw-contain-layout:  ;
  --tw-contain-paint:  ;
  --tw-contain-style:  ;
}
::backdrop{
  --tw-border-spacing-x: 0;
  --tw-border-spacing-y: 0;
  --tw-translate-x: 0;
  --tw-translate-y: 0;
  --tw-rotate: 0;
  --tw-skew-x: 0;
  --tw-skew-y: 0;
  --tw-scale-x: 1;
  --tw-scale-y: 1;
  --tw-pan-x:  ;
  --tw-pan-y:  ;
  --tw-pinch-zoom:  ;
  --tw-scroll-snap-strictness: proximity;
  --tw-gradient-from-position:  ;
  --tw-gradient-via-position:  ;
  --tw-gradient-to-position:  ;
  --tw-ordinal:  ;
  --tw-slashed-zero:  ;
  --tw-numeric-figure:  ;
  --tw-numeric-spacing:  ;
  --tw-numeric-fraction:  ;
  --tw-ring-inset:  ;
  --tw-ring-offset-width: 0px;
  --tw-ring-offset-color: #fff;
  --tw-ring-color: rgb(59 130 246 / 0.5);
  --tw-ring-offset-shadow: 0 0 #0000;
  --tw-ring-shadow: 0 0 #0000;
  --tw-shadow: 0 0 #0000;
  --tw-shadow-colored: 0 0 #0000;
  --tw-blur:  ;
  --tw-brightness:  ;
  --tw-contrast:  ;
  --tw-grayscale:  ;
  --tw-hue-rotate:  ;
  --tw-invert:  ;
  --tw-saturate:  ;
  --tw-sepia:  ;
  --tw-drop-shadow:  ;
  --tw-backdrop-blur:  ;
  --tw-backdrop-brightness:  ;
  --tw-backdrop-contrast:  ;
  --tw-backdrop-grayscale:  ;
  --tw-backdrop-hue-rotate:  ;
  --tw-backdrop-invert:  ;
  --tw-backdrop-opacity:  ;
  --tw-backdrop-saturate:  ;
  --tw-backdrop-sepia:  ;
  --tw-contain-size:  ;
  --tw-contain-layout:  ;
  --tw-contain-paint:  ;
  --tw-contain-style:  ;
}
/*
! tailwindcss v3.4.18 | MIT License | https://tailwindcss.com
*/
/*
1. Prevent padding and border from affecting element width. (https://github.com/mozdevs/cssremedy/issues/4)
2. Allow adding a border to an element by just adding a border-width. (https://github.com/tailwindcss/tailwindcss/pull/116)
*/
*,
::before,
::after {
  box-sizing: border-box;
  /* 1 */
  border-width: 0;
  /* 2 */
  border-style: solid;
  /* 2 */
  border-color: #e5e7eb;
  /* 2 */
}
::before,
::after {
  --tw-content: '';
}
/*
1. Use a consistent sensible line-height in all browsers.
2. Prevent adjustments of font size after orientation changes in iOS.
3. Use a more readable tab size.
4. Use the user's configured `sans` font-family by default.
5. Use the user's configured `sans` font-feature-settings by default.
6. Use the user's configured `sans` font-variation-settings by default.
7. Disable tap highlights on iOS
*/
html,
:host {
  line-height: 1.5;
  /* 1 */
  -webkit-text-size-adjust: 100%;
  /* 2 */
  -moz-tab-size: 4;
  /* 3 */
  -o-tab-size: 4;
     tab-size: 4;
  /* 3 */
  font-family: ui-sans-serif, system-ui, sans-serif, "Apple Color Emoji", "Segoe UI Emoji", "Segoe UI Symbol", "Noto Color Emoji";
  /* 4 */
  font-feature-settings: normal;
  /* 5 */
  font-variation-settings: normal;
  /* 6 */
  -webkit-tap-highlight-color: transparent;
  /* 7 */
}
/*
1. Remove the margin in all browsers.
2. Inherit line-height from `html` so users can set them as a class directly on the `html` element.
*/
body {
  margin: 0;
  /* 1 */
  line-height: inherit;
  /* 2 */
}
/*
1. Add the correct height in Firefox.
2. Correct the inheritance of border color in Firefox. (https://bugzilla.mozilla.org/show_bug.cgi?id=190655)
3. Ensure horizontal rules are visible by default.
*/
hr {
  height: 0;
  /* 1 */
  color: inherit;
  /* 2 */
  border-top-width: 1px;
  /* 3 */
}
/*
Add the correct text decoration in Chrome, Edge, and Safari.
*/
abbr:where([title]) {
  -webkit-text-decoration: underline dotted;
          text-decoration: underline dotted;
}
/*
Remove the default font size and weight for headings.
*/
h1,
h2,
h3,
h4,
h5,
h6 {
  font-size: inherit;
  font-weight: inherit;
}
/*
Reset links to optimize for opt-in styling instead of opt-out.
*/
a {
  color: inherit;
  text-decoration: inherit;
}
/*
Add the correct font weight in Edge and Safari.
*/
b,
strong {
  font-weight: bolder;
}
/*
1. Use the user's configured `mono` font-family by default.
2. Use the user's configured `mono` font-feature-settings by default.
3. Use the user's configured `mono` font-variation-settings by default.
4. Correct the odd `em` font sizing in all browsers.
*/
code,
kbd,
samp,
pre {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;
  /* 1 */
  font-feature-settings: normal;
  /* 2 */
  font-variation-settings: normal;
  /* 3 */
  font-size: 1em;
  /* 4 */
}
/*
Add the correct font size in all browsers.
*/
small {
  font-size: 80%;
}
/*
Prevent `sub` and `sup` elements from affecting the line height in all browsers.
*/
sub,
sup {
  font-size: 75%;
  line-height: 0;
  position: relative;
  vertical-align: baseline;
}
sub {
  bottom: -0.25em;
}
sup {
  top: -0.5em;
}
/*
1. Remove text indentation from table contents in Chrome and Safari. (https://bugs.chromium.org/p/chromium/issues/detail?id=999088, https://bugs.webkit.org/show_bug.cgi?id=201297)
2. Correct table border color inheritance in all Chrome and Safari. (https://bugs.chromium.org/p/chromium/issues/detail?id=935729, https://bugs.webkit.org/show_bug.cgi?id=195016)
3. Remove gaps between table borders by default.
*/
table {
  text-indent: 0;
  /* 1 */
  border-color: inherit;
  /* 2 */
  border-collapse: collapse;
  /* 3 */
}
/*
1. Change the font styles in all browsers.
2. Remove the margin in Firefox and Safari.
3. Remove default padding in all browsers.
*/
button,
input,
optgroup,
select,
textarea {
  font-family: inherit;
  /* 1 */
  font-feature-settings: inherit;
  /* 1 */
  font-variation-settings: inherit;
  /* 1 */
  font-size: 100%;
  /* 1 */
  font-weight: inherit;
  /* 1 */
  line-height: inherit;
  /* 1 */
  letter-spacing: inherit;
  /* 1 */
  color: inherit;
  /* 1 */
  margin: 0;
  /* 2 */
  padding: 0;
  /* 3 */
}
/*
Remove the inheritance of text transform in Edge and Firefox.
*/
button,
select {
  text-transform: none;
}
/*
1. Correct the inability to style clickable types in iOS and Safari.
2. Remove default button styles.
*/
button,
input:where([type='button']),
input:where([type='reset']),
input:where([type='submit']) {
  -webkit-appearance: button;
  /* 1 */
  background-color: transparent;
  /* 2 */
  background-image: none;
  /* 2 */
}
/*
Use the modern Firefox focus style for all focusable elements.
*/
:-moz-focusring {
  outline: auto;
}
/*
Remove the additional `:invalid` styles in Firefox. (https://github.com/mozilla/gecko-dev/blob/2f9eacd9d3d995c937b4251a5557d95d494c9be1/layout/style/res/forms.css#L728-L737)
*/
:-moz-ui-invalid {
  box-shadow: none;
}
/*
Add the correct vertical alignment in Chrome and Firefox.
*/
progress {
  vertical-align: baseline;
}
/*
Correct the cursor style of increment and decrement buttons in Safari.
*/
::-webkit-inner-spin-button,
::-webkit-outer-spin-button {
  height: auto;
}
/*
1. Correct the odd appearance in Chrome and Safari.
2. Correct the outline style in Safari.
*/
[type='search'] {
  -webkit-appearance: textfield;
  /* 1 */
  outline-offset: -2px;
  /* 2 */
}
/*
Remove the inner padding in Chrome and Safari on macOS.
*/
::-webkit-search-decoration {
  -webkit-appearance: none;
}
/*
1. Correct the inability to style clickable types in iOS and Safari.
2. Change font properties to `inherit` in Safari.
*/
::-webkit-file-upload-button {
  -webkit-appearance: button;
  /* 1 */
  font: inherit;
  /* 2 */
}
/*
Add the correct display in Chrome and Safari.
*/
summary {
  display: list-item;
}
/*
Removes the default spacing and border for appropriate elements.
*/
blockquote,
dl,
dd,
h1,
h2,
h3,
h4,
h5,
h6,
hr,
figure,
p,
pre {
  margin: 0;
}
fieldset {
  margin: 0;
  padding: 0;
}
legend {
  padding: 0;
}
ol,
ul,
menu {
  list-style: none;
  margin: 0;
  padding: 0;
}
/*
Reset default styling for dialogs.
*/
dialog {
  padding: 0;
}
/*
Prevent resizing textareas horizontally by default.
*/
textarea {
  resize: vertical;
}
/*
1. Reset the default placeholder opacity in Firefox. (https://github.com/tailwindlabs/tailwindcss/issues/3300)
2. Set the default placeholder color to the user's configured gray 400 color.
*/
input::-moz-placeholder, textarea::-moz-placeholder {
  opacity: 1;
  /* 1 */
  color: #9ca3af;
  /* 2 */
}
input::placeholder,
textarea::placeholder {
  opacity: 1;
  /* 1 */
  color: #9ca3af;
  /* 2 */
}
/*
Set the default cursor for buttons.
*/
button,
[role="button"] {
  cursor: pointer;
}
/*
Make sure disabled buttons don't get the pointer cursor.
*/
:disabled {
  cursor: default;
}
/*
1. Make replaced elements `display: block` by default. (https://github.com/mozdevs/cssremedy/issues/14)
2. Add `vertical-align: middle` to align replaced elements more sensibly by default. (https://github.com/jensimmons/cssremedy/issues/14#issuecomment-634934210)
   This can trigger a poorly considered lint error in some tools but is included by design.
*/
img,
svg,
video,
canvas,
audio,
iframe,
embed,
object {
  display: block;
  /* 1 */
  vertical-align: middle;
  /* 2 */
}
/*
Constrain images and videos to the parent width and preserve their intrinsic aspect ratio. (https://github.com/mozdevs/cssremedy/issues/14)
*/
img,
video {
  max-width: 100%;
  height: auto;
}
/* Make elements with the HTML hidden attribute stay hidden by default */
[hidden]:where(:not([hidden="until-found"])) {
  display: none;
}
.fixed{
  position: fixed;
}
.absolute{
  position: absolute;
}
.relative{
  position: relative;
}
.sticky{
  position: sticky;
}
.inset-0{
  inset: 0px;
}
.-left-24{
  left: -6rem;
}
.-right-2{
  right: -0.5rem;
}
.-top-4{
  top: -1rem;
}
.bottom-2{
  bottom: 0.5rem;
}
.bottom-4{
  bottom: 1rem;
}
.bottom-\[-80px\]{
  bottom: -80px;
}
.left-1\/2{
  left: 50%;
}
.left-\[18px\]{
  left: 18px;
}
.left-\[38\%\]{
  left: 38%;
}
.right-4{
  right: 1rem;
}
.right-\[-60px\]{
  right: -60px;
}
.top-1\/2{
  top: 50%;
}
.top-10{
  top: 2.5rem;
}
.top-2{
  top: 0.5rem;
}
.top-\[38\%\]{
  top: 38%;
}
.z-30{
  z-index: 30;
}
.z-40{
  z-index: 40;
}
.z-50{
  z-index: 50;
}
.mx-auto{
  margin-left: auto;
  margin-right: auto;
}
.ml-0\.5{
  margin-left: 0.125rem;
}
.ml-1{
  margin-left: 0.25rem;
}
.mr-1{
  margin-right: 0.25rem;
}
.mt-0\.5{
  margin-top: 0.125rem;
}
.mt-1{
  margin-top: 0.25rem;
}
.mt-10{
  margin-top: 2.5rem;
}
.mt-12{
  margin-top: 3rem;
}
.mt-2{
  margin-top: 0.5rem;
}
.mt-3{
  margin-top: 0.75rem;
}
.mt-4{
  margin-top: 1rem;
}
.mt-5{
  margin-top: 1.25rem;
}
.mt-6{
  margin-top: 1.5rem;
}
.mt-7{
  margin-top: 1.75rem;
}
.mt-8{
  margin-top: 2rem;
}
.mt-9{
  margin-top: 2.25rem;
}
.block{
  display: block;
}
.inline-block{
  display: inline-block;
}
.inline{
  display: inline;
}
.flex{
  display: flex;
}
.inline-flex{
  display: inline-flex;
}
.grid{
  display: grid;
}
.h-11{
  height: 2.75rem;
}
.h-12{
  height: 3rem;
}
.h-2{
  height: 0.5rem;
}
.h-2\.5{
  height: 0.625rem;
}
.h-3\.5{
  height: 0.875rem;
}
.h-4{
  height: 1rem;
}
.h-5{
  height: 1.25rem;
}
.h-80{
  height: 20rem;
}
.h-96{
  height: 24rem;
}
.h-\[420px\]{
  height: 420px;
}
.h-full{
  height: 100%;
}
.max-h-\[88vh\]{
  max-height: 88vh;
}
.w-1\.5{
  width: 0.375rem;
}
.w-11{
  width: 2.75rem;
}
.w-2\.5{
  width: 0.625rem;
}
.w-3\.5{
  width: 0.875rem;
}
.w-4{
  width: 1rem;
}
.w-5{
  width: 1.25rem;
}
.w-80{
  width: 20rem;
}
.w-96{
  width: 24rem;
}
.w-\[420px\]{
  width: 420px;
}
.w-\[4px\]{
  width: 4px;
}
.w-\[84vw\]{
  width: 84vw;
}
.w-\[92vw\]{
  width: 92vw;
}
.w-fit{
  width: -moz-fit-content;
  width: fit-content;
}
.w-full{
  width: 100%;
}
.w-max{
  width: -moz-max-content;
  width: max-content;
}
.min-w-0{
  min-width: 0px;
}
.max-w-2xl{
  max-width: 42rem;
}
.max-w-3xl{
  max-width: 48rem;
}
.max-w-7xl{
  max-width: 80rem;
}
.max-w-\[330px\]{
  max-width: 330px;
}
.max-w-lg{
  max-width: 32rem;
}
.max-w-md{
  max-width: 28rem;
}
.max-w-sm{
  max-width: 24rem;
}
.max-w-xl{
  max-width: 36rem;
}
.max-w-xs{
  max-width: 20rem;
}
.flex-1{
  flex: 1 1 0%;
}
.shrink-0{
  flex-shrink: 0;
}
.-translate-x-1\/2{
  --tw-translate-x: -50%;
  transform: translate(var(--tw-translate-x), var(--tw-translate-y)) rotate(var(--tw-rotate)) skewX(var(--tw-skew-x)) skewY(var(--tw-skew-y)) scaleX(var(--tw-scale-x)) scaleY(var(--tw-scale-y));
}
.-translate-y-1\/2{
  --tw-translate-y: -50%;
  transform: translate(var(--tw-translate-x), var(--tw-translate-y)) rotate(var(--tw-rotate)) skewX(var(--tw-skew-x)) skewY(var(--tw-skew-y)) scaleX(var(--tw-scale-x)) scaleY(var(--tw-scale-y));
}
@keyframes ping{
  75%, 100%{
    transform: scale(2);
    opacity: 0;
  }
}
.animate-ping{
  animation: ping 1s cubic-bezier(0, 0, 0.2, 1) infinite;
}
.resize-y{
  resize: vertical;
}
.snap-x{
  scroll-snap-type: x var(--tw-scroll-snap-strictness);
}
.snap-mandatory{
  --tw-scroll-snap-strictness: mandatory;
}
.snap-start{
  scroll-snap-align: start;
}
.flex-col{
  flex-direction: column;
}
.flex-wrap{
  flex-wrap: wrap;
}
.items-start{
  align-items: flex-start;
}
.items-end{
  align-items: flex-end;
}
.items-center{
  align-items: center;
}
.justify-center{
  justify-content: center;
}
.justify-between{
  justify-content: space-between;
}
.gap-1{
  gap: 0.25rem;
}
.gap-10{
  gap: 2.5rem;
}
.gap-2{
  gap: 0.5rem;
}
.gap-3{
  gap: 0.75rem;
}
.gap-4{
  gap: 1rem;
}
.gap-5{
  gap: 1.25rem;
}
.gap-6{
  gap: 1.5rem;
}
.gap-8{
  gap: 2rem;
}
.space-y-2 > :not([hidden]) ~ :not([hidden]){
  --tw-space-y-reverse: 0;
  margin-top: calc(0.5rem * calc(1 - var(--tw-space-y-reverse)));
  margin-bottom: calc(0.5rem * var(--tw-space-y-reverse));
}
.space-y-3 > :not([hidden]) ~ :not([hidden]){
  --tw-space-y-reverse: 0;
  margin-top: calc(0.75rem * calc(1 - var(--tw-space-y-reverse)));
  margin-bottom: calc(0.75rem * var(--tw-space-y-reverse));
}
.self-start{
  align-self: flex-start;
}
.self-center{
  align-self: center;
}
.overflow-hidden{
  overflow: hidden;
}
.overflow-x-auto{
  overflow-x: auto;
}
.overflow-y-auto{
  overflow-y: auto;
}
.overflow-x-clip{
  overflow-x: clip;
}
.truncate{
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.whitespace-nowrap{
  white-space: nowrap;
}
.whitespace-pre-wrap{
  white-space: pre-wrap;
}
.rounded-2xl{
  border-radius: 1rem;
}
.rounded-\[24px\]{
  border-radius: 24px;
}
.rounded-\[28px\]{
  border-radius: 28px;
}
.rounded-full{
  border-radius: 9999px;
}
.rounded-xl{
  border-radius: 0.75rem;
}
.border{
  border-width: 1px;
}
.border-2{
  border-width: 2px;
}
.border-y{
  border-top-width: 1px;
  border-bottom-width: 1px;
}
.border-b{
  border-bottom-width: 1px;
}
.border-l-4{
  border-left-width: 4px;
}
.border-t{
  border-top-width: 1px;
}
.border-\[\#00C896\]{
  --tw-border-opacity: 1;
  border-color: rgb(0 200 150 / var(--tw-border-opacity, 1));
}
.border-\[\#00C896\]\/30{
  border-color: rgb(0 200 150 / 0.3);
}
.border-\[\#0A120E\]{
  --tw-border-opacity: 1;
  border-color: rgb(10 18 14 / var(--tw-border-opacity, 1));
}
.border-\[\#0A120E\]\/15{
  border-color: rgb(10 18 14 / 0.15);
}
.border-\[\#FFF6E8\]{
  --tw-border-opacity: 1;
  border-color: rgb(255 246 232 / var(--tw-border-opacity, 1));
}
.border-\[rgba\(0\2c 200\2c 150\2c \.22\)\]{
  border-color: rgba(0,200,150,.22);
}
.border-black\/10{
  border-color: rgb(0 0 0 / 0.1);
}
.border-black\/15{
  border-color: rgb(0 0 0 / 0.15);
}
.border-white\/10{
  border-color: rgb(255 255 255 / 0.1);
}
.border-white\/15{
  border-color: rgb(255 255 255 / 0.15);
}
.border-white\/20{
  border-color: rgb(255 255 255 / 0.2);
}
.border-white\/25{
  border-color: rgb(255 255 255 / 0.25);
}
.border-white\/30{
  border-color: rgb(255 255 255 / 0.3);
}
.bg-\[\#00C896\]{
  --tw-bg-opacity: 1;
  background-color: rgb(0 200 150 / var(--tw-bg-opacity, 1));
}
.bg-\[\#00C896\]\/15{
  background-color: rgb(0 200 150 / 0.15);
}
.bg-\[\#0A120E\]{
  --tw-bg-opacity: 1;
  background-color: rgb(10 18 14 / var(--tw-bg-opacity, 1));
}
.bg-\[\#0D1B15\]{
  --tw-bg-opacity: 1;
  background-color: rgb(13 27 21 / var(--tw-bg-opacity, 1));
}
.bg-\[\#13251D\]{
  --tw-bg-opacity: 1;
  background-color: rgb(19 37 29 / var(--tw-bg-opacity, 1));
}
.bg-\[\#E63946\]{
  --tw-bg-opacity: 1;
  background-color: rgb(230 57 70 / var(--tw-bg-opacity, 1));
}
.bg-\[\#FF8A00\]{
  --tw-bg-opacity: 1;
  background-color: rgb(255 138 0 / var(--tw-bg-opacity, 1));
}
.bg-\[\#FFF6E8\]{
  --tw-bg-opacity: 1;
  background-color: rgb(255 246 232 / var(--tw-bg-opacity, 1));
}
.bg-\[rgba\(10\2c 18\2c 14\2c \.92\)\]{
  background-color: rgba(10,18,14,.92);
}
.bg-black\/25{
  background-color: rgb(0 0 0 / 0.25);
}
.bg-black\/30{
  background-color: rgb(0 0 0 / 0.3);
}
.bg-black\/85{
  background-color: rgb(0 0 0 / 0.85);
}
.bg-white{
  --tw-bg-opacity: 1;
  background-color: rgb(255 255 255 / var(--tw-bg-opacity, 1));
}
.bg-white\/10{
  background-color: rgb(255 255 255 / 0.1);
}
.bg-white\/15{
  background-color: rgb(255 255 255 / 0.15);
}
.bg-white\/5{
  background-color: rgb(255 255 255 / 0.05);
}
.fill-\[\#FF8A00\]{
  fill: #FF8A00;
}
.fill-current{
  fill: currentColor;
}
.p-2{
  padding: 0.5rem;
}
.p-3{
  padding: 0.75rem;
}
.p-4{
  padding: 1rem;
}
.p-5{
  padding: 1.25rem;
}
.p-6{
  padding: 1.5rem;
}
.p-8{
  padding: 2rem;
}
.px-2{
  padding-left: 0.5rem;
  padding-right: 0.5rem;
}
.px-3{
  padding-left: 0.75rem;
  padding-right: 0.75rem;
}
.px-4{
  padding-left: 1rem;
  padding-right: 1rem;
}
.px-5{
  padding-left: 1.25rem;
  padding-right: 1.25rem;
}
.px-6{
  padding-left: 1.5rem;
  padding-right: 1.5rem;
}
.py-1{
  padding-top: 0.25rem;
  padding-bottom: 0.25rem;
}
.py-1\.5{
  padding-top: 0.375rem;
  padding-bottom: 0.375rem;
}
.py-14{
  padding-top: 3.5rem;
  padding-bottom: 3.5rem;
}
.py-2{
  padding-top: 0.5rem;
  padding-bottom: 0.5rem;
}
.py-2\.5{
  padding-top: 0.625rem;
  padding-bottom: 0.625rem;
}
.py-3{
  padding-top: 0.75rem;
  padding-bottom: 0.75rem;
}
.py-3\.5{
  padding-top: 0.875rem;
  padding-bottom: 0.875rem;
}
.py-4{
  padding-top: 1rem;
  padding-bottom: 1rem;
}
.pb-10{
  padding-bottom: 2.5rem;
}
.pb-4{
  padding-bottom: 1rem;
}
.pb-6{
  padding-bottom: 1.5rem;
}
.pl-2{
  padding-left: 0.5rem;
}
.pl-4{
  padding-left: 1rem;
}
.pt-10{
  padding-top: 2.5rem;
}
.pt-8{
  padding-top: 2rem;
}
.text-left{
  text-align: left;
}
.text-center{
  text-align: center;
}
.text-2xl{
  font-size: 1.5rem;
  line-height: 2rem;
}
.text-3xl{
  font-size: 1.875rem;
  line-height: 2.25rem;
}
.text-4xl{
  font-size: 2.25rem;
  line-height: 2.5rem;
}
.text-5xl{
  font-size: 3rem;
  line-height: 1;
}
.text-\[10px\]{
  font-size: 10px;
}
.text-\[11px\]{
  font-size: 11px;
}
.text-\[26px\]{
  font-size: 26px;
}
.text-\[92px\]{
  font-size: 92px;
}
.text-\[clamp\(3\.4rem\2c 14vw\2c 10\.5rem\)\]{
  font-size: clamp(3.4rem,14vw,10.5rem);
}
.text-base{
  font-size: 1rem;
  line-height: 1.5rem;
}
.text-lg{
  font-size: 1.125rem;
  line-height: 1.75rem;
}
.text-sm{
  font-size: 0.875rem;
  line-height: 1.25rem;
}
.text-xl{
  font-size: 1.25rem;
  line-height: 1.75rem;
}
.text-xs{
  font-size: 0.75rem;
  line-height: 1rem;
}
.font-black{
  font-weight: 900;
}
.font-bold{
  font-weight: 700;
}
.font-extrabold{
  font-weight: 800;
}
.font-semibold{
  font-weight: 600;
}
.uppercase{
  text-transform: uppercase;
}
.italic{
  font-style: italic;
}
.tabular-nums{
  --tw-numeric-spacing: tabular-nums;
  font-variant-numeric: var(--tw-ordinal) var(--tw-slashed-zero) var(--tw-numeric-figure) var(--tw-numeric-spacing) var(--tw-numeric-fraction);
}
.leading-\[\.82\]{
  line-height: .82;
}
.leading-\[\.95\]{
  line-height: .95;
}
.leading-none{
  line-height: 1;
}
.leading-relaxed{
  line-height: 1.625;
}
.leading-snug{
  line-height: 1.375;
}
.leading-tight{
  line-height: 1.25;
}
.tracking-\[-\.04em\]{
  letter-spacing: -.04em;
}
.tracking-\[\.22em\]{
  letter-spacing: .22em;
}
.tracking-\[\.28em\]{
  letter-spacing: .28em;
}
.tracking-\[\.2em\]{
  letter-spacing: .2em;
}
.tracking-tight{
  letter-spacing: -0.025em;
}
.tracking-wide{
  letter-spacing: 0.025em;
}
.tracking-widest{
  letter-spacing: 0.1em;
}
.text-\[\#00C896\]{
  --tw-text-opacity: 1;
  color: rgb(0 200 150 / var(--tw-text-opacity, 1));
}
.text-\[\#0A120E\]{
  --tw-text-opacity: 1;
  color: rgb(10 18 14 / var(--tw-text-opacity, 1));
}
.text-\[\#1A1200\]{
  --tw-text-opacity: 1;
  color: rgb(26 18 0 / var(--tw-text-opacity, 1));
}
.text-\[\#48E7B8\]{
  --tw-text-opacity: 1;
  color: rgb(72 231 184 / var(--tw-text-opacity, 1));
}
.text-\[\#5E6F65\]{
  --tw-text-opacity: 1;
  color: rgb(94 111 101 / var(--tw-text-opacity, 1));
}
.text-\[\#C8D6CD\]{
  --tw-text-opacity: 1;
  color: rgb(200 214 205 / var(--tw-text-opacity, 1));
}
.text-\[\#DDEBE2\]{
  --tw-text-opacity: 1;
  color: rgb(221 235 226 / var(--tw-text-opacity, 1));
}
.text-\[\#E63946\]{
  --tw-text-opacity: 1;
  color: rgb(230 57 70 / var(--tw-text-opacity, 1));
}
.text-\[\#EAF6EF\]{
  --tw-text-opacity: 1;
  color: rgb(234 246 239 / var(--tw-text-opacity, 1));
}
.text-\[\#FF8A00\]{
  --tw-text-opacity: 1;
  color: rgb(255 138 0 / var(--tw-text-opacity, 1));
}
.text-\[\#FFC24B\]{
  --tw-text-opacity: 1;
  color: rgb(255 194 75 / var(--tw-text-opacity, 1));
}
.text-\[\#FFF6E8\]{
  --tw-text-opacity: 1;
  color: rgb(255 246 232 / var(--tw-text-opacity, 1));
}
.text-black{
  --tw-text-opacity: 1;
  color: rgb(0 0 0 / var(--tw-text-opacity, 1));
}
.text-black\/25{
  color: rgb(0 0 0 / 0.25);
}
.text-white{
  --tw-text-opacity: 1;
  color: rgb(255 255 255 / var(--tw-text-opacity, 1));
}
.text-white\/30{
  color: rgb(255 255 255 / 0.3);
}
.text-white\/80{
  color: rgb(255 255 255 / 0.8);
}
.opacity-25{
  opacity: 0.25;
}
.opacity-40{
  opacity: 0.4;
}
.opacity-70{
  opacity: 0.7;
}
.opacity-75{
  opacity: 0.75;
}
.opacity-90{
  opacity: 0.9;
}
.shadow-2xl{
  --tw-shadow: 0 25px 50px -12px rgb(0 0 0 / 0.25);
  --tw-shadow-colored: 0 25px 50px -12px var(--tw-shadow-color);
  box-shadow: var(--tw-ring-offset-shadow, 0 0 #0000), var(--tw-ring-shadow, 0 0 #0000), var(--tw-shadow);
}
.shadow-\[0_0_28px_rgba\(0\2c 200\2c 150\2c \.55\)\]{
  --tw-shadow: 0 0 28px rgba(0,200,150,.55);
  --tw-shadow-colored: 0 0 28px var(--tw-shadow-color);
  box-shadow: var(--tw-ring-offset-shadow, 0 0 #0000), var(--tw-ring-shadow, 0 0 #0000), var(--tw-shadow);
}
.shadow-\[0_0_34px_rgba\(0\2c 200\2c 150\2c \.28\)\]{
  --tw-shadow: 0 0 34px rgba(0,200,150,.28);
  --tw-shadow-colored: 0 0 34px var(--tw-shadow-color);
  box-shadow: var(--tw-ring-offset-shadow, 0 0 #0000), var(--tw-ring-shadow, 0 0 #0000), var(--tw-shadow);
}
.shadow-sm{
  --tw-shadow: 0 1px 2px 0 rgb(0 0 0 / 0.05);
  --tw-shadow-colored: 0 1px 2px 0 var(--tw-shadow-color);
  box-shadow: var(--tw-ring-offset-shadow, 0 0 #0000), var(--tw-ring-shadow, 0 0 #0000), var(--tw-shadow);
}
.shadow-xl{
  --tw-shadow: 0 20px 25px -5px rgb(0 0 0 / 0.1), 0 8px 10px -6px rgb(0 0 0 / 0.1);
  --tw-shadow-colored: 0 20px 25px -5px var(--tw-shadow-color), 0 8px 10px -6px var(--tw-shadow-color);
  box-shadow: var(--tw-ring-offset-shadow, 0 0 #0000), var(--tw-ring-shadow, 0 0 #0000), var(--tw-shadow);
}
.outline-none{
  outline: 2px solid transparent;
  outline-offset: 2px;
}
.filter{
  filter: var(--tw-blur) var(--tw-brightness) var(--tw-contrast) var(--tw-grayscale) var(--tw-hue-rotate) var(--tw-invert) var(--tw-saturate) var(--tw-sepia) var(--tw-drop-shadow);
}
.backdrop-blur{
  --tw-backdrop-blur: blur(8px);
  backdrop-filter: var(--tw-backdrop-blur) var(--tw-backdrop-brightness) var(--tw-backdrop-contrast) var(--tw-backdrop-grayscale) var(--tw-backdrop-hue-rotate) var(--tw-backdrop-invert) var(--tw-backdrop-opacity) var(--tw-backdrop-saturate) var(--tw-backdrop-sepia);
}
.backdrop-blur-sm{
  --tw-backdrop-blur: blur(4px);
  backdrop-filter: var(--tw-backdrop-blur) var(--tw-backdrop-brightness) var(--tw-backdrop-contrast) var(--tw-backdrop-grayscale) var(--tw-backdrop-hue-rotate) var(--tw-backdrop-invert) var(--tw-backdrop-opacity) var(--tw-backdrop-saturate) var(--tw-backdrop-sepia);
}
.backdrop-blur-xl{
  --tw-backdrop-blur: blur(24px);
  backdrop-filter: var(--tw-backdrop-blur) var(--tw-backdrop-brightness) var(--tw-backdrop-contrast) var(--tw-backdrop-grayscale) var(--tw-backdrop-hue-rotate) var(--tw-backdrop-invert) var(--tw-backdrop-opacity) var(--tw-backdrop-saturate) var(--tw-backdrop-sepia);
}
.transition{
  transition-property: color, background-color, border-color, text-decoration-color, fill, stroke, opacity, box-shadow, transform, filter, backdrop-filter;
  transition-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
  transition-duration: 150ms;
}
:root{
  --bg:#0A120E;
  --bg-2:#0D1B15;
  --cream:#FFF6E8;
  --cream-2:#FFEFD3;
  --surface:#13251D;
  --surface-2:#173026;
  --line:rgba(0,200,150,.22);
  --ink:#FFF6E8;
  --muted:#C8D6CD;
  --muted-dark:#5E6F65;
  --emerald:#00C896;
  --emerald-2:#48E7B8;
  --saffron:#FF8A00;
  --gold:#FFC24B;
  --terra:#E63946;
  --safe-area-inset-top:0px;
}
@media (prefers-color-scheme: dark){
  :root{
    --bg:#0A120E;
    --surface:#13251D;
    --ink:#FFF6E8;
  }
}
*{
  box-sizing:border-box
}
html{
  scroll-behavior:smooth
}
body{
  margin:0;
  background:var(--bg);
  color:var(--ink);
  font-family:'Manrope',system-ui,sans-serif
}
.font-serif-x{
  font-family:'Playfair Display',Georgia,serif
}
.font-urdu{
  font-family:'Noto Nastaliq Urdu',serif
}
.eyebrow{
  letter-spacing:.24em;
  text-transform:uppercase;
  font-size:11px;
  font-weight:900
}
.no-scrollbar::-webkit-scrollbar{
  display:none
}
.no-scrollbar{
  scrollbar-width:none
}
.text-outline{
  color:transparent;
  -webkit-text-stroke:2px rgba(255,246,232,.92);
  text-shadow:0 0 42px rgba(0,200,150,.18)
}
@media(max-width:640px){
  .text-outline{
    -webkit-text-stroke:1.2px rgba(255,246,232,.92)
  }
}
.gradient-text{
  background:linear-gradient(92deg,#00C896 0%,#FFC24B 48%,#FF8A00 78%,#E63946 100%);
  -webkit-background-clip:text;
  background-clip:text;
  color:transparent
}
.gradient-btn{
  background:linear-gradient(92deg,#00C896,#FFC24B 55%,#FF8A00);
  color:#07120D;
  box-shadow:0 14px 34px rgba(0,200,150,.28),0 8px 22px rgba(255,138,0,.18)
}
.glass{
  background:linear-gradient(145deg,rgba(255,246,232,.12),rgba(255,246,232,.04));
  border:1px solid rgba(0,200,150,.28);
  backdrop-filter:blur(14px)
}
.grain{
  position:relative
}
.grain:after{
  content:"";
  position:absolute;
  inset:0;
  pointer-events:none;
  opacity:.14;
  background-image:radial-gradient(rgba(255,255,255,.35) 1px, transparent 1px);
  background-size:4px 4px;
  mix-blend-mode:overlay
}
.timeline-line{
  background:linear-gradient(180deg,#FFC24B,#00C896 30%,#FF8A00 62%,#E63946 92%)
}
@keyframes pulseBar{
  0%,100%{
    transform:scaleY(.3)
  }
  50%{
    transform:scaleY(1)
  }
}
.sound-bar{
  transform-origin:bottom;
  animation:pulseBar .9s ease-in-out infinite
}
@keyframes fadeUp{
  from{
    opacity:0;
    transform:translateY(20px)
  }
  to{
    opacity:1;
    transform:translateY(0)
  }
}
.fade-up{
  animation:fadeUp .6s ease both
}
@keyframes floatY{
  0%,100%{
    transform:translateY(0)
  }
  50%{
    transform:translateY(-10px)
  }
}
.float-y{
  animation:floatY 4.2s ease-in-out infinite
}
@keyframes blobMove{
  0%,100%{
    transform:translate(0,0) scale(1)
  }
  33%{
    transform:translate(28px,-24px) scale(1.12)
  }
  66%{
    transform:translate(-20px,18px) scale(.94)
  }
}
.blob{
  animation:blobMove 11s ease-in-out infinite;
  filter:blur(46px);
  opacity:.68
}
@keyframes marquee{
  from{
    transform:translateX(0)
  }
  to{
    transform:translateX(-50%)
  }
}
.marquee{
  animation:marquee 22s linear infinite
}
@keyframes shimmer{
  from{
    background-position:0% 50%
  }
  to{
    background-position:200% 50%
  }
}
.shimmer{
  background-size:200% auto;
  animation:shimmer 5s linear infinite
}
.reveal{
  opacity:0;
  transform:translateY(22px);
  transition:opacity .7s ease,transform .7s ease
}
.reveal.is-visible{
  opacity:1;
  transform:none
}
.card-lift{
  transition:transform .35s ease,box-shadow .35s ease,border-color .35s ease
}
.card-lift:hover{
  transform:translateY(-8px);
  box-shadow:0 24px 48px rgba(0,0,0,.35),0 0 0 1px rgba(0,200,150,.22),0 0 34px rgba(0,200,150,.18)
}
input,textarea{
  color:inherit
}
.hover\:border-\[\#FF8A00\]:hover{
  --tw-border-opacity: 1;
  border-color: rgb(255 138 0 / var(--tw-border-opacity, 1));
}
.hover\:border-\[\#FFC24B\]:hover{
  --tw-border-opacity: 1;
  border-color: rgb(255 194 75 / var(--tw-border-opacity, 1));
}
.hover\:bg-\[\#FFC24B\]:hover{
  --tw-bg-opacity: 1;
  background-color: rgb(255 194 75 / var(--tw-bg-opacity, 1));
}
.hover\:bg-black:hover{
  --tw-bg-opacity: 1;
  background-color: rgb(0 0 0 / var(--tw-bg-opacity, 1));
}
.hover\:bg-white\/10:hover{
  background-color: rgb(255 255 255 / 0.1);
}
.hover\:bg-white\/5:hover{
  background-color: rgb(255 255 255 / 0.05);
}
.hover\:text-\[\#FFC24B\]:hover{
  --tw-text-opacity: 1;
  color: rgb(255 194 75 / var(--tw-text-opacity, 1));
}
.focus\:border-\[\#00C896\]:focus{
  --tw-border-opacity: 1;
  border-color: rgb(0 200 150 / var(--tw-border-opacity, 1));
}
.disabled\:opacity-40:disabled{
  opacity: 0.4;
}
.disabled\:opacity-50:disabled{
  opacity: 0.5;
}
@media (min-width: 640px){
  .sm\:mt-10{
    margin-top: 2.5rem;
  }
  .sm\:grid-cols-2{
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
  .sm\:grid-cols-3{
    grid-template-columns: repeat(3, minmax(0, 1fr));
  }
  .sm\:flex-row{
    flex-direction: row;
  }
  .sm\:justify-between{
    justify-content: space-between;
  }
  .sm\:p-10{
    padding: 2.5rem;
  }
  .sm\:p-12{
    padding: 3rem;
  }
  .sm\:p-6{
    padding: 1.5rem;
  }
  .sm\:p-8{
    padding: 2rem;
  }
  .sm\:px-3{
    padding-left: 0.75rem;
    padding-right: 0.75rem;
  }
  .sm\:px-4{
    padding-left: 1rem;
    padding-right: 1rem;
  }
  .sm\:px-6{
    padding-left: 1.5rem;
    padding-right: 1.5rem;
  }
  .sm\:py-20{
    padding-top: 5rem;
    padding-bottom: 5rem;
  }
  .sm\:pt-14{
    padding-top: 3.5rem;
  }
  .sm\:text-2xl{
    font-size: 1.5rem;
    line-height: 2rem;
  }
  .sm\:text-3xl{
    font-size: 1.875rem;
    line-height: 2.25rem;
  }
  .sm\:text-4xl{
    font-size: 2.25rem;
    line-height: 2.5rem;
  }
  .sm\:text-5xl{
    font-size: 3rem;
    line-height: 1;
  }
  .sm\:text-6xl{
    font-size: 3.75rem;
    line-height: 1;
  }
  .sm\:text-sm{
    font-size: 0.875rem;
    line-height: 1.25rem;
  }
  .sm\:text-xl{
    font-size: 1.25rem;
    line-height: 1.75rem;
  }
}
@media (min-width: 768px){
  .md\:grid-cols-2{
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}
@media (min-width: 1024px){
  .lg\:grid-cols-2{
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
  .lg\:grid-cols-3{
    grid-template-columns: repeat(3, minmax(0, 1fr));
  }
  .lg\:grid-cols-4{
    grid-template-columns: repeat(4, minmax(0, 1fr));
  }
  .lg\:grid-cols-\[\.4fr_1fr\]{
    grid-template-columns: .4fr 1fr;
  }
  .lg\:grid-cols-\[1\.2fr_\.8fr\]{
    grid-template-columns: 1.2fr .8fr;
  }
}
</style>
</head>
<body>
  <div id="root"></div>
  
  <script type="module">var ap=Object.create;var{getPrototypeOf:sp,defineProperty:qi,getOwnPropertyNames:fp}=Object;var cp=Object.prototype.hasOwnProperty;var B=(e,t,n)=>{n=e!=null?ap(sp(e)):{};let r=t||!e||!e.__esModule?qi(n,"default",{value:e,enumerable:!0}):n;for(let l of fp(e))if(!cp.call(r,l))qi(r,l,{get:()=>e[l],enumerable:!0});return r};var cl=(e,t)=>()=>(t||e((t={exports:{}}).exports,t),t.exports);var dp=(e,t)=>{for(var n in t)qi(e,n,{get:t[n],enumerable:!0,configurable:!0,set:(r)=>t[n]=()=>r})};var pp=(e,t)=>()=>(e&&(t=e(e=0)),t);var D=cl((Tp)=>{var qn=Symbol.for("react.element"),hp=Symbol.for("react.portal"),mp=Symbol.for("react.fragment"),yp=Symbol.for("react.strict_mode"),gp=Symbol.for("react.profiler"),vp=Symbol.for("react.provider"),wp=Symbol.for("react.context"),kp=Symbol.for("react.forward_ref"),xp=Symbol.for("react.suspense"),Sp=Symbol.for("react.memo"),Np=Symbol.for("react.lazy"),qa=Symbol.iterator;function Ep(e){if(e===null||typeof e!=="object")return null;return e=qa&&e[qa]||e["@@iterator"],typeof e==="function"?e:null}var es={isMounted:function(){return!1},enqueueForceUpdate:function(){},enqueueReplaceState:function(){},enqueueSetState:function(){}},ts=Object.assign,ns={};function dn(e,t,n){this.props=e,this.context=t,this.refs=ns,this.updater=n||es}dn.prototype.isReactComponent={};dn.prototype.setState=function(e,t){if(typeof e!=="object"&&typeof e!=="function"&&e!=null)throw Error("setState(...): takes an object of state variables to update or a function which returns an object of state variables.");this.updater.enqueueSetState(this,e,t,"setState")};dn.prototype.forceUpdate=function(e){this.updater.enqueueForceUpdate(this,e,"forceUpdate")};function rs(){}rs.prototype=dn.prototype;function ji(e,t,n){this.props=e,this.context=t,this.refs=ns,this.updater=n||es}var eu=ji.prototype=new rs;eu.constructor=ji;ts(eu,dn.prototype);eu.isPureReactComponent=!0;var ba=Array.isArray,ls=Object.prototype.hasOwnProperty,tu={current:null},is={key:!0,ref:!0,__self:!0,__source:!0};function us(e,t,n){var r,l={},i=null,u=null;if(t!=null)for(r in t.ref!==void 0&&(u=t.ref),t.key!==void 0&&(i=""+t.key),t)ls.call(t,r)&&!is.hasOwnProperty(r)&&(l[r]=t[r]);var o=arguments.length-2;if(o===1)l.children=n;else if(1<o){for(var a=Array(o),s=0;s<o;s++)a[s]=arguments[s+2];l.children=a}if(e&&e.defaultProps)for(r in o=e.defaultProps,o)l[r]===void 0&&(l[r]=o[r]);return{$$typeof:qn,type:e,key:i,ref:u,props:l,_owner:tu.current}}function Cp(e,t){return{$$typeof:qn,type:e.type,key:t,ref:e.ref,props:e.props,_owner:e._owner}}function nu(e){return typeof e==="object"&&e!==null&&e.$$typeof===qn}function Fp(e){var t={"=":"=0",":":"=2"};return"$"+e.replace(/[=:]/g,function(n){return t[n]})}var ja=/\/+/g;function bi(e,t){return typeof e==="object"&&e!==null&&e.key!=null?Fp(""+e.key):t.toString(36)}function pl(e,t,n,r,l){var i=typeof e;if(i==="undefined"||i==="boolean")e=null;var u=!1;if(e===null)u=!0;else switch(i){case"string":case"number":u=!0;break;case"object":switch(e.$$typeof){case qn:case hp:u=!0}}if(u)return u=e,l=l(u),e=r===""?"."+bi(u,0):r,ba(l)?(n="",e!=null&&(n=e.replace(ja,"$&/")+"/"),pl(l,t,n,"",function(s){return s})):l!=null&&(nu(l)&&(l=Cp(l,n+(!l.key||u&&u.key===l.key?"":(""+l.key).replace(ja,"$&/")+"/")+e)),t.push(l)),1;if(u=0,r=r===""?".":r+":",ba(e))for(var o=0;o<e.length;o++){i=e[o];var a=r+bi(i,o);u+=pl(i,t,n,a,l)}else if(a=Ep(e),typeof a==="function")for(e=a.call(e),o=0;!(i=e.next()).done;)i=i.value,a=r+bi(i,o++),u+=pl(i,t,n,a,l);else if(i==="object")throw t=String(e),Error("Objects are not valid as a React child (found: "+(t==="[object Object]"?"object with keys {"+Object.keys(e).join(", ")+"}":t)+"). If you meant to render a collection of children, use an array instead.");return u}function dl(e,t,n){if(e==null)return e;var r=[],l=0;return pl(e,r,"","",function(i){return t.call(n,i,l++)}),r}function zp(e){if(e._status===-1){var t=e._result;t=t(),t.then(function(n){if(e._status===0||e._status===-1)e._status=1,e._result=n},function(n){if(e._status===0||e._status===-1)e._status=2,e._result=n}),e._status===-1&&(e._status=0,e._result=t)}if(e._status===1)return e._result.default;throw e._result}var he={current:null},hl={transition:null},Lp={ReactCurrentDispatcher:he,ReactCurrentBatchConfig:hl,ReactCurrentOwner:tu};function os(){throw Error("act(...) is not supported in production builds of React.")}Tp.Children={map:dl,forEach:function(e,t,n){dl(e,function(){t.apply(this,arguments)},n)},count:function(e){var t=0;return dl(e,function(){t++}),t},toArray:function(e){return dl(e,function(t){return t})||[]},only:function(e){if(!nu(e))throw Error("React.Children.only expected to receive a single React element child.");return e}};Tp.Component=dn;Tp.Fragment=mp;Tp.Profiler=gp;Tp.PureComponent=ji;Tp.StrictMode=yp;Tp.Suspense=xp;Tp.__SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED=Lp;Tp.act=os;Tp.cloneElement=function(e,t,n){if(e===null||e===void 0)throw Error("React.cloneElement(...): The argument must be a React element, but you passed "+e+".");var r=ts({},e.props),l=e.key,i=e.ref,u=e._owner;if(t!=null){if(t.ref!==void 0&&(i=t.ref,u=tu.current),t.key!==void 0&&(l=""+t.key),e.type&&e.type.defaultProps)var o=e.type.defaultProps;for(a in t)ls.call(t,a)&&!is.hasOwnProperty(a)&&(r[a]=t[a]===void 0&&o!==void 0?o[a]:t[a])}var a=arguments.length-2;if(a===1)r.children=n;else if(1<a){o=Array(a);for(var s=0;s<a;s++)o[s]=arguments[s+2];r.children=o}return{$$typeof:qn,type:e.type,key:l,ref:i,props:r,_owner:u}};Tp.createContext=function(e){return e={$$typeof:wp,_currentValue:e,_currentValue2:e,_threadCount:0,Provider:null,Consumer:null,_defaultValue:null,_globalName:null},e.Provider={$$typeof:vp,_context:e},e.Consumer=e};Tp.createElement=us;Tp.createFactory=function(e){var t=us.bind(null,e);return t.type=e,t};Tp.createRef=function(){return{current:null}};Tp.forwardRef=function(e){return{$$typeof:kp,render:e}};Tp.isValidElement=nu;Tp.lazy=function(e){return{$$typeof:Np,_payload:{_status:-1,_result:e},_init:zp}};Tp.memo=function(e,t){return{$$typeof:Sp,type:e,compare:t===void 0?null:t}};Tp.startTransition=function(e){var t=hl.transition;hl.transition={};try{e()}finally{hl.transition=t}};Tp.unstable_act=os;Tp.useCallback=function(e,t){return he.current.useCallback(e,t)};Tp.useContext=function(e){return he.current.useContext(e)};Tp.useDebugValue=function(){};Tp.useDeferredValue=function(e){return he.current.useDeferredValue(e)};Tp.useEffect=function(e,t){return he.current.useEffect(e,t)};Tp.useId=function(){return he.current.useId()};Tp.useImperativeHandle=function(e,t,n){return he.current.useImperativeHandle(e,t,n)};Tp.useInsertionEffect=function(e,t){return he.current.useInsertionEffect(e,t)};Tp.useLayoutEffect=function(e,t){return he.current.useLayoutEffect(e,t)};Tp.useMemo=function(e,t){return he.current.useMemo(e,t)};Tp.useReducer=function(e,t,n){return he.current.useReducer(e,t,n)};Tp.useRef=function(e){return he.current.useRef(e)};Tp.useState=function(e){return he.current.useState(e)};Tp.useSyncExternalStore=function(e,t,n){return he.current.useSyncExternalStore(e,t,n)};Tp.useTransition=function(){return he.current.useTransition()};Tp.version="18.3.1"});var hs=cl((ph)=>{function lu(e,t){var n=e.length;e.push(t);e:for(;0<n;){var r=n-1>>>1,l=e[r];if(0<ml(l,t))e[r]=t,e[n]=l,n=r;else break e}}function Ae(e){return e.length===0?null:e[0]}function wl(e){if(e.length===0)return null;var t=e[0],n=e.pop();if(n!==t){e[0]=n;e:for(var r=0,l=e.length,i=l>>>1;r<i;){var u=2*(r+1)-1,o=e[u],a=u+1,s=e[a];if(0>ml(o,n))a<l&&0>ml(s,o)?(e[r]=s,e[a]=n,r=a):(e[r]=o,e[u]=n,r=u);else if(a<l&&0>ml(s,n))e[r]=s,e[a]=n,r=a;else break e}}return t}function ml(e,t){var n=e.sortIndex-t.sortIndex;return n!==0?n:e.id-t.id}if(typeof performance==="object"&&typeof performance.now==="function")iu=performance,ph.unstable_now=function(){return iu.now()};else yl=Date,uu=yl.now(),ph.unstable_now=function(){return yl.now()-uu};var iu,yl,uu,qe=[],yt=[],dh=1,Be=null,ae=3,kl=!1,Rt=!1,jn=!1,ss=typeof setTimeout==="function"?setTimeout:null,fs=typeof clearTimeout==="function"?clearTimeout:null,as=typeof setImmediate<"u"?setImmediate:null;typeof navigator<"u"&&navigator.scheduling!==void 0&&navigator.scheduling.isInputPending!==void 0&&navigator.scheduling.isInputPending.bind(navigator.scheduling);function ou(e){for(var t=Ae(yt);t!==null;){if(t.callback===null)wl(yt);else if(t.startTime<=e)wl(yt),t.sortIndex=t.expirationTime,lu(qe,t);else break;t=Ae(yt)}}function su(e){if(jn=!1,ou(e),!Rt)if(Ae(qe)!==null)Rt=!0,cu(fu);else{var t=Ae(yt);t!==null&&du(su,t.startTime-e)}}function fu(e,t){Rt=!1,jn&&(jn=!1,fs(er),er=-1),kl=!0;var n=ae;try{ou(t);for(Be=Ae(qe);Be!==null&&(!(Be.expirationTime>t)||e&&!ps());){var r=Be.callback;if(typeof r==="function"){Be.callback=null,ae=Be.priorityLevel;var l=r(Be.expirationTime<=t);t=ph.unstable_now(),typeof l==="function"?Be.callback=l:Be===Ae(qe)&&wl(qe),ou(t)}else wl(qe);Be=Ae(qe)}if(Be!==null)var i=!0;else{var u=Ae(yt);u!==null&&du(su,u.startTime-t),i=!1}return i}finally{Be=null,ae=n,kl=!1}}var xl=!1,gl=null,er=-1,cs=5,ds=-1;function ps(){return ph.unstable_now()-ds<cs?!1:!0}function ru(){if(gl!==null){var e=ph.unstable_now();ds=e;var t=!0;try{t=gl(!0,e)}finally{t?bn():(xl=!1,gl=null)}}else xl=!1}var bn;if(typeof as==="function")bn=function(){as(ru)};else if(typeof MessageChannel<"u")vl=new MessageChannel,au=vl.port2,vl.port1.onmessage=ru,bn=function(){au.postMessage(null)};else bn=function(){ss(ru,0)};var vl,au;function cu(e){gl=e,xl||(xl=!0,bn())}function du(e,t){er=ss(function(){e(ph.unstable_now())},t)}ph.unstable_IdlePriority=5;ph.unstable_ImmediatePriority=1;ph.unstable_LowPriority=4;ph.unstable_NormalPriority=3;ph.unstable_Profiling=null;ph.unstable_UserBlockingPriority=2;ph.unstable_cancelCallback=function(e){e.callback=null};ph.unstable_continueExecution=function(){Rt||kl||(Rt=!0,cu(fu))};ph.unstable_forceFrameRate=function(e){0>e||125<e?console.error("forceFrameRate takes a positive int between 0 and 125, forcing frame rates higher than 125 fps is not supported"):cs=0<e?Math.floor(1000/e):5};ph.unstable_getCurrentPriorityLevel=function(){return ae};ph.unstable_getFirstCallbackNode=function(){return Ae(qe)};ph.unstable_next=function(e){switch(ae){case 1:case 2:case 3:var t=3;break;default:t=ae}var n=ae;ae=t;try{return e()}finally{ae=n}};ph.unstable_pauseExecution=function(){};ph.unstable_requestPaint=function(){};ph.unstable_runWithPriority=function(e,t){switch(e){case 1:case 2:case 3:case 4:case 5:break;default:e=3}var n=ae;ae=e;try{return t()}finally{ae=n}};ph.unstable_scheduleCallback=function(e,t,n){var r=ph.unstable_now();switch(typeof n==="object"&&n!==null?(n=n.delay,n=typeof n==="number"&&0<n?r+n:r):n=r,e){case 1:var l=-1;break;case 2:l=250;break;case 5:l=1073741823;break;case 4:l=1e4;break;default:l=5000}return l=n+l,e={id:dh++,callback:t,priorityLevel:e,startTime:n,expirationTime:l,sortIndex:-1},n>r?(e.sortIndex=n,lu(yt,e),Ae(qe)===null&&e===Ae(yt)&&(jn?(fs(er),er=-1):jn=!0,du(su,n-r))):(e.sortIndex=l,lu(qe,e),Rt||kl||(Rt=!0,cu(fu))),e};ph.unstable_shouldYield=ps;ph.unstable_wrapCallback=function(e){var t=ae;return function(){var n=ae;ae=t;try{return e.apply(this,arguments)}finally{ae=n}}}});var wa={};dp(wa,{version:()=>Td,unstable_renderSubtreeIntoContainer:()=>Ld,unstable_batchedUpdates:()=>zd,unmountComponentAtNode:()=>Fd,render:()=>Cd,hydrateRoot:()=>Ed,hydrate:()=>Nd,flushSync:()=>Sd,findDOMNode:()=>xd,createRoot:()=>kd,createPortal:()=>wd,__SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED:()=>vd});function S(e){for(var t="https://reactjs.org/docs/error-decoder.html?invariant="+e,n=1;n<arguments.length;n++)t+="&args[]="+encodeURIComponent(arguments[n]);return"Minified React error #"+e+"; visit "+t+" for the full message or use the non-minified dev environment for full errors and additional helpful warnings."}function rn(e,t){Dn(e,t),Dn(e+"Capture",t)}function Dn(e,t){Cr[e]=t;for(e=0;e<t.length;e++)xf.add(t[e])}function $h(e){if(Du.call(ys,e))return!0;if(Du.call(ms,e))return!1;if(Mh.test(e))return ys[e]=!0;return ms[e]=!0,!1}function Dh(e,t,n,r){if(n!==null&&n.type===0)return!1;switch(typeof t){case"function":case"symbol":return!0;case"boolean":if(r)return!1;if(n!==null)return!n.acceptsBooleans;return e=e.toLowerCase().slice(0,5),e!=="data-"&&e!=="aria-";default:return!1}}function _h(e,t,n,r){if(t===null||typeof t>"u"||Dh(e,t,n,r))return!0;if(r)return!1;if(n!==null)switch(n.type){case 3:return!t;case 4:return t===!1;case 5:return isNaN(t);case 6:return isNaN(t)||1>t}return!1}function ge(e,t,n,r,l,i,u){this.acceptsBooleans=t===2||t===3||t===4,this.attributeName=r,this.attributeNamespace=l,this.mustUseProperty=n,this.propertyName=e,this.type=t,this.sanitizeURL=i,this.removeEmptyString=u}function Po(e){return e[1].toUpperCase()}function Bo(e,t,n,r){var l=ue.hasOwnProperty(t)?ue[t]:null;if(l!==null?l.type!==0:r||!(2<t.length)||t[0]!=="o"&&t[0]!=="O"||t[1]!=="n"&&t[1]!=="N")_h(t,n,l,r)&&(n=null),r||l===null?$h(t)&&(n===null?e.removeAttribute(t):e.setAttribute(t,""+n)):l.mustUseProperty?e[l.propertyName]=n===null?l.type===3?!1:"":n:(t=l.attributeName,r=l.attributeNamespace,n===null?e.removeAttribute(t):(l=l.type,n=l===3||l===4&&n===!0?"":""+n,r?e.setAttributeNS(r,t,n):e.setAttribute(t,n)))}function tr(e){if(e===null||typeof e!=="object")return null;return e=gs&&e[gs]||e["@@iterator"],typeof e==="function"?e:null}function ar(e){if(pu===void 0)try{throw Error()}catch(n){var t=n.stack.trim().match(/\n( *(at )?)/);pu=t&&t[1]||""}return`
`+pu+e}function mu(e,t){if(!e||hu)return"";hu=!0;var n=Error.prepareStackTrace;Error.prepareStackTrace=void 0;try{if(t)if(t=function(){throw Error()},Object.defineProperty(t.prototype,"props",{set:function(){throw Error()}}),typeof Reflect==="object"&&Reflect.construct){try{Reflect.construct(t,[])}catch(s){var r=s}Reflect.construct(e,[],t)}else{try{t.call()}catch(s){r=s}e.call(t.prototype)}else{try{throw Error()}catch(s){r=s}e()}}catch(s){if(s&&r&&typeof s.stack==="string"){for(var l=s.stack.split(`
`),i=r.stack.split(`
`),u=l.length-1,o=i.length-1;1<=u&&0<=o&&l[u]!==i[o];)o--;for(;1<=u&&0<=o;u--,o--)if(l[u]!==i[o]){if(u!==1||o!==1)do if(u--,o--,0>o||l[u]!==i[o]){var a=`
`+l[u].replace(" at new "," at ");return e.displayName&&a.includes("<anonymous>")&&(a=a.replace("<anonymous>",e.displayName)),a}while(1<=u&&0<=o);break}}}finally{hu=!1,Error.prepareStackTrace=n}return(e=e?e.displayName||e.name:"")?ar(e):""}function Oh(e){switch(e.tag){case 5:return ar(e.type);case 16:return ar("Lazy");case 13:return ar("Suspense");case 19:return ar("SuspenseList");case 0:case 2:case 15:return e=mu(e.type,!1),e;case 11:return e=mu(e.type.render,!1),e;case 1:return e=mu(e.type,!0),e;default:return""}}function Hu(e){if(e==null)return null;if(typeof e==="function")return e.displayName||e.name||null;if(typeof e==="string")return e;switch(e){case gn:return"Fragment";case yn:return"Portal";case _u:return"Profiler";case Mo:return"StrictMode";case Ou:return"Suspense";case Uu:return"SuspenseList"}if(typeof e==="object")switch(e.$$typeof){case Nf:return(e.displayName||"Context")+".Consumer";case Sf:return(e._context.displayName||"Context")+".Provider";case $o:var t=e.render;return e=e.displayName,e||(e=t.displayName||t.name||"",e=e!==""?"ForwardRef("+e+")":"ForwardRef"),e;case Do:return t=e.displayName||null,t!==null?t:Hu(e.type)||"Memo";case vt:t=e._payload,e=e._init;try{return Hu(e(t))}catch(n){}}return null}function Uh(e){var t=e.type;switch(e.tag){case 24:return"Cache";case 9:return(t.displayName||"Context")+".Consumer";case 10:return(t._context.displayName||"Context")+".Provider";case 18:return"DehydratedFragment";case 11:return e=t.render,e=e.displayName||e.name||"",t.displayName||(e!==""?"ForwardRef("+e+")":"ForwardRef");case 7:return"Fragment";case 5:return t;case 4:return"Portal";case 3:return"Root";case 6:return"Text";case 16:return Hu(t);case 8:return t===Mo?"StrictMode":"Mode";case 22:return"Offscreen";case 12:return"Profiler";case 21:return"Scope";case 13:return"Suspense";case 19:return"SuspenseList";case 25:return"TracingMarker";case 1:case 0:case 17:case 2:case 14:case 15:if(typeof t==="function")return t.displayName||t.name||null;if(typeof t==="string")return t}return null}function Mt(e){switch(typeof e){case"boolean":case"number":case"string":case"undefined":return e;case"object":return e;default:return""}}function Cf(e){var t=e.type;return(e=e.nodeName)&&e.toLowerCase()==="input"&&(t==="checkbox"||t==="radio")}function Hh(e){var t=Cf(e)?"checked":"value",n=Object.getOwnPropertyDescriptor(e.constructor.prototype,t),r=""+e[t];if(!e.hasOwnProperty(t)&&typeof n<"u"&&typeof n.get==="function"&&typeof n.set==="function"){var{get:l,set:i}=n;return Object.defineProperty(e,t,{configurable:!0,get:function(){return l.call(this)},set:function(u){r=""+u,i.call(this,u)}}),Object.defineProperty(e,t,{enumerable:n.enumerable}),{getValue:function(){return r},setValue:function(u){r=""+u},stopTracking:function(){e._valueTracker=null,delete e[t]}}}}function Nl(e){e._valueTracker||(e._valueTracker=Hh(e))}function Ff(e){if(!e)return!1;var t=e._valueTracker;if(!t)return!0;var n=t.getValue(),r="";return e&&(r=Cf(e)?e.checked?"true":"false":e.value),e=r,e!==n?(t.setValue(e),!0):!1}function Zl(e){if(e=e||(typeof document<"u"?document:void 0),typeof e>"u")return null;try{return e.activeElement||e.body}catch(t){return e.body}}function Qu(e,t){var n=t.checked;return Z({},t,{defaultChecked:void 0,defaultValue:void 0,value:void 0,checked:n!=null?n:e._wrapperState.initialChecked})}function vs(e,t){var n=t.defaultValue==null?"":t.defaultValue,r=t.checked!=null?t.checked:t.defaultChecked;n=Mt(t.value!=null?t.value:n),e._wrapperState={initialChecked:r,initialValue:n,controlled:t.type==="checkbox"||t.type==="radio"?t.checked!=null:t.value!=null}}function zf(e,t){t=t.checked,t!=null&&Bo(e,"checked",t,!1)}function Wu(e,t){zf(e,t);var n=Mt(t.value),r=t.type;if(n!=null)if(r==="number"){if(n===0&&e.value===""||e.value!=n)e.value=""+n}else e.value!==""+n&&(e.value=""+n);else if(r==="submit"||r==="reset"){e.removeAttribute("value");return}t.hasOwnProperty("value")?Au(e,t.type,n):t.hasOwnProperty("defaultValue")&&Au(e,t.type,Mt(t.defaultValue)),t.checked==null&&t.defaultChecked!=null&&(e.defaultChecked=!!t.defaultChecked)}function ws(e,t,n){if(t.hasOwnProperty("value")||t.hasOwnProperty("defaultValue")){var r=t.type;if(!(r!=="submit"&&r!=="reset"||t.value!==void 0&&t.value!==null))return;t=""+e._wrapperState.initialValue,n||t===e.value||(e.value=t),e.defaultValue=t}n=e.name,n!==""&&(e.name=""),e.defaultChecked=!!e._wrapperState.initialChecked,n!==""&&(e.name=n)}function Au(e,t,n){if(t!=="number"||Zl(e.ownerDocument)!==e)n==null?e.defaultValue=""+e._wrapperState.initialValue:e.defaultValue!==""+n&&(e.defaultValue=""+n)}function Ln(e,t,n,r){if(e=e.options,t){t={};for(var l=0;l<n.length;l++)t["$"+n[l]]=!0;for(n=0;n<e.length;n++)l=t.hasOwnProperty("$"+e[n].value),e[n].selected!==l&&(e[n].selected=l),l&&r&&(e[n].defaultSelected=!0)}else{n=""+Mt(n),t=null;for(l=0;l<e.length;l++){if(e[l].value===n){e[l].selected=!0,r&&(e[l].defaultSelected=!0);return}t!==null||e[l].disabled||(t=e[l])}t!==null&&(t.selected=!0)}}function Iu(e,t){if(t.dangerouslySetInnerHTML!=null)throw Error(S(91));return Z({},t,{value:void 0,defaultValue:void 0,children:""+e._wrapperState.initialValue})}function ks(e,t){var n=t.value;if(n==null){if(n=t.children,t=t.defaultValue,n!=null){if(t!=null)throw Error(S(92));if(sr(n)){if(1<n.length)throw Error(S(93));n=n[0]}t=n}t==null&&(t=""),n=t}e._wrapperState={initialValue:Mt(n)}}function Lf(e,t){var n=Mt(t.value),r=Mt(t.defaultValue);n!=null&&(n=""+n,n!==e.value&&(e.value=n),t.defaultValue==null&&e.defaultValue!==n&&(e.defaultValue=n)),r!=null&&(e.defaultValue=""+r)}function xs(e){var t=e.textContent;t===e._wrapperState.initialValue&&t!==""&&t!==null&&(e.value=t)}function Tf(e){switch(e){case"svg":return"http://www.w3.org/2000/svg";case"math":return"http://www.w3.org/1998/Math/MathML";default:return"http://www.w3.org/1999/xhtml"}}function Ru(e,t){return e==null||e==="http://www.w3.org/1999/xhtml"?Tf(t):e==="http://www.w3.org/2000/svg"&&t==="foreignObject"?"http://www.w3.org/1999/xhtml":e}function Fr(e,t){if(t){var n=e.firstChild;if(n&&n===e.lastChild&&n.nodeType===3){n.nodeValue=t;return}}e.textContent=t}function Bf(e,t,n){return t==null||typeof t==="boolean"||t===""?"":n||typeof t!=="number"||t===0||yr.hasOwnProperty(e)&&yr[e]?(""+t).trim():t+"px"}function Mf(e,t){e=e.style;for(var n in t)if(t.hasOwnProperty(n)){var r=n.indexOf("--")===0,l=Bf(n,t[n],r);n==="float"&&(n="cssFloat"),r?e.setProperty(n,l):e[n]=l}}function Vu(e,t){if(t){if(Wh[e]&&(t.children!=null||t.dangerouslySetInnerHTML!=null))throw Error(S(137,e));if(t.dangerouslySetInnerHTML!=null){if(t.children!=null)throw Error(S(60));if(typeof t.dangerouslySetInnerHTML!=="object"||!("__html"in t.dangerouslySetInnerHTML))throw Error(S(61))}if(t.style!=null&&typeof t.style!=="object")throw Error(S(62))}}function Yu(e,t){if(e.indexOf("-")===-1)return typeof t.is==="string";switch(e){case"annotation-xml":case"color-profile":case"font-face":case"font-face-src":case"font-face-uri":case"font-face-format":case"font-face-name":case"missing-glyph":return!1;default:return!0}}function _o(e){return e=e.target||e.srcElement||window,e.correspondingUseElement&&(e=e.correspondingUseElement),e.nodeType===3?e.parentNode:e}function Ss(e){if(e=Vr(e)){if(typeof Xu!=="function")throw Error(S(280));var t=e.stateNode;t&&(t=Si(t),Xu(e.stateNode,e.type,t))}}function $f(e){Tn?Pn?Pn.push(e):Pn=[e]:Tn=e}function Df(){if(Tn){var e=Tn,t=Pn;if(Pn=Tn=null,Ss(e),t)for(e=0;e<t.length;e++)Ss(t[e])}}function _f(e,t){return e(t)}function Of(){}function Uf(e,t,n){if(yu)return e(t,n);yu=!0;try{return _f(e,t,n)}finally{if(yu=!1,Tn!==null||Pn!==null)Of(),Df()}}function zr(e,t){var n=e.stateNode;if(n===null)return null;var r=Si(n);if(r===null)return null;n=r[t];e:switch(t){case"onClick":case"onClickCapture":case"onDoubleClick":case"onDoubleClickCapture":case"onMouseDown":case"onMouseDownCapture":case"onMouseMove":case"onMouseMoveCapture":case"onMouseUp":case"onMouseUpCapture":case"onMouseEnter":(r=!r.disabled)||(e=e.type,r=!(e==="button"||e==="input"||e==="select"||e==="textarea")),e=!r;break e;default:e=!1}if(e)return null;if(n&&typeof n!=="function")throw Error(S(231,t,typeof n));return n}function Ah(e,t,n,r,l,i,u,o,a){var s=Array.prototype.slice.call(arguments,3);try{t.apply(n,s)}catch(h){this.onError(h)}}function Rh(e,t,n,r,l,i,u,o,a){gr=!1,Jl=null,Ah.apply(Ih,arguments)}function Vh(e,t,n,r,l,i,u,o,a){if(Rh.apply(this,arguments),gr){if(gr){var s=Jl;gr=!1,Jl=null}else throw Error(S(198));Gl||(Gl=!0,Ju=s)}}function ln(e){var t=e,n=e;if(e.alternate)for(;t.return;)t=t.return;else{e=t;do t=e,(t.flags&4098)!==0&&(n=t.return),e=t.return;while(e)}return t.tag===3?n:null}function Hf(e){if(e.tag===13){var t=e.memoizedState;if(t===null&&(e=e.alternate,e!==null&&(t=e.memoizedState)),t!==null)return t.dehydrated}return null}function Ns(e){if(ln(e)!==e)throw Error(S(188))}function Yh(e){var t=e.alternate;if(!t){if(t=ln(e),t===null)throw Error(S(188));return t!==e?null:e}for(var n=e,r=t;;){var l=n.return;if(l===null)break;var i=l.alternate;if(i===null){if(r=l.return,r!==null){n=r;continue}break}if(l.child===i.child){for(i=l.child;i;){if(i===n)return Ns(l),e;if(i===r)return Ns(l),t;i=i.sibling}throw Error(S(188))}if(n.return!==r.return)n=l,r=i;else{for(var u=!1,o=l.child;o;){if(o===n){u=!0,n=l,r=i;break}if(o===r){u=!0,r=l,n=i;break}o=o.sibling}if(!u){for(o=i.child;o;){if(o===n){u=!0,n=i,r=l;break}if(o===r){u=!0,r=i,n=l;break}o=o.sibling}if(!u)throw Error(S(189))}}if(n.alternate!==r)throw Error(S(190))}if(n.tag!==3)throw Error(S(188));return n.stateNode.current===n?e:t}function Qf(e){return e=Yh(e),e!==null?Wf(e):null}function Wf(e){if(e.tag===5||e.tag===6)return e;for(e=e.child;e!==null;){var t=Wf(e);if(t!==null)return t;e=e.sibling}return null}function Gh(e){if(tt&&typeof tt.onCommitFiberRoot==="function")try{tt.onCommitFiberRoot(vi,e,void 0,(e.current.flags&128)===128)}catch(t){}}function jh(e){return e>>>=0,e===0?32:31-(qh(e)/bh|0)|0}function fr(e){switch(e&-e){case 1:return 1;case 2:return 2;case 4:return 4;case 8:return 8;case 16:return 16;case 32:return 32;case 64:case 128:case 256:case 512:case 1024:case 2048:case 4096:case 8192:case 16384:case 32768:case 65536:case 131072:case 262144:case 524288:case 1048576:case 2097152:return e&4194240;case 4194304:case 8388608:case 16777216:case 33554432:case 67108864:return e&130023424;case 134217728:return 134217728;case 268435456:return 268435456;case 536870912:return 536870912;case 1073741824:return 1073741824;default:return e}}function bl(e,t){var n=e.pendingLanes;if(n===0)return 0;var r=0,l=e.suspendedLanes,i=e.pingedLanes,u=n&268435455;if(u!==0){var o=u&~l;o!==0?r=fr(o):(i&=u,i!==0&&(r=fr(i)))}else u=n&~l,u!==0?r=fr(u):i!==0&&(r=fr(i));if(r===0)return 0;if(t!==0&&t!==r&&(t&l)===0&&(l=r&-r,i=t&-t,l>=i||l===16&&(i&4194240)!==0))return t;if((r&4)!==0&&(r|=n&16),t=e.entangledLanes,t!==0)for(e=e.entanglements,t&=r;0<t;)n=31-Ke(t),l=1<<n,r|=e[n],t&=~l;return r}function em(e,t){switch(e){case 1:case 2:case 4:return t+250;case 8:case 16:case 32:case 64:case 128:case 256:case 512:case 1024:case 2048:case 4096:case 8192:case 16384:case 32768:case 65536:case 131072:case 262144:case 524288:case 1048576:case 2097152:return t+5000;case 4194304:case 8388608:case 16777216:case 33554432:case 67108864:return-1;case 134217728:case 268435456:case 536870912:case 1073741824:return-1;default:return-1}}function tm(e,t){for(var{suspendedLanes:n,pingedLanes:r,expirationTimes:l,pendingLanes:i}=e;0<i;){var u=31-Ke(i),o=1<<u,a=l[u];if(a===-1){if((o&n)===0||(o&r)!==0)l[u]=em(o,t)}else a<=t&&(e.expiredLanes|=o);i&=~o}}function Gu(e){return e=e.pendingLanes&-1073741825,e!==0?e:e&1073741824?1073741824:0}function Vf(){var e=Cl;return Cl<<=1,(Cl&4194240)===0&&(Cl=64),e}function gu(e){for(var t=[],n=0;31>n;n++)t.push(e);return t}function Ir(e,t,n){e.pendingLanes|=t,t!==536870912&&(e.suspendedLanes=0,e.pingedLanes=0),e=e.eventTimes,t=31-Ke(t),e[t]=n}function nm(e,t){var n=e.pendingLanes&~t;e.pendingLanes=t,e.suspendedLanes=0,e.pingedLanes=0,e.expiredLanes&=t,e.mutableReadLanes&=t,e.entangledLanes&=t,t=e.entanglements;var r=e.eventTimes;for(e=e.expirationTimes;0<n;){var l=31-Ke(n),i=1<<l;t[l]=0,r[l]=-1,e[l]=-1,n&=~i}}function Uo(e,t){var n=e.entangledLanes|=t;for(e=e.entanglements;n;){var r=31-Ke(n),l=1<<r;l&t|e[r]&t&&(e[r]|=t),n&=~l}}function Yf(e){return e&=-e,1<e?4<e?(e&268435455)!==0?16:536870912:4:1}function Cs(e,t){switch(e){case"focusin":case"focusout":Et=null;break;case"dragenter":case"dragleave":Ct=null;break;case"mouseover":case"mouseout":Ft=null;break;case"pointerover":case"pointerout":Lr.delete(t.pointerId);break;case"gotpointercapture":case"lostpointercapture":Tr.delete(t.pointerId)}}function nr(e,t,n,r,l,i){if(e===null||e.nativeEvent!==i)return e={blockedOn:t,domEventName:n,eventSystemFlags:r,nativeEvent:i,targetContainers:[l]},t!==null&&(t=Vr(t),t!==null&&Ho(t)),e;return e.eventSystemFlags|=r,t=e.targetContainers,l!==null&&t.indexOf(l)===-1&&t.push(l),e}function lm(e,t,n,r,l){switch(t){case"focusin":return Et=nr(Et,e,t,n,r,l),!0;case"dragenter":return Ct=nr(Ct,e,t,n,r,l),!0;case"mouseover":return Ft=nr(Ft,e,t,n,r,l),!0;case"pointerover":var i=l.pointerId;return Lr.set(i,nr(Lr.get(i)||null,e,t,n,r,l)),!0;case"gotpointercapture":return i=l.pointerId,Tr.set(i,nr(Tr.get(i)||null,e,t,n,r,l)),!0}return!1}function Gf(e){var t=Xt(e.target);if(t!==null){var n=ln(t);if(n!==null){if(t=n.tag,t===13){if(t=Hf(n),t!==null){e.blockedOn=t,Jf(e.priority,function(){Xf(n)});return}}else if(t===3&&n.stateNode.current.memoizedState.isDehydrated){e.blockedOn=n.tag===3?n.stateNode.containerInfo:null;return}}}e.blockedOn=null}function Ul(e){if(e.blockedOn!==null)return!1;for(var t=e.targetContainers;0<t.length;){var n=bu(e.domEventName,e.eventSystemFlags,t[0],e.nativeEvent);if(n===null){n=e.nativeEvent;var r=new n.constructor(n.type,n);Ku=r,n.target.dispatchEvent(r),Ku=null}else return t=Vr(n),t!==null&&Ho(t),e.blockedOn=n,!1;t.shift()}return!0}function Fs(e,t,n){Ul(e)&&n.delete(t)}function im(){qu=!1,Et!==null&&Ul(Et)&&(Et=null),Ct!==null&&Ul(Ct)&&(Ct=null),Ft!==null&&Ul(Ft)&&(Ft=null),Lr.forEach(Fs),Tr.forEach(Fs)}function rr(e,t){e.blockedOn===t&&(e.blockedOn=null,qu||(qu=!0,Y.unstable_scheduleCallback(Y.unstable_NormalPriority,im)))}function Pr(e){function t(l){return rr(l,e)}if(0<zl.length){rr(zl[0],e);for(var n=1;n<zl.length;n++){var r=zl[n];r.blockedOn===e&&(r.blockedOn=null)}}Et!==null&&rr(Et,e),Ct!==null&&rr(Ct,e),Ft!==null&&rr(Ft,e),Lr.forEach(t),Tr.forEach(t);for(n=0;n<kt.length;n++)r=kt[n],r.blockedOn===e&&(r.blockedOn=null);for(;0<kt.length&&(n=kt[0],n.blockedOn===null);)Gf(n),n.blockedOn===null&&kt.shift()}function um(e,t,n,r){var l=_,i=Bn.transition;Bn.transition=null;try{_=1,Qo(e,t,n,r)}finally{_=l,Bn.transition=i}}function om(e,t,n,r){var l=_,i=Bn.transition;Bn.transition=null;try{_=4,Qo(e,t,n,r)}finally{_=l,Bn.transition=i}}function Qo(e,t,n,r){if(jl){var l=bu(e,t,n,r);if(l===null)Nu(e,t,r,ei,n),Cs(e,r);else if(lm(l,e,t,n,r))r.stopPropagation();else if(Cs(e,r),t&4&&-1<rm.indexOf(e)){for(;l!==null;){var i=Vr(l);if(i!==null&&Kf(i),i=bu(e,t,n,r),i===null&&Nu(e,t,r,ei,n),i===l)break;l=i}l!==null&&r.stopPropagation()}else Nu(e,t,r,null,n)}}function bu(e,t,n,r){if(ei=null,e=_o(r),e=Xt(e),e!==null)if(t=ln(e),t===null)e=null;else if(n=t.tag,n===13){if(e=Hf(t),e!==null)return e;e=null}else if(n===3){if(t.stateNode.current.memoizedState.isDehydrated)return t.tag===3?t.stateNode.containerInfo:null;e=null}else t!==e&&(e=null);return ei=e,null}function qf(e){switch(e){case"cancel":case"click":case"close":case"contextmenu":case"copy":case"cut":case"auxclick":case"dblclick":case"dragend":case"dragstart":case"drop":case"focusin":case"focusout":case"input":case"invalid":case"keydown":case"keypress":case"keyup":case"mousedown":case"mouseup":case"paste":case"pause":case"play":case"pointercancel":case"pointerdown":case"pointerup":case"ratechange":case"reset":case"resize":case"seeked":case"submit":case"touchcancel":case"touchend":case"touchstart":case"volumechange":case"change":case"selectionchange":case"textInput":case"compositionstart":case"compositionend":case"compositionupdate":case"beforeblur":case"afterblur":case"beforeinput":case"blur":case"fullscreenchange":case"focus":case"hashchange":case"popstate":case"select":case"selectstart":return 1;case"drag":case"dragenter":case"dragexit":case"dragleave":case"dragover":case"mousemove":case"mouseout":case"mouseover":case"pointermove":case"pointerout":case"pointerover":case"scroll":case"toggle":case"touchmove":case"wheel":case"mouseenter":case"mouseleave":case"pointerenter":case"pointerleave":return 4;case"message":switch(Zh()){case Oo:return 1;case If:return 4;case ql:case Jh:return 16;case Rf:return 536870912;default:return 16}default:return 16}}function bf(){if(Hl)return Hl;var e,t=Wo,n=t.length,r,l="value"in St?St.value:St.textContent,i=l.length;for(e=0;e<n&&t[e]===l[e];e++);var u=n-e;for(r=1;r<=u&&t[n-r]===l[i-r];r++);return Hl=l.slice(e,1<r?1-r:void 0)}function Ql(e){var t=e.keyCode;return"charCode"in e?(e=e.charCode,e===0&&t===13&&(e=13)):e=t,e===10&&(e=13),32<=e||e===13?e:0}function Ll(){return!0}function zs(){return!1}function Te(e){function t(n,r,l,i,u){this._reactName=n,this._targetInst=l,this.type=r,this.nativeEvent=i,this.target=u,this.currentTarget=null;for(var o in e)e.hasOwnProperty(o)&&(n=e[o],this[o]=n?n(i):i[o]);return this.isDefaultPrevented=(i.defaultPrevented!=null?i.defaultPrevented:i.returnValue===!1)?Ll:zs,this.isPropagationStopped=zs,this}return Z(t.prototype,{preventDefault:function(){this.defaultPrevented=!0;var n=this.nativeEvent;n&&(n.preventDefault?n.preventDefault():typeof n.returnValue!=="unknown"&&(n.returnValue=!1),this.isDefaultPrevented=Ll)},stopPropagation:function(){var n=this.nativeEvent;n&&(n.stopPropagation?n.stopPropagation():typeof n.cancelBubble!=="unknown"&&(n.cancelBubble=!0),this.isPropagationStopped=Ll)},persist:function(){},isPersistent:Ll}),t}function km(e){var t=this.nativeEvent;return t.getModifierState?t.getModifierState(e):(e=wm[e])?!!t[e]:!1}function Io(){return km}function ec(e,t){switch(e){case"keyup":return Pm.indexOf(t.keyCode)!==-1;case"keydown":return t.keyCode!==229;case"keypress":case"mousedown":case"focusout":return!0;default:return!1}}function tc(e){return e=e.detail,typeof e==="object"&&"data"in e?e.data:null}function Mm(e,t){switch(e){case"compositionend":return tc(t);case"keypress":if(t.which!==32)return null;return Ms=!0,Bs;case"textInput":return e=t.data,e===Bs&&Ms?null:e;default:return null}}function $m(e,t){if(vn)return e==="compositionend"||!Ro&&ec(e,t)?(e=bf(),Hl=Wo=St=null,vn=!1,e):null;switch(e){case"paste":return null;case"keypress":if(!(t.ctrlKey||t.altKey||t.metaKey)||t.ctrlKey&&t.altKey){if(t.char&&1<t.char.length)return t.char;if(t.which)return String.fromCharCode(t.which)}return null;case"compositionend":return jf&&t.locale!=="ko"?null:t.data;default:return null}}function $s(e){var t=e&&e.nodeName&&e.nodeName.toLowerCase();return t==="input"?!!Dm[e.type]:t==="textarea"?!0:!1}function nc(e,t,n,r){$f(r),t=ti(t,"onChange"),0<t.length&&(n=new Ao("onChange","change",null,n,r),e.push({event:n,listeners:t}))}function _m(e){pc(e,0)}function ki(e){var t=xn(e);if(Ff(t))return e}function Om(e,t){if(e==="change")return t}function Ds(){wr&&(wr.detachEvent("onpropertychange",lc),Br=wr=null)}function lc(e){if(e.propertyName==="value"&&ki(Br)){var t=[];nc(t,Br,e,_o(e)),Uf(_m,t)}}function Um(e,t,n){e==="focusin"?(Ds(),wr=t,Br=n,wr.attachEvent("onpropertychange",lc)):e==="focusout"&&Ds()}function Hm(e){if(e==="selectionchange"||e==="keyup"||e==="keydown")return ki(Br)}function Qm(e,t){if(e==="click")return ki(t)}function Wm(e,t){if(e==="input"||e==="change")return ki(t)}function Am(e,t){return e===t&&(e!==0||1/e===1/t)||e!==e&&t!==t}function Mr(e,t){if(Ze(e,t))return!0;if(typeof e!=="object"||e===null||typeof t!=="object"||t===null)return!1;var n=Object.keys(e),r=Object.keys(t);if(n.length!==r.length)return!1;for(r=0;r<n.length;r++){var l=n[r];if(!Du.call(t,l)||!Ze(e[l],t[l]))return!1}return!0}function _s(e){for(;e&&e.firstChild;)e=e.firstChild;return e}function Os(e,t){var n=_s(e);e=0;for(var r;n;){if(n.nodeType===3){if(r=e+n.textContent.length,e<=t&&r>=t)return{node:n,offset:t-e};e=r}e:{for(;n;){if(n.nextSibling){n=n.nextSibling;break e}n=n.parentNode}n=void 0}n=_s(n)}}function ic(e,t){return e&&t?e===t?!0:e&&e.nodeType===3?!1:t&&t.nodeType===3?ic(e,t.parentNode):("contains"in e)?e.contains(t):e.compareDocumentPosition?!!(e.compareDocumentPosition(t)&16):!1:!1}function uc(){for(var e=window,t=Zl();t instanceof e.HTMLIFrameElement;){try{var n=typeof t.contentWindow.location.href==="string"}catch(r){n=!1}if(n)e=t.contentWindow;else break;t=Zl(e.document)}return t}function Vo(e){var t=e&&e.nodeName&&e.nodeName.toLowerCase();return t&&(t==="input"&&(e.type==="text"||e.type==="search"||e.type==="tel"||e.type==="url"||e.type==="password")||t==="textarea"||e.contentEditable==="true")}function Im(e){var t=uc(),n=e.focusedElem,r=e.selectionRange;if(t!==n&&n&&n.ownerDocument&&ic(n.ownerDocument.documentElement,n)){if(r!==null&&Vo(n)){if(t=r.start,e=r.end,e===void 0&&(e=t),"selectionStart"in n)n.selectionStart=t,n.selectionEnd=Math.min(e,n.value.length);else if(e=(t=n.ownerDocument||document)&&t.defaultView||window,e.getSelection){e=e.getSelection();var l=n.textContent.length,i=Math.min(r.start,l);r=r.end===void 0?i:Math.min(r.end,l),!e.extend&&i>r&&(l=r,r=i,i=l),l=Os(n,i);var u=Os(n,r);l&&u&&(e.rangeCount!==1||e.anchorNode!==l.node||e.anchorOffset!==l.offset||e.focusNode!==u.node||e.focusOffset!==u.offset)&&(t=t.createRange(),t.setStart(l.node,l.offset),e.removeAllRanges(),i>r?(e.addRange(t),e.extend(u.node,u.offset)):(t.setEnd(u.node,u.offset),e.addRange(t)))}}t=[];for(e=n;e=e.parentNode;)e.nodeType===1&&t.push({element:e,left:e.scrollLeft,top:e.scrollTop});typeof n.focus==="function"&&n.focus();for(n=0;n<t.length;n++)e=t[n],e.element.scrollLeft=e.left,e.element.scrollTop=e.top}}function Us(e,t,n){var r=n.window===n?n.document:n.nodeType===9?n:n.ownerDocument;eo||wn==null||wn!==Zl(r)||(r=wn,("selectionStart"in r)&&Vo(r)?r={start:r.selectionStart,end:r.selectionEnd}:(r=(r.ownerDocument&&r.ownerDocument.defaultView||window).getSelection(),r={anchorNode:r.anchorNode,anchorOffset:r.anchorOffset,focusNode:r.focusNode,focusOffset:r.focusOffset}),kr&&Mr(kr,r)||(kr=r,r=ti(ju,"onSelect"),0<r.length&&(t=new Ao("onSelect","select",null,t,n),e.push({event:t,listeners:r}),t.target=wn)))}function Tl(e,t){var n={};return n[e.toLowerCase()]=t.toLowerCase(),n["Webkit"+e]="webkit"+t,n["Moz"+e]="moz"+t,n}function xi(e){if(xu[e])return xu[e];if(!kn[e])return e;var t=kn[e],n;for(n in t)if(t.hasOwnProperty(n)&&n in oc)return xu[e]=t[n];return e}function Dt(e,t){dc.set(e,t),rn(t,[e])}function Qs(e,t,n){var r=e.type||"unknown-event";e.currentTarget=n,Vh(r,t,void 0,e),e.currentTarget=null}function pc(e,t){t=(t&4)!==0;for(var n=0;n<e.length;n++){var r=e[n],l=r.event;r=r.listeners;e:{var i=void 0;if(t)for(var u=r.length-1;0<=u;u--){var o=r[u],a=o.instance,s=o.currentTarget;if(o=o.listener,a!==i&&l.isPropagationStopped())break e;Qs(l,o,s),i=a}else for(u=0;u<r.length;u++){if(o=r[u],a=o.instance,s=o.currentTarget,o=o.listener,a!==i&&l.isPropagationStopped())break e;Qs(l,o,s),i=a}}}if(Gl)throw e=Ju,Gl=!1,Ju=null,e}function A(e,t){var n=t[oo];n===void 0&&(n=t[oo]=new Set);var r=e+"__bubble";n.has(r)||(hc(t,e,2,!1),n.add(r))}function Su(e,t,n){var r=0;t&&(r|=4),hc(n,e,r,t)}function $r(e){if(!e[Pl]){e[Pl]=!0,xf.forEach(function(n){n!=="selectionchange"&&(Vm.has(n)||Su(n,!1,e),Su(n,!0,e))});var t=e.nodeType===9?e:e.ownerDocument;t===null||t[Pl]||(t[Pl]=!0,Su("selectionchange",!1,t))}}function hc(e,t,n,r){switch(qf(t)){case 1:var l=um;break;case 4:l=om;break;default:l=Qo}n=l.bind(null,t,n,e),l=void 0,!Zu||t!=="touchstart"&&t!=="touchmove"&&t!=="wheel"||(l=!0),r?l!==void 0?e.addEventListener(t,n,{capture:!0,passive:l}):e.addEventListener(t,n,!0):l!==void 0?e.addEventListener(t,n,{passive:l}):e.addEventListener(t,n,!1)}function Nu(e,t,n,r,l){var i=r;if((t&1)===0&&(t&2)===0&&r!==null)e:for(;;){if(r===null)return;var u=r.tag;if(u===3||u===4){var o=r.stateNode.containerInfo;if(o===l||o.nodeType===8&&o.parentNode===l)break;if(u===4)for(u=r.return;u!==null;){var a=u.tag;if(a===3||a===4){if(a=u.stateNode.containerInfo,a===l||a.nodeType===8&&a.parentNode===l)return}u=u.return}for(;o!==null;){if(u=Xt(o),u===null)return;if(a=u.tag,a===5||a===6){r=i=u;continue e}o=o.parentNode}}r=r.return}Uf(function(){var s=i,h=_o(n),m=[];e:{var y=dc.get(e);if(y!==void 0){var x=Ao,N=e;switch(e){case"keypress":if(Ql(n)===0)break e;case"keydown":case"keyup":x=Sm;break;case"focusin":N="focus",x=ku;break;case"focusout":N="blur",x=ku;break;case"beforeblur":case"afterblur":x=ku;break;case"click":if(n.button===2)break e;case"auxclick":case"dblclick":case"mousedown":case"mousemove":case"mouseup":case"mouseout":case"mouseover":case"contextmenu":x=Ls;break;case"drag":case"dragend":case"dragenter":case"dragexit":case"dragleave":case"dragover":case"dragstart":case"drop":x=fm;break;case"touchcancel":case"touchend":case"touchmove":case"touchstart":x=Cm;break;case ac:case sc:case fc:x=pm;break;case cc:x=zm;break;case"scroll":x=am;break;case"wheel":x=Tm;break;case"copy":case"cut":case"paste":x=mm;break;case"gotpointercapture":case"lostpointercapture":case"pointercancel":case"pointerdown":case"pointermove":case"pointerout":case"pointerover":case"pointerup":x=Ps}var g=(t&4)!==0,L=!g&&e==="scroll",c=g?y!==null?y+"Capture":null:y;g=[];for(var f=s,p;f!==null;){p=f;var w=p.stateNode;if(p.tag===5&&w!==null&&(p=w,c!==null&&(w=zr(f,c),w!=null&&g.push(Dr(f,w,p)))),L)break;f=f.return}0<g.length&&(y=new x(y,N,null,n,h),m.push({event:y,listeners:g}))}}if((t&7)===0){e:{if(y=e==="mouseover"||e==="pointerover",x=e==="mouseout"||e==="pointerout",y&&n!==Ku&&(N=n.relatedTarget||n.fromElement)&&(Xt(N)||N[ct]))break e;if(x||y){if(y=h.window===h?h:(y=h.ownerDocument)?y.defaultView||y.parentWindow:window,x){if(N=n.relatedTarget||n.toElement,x=s,N=N?Xt(N):null,N!==null&&(L=ln(N),N!==L||N.tag!==5&&N.tag!==6))N=null}else x=null,N=s;if(x!==N){if(g=Ls,w="onMouseLeave",c="onMouseEnter",f="mouse",e==="pointerout"||e==="pointerover")g=Ps,w="onPointerLeave",c="onPointerEnter",f="pointer";if(L=x==null?y:xn(x),p=N==null?y:xn(N),y=new g(w,f+"leave",x,n,h),y.target=L,y.relatedTarget=p,w=null,Xt(h)===s&&(g=new g(c,f+"enter",N,n,h),g.target=p,g.relatedTarget=L,w=g),L=w,x&&N)t:{g=x,c=N,f=0;for(p=g;p;p=hn(p))f++;p=0;for(w=c;w;w=hn(w))p++;for(;0<f-p;)g=hn(g),f--;for(;0<p-f;)c=hn(c),p--;for(;f--;){if(g===c||c!==null&&g===c.alternate)break t;g=hn(g),c=hn(c)}g=null}else g=null;x!==null&&Ws(m,y,x,g,!1),N!==null&&L!==null&&Ws(m,L,N,g,!0)}}}e:{if(y=s?xn(s):window,x=y.nodeName&&y.nodeName.toLowerCase(),x==="select"||x==="input"&&y.type==="file")var C=Om;else if($s(y))if(rc)C=Wm;else{C=Hm;var F=Um}else(x=y.nodeName)&&x.toLowerCase()==="input"&&(y.type==="checkbox"||y.type==="radio")&&(C=Qm);if(C&&(C=C(e,s))){nc(m,C,n,h);break e}F&&F(e,y,s),e==="focusout"&&(F=y._wrapperState)&&F.controlled&&y.type==="number"&&Au(y,"number",y.value)}switch(F=s?xn(s):window,e){case"focusin":if($s(F)||F.contentEditable==="true")wn=F,ju=s,kr=null;break;case"focusout":kr=ju=wn=null;break;case"mousedown":eo=!0;break;case"contextmenu":case"mouseup":case"dragend":eo=!1,Us(m,n,h);break;case"selectionchange":if(Rm)break;case"keydown":case"keyup":Us(m,n,h)}var E;if(Ro)e:{switch(e){case"compositionstart":var T="onCompositionStart";break e;case"compositionend":T="onCompositionEnd";break e;case"compositionupdate":T="onCompositionUpdate";break e}T=void 0}else vn?ec(e,n)&&(T="onCompositionEnd"):e==="keydown"&&n.keyCode===229&&(T="onCompositionStart");if(T&&(jf&&n.locale!=="ko"&&(vn||T!=="onCompositionStart"?T==="onCompositionEnd"&&vn&&(E=bf()):(St=h,Wo=("value"in St)?St.value:St.textContent,vn=!0)),F=ti(s,T),0<F.length&&(T=new Ts(T,e,null,n,h),m.push({event:T,listeners:F}),E?T.data=E:(E=tc(n),E!==null&&(T.data=E)))),E=Bm?Mm(e,n):$m(e,n))s=ti(s,"onBeforeInput"),0<s.length&&(h=new Ts("onBeforeInput","beforeinput",null,n,h),m.push({event:h,listeners:s}),h.data=E)}pc(m,t)})}function Dr(e,t,n){return{instance:e,listener:t,currentTarget:n}}function ti(e,t){for(var n=t+"Capture",r=[];e!==null;){var l=e,i=l.stateNode;l.tag===5&&i!==null&&(l=i,i=zr(e,n),i!=null&&r.unshift(Dr(e,i,l)),i=zr(e,t),i!=null&&r.push(Dr(e,i,l))),e=e.return}return r}function hn(e){if(e===null)return null;do e=e.return;while(e&&e.tag!==5);return e?e:null}function Ws(e,t,n,r,l){for(var i=t._reactName,u=[];n!==null&&n!==r;){var o=n,a=o.alternate,s=o.stateNode;if(a!==null&&a===r)break;o.tag===5&&s!==null&&(o=s,l?(a=zr(n,i),a!=null&&u.unshift(Dr(n,a,o))):l||(a=zr(n,i),a!=null&&u.push(Dr(n,a,o)))),n=n.return}u.length!==0&&e.push({event:t,listeners:u})}function As(e){return(typeof e==="string"?e:""+e).replace(Ym,`
`).replace(Km,"")}function Bl(e,t,n){if(t=As(t),As(e)!==t&&n)throw Error(S(425))}function ni(){}function io(e,t){return e==="textarea"||e==="noscript"||typeof t.children==="string"||typeof t.children==="number"||typeof t.dangerouslySetInnerHTML==="object"&&t.dangerouslySetInnerHTML!==null&&t.dangerouslySetInnerHTML.__html!=null}function Jm(e){setTimeout(function(){throw e})}function Eu(e,t){var n=t,r=0;do{var l=n.nextSibling;if(e.removeChild(n),l&&l.nodeType===8)if(n=l.data,n==="/$"){if(r===0){e.removeChild(l),Pr(t);return}r--}else n!=="$"&&n!=="$?"&&n!=="$!"||r++;n=l}while(n);Pr(t)}function zt(e){for(;e!=null;e=e.nextSibling){var t=e.nodeType;if(t===1||t===3)break;if(t===8){if(t=e.data,t==="$"||t==="$!"||t==="$?")break;if(t==="/$")return null}}return e}function Rs(e){e=e.previousSibling;for(var t=0;e;){if(e.nodeType===8){var n=e.data;if(n==="$"||n==="$!"||n==="$?"){if(t===0)return e;t--}else n==="/$"&&t++}e=e.previousSibling}return null}function Xt(e){var t=e[et];if(t)return t;for(var n=e.parentNode;n;){if(t=n[ct]||n[et]){if(n=t.alternate,t.child!==null||n!==null&&n.child!==null)for(e=Rs(e);e!==null;){if(n=e[et])return n;e=Rs(e)}return t}e=n,n=e.parentNode}return null}function Vr(e){return e=e[et]||e[ct],!e||e.tag!==5&&e.tag!==6&&e.tag!==13&&e.tag!==3?null:e}function xn(e){if(e.tag===5||e.tag===6)return e.stateNode;throw Error(S(33))}function Si(e){return e[_r]||null}function _t(e){return{current:e}}function I(e){0>Sn||(e.current=ao[Sn],ao[Sn]=null,Sn--)}function H(e,t){Sn++,ao[Sn]=e.current,e.current=t}function _n(e,t){var n=e.type.contextTypes;if(!n)return $t;var r=e.stateNode;if(r&&r.__reactInternalMemoizedUnmaskedChildContext===t)return r.__reactInternalMemoizedMaskedChildContext;var l={},i;for(i in n)l[i]=t[i];return r&&(e=e.stateNode,e.__reactInternalMemoizedUnmaskedChildContext=t,e.__reactInternalMemoizedMaskedChildContext=l),l}function Se(e){return e=e.childContextTypes,e!==null&&e!==void 0}function ri(){I(xe),I(de)}function Vs(e,t,n){if(de.current!==$t)throw Error(S(168));H(de,t),H(xe,n)}function mc(e,t,n){var r=e.stateNode;if(t=t.childContextTypes,typeof r.getChildContext!=="function")return n;r=r.getChildContext();for(var l in r)if(!(l in t))throw Error(S(108,Uh(e)||"Unknown",l));return Z({},n,r)}function li(e){return e=(e=e.stateNode)&&e.__reactInternalMemoizedMergedChildContext||$t,bt=de.current,H(de,e),H(xe,xe.current),!0}function Ys(e,t,n){var r=e.stateNode;if(!r)throw Error(S(169));n?(e=mc(e,t,bt),r.__reactInternalMemoizedMergedChildContext=e,I(xe),I(de),H(de,e)):I(xe),H(xe,n)}function yc(e){ut===null?ut=[e]:ut.push(e)}function bm(e){Ni=!0,yc(e)}function Ot(){if(!Cu&&ut!==null){Cu=!0;var e=0,t=_;try{var n=ut;for(_=1;e<n.length;e++){var r=n[e];do r=r(!0);while(r!==null)}ut=null,Ni=!1}catch(l){throw ut!==null&&(ut=ut.slice(e+1)),Af(Oo,Ot),l}finally{_=t,Cu=!1}}return null}function Yt(e,t){Nn[En++]=ui,Nn[En++]=ii,ii=e,ui=t}function gc(e,t,n){Me[$e++]=ot,Me[$e++]=at,Me[$e++]=jt,jt=e;var r=ot;e=at;var l=32-Ke(r)-1;r&=~(1<<l),n+=1;var i=32-Ke(t)+l;if(30<i){var u=l-l%5;i=(r&(1<<u)-1).toString(32),r>>=u,l-=u,ot=1<<32-Ke(t)+l|n<<l|r,at=i+e}else ot=1<<i|n<<l|r,at=e}function Yo(e){e.return!==null&&(Yt(e,1),gc(e,1,0))}function Ko(e){for(;e===ii;)ii=Nn[--En],Nn[En]=null,ui=Nn[--En],Nn[En]=null;for(;e===jt;)jt=Me[--$e],Me[$e]=null,at=Me[--$e],Me[$e]=null,ot=Me[--$e],Me[$e]=null}function vc(e,t){var n=De(5,null,null,0);n.elementType="DELETED",n.stateNode=t,n.return=e,t=e.deletions,t===null?(e.deletions=[n],e.flags|=16):t.push(n)}function Ks(e,t){switch(e.tag){case 5:var n=e.type;return t=t.nodeType!==1||n.toLowerCase()!==t.nodeName.toLowerCase()?null:t,t!==null?(e.stateNode=t,Le=e,ze=zt(t.firstChild),!0):!1;case 6:return t=e.pendingProps===""||t.nodeType!==3?null:t,t!==null?(e.stateNode=t,Le=e,ze=null,!0):!1;case 13:return t=t.nodeType!==8?null:t,t!==null?(n=jt!==null?{id:ot,overflow:at}:null,e.memoizedState={dehydrated:t,treeContext:n,retryLane:1073741824},n=De(18,null,null,0),n.stateNode=t,n.return=e,e.child=n,Le=e,ze=null,!0):!1;default:return!1}}function so(e){return(e.mode&1)!==0&&(e.flags&128)===0}function fo(e){if(V){var t=ze;if(t){var n=t;if(!Ks(e,t)){if(so(e))throw Error(S(418));t=zt(n.nextSibling);var r=Le;t&&Ks(e,t)?vc(r,n):(e.flags=e.flags&-4097|2,V=!1,Le=e)}}else{if(so(e))throw Error(S(418));e.flags=e.flags&-4097|2,V=!1,Le=e}}}function Xs(e){for(e=e.return;e!==null&&e.tag!==5&&e.tag!==3&&e.tag!==13;)e=e.return;Le=e}function Ml(e){if(e!==Le)return!1;if(!V)return Xs(e),V=!0,!1;var t;if((t=e.tag!==3)&&!(t=e.tag!==5)&&(t=e.type,t=t!=="head"&&t!=="body"&&!io(e.type,e.memoizedProps)),t&&(t=ze)){if(so(e))throw wc(),Error(S(418));for(;t;)vc(e,t),t=zt(t.nextSibling)}if(Xs(e),e.tag===13){if(e=e.memoizedState,e=e!==null?e.dehydrated:null,!e)throw Error(S(317));e:{e=e.nextSibling;for(t=0;e;){if(e.nodeType===8){var n=e.data;if(n==="/$"){if(t===0){ze=zt(e.nextSibling);break e}t--}else n!=="$"&&n!=="$!"&&n!=="$?"||t++}e=e.nextSibling}ze=null}}else ze=Le?zt(e.stateNode.nextSibling):null;return!0}function wc(){for(var e=ze;e;)e=zt(e.nextSibling)}function On(){ze=Le=null,V=!1}function Xo(e){Ye===null?Ye=[e]:Ye.push(e)}function ir(e,t,n){if(e=n.ref,e!==null&&typeof e!=="function"&&typeof e!=="object"){if(n._owner){if(n=n._owner,n){if(n.tag!==1)throw Error(S(309));var r=n.stateNode}if(!r)throw Error(S(147,e));var l=r,i=""+e;if(t!==null&&t.ref!==null&&typeof t.ref==="function"&&t.ref._stringRef===i)return t.ref;return t=function(u){var o=l.refs;u===null?delete o[i]:o[i]=u},t._stringRef=i,t}if(typeof e!=="string")throw Error(S(284));if(!n._owner)throw Error(S(290,e))}return e}function $l(e,t){throw e=Object.prototype.toString.call(t),Error(S(31,e==="[object Object]"?"object with keys {"+Object.keys(t).join(", ")+"}":e))}function Zs(e){var t=e._init;return t(e._payload)}function kc(e){function t(c,f){if(e){var p=c.deletions;p===null?(c.deletions=[f],c.flags|=16):p.push(f)}}function n(c,f){if(!e)return null;for(;f!==null;)t(c,f),f=f.sibling;return null}function r(c,f){for(c=new Map;f!==null;)f.key!==null?c.set(f.key,f):c.set(f.index,f),f=f.sibling;return c}function l(c,f){return c=Bt(c,f),c.index=0,c.sibling=null,c}function i(c,f,p){if(c.index=p,!e)return c.flags|=1048576,f;if(p=c.alternate,p!==null)return p=p.index,p<f?(c.flags|=2,f):p;return c.flags|=2,f}function u(c){return e&&c.alternate===null&&(c.flags|=2),c}function o(c,f,p,w){if(f===null||f.tag!==6)return f=Mu(p,c.mode,w),f.return=c,f;return f=l(f,p),f.return=c,f}function a(c,f,p,w){var C=p.type;if(C===gn)return h(c,f,p.props.children,w,p.key);if(f!==null&&(f.elementType===C||typeof C==="object"&&C!==null&&C.$$typeof===vt&&Zs(C)===f.type))return w=l(f,p.props),w.ref=ir(c,f,p),w.return=c,w;return w=Xl(p.type,p.key,p.props,null,c.mode,w),w.ref=ir(c,f,p),w.return=c,w}function s(c,f,p,w){if(f===null||f.tag!==4||f.stateNode.containerInfo!==p.containerInfo||f.stateNode.implementation!==p.implementation)return f=$u(p,c.mode,w),f.return=c,f;return f=l(f,p.children||[]),f.return=c,f}function h(c,f,p,w,C){if(f===null||f.tag!==7)return f=qt(p,c.mode,w,C),f.return=c,f;return f=l(f,p),f.return=c,f}function m(c,f,p){if(typeof f==="string"&&f!==""||typeof f==="number")return f=Mu(""+f,c.mode,p),f.return=c,f;if(typeof f==="object"&&f!==null){switch(f.$$typeof){case Sl:return p=Xl(f.type,f.key,f.props,null,c.mode,p),p.ref=ir(c,null,f),p.return=c,p;case yn:return f=$u(f,c.mode,p),f.return=c,f;case vt:var w=f._init;return m(c,w(f._payload),p)}if(sr(f)||tr(f))return f=qt(f,c.mode,p,null),f.return=c,f;$l(c,f)}return null}function y(c,f,p,w){var C=f!==null?f.key:null;if(typeof p==="string"&&p!==""||typeof p==="number")return C!==null?null:o(c,f,""+p,w);if(typeof p==="object"&&p!==null){switch(p.$$typeof){case Sl:return p.key===C?a(c,f,p,w):null;case yn:return p.key===C?s(c,f,p,w):null;case vt:return C=p._init,y(c,f,C(p._payload),w)}if(sr(p)||tr(p))return C!==null?null:h(c,f,p,w,null);$l(c,p)}return null}function x(c,f,p,w,C){if(typeof w==="string"&&w!==""||typeof w==="number")return c=c.get(p)||null,o(f,c,""+w,C);if(typeof w==="object"&&w!==null){switch(w.$$typeof){case Sl:return c=c.get(w.key===null?p:w.key)||null,a(f,c,w,C);case yn:return c=c.get(w.key===null?p:w.key)||null,s(f,c,w,C);case vt:var F=w._init;return x(c,f,p,F(w._payload),C)}if(sr(w)||tr(w))return c=c.get(p)||null,h(f,c,w,C,null);$l(f,w)}return null}function N(c,f,p,w){for(var C=null,F=null,E=f,T=f=0,O=null;E!==null&&T<p.length;T++){E.index>T?(O=E,E=null):O=E.sibling;var P=y(c,E,p[T],w);if(P===null){E===null&&(E=O);break}e&&E&&P.alternate===null&&t(c,E),f=i(P,f,T),F===null?C=P:F.sibling=P,F=P,E=O}if(T===p.length)return n(c,E),V&&Yt(c,T),C;if(E===null){for(;T<p.length;T++)E=m(c,p[T],w),E!==null&&(f=i(E,f,T),F===null?C=E:F.sibling=E,F=E);return V&&Yt(c,T),C}for(E=r(c,E);T<p.length;T++)O=x(E,c,T,p[T],w),O!==null&&(e&&O.alternate!==null&&E.delete(O.key===null?T:O.key),f=i(O,f,T),F===null?C=O:F.sibling=O,F=O);return e&&E.forEach(function(ve){return t(c,ve)}),V&&Yt(c,T),C}function g(c,f,p,w){var C=tr(p);if(typeof C!=="function")throw Error(S(150));if(p=C.call(p),p==null)throw Error(S(151));for(var F=C=null,E=f,T=f=0,O=null,P=p.next();E!==null&&!P.done;T++,P=p.next()){E.index>T?(O=E,E=null):O=E.sibling;var ve=y(c,E,P.value,w);if(ve===null){E===null&&(E=O);break}e&&E&&ve.alternate===null&&t(c,E),f=i(ve,f,T),F===null?C=ve:F.sibling=ve,F=ve,E=O}if(P.done)return n(c,E),V&&Yt(c,T),C;if(E===null){for(;!P.done;T++,P=p.next())P=m(c,P.value,w),P!==null&&(f=i(P,f,T),F===null?C=P:F.sibling=P,F=P);return V&&Yt(c,T),C}for(E=r(c,E);!P.done;T++,P=p.next())P=x(E,c,T,P.value,w),P!==null&&(e&&P.alternate!==null&&E.delete(P.key===null?T:P.key),f=i(P,f,T),F===null?C=P:F.sibling=P,F=P);return e&&E.forEach(function(It){return t(c,It)}),V&&Yt(c,T),C}function L(c,f,p,w){if(typeof p==="object"&&p!==null&&p.type===gn&&p.key===null&&(p=p.props.children),typeof p==="object"&&p!==null){switch(p.$$typeof){case Sl:e:{for(var C=p.key,F=f;F!==null;){if(F.key===C){if(C=p.type,C===gn){if(F.tag===7){n(c,F.sibling),f=l(F,p.props.children),f.return=c,c=f;break e}}else if(F.elementType===C||typeof C==="object"&&C!==null&&C.$$typeof===vt&&Zs(C)===F.type){n(c,F.sibling),f=l(F,p.props),f.ref=ir(c,F,p),f.return=c,c=f;break e}n(c,F);break}else t(c,F);F=F.sibling}p.type===gn?(f=qt(p.props.children,c.mode,w,p.key),f.return=c,c=f):(w=Xl(p.type,p.key,p.props,null,c.mode,w),w.ref=ir(c,f,p),w.return=c,c=w)}return u(c);case yn:e:{for(F=p.key;f!==null;){if(f.key===F)if(f.tag===4&&f.stateNode.containerInfo===p.containerInfo&&f.stateNode.implementation===p.implementation){n(c,f.sibling),f=l(f,p.children||[]),f.return=c,c=f;break e}else{n(c,f);break}else t(c,f);f=f.sibling}f=$u(p,c.mode,w),f.return=c,c=f}return u(c);case vt:return F=p._init,L(c,f,F(p._payload),w)}if(sr(p))return N(c,f,p,w);if(tr(p))return g(c,f,p,w);$l(c,p)}return typeof p==="string"&&p!==""||typeof p==="number"?(p=""+p,f!==null&&f.tag===6?(n(c,f.sibling),f=l(f,p),f.return=c,c=f):(n(c,f),f=Mu(p,c.mode,w),f.return=c,c=f),u(c)):n(c,f)}return L}function Jo(){Zo=Cn=ai=null}function Go(e){var t=oi.current;I(oi),e._currentValue=t}function co(e,t,n){for(;e!==null;){var r=e.alternate;if((e.childLanes&t)!==t?(e.childLanes|=t,r!==null&&(r.childLanes|=t)):r!==null&&(r.childLanes&t)!==t&&(r.childLanes|=t),e===n)break;e=e.return}}function Mn(e,t){ai=e,Zo=Cn=null,e=e.dependencies,e!==null&&e.firstContext!==null&&((e.lanes&t)!==0&&(ke=!0),e.firstContext=null)}function Oe(e){var t=e._currentValue;if(Zo!==e)if(e={context:e,memoizedValue:t,next:null},Cn===null){if(ai===null)throw Error(S(308));Cn=e,ai.dependencies={lanes:0,firstContext:e}}else Cn=Cn.next=e;return t}function qo(e){Zt===null?Zt=[e]:Zt.push(e)}function Sc(e,t,n,r){var l=t.interleaved;return l===null?(n.next=n,qo(t)):(n.next=l.next,l.next=n),t.interleaved=n,dt(e,r)}function dt(e,t){e.lanes|=t;var n=e.alternate;n!==null&&(n.lanes|=t),n=e;for(e=e.return;e!==null;)e.childLanes|=t,n=e.alternate,n!==null&&(n.childLanes|=t),n=e,e=e.return;return n.tag===3?n.stateNode:null}function bo(e){e.updateQueue={baseState:e.memoizedState,firstBaseUpdate:null,lastBaseUpdate:null,shared:{pending:null,interleaved:null,lanes:0},effects:null}}function Nc(e,t){e=e.updateQueue,t.updateQueue===e&&(t.updateQueue={baseState:e.baseState,firstBaseUpdate:e.firstBaseUpdate,lastBaseUpdate:e.lastBaseUpdate,shared:e.shared,effects:e.effects})}function st(e,t){return{eventTime:e,lane:t,tag:0,payload:null,callback:null,next:null}}function Lt(e,t,n){var r=e.updateQueue;if(r===null)return null;if(r=r.shared,($&2)!==0){var l=r.pending;return l===null?t.next=t:(t.next=l.next,l.next=t),r.pending=t,dt(e,n)}return l=r.interleaved,l===null?(t.next=t,qo(r)):(t.next=l.next,l.next=t),r.interleaved=t,dt(e,n)}function Al(e,t,n){if(t=t.updateQueue,t!==null&&(t=t.shared,(n&4194240)!==0)){var r=t.lanes;r&=e.pendingLanes,n|=r,t.lanes=n,Uo(e,n)}}function Js(e,t){var{updateQueue:n,alternate:r}=e;if(r!==null&&(r=r.updateQueue,n===r)){var l=null,i=null;if(n=n.firstBaseUpdate,n!==null){do{var u={eventTime:n.eventTime,lane:n.lane,tag:n.tag,payload:n.payload,callback:n.callback,next:null};i===null?l=i=u:i=i.next=u,n=n.next}while(n!==null);i===null?l=i=t:i=i.next=t}else l=i=t;n={baseState:r.baseState,firstBaseUpdate:l,lastBaseUpdate:i,shared:r.shared,effects:r.effects},e.updateQueue=n;return}e=n.lastBaseUpdate,e===null?n.firstBaseUpdate=t:e.next=t,n.lastBaseUpdate=t}function si(e,t,n,r){var l=e.updateQueue;wt=!1;var{firstBaseUpdate:i,lastBaseUpdate:u}=l,o=l.shared.pending;if(o!==null){l.shared.pending=null;var a=o,s=a.next;a.next=null,u===null?i=s:u.next=s,u=a;var h=e.alternate;h!==null&&(h=h.updateQueue,o=h.lastBaseUpdate,o!==u&&(o===null?h.firstBaseUpdate=s:o.next=s,h.lastBaseUpdate=a))}if(i!==null){var m=l.baseState;u=0,h=s=a=null,o=i;do{var{lane:y,eventTime:x}=o;if((r&y)===y){h!==null&&(h=h.next={eventTime:x,lane:0,tag:o.tag,payload:o.payload,callback:o.callback,next:null});e:{var N=e,g=o;switch(y=t,x=n,g.tag){case 1:if(N=g.payload,typeof N==="function"){m=N.call(x,m,y);break e}m=N;break e;case 3:N.flags=N.flags&-65537|128;case 0:if(N=g.payload,y=typeof N==="function"?N.call(x,m,y):N,y===null||y===void 0)break e;m=Z({},m,y);break e;case 2:wt=!0}}o.callback!==null&&o.lane!==0&&(e.flags|=64,y=l.effects,y===null?l.effects=[o]:y.push(o))}else x={eventTime:x,lane:y,tag:o.tag,payload:o.payload,callback:o.callback,next:null},h===null?(s=h=x,a=m):h=h.next=x,u|=y;if(o=o.next,o===null)if(o=l.shared.pending,o===null)break;else y=o,o=y.next,y.next=null,l.lastBaseUpdate=y,l.shared.pending=null}while(1);if(h===null&&(a=m),l.baseState=a,l.firstBaseUpdate=s,l.lastBaseUpdate=h,t=l.shared.interleaved,t!==null){l=t;do u|=l.lane,l=l.next;while(l!==t)}else i===null&&(l.shared.lanes=0);tn|=u,e.lanes=u,e.memoizedState=m}}function Gs(e,t,n){if(e=t.effects,t.effects=null,e!==null)for(t=0;t<e.length;t++){var r=e[t],l=r.callback;if(l!==null){if(r.callback=null,r=n,typeof l!=="function")throw Error(S(191,l));l.call(r)}}}function Jt(e){if(e===Yr)throw Error(S(174));return e}function jo(e,t){switch(H(Ur,t),H(Or,e),H(nt,Yr),e=t.nodeType,e){case 9:case 11:t=(t=t.documentElement)?t.namespaceURI:Ru(null,"");break;default:e=e===8?t.parentNode:t,t=e.namespaceURI||null,e=e.tagName,t=Ru(t,e)}I(nt),H(nt,t)}function Hn(){I(nt),I(Or),I(Ur)}function Ec(e){Jt(Ur.current);var t=Jt(nt.current),n=Ru(t,e.type);t!==n&&(H(Or,e),H(nt,n))}function ea(e){Or.current===e&&(I(nt),I(Or))}function fi(e){for(var t=e;t!==null;){if(t.tag===13){var n=t.memoizedState;if(n!==null&&(n=n.dehydrated,n===null||n.data==="$?"||n.data==="$!"))return t}else if(t.tag===19&&t.memoizedProps.revealOrder!==void 0){if((t.flags&128)!==0)return t}else if(t.child!==null){t.child.return=t,t=t.child;continue}if(t===e)break;for(;t.sibling===null;){if(t.return===null||t.return===e)return null;t=t.return}t.sibling.return=t.return,t=t.sibling}return null}function ta(){for(var e=0;e<Fu.length;e++)Fu[e]._workInProgressVersionPrimary=null;Fu.length=0}function se(){throw Error(S(321))}function na(e,t){if(t===null)return!1;for(var n=0;n<t.length&&n<e.length;n++)if(!Ze(e[n],t[n]))return!1;return!0}function ra(e,t,n,r,l,i){if(en=i,X=t,t.memoizedState=null,t.updateQueue=null,t.lanes=0,Il.current=e===null||e.memoizedState===null?l1:i1,e=n(r,l),xr){i=0;do{if(xr=!1,Hr=0,25<=i)throw Error(S(301));i+=1,ne=j=null,t.updateQueue=null,Il.current=u1,e=n(r,l)}while(xr)}if(Il.current=di,t=j!==null&&j.next!==null,en=0,ne=j=X=null,ci=!1,t)throw Error(S(300));return e}function la(){var e=Hr!==0;return Hr=0,e}function je(){var e={memoizedState:null,baseState:null,baseQueue:null,queue:null,next:null};return ne===null?X.memoizedState=ne=e:ne=ne.next=e,ne}function Ue(){if(j===null){var e=X.alternate;e=e!==null?e.memoizedState:null}else e=j.next;var t=ne===null?X.memoizedState:ne.next;if(t!==null)ne=t,j=e;else{if(e===null)throw Error(S(310));j=e,e={memoizedState:j.memoizedState,baseState:j.baseState,baseQueue:j.baseQueue,queue:j.queue,next:null},ne===null?X.memoizedState=ne=e:ne=ne.next=e}return ne}function Qr(e,t){return typeof t==="function"?t(e):t}function Lu(e){var t=Ue(),n=t.queue;if(n===null)throw Error(S(311));n.lastRenderedReducer=e;var r=j,l=r.baseQueue,i=n.pending;if(i!==null){if(l!==null){var u=l.next;l.next=i.next,i.next=u}r.baseQueue=l=i,n.pending=null}if(l!==null){i=l.next,r=r.baseState;var o=u=null,a=null,s=i;do{var h=s.lane;if((en&h)===h)a!==null&&(a=a.next={lane:0,action:s.action,hasEagerState:s.hasEagerState,eagerState:s.eagerState,next:null}),r=s.hasEagerState?s.eagerState:e(r,s.action);else{var m={lane:h,action:s.action,hasEagerState:s.hasEagerState,eagerState:s.eagerState,next:null};a===null?(o=a=m,u=r):a=a.next=m,X.lanes|=h,tn|=h}s=s.next}while(s!==null&&s!==i);a===null?u=r:a.next=o,Ze(r,t.memoizedState)||(ke=!0),t.memoizedState=r,t.baseState=u,t.baseQueue=a,n.lastRenderedState=r}if(e=n.interleaved,e!==null){l=e;do i=l.lane,X.lanes|=i,tn|=i,l=l.next;while(l!==e)}else l===null&&(n.lanes=0);return[t.memoizedState,n.dispatch]}function Tu(e){var t=Ue(),n=t.queue;if(n===null)throw Error(S(311));n.lastRenderedReducer=e;var{dispatch:r,pending:l}=n,i=t.memoizedState;if(l!==null){n.pending=null;var u=l=l.next;do i=e(i,u.action),u=u.next;while(u!==l);Ze(i,t.memoizedState)||(ke=!0),t.memoizedState=i,t.baseQueue===null&&(t.baseState=i),n.lastRenderedState=i}return[i,r]}function Cc(){}function Fc(e,t){var n=X,r=Ue(),l=t(),i=!Ze(r.memoizedState,l);if(i&&(r.memoizedState=l,ke=!0),r=r.queue,ia(Tc.bind(null,n,r,e),[e]),r.getSnapshot!==t||i||ne!==null&&ne.memoizedState.tag&1){if(n.flags|=2048,Wr(9,Lc.bind(null,n,r,l,t),void 0,null),re===null)throw Error(S(349));(en&30)!==0||zc(n,t,l)}return l}function zc(e,t,n){e.flags|=16384,e={getSnapshot:t,value:n},t=X.updateQueue,t===null?(t={lastEffect:null,stores:null},X.updateQueue=t,t.stores=[e]):(n=t.stores,n===null?t.stores=[e]:n.push(e))}function Lc(e,t,n,r){t.value=n,t.getSnapshot=r,Pc(t)&&Bc(e)}function Tc(e,t,n){return n(function(){Pc(t)&&Bc(e)})}function Pc(e){var t=e.getSnapshot;e=e.value;try{var n=t();return!Ze(e,n)}catch(r){return!0}}function Bc(e){var t=dt(e,1);t!==null&&Xe(t,e,1,-1)}function qs(e){var t=je();return typeof e==="function"&&(e=e()),t.memoizedState=t.baseState=e,e={pending:null,interleaved:null,lanes:0,dispatch:null,lastRenderedReducer:Qr,lastRenderedState:e},t.queue=e,e=e.dispatch=r1.bind(null,X,e),[t.memoizedState,e]}function Wr(e,t,n,r){return e={tag:e,create:t,destroy:n,deps:r,next:null},t=X.updateQueue,t===null?(t={lastEffect:null,stores:null},X.updateQueue=t,t.lastEffect=e.next=e):(n=t.lastEffect,n===null?t.lastEffect=e.next=e:(r=n.next,n.next=e,e.next=r,t.lastEffect=e)),e}function Mc(){return Ue().memoizedState}function Rl(e,t,n,r){var l=je();X.flags|=e,l.memoizedState=Wr(1|t,n,void 0,r===void 0?null:r)}function Ei(e,t,n,r){var l=Ue();r=r===void 0?null:r;var i=void 0;if(j!==null){var u=j.memoizedState;if(i=u.destroy,r!==null&&na(r,u.deps)){l.memoizedState=Wr(t,n,i,r);return}}X.flags|=e,l.memoizedState=Wr(1|t,n,i,r)}function bs(e,t){return Rl(8390656,8,e,t)}function ia(e,t){return Ei(2048,8,e,t)}function $c(e,t){return Ei(4,2,e,t)}function Dc(e,t){return Ei(4,4,e,t)}function _c(e,t){if(typeof t==="function")return e=e(),t(e),function(){t(null)};if(t!==null&&t!==void 0)return e=e(),t.current=e,function(){t.current=null}}function Oc(e,t,n){return n=n!==null&&n!==void 0?n.concat([e]):null,Ei(4,4,_c.bind(null,t,e),n)}function ua(){}function Uc(e,t){var n=Ue();t=t===void 0?null:t;var r=n.memoizedState;if(r!==null&&t!==null&&na(t,r[1]))return r[0];return n.memoizedState=[e,t],e}function Hc(e,t){var n=Ue();t=t===void 0?null:t;var r=n.memoizedState;if(r!==null&&t!==null&&na(t,r[1]))return r[0];return e=e(),n.memoizedState=[e,t],e}function Qc(e,t,n){if((en&21)===0)return e.baseState&&(e.baseState=!1,ke=!0),e.memoizedState=n;return Ze(n,t)||(n=Vf(),X.lanes|=n,tn|=n,e.baseState=!0),t}function t1(e,t){var n=_;_=n!==0&&4>n?n:4,e(!0);var r=zu.transition;zu.transition={};try{e(!1),t()}finally{_=n,zu.transition=r}}function Wc(){return Ue().memoizedState}function n1(e,t,n){var r=Pt(e);if(n={lane:r,action:n,hasEagerState:!1,eagerState:null,next:null},Ac(e))Ic(t,n);else if(n=Sc(e,t,n,r),n!==null){var l=ye();Xe(n,e,r,l),Rc(n,t,r)}}function r1(e,t,n){var r=Pt(e),l={lane:r,action:n,hasEagerState:!1,eagerState:null,next:null};if(Ac(e))Ic(t,l);else{var i=e.alternate;if(e.lanes===0&&(i===null||i.lanes===0)&&(i=t.lastRenderedReducer,i!==null))try{var u=t.lastRenderedState,o=i(u,n);if(l.hasEagerState=!0,l.eagerState=o,Ze(o,u)){var a=t.interleaved;a===null?(l.next=l,qo(t)):(l.next=a.next,a.next=l),t.interleaved=l;return}}catch(s){}finally{}n=Sc(e,t,l,r),n!==null&&(l=ye(),Xe(n,e,r,l),Rc(n,t,r))}}function Ac(e){var t=e.alternate;return e===X||t!==null&&t===X}function Ic(e,t){xr=ci=!0;var n=e.pending;n===null?t.next=t:(t.next=n.next,n.next=t),e.pending=t}function Rc(e,t,n){if((n&4194240)!==0){var r=t.lanes;r&=e.pendingLanes,n|=r,t.lanes=n,Uo(e,n)}}function Re(e,t){if(e&&e.defaultProps){t=Z({},t),e=e.defaultProps;for(var n in e)t[n]===void 0&&(t[n]=e[n]);return t}return t}function po(e,t,n,r){t=e.memoizedState,n=n(r,t),n=n===null||n===void 0?t:Z({},t,n),e.memoizedState=n,e.lanes===0&&(e.updateQueue.baseState=n)}function js(e,t,n,r,l,i,u){return e=e.stateNode,typeof e.shouldComponentUpdate==="function"?e.shouldComponentUpdate(r,i,u):t.prototype&&t.prototype.isPureReactComponent?!Mr(n,r)||!Mr(l,i):!0}function Vc(e,t,n){var r=!1,l=$t,i=t.contextType;return typeof i==="object"&&i!==null?i=Oe(i):(l=Se(t)?bt:de.current,r=t.contextTypes,i=(r=r!==null&&r!==void 0)?_n(e,l):$t),t=new t(n,i),e.memoizedState=t.state!==null&&t.state!==void 0?t.state:null,t.updater=Ci,e.stateNode=t,t._reactInternals=e,r&&(e=e.stateNode,e.__reactInternalMemoizedUnmaskedChildContext=l,e.__reactInternalMemoizedMaskedChildContext=i),t}function ef(e,t,n,r){e=t.state,typeof t.componentWillReceiveProps==="function"&&t.componentWillReceiveProps(n,r),typeof t.UNSAFE_componentWillReceiveProps==="function"&&t.UNSAFE_componentWillReceiveProps(n,r),t.state!==e&&Ci.enqueueReplaceState(t,t.state,null)}function ho(e,t,n,r){var l=e.stateNode;l.props=n,l.state=e.memoizedState,l.refs={},bo(e);var i=t.contextType;typeof i==="object"&&i!==null?l.context=Oe(i):(i=Se(t)?bt:de.current,l.context=_n(e,i)),l.state=e.memoizedState,i=t.getDerivedStateFromProps,typeof i==="function"&&(po(e,t,i,n),l.state=e.memoizedState),typeof t.getDerivedStateFromProps==="function"||typeof l.getSnapshotBeforeUpdate==="function"||typeof l.UNSAFE_componentWillMount!=="function"&&typeof l.componentWillMount!=="function"||(t=l.state,typeof l.componentWillMount==="function"&&l.componentWillMount(),typeof l.UNSAFE_componentWillMount==="function"&&l.UNSAFE_componentWillMount(),t!==l.state&&Ci.enqueueReplaceState(l,l.state,null),si(e,n,l,r),l.state=e.memoizedState),typeof l.componentDidMount==="function"&&(e.flags|=4194308)}function Qn(e,t){try{var n="",r=t;do n+=Oh(r),r=r.return;while(r);var l=n}catch(i){l=`
Error generating stack: `+i.message+`
`+i.stack}return{value:e,source:t,stack:l,digest:null}}function Pu(e,t,n){return{value:e,source:null,stack:n!=null?n:null,digest:t!=null?t:null}}function mo(e,t){try{console.error(t.value)}catch(n){setTimeout(function(){throw n})}}function Yc(e,t,n){n=st(-1,n),n.tag=3,n.payload={element:null};var r=t.value;return n.callback=function(){hi||(hi=!0,Co=r),mo(e,t)},n}function Kc(e,t,n){n=st(-1,n),n.tag=3;var r=e.type.getDerivedStateFromError;if(typeof r==="function"){var l=t.value;n.payload=function(){return r(l)},n.callback=function(){mo(e,t)}}var i=e.stateNode;return i!==null&&typeof i.componentDidCatch==="function"&&(n.callback=function(){mo(e,t),typeof r!=="function"&&(Tt===null?Tt=new Set([this]):Tt.add(this));var u=t.stack;this.componentDidCatch(t.value,{componentStack:u!==null?u:""})}),n}function tf(e,t,n){var r=e.pingCache;if(r===null){r=e.pingCache=new o1;var l=new Set;r.set(t,l)}else l=r.get(t),l===void 0&&(l=new Set,r.set(t,l));l.has(n)||(l.add(n),e=x1.bind(null,e,t,n),t.then(e,e))}function nf(e){do{var t;if(t=e.tag===13)t=e.memoizedState,t=t!==null?t.dehydrated!==null?!0:!1:!0;if(t)return e;e=e.return}while(e!==null);return null}function rf(e,t,n,r,l){if((e.mode&1)===0)return e===t?e.flags|=65536:(e.flags|=128,n.flags|=131072,n.flags&=-52805,n.tag===1&&(n.alternate===null?n.tag=17:(t=st(-1,1),t.tag=2,Lt(n,t,1))),n.lanes|=1),e;return e.flags|=65536,e.lanes=l,e}function me(e,t,n,r){t.child=e===null?xc(t,null,n,r):Un(t,e.child,n,r)}function lf(e,t,n,r,l){n=n.render;var i=t.ref;if(Mn(t,l),r=ra(e,t,n,r,i,l),n=la(),e!==null&&!ke)return t.updateQueue=e.updateQueue,t.flags&=-2053,e.lanes&=~l,pt(e,t,l);return V&&n&&Yo(t),t.flags|=1,me(e,t,r,l),t.child}function uf(e,t,n,r,l){if(e===null){var i=n.type;if(typeof i==="function"&&!ha(i)&&i.defaultProps===void 0&&n.compare===null&&n.defaultProps===void 0)return t.tag=15,t.type=i,Xc(e,t,i,r,l);return e=Xl(n.type,null,r,t,t.mode,l),e.ref=t.ref,e.return=t,t.child=e}if(i=e.child,(e.lanes&l)===0){var u=i.memoizedProps;if(n=n.compare,n=n!==null?n:Mr,n(u,r)&&e.ref===t.ref)return pt(e,t,l)}return t.flags|=1,e=Bt(i,r),e.ref=t.ref,e.return=t,t.child=e}function Xc(e,t,n,r,l){if(e!==null){var i=e.memoizedProps;if(Mr(i,r)&&e.ref===t.ref)if(ke=!1,t.pendingProps=r=i,(e.lanes&l)!==0)(e.flags&131072)!==0&&(ke=!0);else return t.lanes=e.lanes,pt(e,t,l)}return yo(e,t,n,r,l)}function Zc(e,t,n){var r=t.pendingProps,l=r.children,i=e!==null?e.memoizedState:null;if(r.mode==="hidden")if((t.mode&1)===0)t.memoizedState={baseLanes:0,cachePool:null,transitions:null},H(zn,Fe),Fe|=n;else{if((n&1073741824)===0)return e=i!==null?i.baseLanes|n:n,t.lanes=t.childLanes=1073741824,t.memoizedState={baseLanes:e,cachePool:null,transitions:null},t.updateQueue=null,H(zn,Fe),Fe|=e,null;t.memoizedState={baseLanes:0,cachePool:null,transitions:null},r=i!==null?i.baseLanes:n,H(zn,Fe),Fe|=r}else i!==null?(r=i.baseLanes|n,t.memoizedState=null):r=n,H(zn,Fe),Fe|=r;return me(e,t,l,n),t.child}function Jc(e,t){var n=t.ref;if(e===null&&n!==null||e!==null&&e.ref!==n)t.flags|=512,t.flags|=2097152}function yo(e,t,n,r,l){var i=Se(n)?bt:de.current;if(i=_n(t,i),Mn(t,l),n=ra(e,t,n,r,i,l),r=la(),e!==null&&!ke)return t.updateQueue=e.updateQueue,t.flags&=-2053,e.lanes&=~l,pt(e,t,l);return V&&r&&Yo(t),t.flags|=1,me(e,t,n,l),t.child}function of(e,t,n,r,l){if(Se(n)){var i=!0;li(t)}else i=!1;if(Mn(t,l),t.stateNode===null)Vl(e,t),Vc(t,n,r),ho(t,n,r,l),r=!0;else if(e===null){var{stateNode:u,memoizedProps:o}=t;u.props=o;var a=u.context,s=n.contextType;typeof s==="object"&&s!==null?s=Oe(s):(s=Se(n)?bt:de.current,s=_n(t,s));var h=n.getDerivedStateFromProps,m=typeof h==="function"||typeof u.getSnapshotBeforeUpdate==="function";m||typeof u.UNSAFE_componentWillReceiveProps!=="function"&&typeof u.componentWillReceiveProps!=="function"||(o!==r||a!==s)&&ef(t,u,r,s),wt=!1;var y=t.memoizedState;u.state=y,si(t,r,u,l),a=t.memoizedState,o!==r||y!==a||xe.current||wt?(typeof h==="function"&&(po(t,n,h,r),a=t.memoizedState),(o=wt||js(t,n,o,r,y,a,s))?(m||typeof u.UNSAFE_componentWillMount!=="function"&&typeof u.componentWillMount!=="function"||(typeof u.componentWillMount==="function"&&u.componentWillMount(),typeof u.UNSAFE_componentWillMount==="function"&&u.UNSAFE_componentWillMount()),typeof u.componentDidMount==="function"&&(t.flags|=4194308)):(typeof u.componentDidMount==="function"&&(t.flags|=4194308),t.memoizedProps=r,t.memoizedState=a),u.props=r,u.state=a,u.context=s,r=o):(typeof u.componentDidMount==="function"&&(t.flags|=4194308),r=!1)}else{u=t.stateNode,Nc(e,t),o=t.memoizedProps,s=t.type===t.elementType?o:Re(t.type,o),u.props=s,m=t.pendingProps,y=u.context,a=n.contextType,typeof a==="object"&&a!==null?a=Oe(a):(a=Se(n)?bt:de.current,a=_n(t,a));var x=n.getDerivedStateFromProps;(h=typeof x==="function"||typeof u.getSnapshotBeforeUpdate==="function")||typeof u.UNSAFE_componentWillReceiveProps!=="function"&&typeof u.componentWillReceiveProps!=="function"||(o!==m||y!==a)&&ef(t,u,r,a),wt=!1,y=t.memoizedState,u.state=y,si(t,r,u,l);var N=t.memoizedState;o!==m||y!==N||xe.current||wt?(typeof x==="function"&&(po(t,n,x,r),N=t.memoizedState),(s=wt||js(t,n,s,r,y,N,a)||!1)?(h||typeof u.UNSAFE_componentWillUpdate!=="function"&&typeof u.componentWillUpdate!=="function"||(typeof u.componentWillUpdate==="function"&&u.componentWillUpdate(r,N,a),typeof u.UNSAFE_componentWillUpdate==="function"&&u.UNSAFE_componentWillUpdate(r,N,a)),typeof u.componentDidUpdate==="function"&&(t.flags|=4),typeof u.getSnapshotBeforeUpdate==="function"&&(t.flags|=1024)):(typeof u.componentDidUpdate!=="function"||o===e.memoizedProps&&y===e.memoizedState||(t.flags|=4),typeof u.getSnapshotBeforeUpdate!=="function"||o===e.memoizedProps&&y===e.memoizedState||(t.flags|=1024),t.memoizedProps=r,t.memoizedState=N),u.props=r,u.state=N,u.context=a,r=s):(typeof u.componentDidUpdate!=="function"||o===e.memoizedProps&&y===e.memoizedState||(t.flags|=4),typeof u.getSnapshotBeforeUpdate!=="function"||o===e.memoizedProps&&y===e.memoizedState||(t.flags|=1024),r=!1)}return go(e,t,n,r,i,l)}function go(e,t,n,r,l,i){Jc(e,t);var u=(t.flags&128)!==0;if(!r&&!u)return l&&Ys(t,n,!1),pt(e,t,i);r=t.stateNode,a1.current=t;var o=u&&typeof n.getDerivedStateFromError!=="function"?null:r.render();return t.flags|=1,e!==null&&u?(t.child=Un(t,e.child,null,i),t.child=Un(t,null,o,i)):me(e,t,o,i),t.memoizedState=r.state,l&&Ys(t,n,!0),t.child}function Gc(e){var t=e.stateNode;t.pendingContext?Vs(e,t.pendingContext,t.pendingContext!==t.context):t.context&&Vs(e,t.context,!1),jo(e,t.containerInfo)}function af(e,t,n,r,l){return On(),Xo(l),t.flags|=256,me(e,t,n,r),t.child}function wo(e){return{baseLanes:e,cachePool:null,transitions:null}}function qc(e,t,n){var r=t.pendingProps,l=K.current,i=!1,u=(t.flags&128)!==0,o;if((o=u)||(o=e!==null&&e.memoizedState===null?!1:(l&2)!==0),o)i=!0,t.flags&=-129;else if(e===null||e.memoizedState!==null)l|=1;if(H(K,l&1),e===null){if(fo(t),e=t.memoizedState,e!==null&&(e=e.dehydrated,e!==null))return(t.mode&1)===0?t.lanes=1:e.data==="$!"?t.lanes=8:t.lanes=1073741824,null;return u=r.children,e=r.fallback,i?(r=t.mode,i=t.child,u={mode:"hidden",children:u},(r&1)===0&&i!==null?(i.childLanes=0,i.pendingProps=u):i=Li(u,r,0,null),e=qt(e,r,n,null),i.return=t,e.return=t,i.sibling=e,t.child=i,t.child.memoizedState=wo(n),t.memoizedState=vo,e):oa(t,u)}if(l=e.memoizedState,l!==null&&(o=l.dehydrated,o!==null))return s1(e,t,u,r,o,l,n);if(i){i=r.fallback,u=t.mode,l=e.child,o=l.sibling;var a={mode:"hidden",children:r.children};return(u&1)===0&&t.child!==l?(r=t.child,r.childLanes=0,r.pendingProps=a,t.deletions=null):(r=Bt(l,a),r.subtreeFlags=l.subtreeFlags&14680064),o!==null?i=Bt(o,i):(i=qt(i,u,n,null),i.flags|=2),i.return=t,r.return=t,r.sibling=i,t.child=r,r=i,i=t.child,u=e.child.memoizedState,u=u===null?wo(n):{baseLanes:u.baseLanes|n,cachePool:null,transitions:u.transitions},i.memoizedState=u,i.childLanes=e.childLanes&~n,t.memoizedState=vo,r}return i=e.child,e=i.sibling,r=Bt(i,{mode:"visible",children:r.children}),(t.mode&1)===0&&(r.lanes=n),r.return=t,r.sibling=null,e!==null&&(n=t.deletions,n===null?(t.deletions=[e],t.flags|=16):n.push(e)),t.child=r,t.memoizedState=null,r}function oa(e,t){return t=Li({mode:"visible",children:t},e.mode,0,null),t.return=e,e.child=t}function Dl(e,t,n,r){return r!==null&&Xo(r),Un(t,e.child,null,n),e=oa(t,t.pendingProps.children),e.flags|=2,t.memoizedState=null,e}function s1(e,t,n,r,l,i,u){if(n){if(t.flags&256)return t.flags&=-257,r=Pu(Error(S(422))),Dl(e,t,u,r);if(t.memoizedState!==null)return t.child=e.child,t.flags|=128,null;return i=r.fallback,l=t.mode,r=Li({mode:"visible",children:r.children},l,0,null),i=qt(i,l,u,null),i.flags|=2,r.return=t,i.return=t,r.sibling=i,t.child=r,(t.mode&1)!==0&&Un(t,e.child,null,u),t.child.memoizedState=wo(u),t.memoizedState=vo,i}if((t.mode&1)===0)return Dl(e,t,u,null);if(l.data==="$!"){if(r=l.nextSibling&&l.nextSibling.dataset,r)var o=r.dgst;return r=o,i=Error(S(419)),r=Pu(i,r,void 0),Dl(e,t,u,r)}if(o=(u&e.childLanes)!==0,ke||o){if(r=re,r!==null){switch(u&-u){case 4:l=2;break;case 16:l=8;break;case 64:case 128:case 256:case 512:case 1024:case 2048:case 4096:case 8192:case 16384:case 32768:case 65536:case 131072:case 262144:case 524288:case 1048576:case 2097152:case 4194304:case 8388608:case 16777216:case 33554432:case 67108864:l=32;break;case 536870912:l=268435456;break;default:l=0}l=(l&(r.suspendedLanes|u))!==0?0:l,l!==0&&l!==i.retryLane&&(i.retryLane=l,dt(e,l),Xe(r,e,l,-1))}return pa(),r=Pu(Error(S(421))),Dl(e,t,u,r)}if(l.data==="$?")return t.flags|=128,t.child=e.child,t=S1.bind(null,e),l._reactRetry=t,null;return e=i.treeContext,ze=zt(l.nextSibling),Le=t,V=!0,Ye=null,e!==null&&(Me[$e++]=ot,Me[$e++]=at,Me[$e++]=jt,ot=e.id,at=e.overflow,jt=t),t=oa(t,r.children),t.flags|=4096,t}function sf(e,t,n){e.lanes|=t;var r=e.alternate;r!==null&&(r.lanes|=t),co(e.return,t,n)}function Bu(e,t,n,r,l){var i=e.memoizedState;i===null?e.memoizedState={isBackwards:t,rendering:null,renderingStartTime:0,last:r,tail:n,tailMode:l}:(i.isBackwards=t,i.rendering=null,i.renderingStartTime=0,i.last=r,i.tail=n,i.tailMode=l)}function bc(e,t,n){var r=t.pendingProps,l=r.revealOrder,i=r.tail;if(me(e,t,r.children,n),r=K.current,(r&2)!==0)r=r&1|2,t.flags|=128;else{if(e!==null&&(e.flags&128)!==0)e:for(e=t.child;e!==null;){if(e.tag===13)e.memoizedState!==null&&sf(e,n,t);else if(e.tag===19)sf(e,n,t);else if(e.child!==null){e.child.return=e,e=e.child;continue}if(e===t)break e;for(;e.sibling===null;){if(e.return===null||e.return===t)break e;e=e.return}e.sibling.return=e.return,e=e.sibling}r&=1}if(H(K,r),(t.mode&1)===0)t.memoizedState=null;else switch(l){case"forwards":n=t.child;for(l=null;n!==null;)e=n.alternate,e!==null&&fi(e)===null&&(l=n),n=n.sibling;n=l,n===null?(l=t.child,t.child=null):(l=n.sibling,n.sibling=null),Bu(t,!1,l,n,i);break;case"backwards":n=null,l=t.child;for(t.child=null;l!==null;){if(e=l.alternate,e!==null&&fi(e)===null){t.child=l;break}e=l.sibling,l.sibling=n,n=l,l=e}Bu(t,!0,n,null,i);break;case"together":Bu(t,!1,null,null,void 0);break;default:t.memoizedState=null}return t.child}function Vl(e,t){(t.mode&1)===0&&e!==null&&(e.alternate=null,t.alternate=null,t.flags|=2)}function pt(e,t,n){if(e!==null&&(t.dependencies=e.dependencies),tn|=t.lanes,(n&t.childLanes)===0)return null;if(e!==null&&t.child!==e.child)throw Error(S(153));if(t.child!==null){e=t.child,n=Bt(e,e.pendingProps),t.child=n;for(n.return=t;e.sibling!==null;)e=e.sibling,n=n.sibling=Bt(e,e.pendingProps),n.return=t;n.sibling=null}return t.child}function f1(e,t,n){switch(t.tag){case 3:Gc(t),On();break;case 5:Ec(t);break;case 1:Se(t.type)&&li(t);break;case 4:jo(t,t.stateNode.containerInfo);break;case 10:var r=t.type._context,l=t.memoizedProps.value;H(oi,r._currentValue),r._currentValue=l;break;case 13:if(r=t.memoizedState,r!==null){if(r.dehydrated!==null)return H(K,K.current&1),t.flags|=128,null;if((n&t.child.childLanes)!==0)return qc(e,t,n);return H(K,K.current&1),e=pt(e,t,n),e!==null?e.sibling:null}H(K,K.current&1);break;case 19:if(r=(n&t.childLanes)!==0,(e.flags&128)!==0){if(r)return bc(e,t,n);t.flags|=128}if(l=t.memoizedState,l!==null&&(l.rendering=null,l.tail=null,l.lastEffect=null),H(K,K.current),r)break;else return null;case 22:case 23:return t.lanes=0,Zc(e,t,n)}return pt(e,t,n)}function ur(e,t){if(!V)switch(e.tailMode){case"hidden":t=e.tail;for(var n=null;t!==null;)t.alternate!==null&&(n=t),t=t.sibling;n===null?e.tail=null:n.sibling=null;break;case"collapsed":n=e.tail;for(var r=null;n!==null;)n.alternate!==null&&(r=n),n=n.sibling;r===null?t||e.tail===null?e.tail=null:e.tail.sibling=null:r.sibling=null}}function fe(e){var t=e.alternate!==null&&e.alternate.child===e.child,n=0,r=0;if(t)for(var l=e.child;l!==null;)n|=l.lanes|l.childLanes,r|=l.subtreeFlags&14680064,r|=l.flags&14680064,l.return=e,l=l.sibling;else for(l=e.child;l!==null;)n|=l.lanes|l.childLanes,r|=l.subtreeFlags,r|=l.flags,l.return=e,l=l.sibling;return e.subtreeFlags|=r,e.childLanes=n,t}function c1(e,t,n){var r=t.pendingProps;switch(Ko(t),t.tag){case 2:case 16:case 15:case 0:case 11:case 7:case 8:case 12:case 9:case 14:return fe(t),null;case 1:return Se(t.type)&&ri(),fe(t),null;case 3:if(r=t.stateNode,Hn(),I(xe),I(de),ta(),r.pendingContext&&(r.context=r.pendingContext,r.pendingContext=null),e===null||e.child===null)Ml(t)?t.flags|=4:e===null||e.memoizedState.isDehydrated&&(t.flags&256)===0||(t.flags|=1024,Ye!==null&&(Lo(Ye),Ye=null));return ko(e,t),fe(t),null;case 5:ea(t);var l=Jt(Ur.current);if(n=t.type,e!==null&&t.stateNode!=null)ed(e,t,n,r,l),e.ref!==t.ref&&(t.flags|=512,t.flags|=2097152);else{if(!r){if(t.stateNode===null)throw Error(S(166));return fe(t),null}if(e=Jt(nt.current),Ml(t)){r=t.stateNode,n=t.type;var i=t.memoizedProps;switch(r[et]=t,r[_r]=i,e=(t.mode&1)!==0,n){case"dialog":A("cancel",r),A("close",r);break;case"iframe":case"object":case"embed":A("load",r);break;case"video":case"audio":for(l=0;l<mr.length;l++)A(mr[l],r);break;case"source":A("error",r);break;case"img":case"image":case"link":A("error",r),A("load",r);break;case"details":A("toggle",r);break;case"input":vs(r,i),A("invalid",r);break;case"select":r._wrapperState={wasMultiple:!!i.multiple},A("invalid",r);break;case"textarea":ks(r,i),A("invalid",r)}Vu(n,i),l=null;for(var u in i)if(i.hasOwnProperty(u)){var o=i[u];u==="children"?typeof o==="string"?r.textContent!==o&&(i.suppressHydrationWarning!==!0&&Bl(r.textContent,o,e),l=["children",o]):typeof o==="number"&&r.textContent!==""+o&&(i.suppressHydrationWarning!==!0&&Bl(r.textContent,o,e),l=["children",""+o]):Cr.hasOwnProperty(u)&&o!=null&&u==="onScroll"&&A("scroll",r)}switch(n){case"input":Nl(r),ws(r,i,!0);break;case"textarea":Nl(r),xs(r);break;case"select":case"option":break;default:typeof i.onClick==="function"&&(r.onclick=ni)}r=l,t.updateQueue=r,r!==null&&(t.flags|=4)}else{u=l.nodeType===9?l:l.ownerDocument,e==="http://www.w3.org/1999/xhtml"&&(e=Tf(n)),e==="http://www.w3.org/1999/xhtml"?n==="script"?(e=u.createElement("div"),e.innerHTML="<script><\/script>",e=e.removeChild(e.firstChild)):typeof r.is==="string"?e=u.createElement(n,{is:r.is}):(e=u.createElement(n),n==="select"&&(u=e,r.multiple?u.multiple=!0:r.size&&(u.size=r.size))):e=u.createElementNS(e,n),e[et]=t,e[_r]=r,jc(e,t,!1,!1),t.stateNode=e;e:{switch(u=Yu(n,r),n){case"dialog":A("cancel",e),A("close",e),l=r;break;case"iframe":case"object":case"embed":A("load",e),l=r;break;case"video":case"audio":for(l=0;l<mr.length;l++)A(mr[l],e);l=r;break;case"source":A("error",e),l=r;break;case"img":case"image":case"link":A("error",e),A("load",e),l=r;break;case"details":A("toggle",e),l=r;break;case"input":vs(e,r),l=Qu(e,r),A("invalid",e);break;case"option":l=r;break;case"select":e._wrapperState={wasMultiple:!!r.multiple},l=Z({},r,{value:void 0}),A("invalid",e);break;case"textarea":ks(e,r),l=Iu(e,r),A("invalid",e);break;default:l=r}Vu(n,l),o=l;for(i in o)if(o.hasOwnProperty(i)){var a=o[i];i==="style"?Mf(e,a):i==="dangerouslySetInnerHTML"?(a=a?a.__html:void 0,a!=null&&Pf(e,a)):i==="children"?typeof a==="string"?(n!=="textarea"||a!=="")&&Fr(e,a):typeof a==="number"&&Fr(e,""+a):i!=="suppressContentEditableWarning"&&i!=="suppressHydrationWarning"&&i!=="autoFocus"&&(Cr.hasOwnProperty(i)?a!=null&&i==="onScroll"&&A("scroll",e):a!=null&&Bo(e,i,a,u))}switch(n){case"input":Nl(e),ws(e,r,!1);break;case"textarea":Nl(e),xs(e);break;case"option":r.value!=null&&e.setAttribute("value",""+Mt(r.value));break;case"select":e.multiple=!!r.multiple,i=r.value,i!=null?Ln(e,!!r.multiple,i,!1):r.defaultValue!=null&&Ln(e,!!r.multiple,r.defaultValue,!0);break;default:typeof l.onClick==="function"&&(e.onclick=ni)}switch(n){case"button":case"input":case"select":case"textarea":r=!!r.autoFocus;break e;case"img":r=!0;break e;default:r=!1}}r&&(t.flags|=4)}t.ref!==null&&(t.flags|=512,t.flags|=2097152)}return fe(t),null;case 6:if(e&&t.stateNode!=null)td(e,t,e.memoizedProps,r);else{if(typeof r!=="string"&&t.stateNode===null)throw Error(S(166));if(n=Jt(Ur.current),Jt(nt.current),Ml(t)){if(r=t.stateNode,n=t.memoizedProps,r[et]=t,i=r.nodeValue!==n){if(e=Le,e!==null)switch(e.tag){case 3:Bl(r.nodeValue,n,(e.mode&1)!==0);break;case 5:e.memoizedProps.suppressHydrationWarning!==!0&&Bl(r.nodeValue,n,(e.mode&1)!==0)}}i&&(t.flags|=4)}else r=(n.nodeType===9?n:n.ownerDocument).createTextNode(r),r[et]=t,t.stateNode=r}return fe(t),null;case 13:if(I(K),r=t.memoizedState,e===null||e.memoizedState!==null&&e.memoizedState.dehydrated!==null){if(V&&ze!==null&&(t.mode&1)!==0&&(t.flags&128)===0)wc(),On(),t.flags|=98560,i=!1;else if(i=Ml(t),r!==null&&r.dehydrated!==null){if(e===null){if(!i)throw Error(S(318));if(i=t.memoizedState,i=i!==null?i.dehydrated:null,!i)throw Error(S(317));i[et]=t}else On(),(t.flags&128)===0&&(t.memoizedState=null),t.flags|=4;fe(t),i=!1}else Ye!==null&&(Lo(Ye),Ye=null),i=!0;if(!i)return t.flags&65536?t:null}if((t.flags&128)!==0)return t.lanes=n,t;return r=r!==null,r!==(e!==null&&e.memoizedState!==null)&&r&&(t.child.flags|=8192,(t.mode&1)!==0&&(e===null||(K.current&1)!==0?ee===0&&(ee=3):pa())),t.updateQueue!==null&&(t.flags|=4),fe(t),null;case 4:return Hn(),ko(e,t),e===null&&$r(t.stateNode.containerInfo),fe(t),null;case 10:return Go(t.type._context),fe(t),null;case 17:return Se(t.type)&&ri(),fe(t),null;case 19:if(I(K),i=t.memoizedState,i===null)return fe(t),null;if(r=(t.flags&128)!==0,u=i.rendering,u===null)if(r)ur(i,!1);else{if(ee!==0||e!==null&&(e.flags&128)!==0)for(e=t.child;e!==null;){if(u=fi(e),u!==null){t.flags|=128,ur(i,!1),r=u.updateQueue,r!==null&&(t.updateQueue=r,t.flags|=4),t.subtreeFlags=0,r=n;for(n=t.child;n!==null;)i=n,e=r,i.flags&=14680066,u=i.alternate,u===null?(i.childLanes=0,i.lanes=e,i.child=null,i.subtreeFlags=0,i.memoizedProps=null,i.memoizedState=null,i.updateQueue=null,i.dependencies=null,i.stateNode=null):(i.childLanes=u.childLanes,i.lanes=u.lanes,i.child=u.child,i.subtreeFlags=0,i.deletions=null,i.memoizedProps=u.memoizedProps,i.memoizedState=u.memoizedState,i.updateQueue=u.updateQueue,i.type=u.type,e=u.dependencies,i.dependencies=e===null?null:{lanes:e.lanes,firstContext:e.firstContext}),n=n.sibling;return H(K,K.current&1|2),t.child}e=e.sibling}i.tail!==null&&q()>Wn&&(t.flags|=128,r=!0,ur(i,!1),t.lanes=4194304)}else{if(!r)if(e=fi(u),e!==null){if(t.flags|=128,r=!0,n=e.updateQueue,n!==null&&(t.updateQueue=n,t.flags|=4),ur(i,!0),i.tail===null&&i.tailMode==="hidden"&&!u.alternate&&!V)return fe(t),null}else 2*q()-i.renderingStartTime>Wn&&n!==1073741824&&(t.flags|=128,r=!0,ur(i,!1),t.lanes=4194304);i.isBackwards?(u.sibling=t.child,t.child=u):(n=i.last,n!==null?n.sibling=u:t.child=u,i.last=u)}if(i.tail!==null)return t=i.tail,i.rendering=t,i.tail=t.sibling,i.renderingStartTime=q(),t.sibling=null,n=K.current,H(K,r?n&1|2:n&1),t;return fe(t),null;case 22:case 23:return da(),r=t.memoizedState!==null,e!==null&&e.memoizedState!==null!==r&&(t.flags|=8192),r&&(t.mode&1)!==0?(Fe&1073741824)!==0&&(fe(t),t.subtreeFlags&6&&(t.flags|=8192)):fe(t),null;case 24:return null;case 25:return null}throw Error(S(156,t.tag))}function d1(e,t){switch(Ko(t),t.tag){case 1:return Se(t.type)&&ri(),e=t.flags,e&65536?(t.flags=e&-65537|128,t):null;case 3:return Hn(),I(xe),I(de),ta(),e=t.flags,(e&65536)!==0&&(e&128)===0?(t.flags=e&-65537|128,t):null;case 5:return ea(t),null;case 13:if(I(K),e=t.memoizedState,e!==null&&e.dehydrated!==null){if(t.alternate===null)throw Error(S(340));On()}return e=t.flags,e&65536?(t.flags=e&-65537|128,t):null;case 19:return I(K),null;case 4:return Hn(),null;case 10:return Go(t.type._context),null;case 22:case 23:return da(),null;case 24:return null;default:return null}}function Fn(e,t){var n=e.ref;if(n!==null)if(typeof n==="function")try{n(null)}catch(r){G(e,t,r)}else n.current=null}function xo(e,t,n){try{n()}catch(r){G(e,t,r)}}function h1(e,t){if(ro=jl,e=uc(),Vo(e)){if("selectionStart"in e)var n={start:e.selectionStart,end:e.selectionEnd};else e:{n=(n=e.ownerDocument)&&n.defaultView||window;var r=n.getSelection&&n.getSelection();if(r&&r.rangeCount!==0){n=r.anchorNode;var{anchorOffset:l,focusNode:i}=r;r=r.focusOffset;try{n.nodeType,i.nodeType}catch(w){n=null;break e}var u=0,o=-1,a=-1,s=0,h=0,m=e,y=null;t:for(;;){for(var x;;){if(m!==n||l!==0&&m.nodeType!==3||(o=u+l),m!==i||r!==0&&m.nodeType!==3||(a=u+r),m.nodeType===3&&(u+=m.nodeValue.length),(x=m.firstChild)===null)break;y=m,m=x}for(;;){if(m===e)break t;if(y===n&&++s===l&&(o=u),y===i&&++h===r&&(a=u),(x=m.nextSibling)!==null)break;m=y,y=m.parentNode}m=x}n=o===-1||a===-1?null:{start:o,end:a}}else n=null}n=n||{start:0,end:0}}else n=null;lo={focusedElem:e,selectionRange:n},jl=!1;for(z=t;z!==null;)if(t=z,e=t.child,(t.subtreeFlags&1028)!==0&&e!==null)e.return=t,z=e;else for(;z!==null;){t=z;try{var N=t.alternate;if((t.flags&1024)!==0)switch(t.tag){case 0:case 11:case 15:break;case 1:if(N!==null){var{memoizedProps:g,memoizedState:L}=N,c=t.stateNode,f=c.getSnapshotBeforeUpdate(t.elementType===t.type?g:Re(t.type,g),L);c.__reactInternalSnapshotBeforeUpdate=f}break;case 3:var p=t.stateNode.containerInfo;p.nodeType===1?p.textContent="":p.nodeType===9&&p.documentElement&&p.removeChild(p.documentElement);break;case 5:case 6:case 4:case 17:break;default:throw Error(S(163))}}catch(w){G(t,t.return,w)}if(e=t.sibling,e!==null){e.return=t.return,z=e;break}z=t.return}return N=ff,ff=!1,N}function Sr(e,t,n){var r=t.updateQueue;if(r=r!==null?r.lastEffect:null,r!==null){var l=r=r.next;do{if((l.tag&e)===e){var i=l.destroy;l.destroy=void 0,i!==void 0&&xo(t,n,i)}l=l.next}while(l!==r)}}function Fi(e,t){if(t=t.updateQueue,t=t!==null?t.lastEffect:null,t!==null){var n=t=t.next;do{if((n.tag&e)===e){var r=n.create;n.destroy=r()}n=n.next}while(n!==t)}}function So(e){var t=e.ref;if(t!==null){var n=e.stateNode;switch(e.tag){case 5:e=n;break;default:e=n}typeof t==="function"?t(e):t.current=e}}function nd(e){var t=e.alternate;t!==null&&(e.alternate=null,nd(t)),e.child=null,e.deletions=null,e.sibling=null,e.tag===5&&(t=e.stateNode,t!==null&&(delete t[et],delete t[_r],delete t[oo],delete t[Gm],delete t[qm])),e.stateNode=null,e.return=null,e.dependencies=null,e.memoizedProps=null,e.memoizedState=null,e.pendingProps=null,e.stateNode=null,e.updateQueue=null}function rd(e){return e.tag===5||e.tag===3||e.tag===4}function cf(e){e:for(;;){for(;e.sibling===null;){if(e.return===null||rd(e.return))return null;e=e.return}e.sibling.return=e.return;for(e=e.sibling;e.tag!==5&&e.tag!==6&&e.tag!==18;){if(e.flags&2)continue e;if(e.child===null||e.tag===4)continue e;else e.child.return=e,e=e.child}if(!(e.flags&2))return e.stateNode}}function No(e,t,n){var r=e.tag;if(r===5||r===6)e=e.stateNode,t?n.nodeType===8?n.parentNode.insertBefore(e,t):n.insertBefore(e,t):(n.nodeType===8?(t=n.parentNode,t.insertBefore(e,n)):(t=n,t.appendChild(e)),n=n._reactRootContainer,n!==null&&n!==void 0||t.onclick!==null||(t.onclick=ni));else if(r!==4&&(e=e.child,e!==null))for(No(e,t,n),e=e.sibling;e!==null;)No(e,t,n),e=e.sibling}function Eo(e,t,n){var r=e.tag;if(r===5||r===6)e=e.stateNode,t?n.insertBefore(e,t):n.appendChild(e);else if(r!==4&&(e=e.child,e!==null))for(Eo(e,t,n),e=e.sibling;e!==null;)Eo(e,t,n),e=e.sibling}function gt(e,t,n){for(n=n.child;n!==null;)ld(e,t,n),n=n.sibling}function ld(e,t,n){if(tt&&typeof tt.onCommitFiberUnmount==="function")try{tt.onCommitFiberUnmount(vi,n)}catch(o){}switch(n.tag){case 5:ce||Fn(n,t);case 6:var r=le,l=Ve;le=null,gt(e,t,n),le=r,Ve=l,le!==null&&(Ve?(e=le,n=n.stateNode,e.nodeType===8?e.parentNode.removeChild(n):e.removeChild(n)):le.removeChild(n.stateNode));break;case 18:le!==null&&(Ve?(e=le,n=n.stateNode,e.nodeType===8?Eu(e.parentNode,n):e.nodeType===1&&Eu(e,n),Pr(e)):Eu(le,n.stateNode));break;case 4:r=le,l=Ve,le=n.stateNode.containerInfo,Ve=!0,gt(e,t,n),le=r,Ve=l;break;case 0:case 11:case 14:case 15:if(!ce&&(r=n.updateQueue,r!==null&&(r=r.lastEffect,r!==null))){l=r=r.next;do{var i=l,u=i.destroy;i=i.tag,u!==void 0&&((i&2)!==0?xo(n,t,u):(i&4)!==0&&xo(n,t,u)),l=l.next}while(l!==r)}gt(e,t,n);break;case 1:if(!ce&&(Fn(n,t),r=n.stateNode,typeof r.componentWillUnmount==="function"))try{r.props=n.memoizedProps,r.state=n.memoizedState,r.componentWillUnmount()}catch(o){G(n,t,o)}gt(e,t,n);break;case 21:gt(e,t,n);break;case 22:n.mode&1?(ce=(r=ce)||n.memoizedState!==null,gt(e,t,n),ce=r):gt(e,t,n);break;default:gt(e,t,n)}}function df(e){var t=e.updateQueue;if(t!==null){e.updateQueue=null;var n=e.stateNode;n===null&&(n=e.stateNode=new p1),t.forEach(function(r){var l=N1.bind(null,e,r);n.has(r)||(n.add(r),r.then(l,l))})}}function Ie(e,t){var n=t.deletions;if(n!==null)for(var r=0;r<n.length;r++){var l=n[r];try{var i=e,u=t,o=u;e:for(;o!==null;){switch(o.tag){case 5:le=o.stateNode,Ve=!1;break e;case 3:le=o.stateNode.containerInfo,Ve=!0;break e;case 4:le=o.stateNode.containerInfo,Ve=!0;break e}o=o.return}if(le===null)throw Error(S(160));ld(i,u,l),le=null,Ve=!1;var a=l.alternate;a!==null&&(a.return=null),l.return=null}catch(s){G(l,t,s)}}if(t.subtreeFlags&12854)for(t=t.child;t!==null;)id(t,e),t=t.sibling}function id(e,t){var{alternate:n,flags:r}=e;switch(e.tag){case 0:case 11:case 14:case 15:if(Ie(t,e),be(e),r&4){try{Sr(3,e,e.return),Fi(3,e)}catch(g){G(e,e.return,g)}try{Sr(5,e,e.return)}catch(g){G(e,e.return,g)}}break;case 1:Ie(t,e),be(e),r&512&&n!==null&&Fn(n,n.return);break;case 5:if(Ie(t,e),be(e),r&512&&n!==null&&Fn(n,n.return),e.flags&32){var l=e.stateNode;try{Fr(l,"")}catch(g){G(e,e.return,g)}}if(r&4&&(l=e.stateNode,l!=null)){var i=e.memoizedProps,u=n!==null?n.memoizedProps:i,o=e.type,a=e.updateQueue;if(e.updateQueue=null,a!==null)try{o==="input"&&i.type==="radio"&&i.name!=null&&zf(l,i),Yu(o,u);var s=Yu(o,i);for(u=0;u<a.length;u+=2){var h=a[u],m=a[u+1];h==="style"?Mf(l,m):h==="dangerouslySetInnerHTML"?Pf(l,m):h==="children"?Fr(l,m):Bo(l,h,m,s)}switch(o){case"input":Wu(l,i);break;case"textarea":Lf(l,i);break;case"select":var y=l._wrapperState.wasMultiple;l._wrapperState.wasMultiple=!!i.multiple;var x=i.value;x!=null?Ln(l,!!i.multiple,x,!1):y!==!!i.multiple&&(i.defaultValue!=null?Ln(l,!!i.multiple,i.defaultValue,!0):Ln(l,!!i.multiple,i.multiple?[]:"",!1))}l[_r]=i}catch(g){G(e,e.return,g)}}break;case 6:if(Ie(t,e),be(e),r&4){if(e.stateNode===null)throw Error(S(162));l=e.stateNode,i=e.memoizedProps;try{l.nodeValue=i}catch(g){G(e,e.return,g)}}break;case 3:if(Ie(t,e),be(e),r&4&&n!==null&&n.memoizedState.isDehydrated)try{Pr(t.containerInfo)}catch(g){G(e,e.return,g)}break;case 4:Ie(t,e),be(e);break;case 13:Ie(t,e),be(e),l=e.child,l.flags&8192&&(i=l.memoizedState!==null,l.stateNode.isHidden=i,!i||l.alternate!==null&&l.alternate.memoizedState!==null||(fa=q())),r&4&&df(e);break;case 22:if(h=n!==null&&n.memoizedState!==null,e.mode&1?(ce=(s=ce)||h,Ie(t,e),ce=s):Ie(t,e),be(e),r&8192){if(s=e.memoizedState!==null,(e.stateNode.isHidden=s)&&!h&&(e.mode&1)!==0)for(z=e,h=e.child;h!==null;){for(m=z=h;z!==null;){switch(y=z,x=y.child,y.tag){case 0:case 11:case 14:case 15:Sr(4,y,y.return);break;case 1:Fn(y,y.return);var N=y.stateNode;if(typeof N.componentWillUnmount==="function"){r=y,n=y.return;try{t=r,N.props=t.memoizedProps,N.state=t.memoizedState,N.componentWillUnmount()}catch(g){G(r,n,g)}}break;case 5:Fn(y,y.return);break;case 22:if(y.memoizedState!==null){hf(m);continue}}x!==null?(x.return=y,z=x):hf(m)}h=h.sibling}e:for(h=null,m=e;;){if(m.tag===5){if(h===null){h=m;try{l=m.stateNode,s?(i=l.style,typeof i.setProperty==="function"?i.setProperty("display","none","important"):i.display="none"):(o=m.stateNode,a=m.memoizedProps.style,u=a!==void 0&&a!==null&&a.hasOwnProperty("display")?a.display:null,o.style.display=Bf("display",u))}catch(g){G(e,e.return,g)}}}else if(m.tag===6){if(h===null)try{m.stateNode.nodeValue=s?"":m.memoizedProps}catch(g){G(e,e.return,g)}}else if((m.tag!==22&&m.tag!==23||m.memoizedState===null||m===e)&&m.child!==null){m.child.return=m,m=m.child;continue}if(m===e)break e;for(;m.sibling===null;){if(m.return===null||m.return===e)break e;h===m&&(h=null),m=m.return}h===m&&(h=null),m.sibling.return=m.return,m=m.sibling}}break;case 19:Ie(t,e),be(e),r&4&&df(e);break;case 21:break;default:Ie(t,e),be(e)}}function be(e){var t=e.flags;if(t&2){try{e:{for(var n=e.return;n!==null;){if(rd(n)){var r=n;break e}n=n.return}throw Error(S(160))}switch(r.tag){case 5:var l=r.stateNode;r.flags&32&&(Fr(l,""),r.flags&=-33);var i=cf(e);Eo(e,i,l);break;case 3:case 4:var u=r.stateNode.containerInfo,o=cf(e);No(e,o,u);break;default:throw Error(S(161))}}catch(a){G(e,e.return,a)}e.flags&=-3}t&4096&&(e.flags&=-4097)}function m1(e,t,n){z=e,ud(e,t,n)}function ud(e,t,n){for(var r=(e.mode&1)!==0;z!==null;){var l=z,i=l.child;if(l.tag===22&&r){var u=l.memoizedState!==null||_l;if(!u){var o=l.alternate,a=o!==null&&o.memoizedState!==null||ce;o=_l;var s=ce;if(_l=u,(ce=a)&&!s)for(z=l;z!==null;)u=z,a=u.child,u.tag===22&&u.memoizedState!==null?mf(l):a!==null?(a.return=u,z=a):mf(l);for(;i!==null;)z=i,ud(i,t,n),i=i.sibling;z=l,_l=o,ce=s}pf(e,t,n)}else(l.subtreeFlags&8772)!==0&&i!==null?(i.return=l,z=i):pf(e,t,n)}}function pf(e){for(;z!==null;){var t=z;if((t.flags&8772)!==0){var n=t.alternate;try{if((t.flags&8772)!==0)switch(t.tag){case 0:case 11:case 15:ce||Fi(5,t);break;case 1:var r=t.stateNode;if(t.flags&4&&!ce)if(n===null)r.componentDidMount();else{var l=t.elementType===t.type?n.memoizedProps:Re(t.type,n.memoizedProps);r.componentDidUpdate(l,n.memoizedState,r.__reactInternalSnapshotBeforeUpdate)}var i=t.updateQueue;i!==null&&Gs(t,i,r);break;case 3:var u=t.updateQueue;if(u!==null){if(n=null,t.child!==null)switch(t.child.tag){case 5:n=t.child.stateNode;break;case 1:n=t.child.stateNode}Gs(t,u,n)}break;case 5:var o=t.stateNode;if(n===null&&t.flags&4){n=o;var a=t.memoizedProps;switch(t.type){case"button":case"input":case"select":case"textarea":a.autoFocus&&n.focus();break;case"img":a.src&&(n.src=a.src)}}break;case 6:break;case 4:break;case 12:break;case 13:if(t.memoizedState===null){var s=t.alternate;if(s!==null){var h=s.memoizedState;if(h!==null){var m=h.dehydrated;m!==null&&Pr(m)}}}break;case 19:case 17:case 21:case 22:case 23:case 25:break;default:throw Error(S(163))}ce||t.flags&512&&So(t)}catch(y){G(t,t.return,y)}}if(t===e){z=null;break}if(n=t.sibling,n!==null){n.return=t.return,z=n;break}z=t.return}}function hf(e){for(;z!==null;){var t=z;if(t===e){z=null;break}var n=t.sibling;if(n!==null){n.return=t.return,z=n;break}z=t.return}}function mf(e){for(;z!==null;){var t=z;try{switch(t.tag){case 0:case 11:case 15:var n=t.return;try{Fi(4,t)}catch(a){G(t,n,a)}break;case 1:var r=t.stateNode;if(typeof r.componentDidMount==="function"){var l=t.return;try{r.componentDidMount()}catch(a){G(t,l,a)}}var i=t.return;try{So(t)}catch(a){G(t,i,a)}break;case 5:var u=t.return;try{So(t)}catch(a){G(t,u,a)}}}catch(a){G(t,t.return,a)}if(t===e){z=null;break}var o=t.sibling;if(o!==null){o.return=t.return,z=o;break}z=t.return}}function ye(){return($&6)!==0?q():Yl!==-1?Yl:Yl=q()}function Pt(e){if((e.mode&1)===0)return 1;if(($&2)!==0&&ie!==0)return ie&-ie;if(jm.transition!==null)return Kl===0&&(Kl=Vf()),Kl;if(e=_,e!==0)return e;return e=window.event,e=e===void 0?16:qf(e.type),e}function Xe(e,t,n,r){if(50<Er)throw Er=0,Fo=null,Error(S(185));if(Ir(e,n,r),($&2)===0||e!==re)e===re&&(($&2)===0&&(zi|=n),ee===4&&xt(e,ie)),Ne(e,r),n===1&&$===0&&(t.mode&1)===0&&(Wn=q()+500,Ni&&Ot())}function Ne(e,t){var n=e.callbackNode;tm(e,t);var r=bl(e,e===re?ie:0);if(r===0)n!==null&&Es(n),e.callbackNode=null,e.callbackPriority=0;else if(t=r&-r,e.callbackPriority!==t){if(n!=null&&Es(n),t===1)e.tag===0?bm(yf.bind(null,e)):yc(yf.bind(null,e)),Zm(function(){($&6)===0&&Ot()}),n=null;else{switch(Yf(r)){case 1:n=Oo;break;case 4:n=If;break;case 16:n=ql;break;case 536870912:n=Rf;break;default:n=ql}n=hd(n,od.bind(null,e))}e.callbackPriority=t,e.callbackNode=n}}function od(e,t){if(Yl=-1,Kl=0,($&6)!==0)throw Error(S(327));var n=e.callbackNode;if($n()&&e.callbackNode!==n)return null;var r=bl(e,e===re?ie:0);if(r===0)return null;if((r&30)!==0||(r&e.expiredLanes)!==0||t)t=yi(e,r);else{t=r;var l=$;$|=2;var i=sd();if(re!==e||ie!==t)it=null,Wn=q()+500,Gt(e,t);do try{w1();break}catch(o){ad(e,o)}while(1);Jo(),pi.current=i,$=l,b!==null?t=0:(re=null,ie=0,t=ee)}if(t!==0){if(t===2&&(l=Gu(e),l!==0&&(r=l,t=zo(e,l))),t===1)throw n=Ar,Gt(e,0),xt(e,r),Ne(e,q()),n;if(t===6)xt(e,r);else{if(l=e.current.alternate,(r&30)===0&&!g1(l)&&(t=yi(e,r),t===2&&(i=Gu(e),i!==0&&(r=i,t=zo(e,i))),t===1))throw n=Ar,Gt(e,0),xt(e,r),Ne(e,q()),n;switch(e.finishedWork=l,e.finishedLanes=r,t){case 0:case 1:throw Error(S(345));case 2:Kt(e,we,it);break;case 3:if(xt(e,r),(r&130023424)===r&&(t=fa+500-q(),10<t)){if(bl(e,0)!==0)break;if(l=e.suspendedLanes,(l&r)!==r){ye(),e.pingedLanes|=e.suspendedLanes&l;break}e.timeoutHandle=uo(Kt.bind(null,e,we,it),t);break}Kt(e,we,it);break;case 4:if(xt(e,r),(r&4194240)===r)break;t=e.eventTimes;for(l=-1;0<r;){var u=31-Ke(r);i=1<<u,u=t[u],u>l&&(l=u),r&=~i}if(r=l,r=q()-r,r=(120>r?120:480>r?480:1080>r?1080:1920>r?1920:3000>r?3000:4320>r?4320:1960*y1(r/1960))-r,10<r){e.timeoutHandle=uo(Kt.bind(null,e,we,it),r);break}Kt(e,we,it);break;case 5:Kt(e,we,it);break;default:throw Error(S(329))}}}return Ne(e,q()),e.callbackNode===n?od.bind(null,e):null}function zo(e,t){var n=Nr;return e.current.memoizedState.isDehydrated&&(Gt(e,t).flags|=256),e=yi(e,t),e!==2&&(t=we,we=n,t!==null&&Lo(t)),e}function Lo(e){we===null?we=e:we.push.apply(we,e)}function g1(e){for(var t=e;;){if(t.flags&16384){var n=t.updateQueue;if(n!==null&&(n=n.stores,n!==null))for(var r=0;r<n.length;r++){var l=n[r],i=l.getSnapshot;l=l.value;try{if(!Ze(i(),l))return!1}catch(u){return!1}}}if(n=t.child,t.subtreeFlags&16384&&n!==null)n.return=t,t=n;else{if(t===e)break;for(;t.sibling===null;){if(t.return===null||t.return===e)return!0;t=t.return}t.sibling.return=t.return,t=t.sibling}}return!0}function xt(e,t){t&=~sa,t&=~zi,e.suspendedLanes|=t,e.pingedLanes&=~t;for(e=e.expirationTimes;0<t;){var n=31-Ke(t),r=1<<n;e[n]=-1,t&=~r}}function yf(e){if(($&6)!==0)throw Error(S(327));$n();var t=bl(e,0);if((t&1)===0)return Ne(e,q()),null;var n=yi(e,t);if(e.tag!==0&&n===2){var r=Gu(e);r!==0&&(t=r,n=zo(e,r))}if(n===1)throw n=Ar,Gt(e,0),xt(e,t),Ne(e,q()),n;if(n===6)throw Error(S(345));return e.finishedWork=e.current.alternate,e.finishedLanes=t,Kt(e,we,it),Ne(e,q()),null}function ca(e,t){var n=$;$|=1;try{return e(t)}finally{$=n,$===0&&(Wn=q()+500,Ni&&Ot())}}function nn(e){Nt!==null&&Nt.tag===0&&($&6)===0&&$n();var t=$;$|=1;var n=_e.transition,r=_;try{if(_e.transition=null,_=1,e)return e()}finally{_=r,_e.transition=n,$=t,($&6)===0&&Ot()}}function da(){Fe=zn.current,I(zn)}function Gt(e,t){e.finishedWork=null,e.finishedLanes=0;var n=e.timeoutHandle;if(n!==-1&&(e.timeoutHandle=-1,Xm(n)),b!==null)for(n=b.return;n!==null;){var r=n;switch(Ko(r),r.tag){case 1:r=r.type.childContextTypes,r!==null&&r!==void 0&&ri();break;case 3:Hn(),I(xe),I(de),ta();break;case 5:ea(r);break;case 4:Hn();break;case 13:I(K);break;case 19:I(K);break;case 10:Go(r.type._context);break;case 22:case 23:da()}n=n.return}if(re=e,b=e=Bt(e.current,null),ie=Fe=t,ee=0,Ar=null,sa=zi=tn=0,we=Nr=null,Zt!==null){for(t=0;t<Zt.length;t++)if(n=Zt[t],r=n.interleaved,r!==null){n.interleaved=null;var l=r.next,i=n.pending;if(i!==null){var u=i.next;i.next=l,r.next=u}n.pending=r}Zt=null}return e}function ad(e,t){do{var n=b;try{if(Jo(),Il.current=di,ci){for(var r=X.memoizedState;r!==null;){var l=r.queue;l!==null&&(l.pending=null),r=r.next}ci=!1}if(en=0,ne=j=X=null,xr=!1,Hr=0,aa.current=null,n===null||n.return===null){ee=1,Ar=t,b=null;break}e:{var i=e,u=n.return,o=n,a=t;if(t=ie,o.flags|=32768,a!==null&&typeof a==="object"&&typeof a.then==="function"){var s=a,h=o,m=h.tag;if((h.mode&1)===0&&(m===0||m===11||m===15)){var y=h.alternate;y?(h.updateQueue=y.updateQueue,h.memoizedState=y.memoizedState,h.lanes=y.lanes):(h.updateQueue=null,h.memoizedState=null)}var x=nf(u);if(x!==null){x.flags&=-257,rf(x,u,o,i,t),x.mode&1&&tf(i,s,t),t=x,a=s;var N=t.updateQueue;if(N===null){var g=new Set;g.add(a),t.updateQueue=g}else N.add(a);break e}else{if((t&1)===0){tf(i,s,t),pa();break e}a=Error(S(426))}}else if(V&&o.mode&1){var L=nf(u);if(L!==null){(L.flags&65536)===0&&(L.flags|=256),rf(L,u,o,i,t),Xo(Qn(a,o));break e}}i=a=Qn(a,o),ee!==4&&(ee=2),Nr===null?Nr=[i]:Nr.push(i),i=u;do{switch(i.tag){case 3:i.flags|=65536,t&=-t,i.lanes|=t;var c=Yc(i,a,t);Js(i,c);break e;case 1:o=a;var{type:f,stateNode:p}=i;if((i.flags&128)===0&&(typeof f.getDerivedStateFromError==="function"||p!==null&&typeof p.componentDidCatch==="function"&&(Tt===null||!Tt.has(p)))){i.flags|=65536,t&=-t,i.lanes|=t;var w=Kc(i,o,t);Js(i,w);break e}}i=i.return}while(i!==null)}cd(n)}catch(C){t=C,b===n&&n!==null&&(b=n=n.return);continue}break}while(1)}function sd(){var e=pi.current;return pi.current=di,e===null?di:e}function pa(){if(ee===0||ee===3||ee===2)ee=4;re===null||(tn&268435455)===0&&(zi&268435455)===0||xt(re,ie)}function yi(e,t){var n=$;$|=2;var r=sd();if(re!==e||ie!==t)it=null,Gt(e,t);do try{v1();break}catch(l){ad(e,l)}while(1);if(Jo(),$=n,pi.current=r,b!==null)throw Error(S(261));return re=null,ie=0,ee}function v1(){for(;b!==null;)fd(b)}function w1(){for(;b!==null&&!Kh();)fd(b)}function fd(e){var t=pd(e.alternate,e,Fe);e.memoizedProps=e.pendingProps,t===null?cd(e):b=t,aa.current=null}function cd(e){var t=e;do{var n=t.alternate;if(e=t.return,(t.flags&32768)===0){if(n=c1(n,t,Fe),n!==null){b=n;return}}else{if(n=d1(n,t),n!==null){n.flags&=32767,b=n;return}if(e!==null)e.flags|=32768,e.subtreeFlags=0,e.deletions=null;else{ee=6,b=null;return}}if(t=t.sibling,t!==null){b=t;return}b=t=e}while(t!==null);ee===0&&(ee=5)}function Kt(e,t,n){var r=_,l=_e.transition;try{_e.transition=null,_=1,k1(e,t,n,r)}finally{_e.transition=l,_=r}return null}function k1(e,t,n,r){do $n();while(Nt!==null);if(($&6)!==0)throw Error(S(327));n=e.finishedWork;var l=e.finishedLanes;if(n===null)return null;if(e.finishedWork=null,e.finishedLanes=0,n===e.current)throw Error(S(177));e.callbackNode=null,e.callbackPriority=0;var i=n.lanes|n.childLanes;if(nm(e,i),e===re&&(b=re=null,ie=0),(n.subtreeFlags&2064)===0&&(n.flags&2064)===0||Ol||(Ol=!0,hd(ql,function(){return $n(),null})),i=(n.flags&15990)!==0,(n.subtreeFlags&15990)!==0||i){i=_e.transition,_e.transition=null;var u=_;_=1;var o=$;$|=4,aa.current=null,h1(e,n),id(n,e),Im(lo),jl=!!ro,lo=ro=null,e.current=n,m1(n,e,l),Xh(),$=o,_=u,_e.transition=i}else e.current=n;if(Ol&&(Ol=!1,Nt=e,mi=l),i=e.pendingLanes,i===0&&(Tt=null),Gh(n.stateNode,r),Ne(e,q()),t!==null)for(r=e.onRecoverableError,n=0;n<t.length;n++)l=t[n],r(l.value,{componentStack:l.stack,digest:l.digest});if(hi)throw hi=!1,e=Co,Co=null,e;return(mi&1)!==0&&e.tag!==0&&$n(),i=e.pendingLanes,(i&1)!==0?e===Fo?Er++:(Er=0,Fo=e):Er=0,Ot(),null}function $n(){if(Nt!==null){var e=Yf(mi),t=_e.transition,n=_;try{if(_e.transition=null,_=16>e?16:e,Nt===null)var r=!1;else{if(e=Nt,Nt=null,mi=0,($&6)!==0)throw Error(S(331));var l=$;$|=4;for(z=e.current;z!==null;){var i=z,u=i.child;if((z.flags&16)!==0){var o=i.deletions;if(o!==null){for(var a=0;a<o.length;a++){var s=o[a];for(z=s;z!==null;){var h=z;switch(h.tag){case 0:case 11:case 15:Sr(8,h,i)}var m=h.child;if(m!==null)m.return=h,z=m;else for(;z!==null;){h=z;var{sibling:y,return:x}=h;if(nd(h),h===s){z=null;break}if(y!==null){y.return=x,z=y;break}z=x}}}var N=i.alternate;if(N!==null){var g=N.child;if(g!==null){N.child=null;do{var L=g.sibling;g.sibling=null,g=L}while(g!==null)}}z=i}}if((i.subtreeFlags&2064)!==0&&u!==null)u.return=i,z=u;else e:for(;z!==null;){if(i=z,(i.flags&2048)!==0)switch(i.tag){case 0:case 11:case 15:Sr(9,i,i.return)}var c=i.sibling;if(c!==null){c.return=i.return,z=c;break e}z=i.return}}var f=e.current;for(z=f;z!==null;){u=z;var p=u.child;if((u.subtreeFlags&2064)!==0&&p!==null)p.return=u,z=p;else e:for(u=f;z!==null;){if(o=z,(o.flags&2048)!==0)try{switch(o.tag){case 0:case 11:case 15:Fi(9,o)}}catch(C){G(o,o.return,C)}if(o===u){z=null;break e}var w=o.sibling;if(w!==null){w.return=o.return,z=w;break e}z=o.return}}if($=l,Ot(),tt&&typeof tt.onPostCommitFiberRoot==="function")try{tt.onPostCommitFiberRoot(vi,e)}catch(C){}r=!0}return r}finally{_=n,_e.transition=t}}return!1}function gf(e,t,n){t=Qn(n,t),t=Yc(e,t,1),e=Lt(e,t,1),t=ye(),e!==null&&(Ir(e,1,t),Ne(e,t))}function G(e,t,n){if(e.tag===3)gf(e,e,n);else for(;t!==null;){if(t.tag===3){gf(t,e,n);break}else if(t.tag===1){var r=t.stateNode;if(typeof t.type.getDerivedStateFromError==="function"||typeof r.componentDidCatch==="function"&&(Tt===null||!Tt.has(r))){e=Qn(n,e),e=Kc(t,e,1),t=Lt(t,e,1),e=ye(),t!==null&&(Ir(t,1,e),Ne(t,e));break}}t=t.return}}function x1(e,t,n){var r=e.pingCache;r!==null&&r.delete(t),t=ye(),e.pingedLanes|=e.suspendedLanes&n,re===e&&(ie&n)===n&&(ee===4||ee===3&&(ie&130023424)===ie&&500>q()-fa?Gt(e,0):sa|=n),Ne(e,t)}function dd(e,t){t===0&&((e.mode&1)===0?t=1:(t=Fl,Fl<<=1,(Fl&130023424)===0&&(Fl=4194304)));var n=ye();e=dt(e,t),e!==null&&(Ir(e,t,n),Ne(e,n))}function S1(e){var t=e.memoizedState,n=0;t!==null&&(n=t.retryLane),dd(e,n)}function N1(e,t){var n=0;switch(e.tag){case 13:var{stateNode:r,memoizedState:l}=e;l!==null&&(n=l.retryLane);break;case 19:r=e.stateNode;break;default:throw Error(S(314))}r!==null&&r.delete(t),dd(e,n)}function hd(e,t){return Af(e,t)}function E1(e,t,n,r){this.tag=e,this.key=n,this.sibling=this.child=this.return=this.stateNode=this.type=this.elementType=null,this.index=0,this.ref=null,this.pendingProps=t,this.dependencies=this.memoizedState=this.updateQueue=this.memoizedProps=null,this.mode=r,this.subtreeFlags=this.flags=0,this.deletions=null,this.childLanes=this.lanes=0,this.alternate=null}function De(e,t,n,r){return new E1(e,t,n,r)}function ha(e){return e=e.prototype,!(!e||!e.isReactComponent)}function C1(e){if(typeof e==="function")return ha(e)?1:0;if(e!==void 0&&e!==null){if(e=e.$$typeof,e===$o)return 11;if(e===Do)return 14}return 2}function Bt(e,t){var n=e.alternate;return n===null?(n=De(e.tag,t,e.key,e.mode),n.elementType=e.elementType,n.type=e.type,n.stateNode=e.stateNode,n.alternate=e,e.alternate=n):(n.pendingProps=t,n.type=e.type,n.flags=0,n.subtreeFlags=0,n.deletions=null),n.flags=e.flags&14680064,n.childLanes=e.childLanes,n.lanes=e.lanes,n.child=e.child,n.memoizedProps=e.memoizedProps,n.memoizedState=e.memoizedState,n.updateQueue=e.updateQueue,t=e.dependencies,n.dependencies=t===null?null:{lanes:t.lanes,firstContext:t.firstContext},n.sibling=e.sibling,n.index=e.index,n.ref=e.ref,n}function Xl(e,t,n,r,l,i){var u=2;if(r=e,typeof e==="function")ha(e)&&(u=1);else if(typeof e==="string")u=5;else e:switch(e){case gn:return qt(n.children,l,i,t);case Mo:u=8,l|=8;break;case _u:return e=De(12,n,t,l|2),e.elementType=_u,e.lanes=i,e;case Ou:return e=De(13,n,t,l),e.elementType=Ou,e.lanes=i,e;case Uu:return e=De(19,n,t,l),e.elementType=Uu,e.lanes=i,e;case Ef:return Li(n,l,i,t);default:if(typeof e==="object"&&e!==null)switch(e.$$typeof){case Sf:u=10;break e;case Nf:u=9;break e;case $o:u=11;break e;case Do:u=14;break e;case vt:u=16,r=null;break e}throw Error(S(130,e==null?e:typeof e,""))}return t=De(u,n,t,l),t.elementType=e,t.type=r,t.lanes=i,t}function qt(e,t,n,r){return e=De(7,e,r,t),e.lanes=n,e}function Li(e,t,n,r){return e=De(22,e,r,t),e.elementType=Ef,e.lanes=n,e.stateNode={isHidden:!1},e}function Mu(e,t,n){return e=De(6,e,null,t),e.lanes=n,e}function $u(e,t,n){return t=De(4,e.children!==null?e.children:[],e.key,t),t.lanes=n,t.stateNode={containerInfo:e.containerInfo,pendingChildren:null,implementation:e.implementation},t}function F1(e,t,n,r,l){this.tag=t,this.containerInfo=e,this.finishedWork=this.pingCache=this.current=this.pendingChildren=null,this.timeoutHandle=-1,this.callbackNode=this.pendingContext=this.context=null,this.callbackPriority=0,this.eventTimes=gu(0),this.expirationTimes=gu(-1),this.entangledLanes=this.finishedLanes=this.mutableReadLanes=this.expiredLanes=this.pingedLanes=this.suspendedLanes=this.pendingLanes=0,this.entanglements=gu(0),this.identifierPrefix=r,this.onRecoverableError=l,this.mutableSourceEagerHydrationData=null}function ma(e,t,n,r,l,i,u,o,a){return e=new F1(e,t,n,o,a),t===1?(t=1,i===!0&&(t|=8)):t=0,i=De(3,null,null,t),e.current=i,i.stateNode=e,i.memoizedState={element:r,isDehydrated:n,cache:null,transitions:null,pendingSuspenseBoundaries:null},bo(i),e}function z1(e,t,n){var r=3<arguments.length&&arguments[3]!==void 0?arguments[3]:null;return{$$typeof:yn,key:r==null?null:""+r,children:e,containerInfo:t,implementation:n}}function md(e){if(!e)return $t;e=e._reactInternals;e:{if(ln(e)!==e||e.tag!==1)throw Error(S(170));var t=e;do{switch(t.tag){case 3:t=t.stateNode.context;break e;case 1:if(Se(t.type)){t=t.stateNode.__reactInternalMemoizedMergedChildContext;break e}}t=t.return}while(t!==null);throw Error(S(171))}if(e.tag===1){var n=e.type;if(Se(n))return mc(e,n,t)}return t}function yd(e,t,n,r,l,i,u,o,a){return e=ma(n,r,!0,e,l,i,u,o,a),e.context=md(null),n=e.current,r=ye(),l=Pt(n),i=st(r,l),i.callback=t!==void 0&&t!==null?t:null,Lt(n,i,l),e.current.lanes=l,Ir(e,l,r),Ne(e,r),e}function Ti(e,t,n,r){var l=t.current,i=ye(),u=Pt(l);return n=md(n),t.context===null?t.context=n:t.pendingContext=n,t=st(i,u),t.payload={element:e},r=r===void 0?null:r,r!==null&&(t.callback=r),e=Lt(l,t,u),e!==null&&(Xe(e,l,u,i),Al(e,l,u)),u}function gi(e){if(e=e.current,!e.child)return null;switch(e.child.tag){case 5:return e.child.stateNode;default:return e.child.stateNode}}function vf(e,t){if(e=e.memoizedState,e!==null&&e.dehydrated!==null){var n=e.retryLane;e.retryLane=n!==0&&n<t?n:t}}function ya(e,t){vf(e,t),(e=e.alternate)&&vf(e,t)}function L1(){return null}function ga(e){this._internalRoot=e}function Pi(e){this._internalRoot=e}function va(e){return!(!e||e.nodeType!==1&&e.nodeType!==9&&e.nodeType!==11)}function Bi(e){return!(!e||e.nodeType!==1&&e.nodeType!==9&&e.nodeType!==11&&(e.nodeType!==8||e.nodeValue!==" react-mount-point-unstable "))}function wf(){}function T1(e,t,n,r,l){if(l){if(typeof r==="function"){var i=r;r=function(){var s=gi(u);i.call(s)}}var u=yd(t,r,e,0,null,!1,!1,"",wf);return e._reactRootContainer=u,e[ct]=u.current,$r(e.nodeType===8?e.parentNode:e),nn(),u}for(;l=e.lastChild;)e.removeChild(l);if(typeof r==="function"){var o=r;r=function(){var s=gi(a);o.call(s)}}var a=ma(e,0,!1,null,null,!1,!1,"",wf);return e._reactRootContainer=a,e[ct]=a.current,$r(e.nodeType===8?e.parentNode:e),nn(function(){Ti(t,a,n,r)}),a}function Mi(e,t,n,r,l){var i=n._reactRootContainer;if(i){var u=i;if(typeof l==="function"){var o=l;l=function(){var a=gi(u);o.call(a)}}Ti(t,u,e,l)}else u=T1(n,t,e,l,r);return gi(u)}var kf,Y,xf,Cr,ft,Du,Mh,ms,ys,ue,To,ht,Sl,yn,gn,Mo,_u,Sf,Nf,$o,Ou,Uu,Do,vt,Ef,gs,Z,pu,hu=!1,sr,El,Pf,yr,Qh,Wh,Ku=null,Xu=null,Tn=null,Pn=null,yu=!1,Zu=!1,Vt,gr=!1,Jl=null,Gl=!1,Ju=null,Ih,Af,Es,Kh,Xh,q,Zh,Oo,If,ql,Jh,Rf,vi=null,tt=null,Ke,qh,bh,Cl=64,Fl=4194304,_=0,Kf,Ho,Xf,Zf,Jf,qu=!1,zl,Et=null,Ct=null,Ft=null,Lr,Tr,kt,rm,Bn,jl=!0,ei=null,St=null,Wo=null,Hl=null,An,Ao,Rr,am,vu,wu,lr,wi,Ls,sm,fm,cm,ku,dm,pm,hm,mm,ym,Ts,gm,vm,wm,xm,Sm,Nm,Ps,Em,Cm,Fm,zm,Lm,Tm,Pm,Ro,vr=null,Bm,jf,Bs,Ms=!1,vn=!1,Dm,wr=null,Br=null,rc=!1,cr,dr,Wl,Ze,Rm,wn=null,ju=null,kr=null,eo=!1,kn,xu,oc,ac,sc,fc,cc,dc,Hs,hr,to,no,pr,mr,Vm,Pl,Ym,Km,ro=null,lo=null,uo,Xm,Is,Zm,In,et,_r,ct,oo,Gm,qm,ao,Sn=-1,$t,de,xe,bt,ut=null,Ni=!1,Cu=!1,Nn,En=0,ii=null,ui=0,Me,$e=0,jt=null,ot=1,at="",Le=null,ze=null,V=!1,Ye=null,jm,Un,xc,oi,ai=null,Cn=null,Zo=null,Zt=null,wt=!1,Yr,nt,Or,Ur,K,Fu,Il,zu,en=0,X=null,j=null,ne=null,ci=!1,xr=!1,Hr=0,e1=0,di,l1,i1,u1,Ci,o1,a1,ke=!1,vo,jc,ko,ed,td,_l=!1,ce=!1,p1,z=null,ff=!1,le=null,Ve=!1,y1,pi,aa,_e,$=0,re=null,b=null,ie=0,Fe=0,zn,ee=0,Ar=null,tn=0,zi=0,sa=0,Nr=null,we=null,fa=0,Wn=1/0,it=null,hi=!1,Co=null,Tt=null,Ol=!1,Nt=null,mi=0,Er=0,Fo=null,Yl=-1,Kl=0,pd,gd,P1,or,B1,mn,vd,wd=function(e,t){var n=2<arguments.length&&arguments[2]!==void 0?arguments[2]:null;if(!va(t))throw Error(S(200));return z1(e,t,null,n)},kd=function(e,t){if(!va(e))throw Error(S(299));var n=!1,r="",l=gd;return t!==null&&t!==void 0&&(t.unstable_strictMode===!0&&(n=!0),t.identifierPrefix!==void 0&&(r=t.identifierPrefix),t.onRecoverableError!==void 0&&(l=t.onRecoverableError)),t=ma(e,1,!1,null,null,n,!1,r,l),e[ct]=t.current,$r(e.nodeType===8?e.parentNode:e),new ga(t)},xd=function(e){if(e==null)return null;if(e.nodeType===1)return e;var t=e._reactInternals;if(t===void 0){if(typeof e.render==="function")throw Error(S(188));throw e=Object.keys(e).join(","),Error(S(268,e))}return e=Qf(t),e=e===null?null:e.stateNode,e},Sd=function(e){return nn(e)},Nd=function(e,t,n){if(!Bi(t))throw Error(S(200));return Mi(null,e,t,!0,n)},Ed=function(e,t,n){if(!va(e))throw Error(S(405));var r=n!=null&&n.hydratedSources||null,l=!1,i="",u=gd;if(n!==null&&n!==void 0&&(n.unstable_strictMode===!0&&(l=!0),n.identifierPrefix!==void 0&&(i=n.identifierPrefix),n.onRecoverableError!==void 0&&(u=n.onRecoverableError)),t=yd(t,null,e,1,n!=null?n:null,l,!1,i,u),e[ct]=t.current,$r(e),r)for(e=0;e<r.length;e++)n=r[e],l=n._getVersion,l=l(n._source),t.mutableSourceEagerHydrationData==null?t.mutableSourceEagerHydrationData=[n,l]:t.mutableSourceEagerHydrationData.push(n,l);return new Pi(t)},Cd=function(e,t,n){if(!Bi(t))throw Error(S(200));return Mi(null,e,t,!1,n)},Fd=function(e){if(!Bi(e))throw Error(S(40));return e._reactRootContainer?(nn(function(){Mi(null,null,e,!1,function(){e._reactRootContainer=null,e[ct]=null})}),!0):!1},zd,Ld=function(e,t,n,r){if(!Bi(n))throw Error(S(200));if(e==null||e._reactInternals===void 0)throw Error(S(38));return Mi(e,t,n,!1,r)},Td="18.3.1-next-f1338f8080-20240426";var Pd=pp(()=>{kf=B(D(),1),Y=B(hs(),1);xf=new Set,Cr={};ft=!(typeof window>"u"||typeof window.document>"u"||typeof window.document.createElement>"u"),Du=Object.prototype.hasOwnProperty,Mh=/^[:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD][:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD\-.0-9\u00B7\u0300-\u036F\u203F-\u2040]*$/,ms={},ys={};ue={};"children dangerouslySetInnerHTML defaultValue defaultChecked innerHTML suppressContentEditableWarning suppressHydrationWarning style".split(" ").forEach(function(e){ue[e]=new ge(e,0,!1,e,null,!1,!1)});[["acceptCharset","accept-charset"],["className","class"],["htmlFor","for"],["httpEquiv","http-equiv"]].forEach(function(e){var t=e[0];ue[t]=new ge(t,1,!1,e[1],null,!1,!1)});["contentEditable","draggable","spellCheck","value"].forEach(function(e){ue[e]=new ge(e,2,!1,e.toLowerCase(),null,!1,!1)});["autoReverse","externalResourcesRequired","focusable","preserveAlpha"].forEach(function(e){ue[e]=new ge(e,2,!1,e,null,!1,!1)});"allowFullScreen async autoFocus autoPlay controls default defer disabled disablePictureInPicture disableRemotePlayback formNoValidate hidden loop noModule noValidate open playsInline readOnly required reversed scoped seamless itemScope".split(" ").forEach(function(e){ue[e]=new ge(e,3,!1,e.toLowerCase(),null,!1,!1)});["checked","multiple","muted","selected"].forEach(function(e){ue[e]=new ge(e,3,!0,e,null,!1,!1)});["capture","download"].forEach(function(e){ue[e]=new ge(e,4,!1,e,null,!1,!1)});["cols","rows","size","span"].forEach(function(e){ue[e]=new ge(e,6,!1,e,null,!1,!1)});["rowSpan","start"].forEach(function(e){ue[e]=new ge(e,5,!1,e.toLowerCase(),null,!1,!1)});To=/[\-:]([a-z])/g;"accent-height alignment-baseline arabic-form baseline-shift cap-height clip-path clip-rule color-interpolation color-interpolation-filters color-profile color-rendering dominant-baseline enable-background fill-opacity fill-rule flood-color flood-opacity font-family font-size font-size-adjust font-stretch font-style font-variant font-weight glyph-name glyph-orientation-horizontal glyph-orientation-vertical horiz-adv-x horiz-origin-x image-rendering letter-spacing lighting-color marker-end marker-mid marker-start overline-position overline-thickness paint-order panose-1 pointer-events rendering-intent shape-rendering stop-color stop-opacity strikethrough-position strikethrough-thickness stroke-dasharray stroke-dashoffset stroke-linecap stroke-linejoin stroke-miterlimit stroke-opacity stroke-width text-anchor text-decoration text-rendering underline-position underline-thickness unicode-bidi unicode-range units-per-em v-alphabetic v-hanging v-ideographic v-mathematical vector-effect vert-adv-y vert-origin-x vert-origin-y word-spacing writing-mode xmlns:xlink x-height".split(" ").forEach(function(e){var t=e.replace(To,Po);ue[t]=new ge(t,1,!1,e,null,!1,!1)});"xlink:actuate xlink:arcrole xlink:role xlink:show xlink:title xlink:type".split(" ").forEach(function(e){var t=e.replace(To,Po);ue[t]=new ge(t,1,!1,e,"http://www.w3.org/1999/xlink",!1,!1)});["xml:base","xml:lang","xml:space"].forEach(function(e){var t=e.replace(To,Po);ue[t]=new ge(t,1,!1,e,"http://www.w3.org/XML/1998/namespace",!1,!1)});["tabIndex","crossOrigin"].forEach(function(e){ue[e]=new ge(e,1,!1,e.toLowerCase(),null,!1,!1)});ue.xlinkHref=new ge("xlinkHref",1,!1,"xlink:href","http://www.w3.org/1999/xlink",!0,!1);["src","href","action","formAction"].forEach(function(e){ue[e]=new ge(e,1,!1,e.toLowerCase(),null,!0,!0)});ht=kf.__SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED,Sl=Symbol.for("react.element"),yn=Symbol.for("react.portal"),gn=Symbol.for("react.fragment"),Mo=Symbol.for("react.strict_mode"),_u=Symbol.for("react.profiler"),Sf=Symbol.for("react.provider"),Nf=Symbol.for("react.context"),$o=Symbol.for("react.forward_ref"),Ou=Symbol.for("react.suspense"),Uu=Symbol.for("react.suspense_list"),Do=Symbol.for("react.memo"),vt=Symbol.for("react.lazy"),Ef=Symbol.for("react.offscreen"),gs=Symbol.iterator;Z=Object.assign;sr=Array.isArray;Pf=function(e){return typeof MSApp<"u"&&MSApp.execUnsafeLocalFunction?function(t,n,r,l){MSApp.execUnsafeLocalFunction(function(){return e(t,n,r,l)})}:e}(function(e,t){if(e.namespaceURI!=="http://www.w3.org/2000/svg"||"innerHTML"in e)e.innerHTML=t;else{El=El||document.createElement("div"),El.innerHTML="<svg>"+t.valueOf().toString()+"</svg>";for(t=El.firstChild;e.firstChild;)e.removeChild(e.firstChild);for(;t.firstChild;)e.appendChild(t.firstChild)}});yr={animationIterationCount:!0,aspectRatio:!0,borderImageOutset:!0,borderImageSlice:!0,borderImageWidth:!0,boxFlex:!0,boxFlexGroup:!0,boxOrdinalGroup:!0,columnCount:!0,columns:!0,flex:!0,flexGrow:!0,flexPositive:!0,flexShrink:!0,flexNegative:!0,flexOrder:!0,gridArea:!0,gridRow:!0,gridRowEnd:!0,gridRowSpan:!0,gridRowStart:!0,gridColumn:!0,gridColumnEnd:!0,gridColumnSpan:!0,gridColumnStart:!0,fontWeight:!0,lineClamp:!0,lineHeight:!0,opacity:!0,order:!0,orphans:!0,tabSize:!0,widows:!0,zIndex:!0,zoom:!0,fillOpacity:!0,floodOpacity:!0,stopOpacity:!0,strokeDasharray:!0,strokeDashoffset:!0,strokeMiterlimit:!0,strokeOpacity:!0,strokeWidth:!0},Qh=["Webkit","ms","Moz","O"];Object.keys(yr).forEach(function(e){Qh.forEach(function(t){t=t+e.charAt(0).toUpperCase()+e.substring(1),yr[t]=yr[e]})});Wh=Z({menuitem:!0},{area:!0,base:!0,br:!0,col:!0,embed:!0,hr:!0,img:!0,input:!0,keygen:!0,link:!0,meta:!0,param:!0,source:!0,track:!0,wbr:!0});if(ft)try{Vt={},Object.defineProperty(Vt,"passive",{get:function(){Zu=!0}}),window.addEventListener("test",Vt,Vt),window.removeEventListener("test",Vt,Vt)}catch(e){Zu=!1}Ih={onError:function(e){gr=!0,Jl=e}};Af=Y.unstable_scheduleCallback,Es=Y.unstable_cancelCallback,Kh=Y.unstable_shouldYield,Xh=Y.unstable_requestPaint,q=Y.unstable_now,Zh=Y.unstable_getCurrentPriorityLevel,Oo=Y.unstable_ImmediatePriority,If=Y.unstable_UserBlockingPriority,ql=Y.unstable_NormalPriority,Jh=Y.unstable_LowPriority,Rf=Y.unstable_IdlePriority;Ke=Math.clz32?Math.clz32:jh,qh=Math.log,bh=Math.LN2;zl=[],Lr=new Map,Tr=new Map,kt=[],rm="mousedown mouseup touchcancel touchend touchstart auxclick dblclick pointercancel pointerdown pointerup dragend dragstart drop compositionend compositionstart keydown keypress keyup input textInput copy cut paste click change contextmenu reset submit".split(" ");Bn=ht.ReactCurrentBatchConfig;An={eventPhase:0,bubbles:0,cancelable:0,timeStamp:function(e){return e.timeStamp||Date.now()},defaultPrevented:0,isTrusted:0},Ao=Te(An),Rr=Z({},An,{view:0,detail:0}),am=Te(Rr),wi=Z({},Rr,{screenX:0,screenY:0,clientX:0,clientY:0,pageX:0,pageY:0,ctrlKey:0,shiftKey:0,altKey:0,metaKey:0,getModifierState:Io,button:0,buttons:0,relatedTarget:function(e){return e.relatedTarget===void 0?e.fromElement===e.srcElement?e.toElement:e.fromElement:e.relatedTarget},movementX:function(e){if("movementX"in e)return e.movementX;return e!==lr&&(lr&&e.type==="mousemove"?(vu=e.screenX-lr.screenX,wu=e.screenY-lr.screenY):wu=vu=0,lr=e),vu},movementY:function(e){return"movementY"in e?e.movementY:wu}}),Ls=Te(wi),sm=Z({},wi,{dataTransfer:0}),fm=Te(sm),cm=Z({},Rr,{relatedTarget:0}),ku=Te(cm),dm=Z({},An,{animationName:0,elapsedTime:0,pseudoElement:0}),pm=Te(dm),hm=Z({},An,{clipboardData:function(e){return"clipboardData"in e?e.clipboardData:window.clipboardData}}),mm=Te(hm),ym=Z({},An,{data:0}),Ts=Te(ym),gm={Esc:"Escape",Spacebar:" ",Left:"ArrowLeft",Up:"ArrowUp",Right:"ArrowRight",Down:"ArrowDown",Del:"Delete",Win:"OS",Menu:"ContextMenu",Apps:"ContextMenu",Scroll:"ScrollLock",MozPrintableKey:"Unidentified"},vm={8:"Backspace",9:"Tab",12:"Clear",13:"Enter",16:"Shift",17:"Control",18:"Alt",19:"Pause",20:"CapsLock",27:"Escape",32:" ",33:"PageUp",34:"PageDown",35:"End",36:"Home",37:"ArrowLeft",38:"ArrowUp",39:"ArrowRight",40:"ArrowDown",45:"Insert",46:"Delete",112:"F1",113:"F2",114:"F3",115:"F4",116:"F5",117:"F6",118:"F7",119:"F8",120:"F9",121:"F10",122:"F11",123:"F12",144:"NumLock",145:"ScrollLock",224:"Meta"},wm={Alt:"altKey",Control:"ctrlKey",Meta:"metaKey",Shift:"shiftKey"};xm=Z({},Rr,{key:function(e){if(e.key){var t=gm[e.key]||e.key;if(t!=="Unidentified")return t}return e.type==="keypress"?(e=Ql(e),e===13?"Enter":String.fromCharCode(e)):e.type==="keydown"||e.type==="keyup"?vm[e.keyCode]||"Unidentified":""},code:0,location:0,ctrlKey:0,shiftKey:0,altKey:0,metaKey:0,repeat:0,locale:0,getModifierState:Io,charCode:function(e){return e.type==="keypress"?Ql(e):0},keyCode:function(e){return e.type==="keydown"||e.type==="keyup"?e.keyCode:0},which:function(e){return e.type==="keypress"?Ql(e):e.type==="keydown"||e.type==="keyup"?e.keyCode:0}}),Sm=Te(xm),Nm=Z({},wi,{pointerId:0,width:0,height:0,pressure:0,tangentialPressure:0,tiltX:0,tiltY:0,twist:0,pointerType:0,isPrimary:0}),Ps=Te(Nm),Em=Z({},Rr,{touches:0,targetTouches:0,changedTouches:0,altKey:0,metaKey:0,ctrlKey:0,shiftKey:0,getModifierState:Io}),Cm=Te(Em),Fm=Z({},An,{propertyName:0,elapsedTime:0,pseudoElement:0}),zm=Te(Fm),Lm=Z({},wi,{deltaX:function(e){return"deltaX"in e?e.deltaX:("wheelDeltaX"in e)?-e.wheelDeltaX:0},deltaY:function(e){return"deltaY"in e?e.deltaY:("wheelDeltaY"in e)?-e.wheelDeltaY:("wheelDelta"in e)?-e.wheelDelta:0},deltaZ:0,deltaMode:0}),Tm=Te(Lm),Pm=[9,13,27,32],Ro=ft&&"CompositionEvent"in window;ft&&"documentMode"in document&&(vr=document.documentMode);Bm=ft&&"TextEvent"in window&&!vr,jf=ft&&(!Ro||vr&&8<vr&&11>=vr),Bs=String.fromCharCode(32);Dm={color:!0,date:!0,datetime:!0,"datetime-local":!0,email:!0,month:!0,number:!0,password:!0,range:!0,search:!0,tel:!0,text:!0,time:!0,url:!0,week:!0};if(ft){if(ft){if(dr="oninput"in document,!dr)Wl=document.createElement("div"),Wl.setAttribute("oninput","return;"),dr=typeof Wl.oninput==="function";cr=dr}else cr=!1;rc=cr&&(!document.documentMode||9<document.documentMode)}Ze=typeof Object.is==="function"?Object.is:Am;Rm=ft&&"documentMode"in document&&11>=document.documentMode;kn={animationend:Tl("Animation","AnimationEnd"),animationiteration:Tl("Animation","AnimationIteration"),animationstart:Tl("Animation","AnimationStart"),transitionend:Tl("Transition","TransitionEnd")},xu={},oc={};ft&&(oc=document.createElement("div").style,("AnimationEvent"in window)||(delete kn.animationend.animation,delete kn.animationiteration.animation,delete kn.animationstart.animation),("TransitionEvent"in window)||delete kn.transitionend.transition);ac=xi("animationend"),sc=xi("animationiteration"),fc=xi("animationstart"),cc=xi("transitionend"),dc=new Map,Hs="abort auxClick cancel canPlay canPlayThrough click close contextMenu copy cut drag dragEnd dragEnter dragExit dragLeave dragOver dragStart drop durationChange emptied encrypted ended error gotPointerCapture input invalid keyDown keyPress keyUp load loadedData loadedMetadata loadStart lostPointerCapture mouseDown mouseMove mouseOut mouseOver mouseUp paste pause play playing pointerCancel pointerDown pointerMove pointerOut pointerOver pointerUp progress rateChange reset resize seeked seeking stalled submit suspend timeUpdate touchCancel touchEnd touchStart volumeChange scroll toggle touchMove waiting wheel".split(" ");for(pr=0;pr<Hs.length;pr++)hr=Hs[pr],to=hr.toLowerCase(),no=hr[0].toUpperCase()+hr.slice(1),Dt(to,"on"+no);Dt(ac,"onAnimationEnd");Dt(sc,"onAnimationIteration");Dt(fc,"onAnimationStart");Dt("dblclick","onDoubleClick");Dt("focusin","onFocus");Dt("focusout","onBlur");Dt(cc,"onTransitionEnd");Dn("onMouseEnter",["mouseout","mouseover"]);Dn("onMouseLeave",["mouseout","mouseover"]);Dn("onPointerEnter",["pointerout","pointerover"]);Dn("onPointerLeave",["pointerout","pointerover"]);rn("onChange","change click focusin focusout input keydown keyup selectionchange".split(" "));rn("onSelect","focusout contextmenu dragend focusin keydown keyup mousedown mouseup selectionchange".split(" "));rn("onBeforeInput",["compositionend","keypress","textInput","paste"]);rn("onCompositionEnd","compositionend focusout keydown keypress keyup mousedown".split(" "));rn("onCompositionStart","compositionstart focusout keydown keypress keyup mousedown".split(" "));rn("onCompositionUpdate","compositionupdate focusout keydown keypress keyup mousedown".split(" "));mr="abort canplay canplaythrough durationchange emptied encrypted ended error loadeddata loadedmetadata loadstart pause play playing progress ratechange resize seeked seeking stalled suspend timeupdate volumechange waiting".split(" "),Vm=new Set("cancel close invalid load scroll toggle".split(" ").concat(mr));Pl="_reactListening"+Math.random().toString(36).slice(2);Ym=/\r\n?/g,Km=/\u0000|\uFFFD/g;uo=typeof setTimeout==="function"?setTimeout:void 0,Xm=typeof clearTimeout==="function"?clearTimeout:void 0,Is=typeof Promise==="function"?Promise:void 0,Zm=typeof queueMicrotask==="function"?queueMicrotask:typeof Is<"u"?function(e){return Is.resolve(null).then(e).catch(Jm)}:uo;In=Math.random().toString(36).slice(2),et="__reactFiber$"+In,_r="__reactProps$"+In,ct="__reactContainer$"+In,oo="__reactEvents$"+In,Gm="__reactListeners$"+In,qm="__reactHandles$"+In;ao=[];$t={},de=_t($t),xe=_t(!1),bt=$t;Nn=[],Me=[];jm=ht.ReactCurrentBatchConfig;Un=kc(!0),xc=kc(!1),oi=_t(null);Yr={},nt=_t(Yr),Or=_t(Yr),Ur=_t(Yr);K=_t(0);Fu=[];Il=ht.ReactCurrentDispatcher,zu=ht.ReactCurrentBatchConfig;di={readContext:Oe,useCallback:se,useContext:se,useEffect:se,useImperativeHandle:se,useInsertionEffect:se,useLayoutEffect:se,useMemo:se,useReducer:se,useRef:se,useState:se,useDebugValue:se,useDeferredValue:se,useTransition:se,useMutableSource:se,useSyncExternalStore:se,useId:se,unstable_isNewReconciler:!1},l1={readContext:Oe,useCallback:function(e,t){return je().memoizedState=[e,t===void 0?null:t],e},useContext:Oe,useEffect:bs,useImperativeHandle:function(e,t,n){return n=n!==null&&n!==void 0?n.concat([e]):null,Rl(4194308,4,_c.bind(null,t,e),n)},useLayoutEffect:function(e,t){return Rl(4194308,4,e,t)},useInsertionEffect:function(e,t){return Rl(4,2,e,t)},useMemo:function(e,t){var n=je();return t=t===void 0?null:t,e=e(),n.memoizedState=[e,t],e},useReducer:function(e,t,n){var r=je();return t=n!==void 0?n(t):t,r.memoizedState=r.baseState=t,e={pending:null,interleaved:null,lanes:0,dispatch:null,lastRenderedReducer:e,lastRenderedState:t},r.queue=e,e=e.dispatch=n1.bind(null,X,e),[r.memoizedState,e]},useRef:function(e){var t=je();return e={current:e},t.memoizedState=e},useState:qs,useDebugValue:ua,useDeferredValue:function(e){return je().memoizedState=e},useTransition:function(){var e=qs(!1),t=e[0];return e=t1.bind(null,e[1]),je().memoizedState=e,[t,e]},useMutableSource:function(){},useSyncExternalStore:function(e,t,n){var r=X,l=je();if(V){if(n===void 0)throw Error(S(407));n=n()}else{if(n=t(),re===null)throw Error(S(349));(en&30)!==0||zc(r,t,n)}l.memoizedState=n;var i={value:n,getSnapshot:t};return l.queue=i,bs(Tc.bind(null,r,i,e),[e]),r.flags|=2048,Wr(9,Lc.bind(null,r,i,n,t),void 0,null),n},useId:function(){var e=je(),t=re.identifierPrefix;if(V){var n=at,r=ot;n=(r&~(1<<32-Ke(r)-1)).toString(32)+n,t=":"+t+"R"+n,n=Hr++,0<n&&(t+="H"+n.toString(32)),t+=":"}else n=e1++,t=":"+t+"r"+n.toString(32)+":";return e.memoizedState=t},unstable_isNewReconciler:!1},i1={readContext:Oe,useCallback:Uc,useContext:Oe,useEffect:ia,useImperativeHandle:Oc,useInsertionEffect:$c,useLayoutEffect:Dc,useMemo:Hc,useReducer:Lu,useRef:Mc,useState:function(){return Lu(Qr)},useDebugValue:ua,useDeferredValue:function(e){var t=Ue();return Qc(t,j.memoizedState,e)},useTransition:function(){var e=Lu(Qr)[0],t=Ue().memoizedState;return[e,t]},useMutableSource:Cc,useSyncExternalStore:Fc,useId:Wc,unstable_isNewReconciler:!1},u1={readContext:Oe,useCallback:Uc,useContext:Oe,useEffect:ia,useImperativeHandle:Oc,useInsertionEffect:$c,useLayoutEffect:Dc,useMemo:Hc,useReducer:Tu,useRef:Mc,useState:function(){return Tu(Qr)},useDebugValue:ua,useDeferredValue:function(e){var t=Ue();return j===null?t.memoizedState=e:Qc(t,j.memoizedState,e)},useTransition:function(){var e=Tu(Qr)[0],t=Ue().memoizedState;return[e,t]},useMutableSource:Cc,useSyncExternalStore:Fc,useId:Wc,unstable_isNewReconciler:!1};Ci={isMounted:function(e){return(e=e._reactInternals)?ln(e)===e:!1},enqueueSetState:function(e,t,n){e=e._reactInternals;var r=ye(),l=Pt(e),i=st(r,l);i.payload=t,n!==void 0&&n!==null&&(i.callback=n),t=Lt(e,i,l),t!==null&&(Xe(t,e,l,r),Al(t,e,l))},enqueueReplaceState:function(e,t,n){e=e._reactInternals;var r=ye(),l=Pt(e),i=st(r,l);i.tag=1,i.payload=t,n!==void 0&&n!==null&&(i.callback=n),t=Lt(e,i,l),t!==null&&(Xe(t,e,l,r),Al(t,e,l))},enqueueForceUpdate:function(e,t){e=e._reactInternals;var n=ye(),r=Pt(e),l=st(n,r);l.tag=2,t!==void 0&&t!==null&&(l.callback=t),t=Lt(e,l,r),t!==null&&(Xe(t,e,r,n),Al(t,e,r))}};o1=typeof WeakMap==="function"?WeakMap:Map;a1=ht.ReactCurrentOwner;vo={dehydrated:null,treeContext:null,retryLane:0};jc=function(e,t){for(var n=t.child;n!==null;){if(n.tag===5||n.tag===6)e.appendChild(n.stateNode);else if(n.tag!==4&&n.child!==null){n.child.return=n,n=n.child;continue}if(n===t)break;for(;n.sibling===null;){if(n.return===null||n.return===t)return;n=n.return}n.sibling.return=n.return,n=n.sibling}};ko=function(){};ed=function(e,t,n,r){var l=e.memoizedProps;if(l!==r){e=t.stateNode,Jt(nt.current);var i=null;switch(n){case"input":l=Qu(e,l),r=Qu(e,r),i=[];break;case"select":l=Z({},l,{value:void 0}),r=Z({},r,{value:void 0}),i=[];break;case"textarea":l=Iu(e,l),r=Iu(e,r),i=[];break;default:typeof l.onClick!=="function"&&typeof r.onClick==="function"&&(e.onclick=ni)}Vu(n,r);var u;n=null;for(s in l)if(!r.hasOwnProperty(s)&&l.hasOwnProperty(s)&&l[s]!=null)if(s==="style"){var o=l[s];for(u in o)o.hasOwnProperty(u)&&(n||(n={}),n[u]="")}else s!=="dangerouslySetInnerHTML"&&s!=="children"&&s!=="suppressContentEditableWarning"&&s!=="suppressHydrationWarning"&&s!=="autoFocus"&&(Cr.hasOwnProperty(s)?i||(i=[]):(i=i||[]).push(s,null));for(s in r){var a=r[s];if(o=l!=null?l[s]:void 0,r.hasOwnProperty(s)&&a!==o&&(a!=null||o!=null))if(s==="style")if(o){for(u in o)!o.hasOwnProperty(u)||a&&a.hasOwnProperty(u)||(n||(n={}),n[u]="");for(u in a)a.hasOwnProperty(u)&&o[u]!==a[u]&&(n||(n={}),n[u]=a[u])}else n||(i||(i=[]),i.push(s,n)),n=a;else s==="dangerouslySetInnerHTML"?(a=a?a.__html:void 0,o=o?o.__html:void 0,a!=null&&o!==a&&(i=i||[]).push(s,a)):s==="children"?typeof a!=="string"&&typeof a!=="number"||(i=i||[]).push(s,""+a):s!=="suppressContentEditableWarning"&&s!=="suppressHydrationWarning"&&(Cr.hasOwnProperty(s)?(a!=null&&s==="onScroll"&&A("scroll",e),i||o===a||(i=[])):(i=i||[]).push(s,a))}n&&(i=i||[]).push("style",n);var s=i;if(t.updateQueue=s)t.flags|=4}};td=function(e,t,n,r){n!==r&&(t.flags|=4)};p1=typeof WeakSet==="function"?WeakSet:Set;y1=Math.ceil,pi=ht.ReactCurrentDispatcher,aa=ht.ReactCurrentOwner,_e=ht.ReactCurrentBatchConfig,zn=_t(0);pd=function(e,t,n){if(e!==null)if(e.memoizedProps!==t.pendingProps||xe.current)ke=!0;else{if((e.lanes&n)===0&&(t.flags&128)===0)return ke=!1,f1(e,t,n);ke=(e.flags&131072)!==0?!0:!1}else ke=!1,V&&(t.flags&1048576)!==0&&gc(t,ui,t.index);switch(t.lanes=0,t.tag){case 2:var r=t.type;Vl(e,t),e=t.pendingProps;var l=_n(t,de.current);Mn(t,n),l=ra(null,t,r,e,l,n);var i=la();return t.flags|=1,typeof l==="object"&&l!==null&&typeof l.render==="function"&&l.$$typeof===void 0?(t.tag=1,t.memoizedState=null,t.updateQueue=null,Se(r)?(i=!0,li(t)):i=!1,t.memoizedState=l.state!==null&&l.state!==void 0?l.state:null,bo(t),l.updater=Ci,t.stateNode=l,l._reactInternals=t,ho(t,r,e,n),t=go(null,t,r,!0,i,n)):(t.tag=0,V&&i&&Yo(t),me(null,t,l,n),t=t.child),t;case 16:r=t.elementType;e:{switch(Vl(e,t),e=t.pendingProps,l=r._init,r=l(r._payload),t.type=r,l=t.tag=C1(r),e=Re(r,e),l){case 0:t=yo(null,t,r,e,n);break e;case 1:t=of(null,t,r,e,n);break e;case 11:t=lf(null,t,r,e,n);break e;case 14:t=uf(null,t,r,Re(r.type,e),n);break e}throw Error(S(306,r,""))}return t;case 0:return r=t.type,l=t.pendingProps,l=t.elementType===r?l:Re(r,l),yo(e,t,r,l,n);case 1:return r=t.type,l=t.pendingProps,l=t.elementType===r?l:Re(r,l),of(e,t,r,l,n);case 3:e:{if(Gc(t),e===null)throw Error(S(387));r=t.pendingProps,i=t.memoizedState,l=i.element,Nc(e,t),si(t,r,null,n);var u=t.memoizedState;if(r=u.element,i.isDehydrated)if(i={element:r,isDehydrated:!1,cache:u.cache,pendingSuspenseBoundaries:u.pendingSuspenseBoundaries,transitions:u.transitions},t.updateQueue.baseState=i,t.memoizedState=i,t.flags&256){l=Qn(Error(S(423)),t),t=af(e,t,r,n,l);break e}else if(r!==l){l=Qn(Error(S(424)),t),t=af(e,t,r,n,l);break e}else for(ze=zt(t.stateNode.containerInfo.firstChild),Le=t,V=!0,Ye=null,n=xc(t,null,r,n),t.child=n;n;)n.flags=n.flags&-3|4096,n=n.sibling;else{if(On(),r===l){t=pt(e,t,n);break e}me(e,t,r,n)}t=t.child}return t;case 5:return Ec(t),e===null&&fo(t),r=t.type,l=t.pendingProps,i=e!==null?e.memoizedProps:null,u=l.children,io(r,l)?u=null:i!==null&&io(r,i)&&(t.flags|=32),Jc(e,t),me(e,t,u,n),t.child;case 6:return e===null&&fo(t),null;case 13:return qc(e,t,n);case 4:return jo(t,t.stateNode.containerInfo),r=t.pendingProps,e===null?t.child=Un(t,null,r,n):me(e,t,r,n),t.child;case 11:return r=t.type,l=t.pendingProps,l=t.elementType===r?l:Re(r,l),lf(e,t,r,l,n);case 7:return me(e,t,t.pendingProps,n),t.child;case 8:return me(e,t,t.pendingProps.children,n),t.child;case 12:return me(e,t,t.pendingProps.children,n),t.child;case 10:e:{if(r=t.type._context,l=t.pendingProps,i=t.memoizedProps,u=l.value,H(oi,r._currentValue),r._currentValue=u,i!==null)if(Ze(i.value,u)){if(i.children===l.children&&!xe.current){t=pt(e,t,n);break e}}else for(i=t.child,i!==null&&(i.return=t);i!==null;){var o=i.dependencies;if(o!==null){u=i.child;for(var a=o.firstContext;a!==null;){if(a.context===r){if(i.tag===1){a=st(-1,n&-n),a.tag=2;var s=i.updateQueue;if(s!==null){s=s.shared;var h=s.pending;h===null?a.next=a:(a.next=h.next,h.next=a),s.pending=a}}i.lanes|=n,a=i.alternate,a!==null&&(a.lanes|=n),co(i.return,n,t),o.lanes|=n;break}a=a.next}}else if(i.tag===10)u=i.type===t.type?null:i.child;else if(i.tag===18){if(u=i.return,u===null)throw Error(S(341));u.lanes|=n,o=u.alternate,o!==null&&(o.lanes|=n),co(u,n,t),u=i.sibling}else u=i.child;if(u!==null)u.return=i;else for(u=i;u!==null;){if(u===t){u=null;break}if(i=u.sibling,i!==null){i.return=u.return,u=i;break}u=u.return}i=u}me(e,t,l.children,n),t=t.child}return t;case 9:return l=t.type,r=t.pendingProps.children,Mn(t,n),l=Oe(l),r=r(l),t.flags|=1,me(e,t,r,n),t.child;case 14:return r=t.type,l=Re(r,t.pendingProps),l=Re(r.type,l),uf(e,t,r,l,n);case 15:return Xc(e,t,t.type,t.pendingProps,n);case 17:return r=t.type,l=t.pendingProps,l=t.elementType===r?l:Re(r,l),Vl(e,t),t.tag=1,Se(r)?(e=!0,li(t)):e=!1,Mn(t,n),Vc(t,r,l),ho(t,r,l,n),go(null,t,r,!0,e,n);case 19:return bc(e,t,n);case 22:return Zc(e,t,n)}throw Error(S(156,t.tag))};gd=typeof reportError==="function"?reportError:function(e){console.error(e)};Pi.prototype.render=ga.prototype.render=function(e){var t=this._internalRoot;if(t===null)throw Error(S(409));Ti(e,t,null,null)};Pi.prototype.unmount=ga.prototype.unmount=function(){var e=this._internalRoot;if(e!==null){this._internalRoot=null;var t=e.containerInfo;nn(function(){Ti(null,e,null,null)}),t[ct]=null}};Pi.prototype.unstable_scheduleHydration=function(e){if(e){var t=Zf();e={blockedOn:null,target:e,priority:t};for(var n=0;n<kt.length&&t!==0&&t<kt[n].priority;n++);kt.splice(n,0,e),n===0&&Gf(e)}};Kf=function(e){switch(e.tag){case 3:var t=e.stateNode;if(t.current.memoizedState.isDehydrated){var n=fr(t.pendingLanes);n!==0&&(Uo(t,n|1),Ne(t,q()),($&6)===0&&(Wn=q()+500,Ot()))}break;case 13:nn(function(){var r=dt(e,1);if(r!==null){var l=ye();Xe(r,e,1,l)}}),ya(e,1)}};Ho=function(e){if(e.tag===13){var t=dt(e,134217728);if(t!==null){var n=ye();Xe(t,e,134217728,n)}ya(e,134217728)}};Xf=function(e){if(e.tag===13){var t=Pt(e),n=dt(e,t);if(n!==null){var r=ye();Xe(n,e,t,r)}ya(e,t)}};Zf=function(){return _};Jf=function(e,t){var n=_;try{return _=e,t()}finally{_=n}};Xu=function(e,t,n){switch(t){case"input":if(Wu(e,n),t=n.name,n.type==="radio"&&t!=null){for(n=e;n.parentNode;)n=n.parentNode;n=n.querySelectorAll("input[name="+JSON.stringify(""+t)+'][type="radio"]');for(t=0;t<n.length;t++){var r=n[t];if(r!==e&&r.form===e.form){var l=Si(r);if(!l)throw Error(S(90));Ff(r),Wu(r,l)}}}break;case"textarea":Lf(e,n);break;case"select":t=n.value,t!=null&&Ln(e,!!n.multiple,t,!1)}};_f=ca;Of=nn;P1={usingClientEntryPoint:!1,Events:[Vr,xn,Si,$f,Df,ca]},or={findFiberByHostInstance:Xt,bundleType:0,version:"18.3.1",rendererPackageName:"react-dom"},B1={bundleType:or.bundleType,version:or.version,rendererPackageName:or.rendererPackageName,rendererConfig:or.rendererConfig,overrideHookState:null,overrideHookStateDeletePath:null,overrideHookStateRenamePath:null,overrideProps:null,overridePropsDeletePath:null,overridePropsRenamePath:null,setErrorHandler:null,setSuspenseHandler:null,scheduleUpdate:null,currentDispatcherRef:ht.ReactCurrentDispatcher,findHostInstanceByFiber:function(e){return e=Qf(e),e===null?null:e.stateNode},findFiberByHostInstance:or.findFiberByHostInstance||L1,findHostInstancesForRefresh:null,scheduleRefresh:null,scheduleRoot:null,setRefreshHandler:null,getCurrentFiber:null,reconcilerVersion:"18.3.1-next-f1338f8080-20240426"};if(typeof __REACT_DEVTOOLS_GLOBAL_HOOK__<"u"){if(mn=__REACT_DEVTOOLS_GLOBAL_HOOK__,!mn.isDisabled&&mn.supportsFiber)try{vi=mn.inject(B1),tt=mn}catch(e){}}vd=P1,zd=ca});var $i=cl((ng,Md)=>{Pd();function Bd(){if(typeof __REACT_DEVTOOLS_GLOBAL_HOOK__>"u"||typeof __REACT_DEVTOOLS_GLOBAL_HOOK__.checkDCE!=="function")return;try{__REACT_DEVTOOLS_GLOBAL_HOOK__.checkDCE(Bd)}catch(e){console.error(e)}}Bd(),Md.exports=wa});var $d=cl(($1)=>{var Kr=B($i(),1);$1.createRoot=Kr.createRoot,$1.hydrateRoot=Kr.hydrateRoot;var M1});var ep=B(D(),1),tp=B($d(),1);var W=B(D(),1);var Q=B(D(),1);var ug=!!(typeof window<"u"&&window.document&&window.document.createElement);function rt(e,t,{checkForDefaultPrevented:n=!0}={}){return function(l){if(e?.(l),n===!1||!l.defaultPrevented)return t?.(l)}}var _d=B(D(),1);function Dd(e,t){if(typeof e==="function")return e(t);else if(e!==null&&e!==void 0)e.current=t}function ka(...e){return(t)=>{let n=!1,r=e.map((l)=>{let i=Dd(l,t);if(!n&&typeof i=="function")n=!0;return i});if(n)return()=>{for(let l=0;l<r.length;l++){let i=r[l];if(typeof i=="function")i();else Dd(e[l],null)}}}}function Je(...e){return _d.useCallback(ka(...e),e)}var He=B(D(),1);var Od=B(D(),1),D1=Symbol.for("react.element"),_1=Symbol.for("react.fragment"),O1=Object.prototype.hasOwnProperty,U1=Od.__SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED.ReactCurrentOwner,H1={key:!0,ref:!0,__self:!0,__source:!0};function Ud(e,t,n){var r,l={},i=null,u=null;n!==void 0&&(i=""+n),t.key!==void 0&&(i=""+t.key),t.ref!==void 0&&(u=t.ref);for(r in t)O1.call(t,r)&&!H1.hasOwnProperty(r)&&(l[r]=t[r]);if(e&&e.defaultProps)for(r in t=e.defaultProps,t)l[r]===void 0&&(l[r]=t[r]);return{$$typeof:D1,type:e,key:i,ref:u,props:l,_owner:U1.current}}var un=_1,d=Ud,v=Ud;function Hd(e,t){let n=He.createContext(t),r=(i)=>{let{children:u,...o}=i,a=He.useMemo(()=>o,Object.values(o));return d(n.Provider,{value:a,children:u})};r.displayName=e+"Provider";function l(i){let u=He.useContext(n);if(u)return u;if(t!==void 0)return t;throw Error(`\`${i}\` must be used within \`${e}\``)}return[r,l]}function Qd(e,t=[]){let n=[];function r(i,u){let o=He.createContext(u),a=n.length;n=[...n,u];let s=(m)=>{let{scope:y,children:x,...N}=m,g=y?.[e]?.[a]||o,L=He.useMemo(()=>N,Object.values(N));return d(g.Provider,{value:L,children:x})};s.displayName=i+"Provider";function h(m,y){let x=y?.[e]?.[a]||o,N=He.useContext(x);if(N)return N;if(u!==void 0)return u;throw Error(`\`${m}\` must be used within \`${i}\``)}return[s,h]}let l=()=>{let i=n.map((u)=>{return He.createContext(u)});return function(o){let a=o?.[e]||i;return He.useMemo(()=>({[`__scope${e}`]:{...o,[e]:a}}),[o,a])}};return l.scopeName=e,[r,Q1(l,...t)]}function Q1(...e){let t=e[0];if(e.length===1)return t;let n=()=>{let r=e.map((l)=>({useScope:l(),scopeName:l.scopeName}));return function(i){let u=r.reduce((o,{useScope:a,scopeName:s})=>{let m=a(i)[`__scope${s}`];return{...o,...m}},{});return He.useMemo(()=>({[`__scope${t.scopeName}`]:u}),[u])}};return n.scopeName=t.scopeName,n}var xa=B(D(),1);var Wd=B(D(),1),mt=globalThis?.document?Wd.useLayoutEffect:()=>{};var W1=xa[" useId ".trim().toString()]||(()=>{return}),A1=0;function Di(e){let[t,n]=xa.useState(W1());return mt(()=>{if(!e)n((r)=>r??String(A1++))},[e]),e||(t?`radix-${t}`:"")}var Qe=B(D(),1);var _i=B(D(),1);var I1=Qe[" useInsertionEffect ".trim().toString()]||mt;function Ad({prop:e,defaultProp:t,onChange:n=()=>{},caller:r}){let[l,i,u]=R1({defaultProp:t,onChange:n}),o=e!==void 0,a=o?e:l;{let h=Qe.useRef(e!==void 0);Qe.useEffect(()=>{let m=h.current;if(m!==o)console.warn(`${r} is changing from ${m?"controlled":"uncontrolled"} to ${o?"controlled":"uncontrolled"}. Components should not switch from controlled to uncontrolled (or vice versa). Decide between using a controlled or uncontrolled value for the lifetime of the component.`);h.current=o},[o,r])}let s=Qe.useCallback((h)=>{if(o){let m=V1(h)?h(e):h;if(m!==e)u.current?.(m)}else i(h)},[o,e,i,u]);return[a,s]}function R1({defaultProp:e,onChange:t}){let[n,r]=Qe.useState(e),l=Qe.useRef(n),i=Qe.useRef(t);return I1(()=>{i.current=t},[t]),Qe.useEffect(()=>{if(l.current!==n)i.current?.(n),l.current=n},[n,l]),[n,r,i]}function V1(e){return typeof e==="function"}var yg=Symbol("RADIX:SYNC_STATE");var J=B(D(),1);var Id=B(D(),1),Rd=B($i(),1);var te=B(D(),1);function Oi(e){let t=Y1(e),n=te.forwardRef((r,l)=>{let{children:i,...u}=r,o=te.Children.toArray(i),a=o.find(X1);if(a){let s=a.props.children,h=o.map((m)=>{if(m===a){if(te.Children.count(s)>1)return te.Children.only(null);return te.isValidElement(s)?s.props.children:null}else return m});return d(t,{...u,ref:l,children:te.isValidElement(s)?te.cloneElement(s,void 0,h):null})}return d(t,{...u,ref:l,children:i})});return n.displayName=`${e}.Slot`,n}function Y1(e){let t=te.forwardRef((n,r)=>{let{children:l,...i}=n;if(te.isValidElement(l)){let u=J1(l),o=Z1(i,l.props);if(l.type!==te.Fragment)o.ref=r?ka(r,u):u;return te.cloneElement(l,o)}return te.Children.count(l)>1?te.Children.only(null):null});return t.displayName=`${e}.SlotClone`,t}var K1=Symbol("radix.slottable");function X1(e){return te.isValidElement(e)&&typeof e.type==="function"&&"__radixId"in e.type&&e.type.__radixId===K1}function Z1(e,t){let n={...t};for(let r in t){let l=e[r],i=t[r];if(/^on[A-Z]/.test(r)){if(l&&i)n[r]=(...o)=>{let a=i(...o);return l(...o),a};else if(l)n[r]=l}else if(r==="style")n[r]={...l,...i};else if(r==="className")n[r]=[l,i].filter(Boolean).join(" ")}return{...e,...n}}function J1(e){let t=Object.getOwnPropertyDescriptor(e.props,"ref")?.get,n=t&&"isReactWarning"in t&&t.isReactWarning;if(n)return e.ref;if(t=Object.getOwnPropertyDescriptor(e,"ref")?.get,n=t&&"isReactWarning"in t&&t.isReactWarning,n)return e.props.ref;return e.props.ref||e.ref}var G1=["a","button","div","form","h2","h3","img","input","label","li","nav","ol","p","select","span","svg","ul"],Pe=G1.reduce((e,t)=>{let n=Oi(`Primitive.${t}`),r=Id.forwardRef((l,i)=>{let{asChild:u,...o}=l,a=u?n:t;if(typeof window<"u")window[Symbol.for("radix-ui")]=!0;return d(a,{...o,ref:i})});return r.displayName=`Primitive.${t}`,{...e,[t]:r}},{});function Vd(e,t){if(e)Rd.flushSync(()=>e.dispatchEvent(t))}var Rn=B(D(),1);function Ut(e){let t=Rn.useRef(e);return Rn.useEffect(()=>{t.current=e}),Rn.useMemo(()=>(...n)=>t.current?.(...n),[])}var Yd=B(D(),1);function Kd(e,t=globalThis?.document){let n=Ut(e);Yd.useEffect(()=>{let r=(l)=>{if(l.key==="Escape")n(l)};return t.addEventListener("keydown",r,{capture:!0}),()=>t.removeEventListener("keydown",r,{capture:!0})},[n,t])}var q1="DismissableLayer",Sa="dismissableLayer.update",b1="dismissableLayer.pointerDownOutside",j1="dismissableLayer.focusOutside",Xd,Jd=J.createContext({layers:new Set,layersWithOutsidePointerEventsDisabled:new Set,branches:new Set}),Na=J.forwardRef((e,t)=>{let{disableOutsidePointerEvents:n=!1,onEscapeKeyDown:r,onPointerDownOutside:l,onFocusOutside:i,onInteractOutside:u,onDismiss:o,...a}=e,s=J.useContext(Jd),[h,m]=J.useState(null),y=h?.ownerDocument??globalThis?.document,[,x]=J.useState({}),N=Je(t,(E)=>m(E)),g=Array.from(s.layers),[L]=[...s.layersWithOutsidePointerEventsDisabled].slice(-1),c=g.indexOf(L),f=h?g.indexOf(h):-1,p=s.layersWithOutsidePointerEventsDisabled.size>0,w=f>=c,C=ny((E)=>{let T=E.target,O=[...s.branches].some((P)=>P.contains(T));if(!w||O)return;if(l?.(E),u?.(E),!E.defaultPrevented)o?.()},y),F=ry((E)=>{let T=E.target;if([...s.branches].some((P)=>P.contains(T)))return;if(i?.(E),u?.(E),!E.defaultPrevented)o?.()},y);return Kd((E)=>{if(f!==s.layers.size-1)return;if(r?.(E),!E.defaultPrevented&&o)E.preventDefault(),o()},y),J.useEffect(()=>{if(!h)return;if(n){if(s.layersWithOutsidePointerEventsDisabled.size===0)Xd=y.body.style.pointerEvents,y.body.style.pointerEvents="none";s.layersWithOutsidePointerEventsDisabled.add(h)}return s.layers.add(h),Zd(),()=>{if(n&&s.layersWithOutsidePointerEventsDisabled.size===1)y.body.style.pointerEvents=Xd}},[h,y,n,s]),J.useEffect(()=>{return()=>{if(!h)return;s.layers.delete(h),s.layersWithOutsidePointerEventsDisabled.delete(h),Zd()}},[h,s]),J.useEffect(()=>{let E=()=>x({});return document.addEventListener(Sa,E),()=>document.removeEventListener(Sa,E)},[]),d(Pe.div,{...a,ref:N,style:{pointerEvents:p?w?"auto":"none":void 0,...e.style},onFocusCapture:rt(e.onFocusCapture,F.onFocusCapture),onBlurCapture:rt(e.onBlurCapture,F.onBlurCapture),onPointerDownCapture:rt(e.onPointerDownCapture,C.onPointerDownCapture)})});Na.displayName=q1;var ey="DismissableLayerBranch",ty=J.forwardRef((e,t)=>{let n=J.useContext(Jd),r=J.useRef(null),l=Je(t,r);return J.useEffect(()=>{let i=r.current;if(i)return n.branches.add(i),()=>{n.branches.delete(i)}},[n.branches]),d(Pe.div,{...e,ref:l})});ty.displayName=ey;function ny(e,t=globalThis?.document){let n=Ut(e),r=J.useRef(!1),l=J.useRef(()=>{});return J.useEffect(()=>{let i=(o)=>{if(o.target&&!r.current){let s=function(){Gd(b1,n,h,{discrete:!0})};var a=s;let h={originalEvent:o};if(o.pointerType==="touch")t.removeEventListener("click",l.current),l.current=s,t.addEventListener("click",l.current,{once:!0});else s()}else t.removeEventListener("click",l.current);r.current=!1},u=window.setTimeout(()=>{t.addEventListener("pointerdown",i)},0);return()=>{window.clearTimeout(u),t.removeEventListener("pointerdown",i),t.removeEventListener("click",l.current)}},[t,n]),{onPointerDownCapture:()=>r.current=!0}}function ry(e,t=globalThis?.document){let n=Ut(e),r=J.useRef(!1);return J.useEffect(()=>{let l=(i)=>{if(i.target&&!r.current)Gd(j1,n,{originalEvent:i},{discrete:!1})};return t.addEventListener("focusin",l),()=>t.removeEventListener("focusin",l)},[t,n]),{onFocusCapture:()=>r.current=!0,onBlurCapture:()=>r.current=!1}}function Zd(){let e=new CustomEvent(Sa);document.dispatchEvent(e)}function Gd(e,t,n,{discrete:r}){let l=n.originalEvent.target,i=new CustomEvent(e,{bubbles:!1,cancelable:!0,detail:n});if(t)l.addEventListener(e,t,{once:!0});if(r)Vd(l,i);else l.dispatchEvent(i)}var We=B(D(),1);var Ea="focusScope.autoFocusOnMount",Ca="focusScope.autoFocusOnUnmount",qd={bubbles:!1,cancelable:!0},ly="FocusScope",Fa=We.forwardRef((e,t)=>{let{loop:n=!1,trapped:r=!1,onMountAutoFocus:l,onUnmountAutoFocus:i,...u}=e,[o,a]=We.useState(null),s=Ut(l),h=Ut(i),m=We.useRef(null),y=Je(t,(g)=>a(g)),x=We.useRef({paused:!1,pause(){this.paused=!0},resume(){this.paused=!1}}).current;We.useEffect(()=>{if(r){let f=function(F){if(x.paused||!o)return;let E=F.target;if(o.contains(E))m.current=E;else Ht(m.current,{select:!0})},p=function(F){if(x.paused||!o)return;let E=F.relatedTarget;if(E===null)return;if(!o.contains(E))Ht(m.current,{select:!0})},w=function(F){if(document.activeElement!==document.body)return;for(let T of F)if(T.removedNodes.length>0)Ht(o)};var g=f,L=p,c=w;document.addEventListener("focusin",f),document.addEventListener("focusout",p);let C=new MutationObserver(w);if(o)C.observe(o,{childList:!0,subtree:!0});return()=>{document.removeEventListener("focusin",f),document.removeEventListener("focusout",p),C.disconnect()}}},[r,o,x.paused]),We.useEffect(()=>{if(o){jd.add(x);let g=document.activeElement;if(!o.contains(g)){let c=new CustomEvent(Ea,qd);if(o.addEventListener(Ea,s),o.dispatchEvent(c),!c.defaultPrevented){if(iy(fy(t0(o)),{select:!0}),document.activeElement===g)Ht(o)}}return()=>{o.removeEventListener(Ea,s),setTimeout(()=>{let c=new CustomEvent(Ca,qd);if(o.addEventListener(Ca,h),o.dispatchEvent(c),!c.defaultPrevented)Ht(g??document.body,{select:!0});o.removeEventListener(Ca,h),jd.remove(x)},0)}}},[o,s,h,x]);let N=We.useCallback((g)=>{if(!n&&!r)return;if(x.paused)return;let L=g.key==="Tab"&&!g.altKey&&!g.ctrlKey&&!g.metaKey,c=document.activeElement;if(L&&c){let f=g.currentTarget,[p,w]=uy(f);if(!(p&&w)){if(c===f)g.preventDefault()}else if(!g.shiftKey&&c===w){if(g.preventDefault(),n)Ht(p,{select:!0})}else if(g.shiftKey&&c===p){if(g.preventDefault(),n)Ht(w,{select:!0})}}},[n,r,x.paused]);return d(Pe.div,{tabIndex:-1,...u,ref:y,onKeyDown:N})});Fa.displayName=ly;function iy(e,{select:t=!1}={}){let n=document.activeElement;for(let r of e)if(Ht(r,{select:t}),document.activeElement!==n)return}function uy(e){let t=t0(e),n=bd(t,e),r=bd(t.reverse(),e);return[n,r]}function t0(e){let t=[],n=document.createTreeWalker(e,NodeFilter.SHOW_ELEMENT,{acceptNode:(r)=>{let l=r.tagName==="INPUT"&&r.type==="hidden";if(r.disabled||r.hidden||l)return NodeFilter.FILTER_SKIP;return r.tabIndex>=0?NodeFilter.FILTER_ACCEPT:NodeFilter.FILTER_SKIP}});while(n.nextNode())t.push(n.currentNode);return t}function bd(e,t){for(let n of e)if(!oy(n,{upTo:t}))return n}function oy(e,{upTo:t}){if(getComputedStyle(e).visibility==="hidden")return!0;while(e){if(t!==void 0&&e===t)return!1;if(getComputedStyle(e).display==="none")return!0;e=e.parentElement}return!1}function ay(e){return e instanceof HTMLInputElement&&"select"in e}function Ht(e,{select:t=!1}={}){if(e&&e.focus){let n=document.activeElement;if(e.focus({preventScroll:!0}),e!==n&&ay(e)&&t)e.select()}}var jd=sy();function sy(){let e=[];return{add(t){let n=e[0];if(t!==n)n?.pause();e=e0(e,t),e.unshift(t)},remove(t){e=e0(e,t),e[0]?.resume()}}}function e0(e,t){let n=[...e],r=n.indexOf(t);if(r!==-1)n.splice(r,1);return n}function fy(e){return e.filter((t)=>t.tagName!=="A")}var Ui=B(D(),1),n0=B($i(),1);var cy="Portal",za=Ui.forwardRef((e,t)=>{let{container:n,...r}=e,[l,i]=Ui.useState(!1);mt(()=>i(!0),[]);let u=n||l&&globalThis?.document?.body;return u?n0.default.createPortal(d(Pe.div,{...r,ref:t}),u):null});za.displayName=cy;var Ee=B(D(),1);var r0=B(D(),1);function dy(e,t){return r0.useReducer((n,r)=>{return t[n][r]??n},e)}var Xr=(e)=>{let{present:t,children:n}=e,r=py(t),l=typeof n==="function"?n({present:r.isPresent}):Ee.Children.only(n),i=Je(r.ref,hy(l));return typeof n==="function"||r.isPresent?Ee.cloneElement(l,{ref:i}):null};Xr.displayName="Presence";function py(e){let[t,n]=Ee.useState(),r=Ee.useRef(null),l=Ee.useRef(e),i=Ee.useRef("none"),u=e?"mounted":"unmounted",[o,a]=dy(u,{mounted:{UNMOUNT:"unmounted",ANIMATION_OUT:"unmountSuspended"},unmountSuspended:{MOUNT:"mounted",ANIMATION_END:"unmounted"},unmounted:{MOUNT:"mounted"}});return Ee.useEffect(()=>{let s=Hi(r.current);i.current=o==="mounted"?s:"none"},[o]),mt(()=>{let s=r.current,h=l.current;if(h!==e){let y=i.current,x=Hi(s);if(e)a("MOUNT");else if(x==="none"||s?.display==="none")a("UNMOUNT");else if(h&&y!==x)a("ANIMATION_OUT");else a("UNMOUNT");l.current=e}},[e,a]),mt(()=>{if(t){let s,h=t.ownerDocument.defaultView??window,m=(x)=>{let g=Hi(r.current).includes(CSS.escape(x.animationName));if(x.target===t&&g){if(a("ANIMATION_END"),!l.current){let L=t.style.animationFillMode;t.style.animationFillMode="forwards",s=h.setTimeout(()=>{if(t.style.animationFillMode==="forwards")t.style.animationFillMode=L})}}},y=(x)=>{if(x.target===t)i.current=Hi(r.current)};return t.addEventListener("animationstart",y),t.addEventListener("animationcancel",m),t.addEventListener("animationend",m),()=>{h.clearTimeout(s),t.removeEventListener("animationstart",y),t.removeEventListener("animationcancel",m),t.removeEventListener("animationend",m)}}else a("ANIMATION_END")},[t,a]),{isPresent:["mounted","unmountSuspended"].includes(o),ref:Ee.useCallback((s)=>{r.current=s?getComputedStyle(s):null,n(s)},[])}}function Hi(e){return e?.animationName||"none"}function hy(e){let t=Object.getOwnPropertyDescriptor(e.props,"ref")?.get,n=t&&"isReactWarning"in t&&t.isReactWarning;if(n)return e.ref;if(t=Object.getOwnPropertyDescriptor(e,"ref")?.get,n=t&&"isReactWarning"in t&&t.isReactWarning,n)return e.props.ref;return e.props.ref||e.ref}var i0=B(D(),1),La=0;function u0(){i0.useEffect(()=>{let e=document.querySelectorAll("[data-radix-focus-guard]");return document.body.insertAdjacentElement("afterbegin",e[0]??l0()),document.body.insertAdjacentElement("beforeend",e[1]??l0()),La++,()=>{if(La===1)document.querySelectorAll("[data-radix-focus-guard]").forEach((t)=>t.remove());La--}},[])}function l0(){let e=document.createElement("span");return e.setAttribute("data-radix-focus-guard",""),e.tabIndex=0,e.style.outline="none",e.style.opacity="0",e.style.position="fixed",e.style.pointerEvents="none",e}var Ce=function(){return Ce=Object.assign||function(t){for(var n,r=1,l=arguments.length;r<l;r++){n=arguments[r];for(var i in n)if(Object.prototype.hasOwnProperty.call(n,i))t[i]=n[i]}return t},Ce.apply(this,arguments)};function Qi(e,t){var n={};for(var r in e)if(Object.prototype.hasOwnProperty.call(e,r)&&t.indexOf(r)<0)n[r]=e[r];if(e!=null&&typeof Object.getOwnPropertySymbols==="function"){for(var l=0,r=Object.getOwnPropertySymbols(e);l<r.length;l++)if(t.indexOf(r[l])<0&&Object.prototype.propertyIsEnumerable.call(e,r[l]))n[r[l]]=e[r[l]]}return n}function o0(e,t,n){if(n||arguments.length===2){for(var r=0,l=t.length,i;r<l;r++)if(i||!(r in t)){if(!i)i=Array.prototype.slice.call(t,0,r);i[r]=t[r]}}return e.concat(i||Array.prototype.slice.call(t))}var Vi=B(D(),1);var pe=B(D(),1);var on="right-scroll-bar-position",an="width-before-scroll-bar",Ta="with-scroll-bars-hidden",Pa="--removed-body-scroll-bar-size";function Wi(e,t){if(typeof e==="function")e(t);else if(e)e.current=t;return e}var a0=B(D(),1);function s0(e,t){var n=a0.useState(function(){return{value:e,callback:t,facade:{get current(){return n.value},set current(r){var l=n.value;if(l!==r)n.value=r,n.callback(r,l)}}}})[0];return n.callback=t,n.facade}var Ai=B(D(),1);var my=typeof window<"u"?Ai.useLayoutEffect:Ai.useEffect,f0=new WeakMap;function Ba(e,t){var n=s0(t||null,function(r){return e.forEach(function(l){return Wi(l,r)})});return my(function(){var r=f0.get(n);if(r){var l=new Set(r),i=new Set(e),u=n.current;l.forEach(function(o){if(!i.has(o))Wi(o,null)}),i.forEach(function(o){if(!l.has(o))Wi(o,u)})}f0.set(n,e)},[e]),n}function yy(e){return e}function gy(e,t){if(t===void 0)t=yy;var n=[],r=!1,l={read:function(){if(r)throw Error("Sidecar: could not `read` from an `assigned` medium. `read` could be used only with `useMedium`.");if(n.length)return n[n.length-1];return e},useMedium:function(i){var u=t(i,r);return n.push(u),function(){n=n.filter(function(o){return o!==u})}},assignSyncMedium:function(i){r=!0;while(n.length){var u=n;n=[],u.forEach(i)}n={push:function(o){return i(o)},filter:function(){return n}}},assignMedium:function(i){r=!0;var u=[];if(n.length){var o=n;n=[],o.forEach(i),u=n}var a=function(){var h=u;u=[],h.forEach(i)},s=function(){return Promise.resolve().then(a)};s(),n={push:function(h){u.push(h),s()},filter:function(h){return u=u.filter(h),n}}}};return l}function Ma(e){if(e===void 0)e={};var t=gy(null);return t.options=Ce({async:!0,ssr:!1},e),t}var c0=B(D(),1),d0=function(e){var t=e.sideCar,n=Qi(e,["sideCar"]);if(!t)throw Error("Sidecar: please provide `sideCar` property to import the right car");var r=t.read();if(!r)throw Error("Sidecar medium not found");return c0.createElement(r,Ce({},n))};d0.isSideCarExport=!0;function $a(e,t){return e.useMedium(t),d0}var Ii=Ma();var Da=function(){return},Zr=pe.forwardRef(function(e,t){var n=pe.useRef(null),r=pe.useState({onScrollCapture:Da,onWheelCapture:Da,onTouchMoveCapture:Da}),l=r[0],i=r[1],u=e.forwardProps,o=e.children,a=e.className,s=e.removeScrollBar,h=e.enabled,m=e.shards,y=e.sideCar,x=e.noRelative,N=e.noIsolation,g=e.inert,L=e.allowPinchZoom,c=e.as,f=c===void 0?"div":c,p=e.gapMode,w=Qi(e,["forwardProps","children","className","removeScrollBar","enabled","shards","sideCar","noRelative","noIsolation","inert","allowPinchZoom","as","gapMode"]),C=y,F=Ba([n,t]),E=Ce(Ce({},w),l);return pe.createElement(pe.Fragment,null,h&&pe.createElement(C,{sideCar:Ii,removeScrollBar:s,shards:m,noRelative:x,noIsolation:N,inert:g,setCallbacks:i,allowPinchZoom:!!L,lockRef:n,gapMode:p}),u?pe.cloneElement(pe.Children.only(o),Ce(Ce({},E),{ref:F})):pe.createElement(f,Ce({},E,{className:a,ref:F}),o))});Zr.defaultProps={enabled:!0,removeScrollBar:!0,inert:!1};Zr.classNames={fullWidth:an,zeroRight:on};var R=B(D(),1);var Yn=B(D(),1);var m0=B(D(),1);var p0;var h0=function(){if(p0)return p0;if(typeof __webpack_nonce__<"u")return __webpack_nonce__;return};function vy(){if(!document)return null;var e=document.createElement("style");e.type="text/css";var t=h0();if(t)e.setAttribute("nonce",t);return e}function wy(e,t){if(e.styleSheet)e.styleSheet.cssText=t;else e.appendChild(document.createTextNode(t))}function ky(e){var t=document.head||document.getElementsByTagName("head")[0];t.appendChild(e)}var _a=function(){var e=0,t=null;return{add:function(n){if(e==0){if(t=vy())wy(t,n),ky(t)}e++},remove:function(){if(e--,!e&&t)t.parentNode&&t.parentNode.removeChild(t),t=null}}};var Oa=function(){var e=_a();return function(t,n){m0.useEffect(function(){return e.add(t),function(){e.remove()}},[t&&n])}};var Jr=function(){var e=Oa(),t=function(n){var{styles:r,dynamic:l}=n;return e(r,l),null};return t};var xy={left:0,top:0,right:0,gap:0},Ua=function(e){return parseInt(e||"",10)||0},Sy=function(e){var t=window.getComputedStyle(document.body),n=t[e==="padding"?"paddingLeft":"marginLeft"],r=t[e==="padding"?"paddingTop":"marginTop"],l=t[e==="padding"?"paddingRight":"marginRight"];return[Ua(n),Ua(r),Ua(l)]},Ha=function(e){if(e===void 0)e="margin";if(typeof window>"u")return xy;var t=Sy(e),n=document.documentElement.clientWidth,r=window.innerWidth;return{left:t[0],top:t[1],right:t[2],gap:Math.max(0,r-n+t[2]-t[0])}};var Ny=Jr(),Vn="data-scroll-locked",Ey=function(e,t,n,r){var{left:l,top:i,right:u,gap:o}=e;if(n===void 0)n="margin";return`
  .`.concat(Ta,` {
   overflow: hidden `).concat(r,`;
   padding-right: `).concat(o,"px ").concat(r,`;
  }
  body[`).concat(Vn,`] {
    overflow: hidden `).concat(r,`;
    overscroll-behavior: contain;
    `).concat([t&&"position: relative ".concat(r,";"),n==="margin"&&`
    padding-left: `.concat(l,`px;
    padding-top: `).concat(i,`px;
    padding-right: `).concat(u,`px;
    margin-left:0;
    margin-top:0;
    margin-right: `).concat(o,"px ").concat(r,`;
    `),n==="padding"&&"padding-right: ".concat(o,"px ").concat(r,";")].filter(Boolean).join(""),`
  }
  
  .`).concat(on,` {
    right: `).concat(o,"px ").concat(r,`;
  }
  
  .`).concat(an,` {
    margin-right: `).concat(o,"px ").concat(r,`;
  }
  
  .`).concat(on," .").concat(on,` {
    right: 0 `).concat(r,`;
  }
  
  .`).concat(an," .").concat(an,` {
    margin-right: 0 `).concat(r,`;
  }
  
  body[`).concat(Vn,`] {
    `).concat(Pa,": ").concat(o,`px;
  }
`)},y0=function(){var e=parseInt(document.body.getAttribute(Vn)||"0",10);return isFinite(e)?e:0},Cy=function(){Yn.useEffect(function(){return document.body.setAttribute(Vn,(y0()+1).toString()),function(){var e=y0()-1;if(e<=0)document.body.removeAttribute(Vn);else document.body.setAttribute(Vn,e.toString())}},[])},Qa=function(e){var{noRelative:t,noImportant:n,gapMode:r}=e,l=r===void 0?"margin":r;Cy();var i=Yn.useMemo(function(){return Ha(l)},[l]);return Yn.createElement(Ny,{styles:Ey(i,!t,l,!n?"!important":"")})};var Wa=!1;if(typeof window<"u")try{Kn=Object.defineProperty({},"passive",{get:function(){return Wa=!0,!0}}),window.addEventListener("test",Kn,Kn),window.removeEventListener("test",Kn,Kn)}catch(e){Wa=!1}var Kn,sn=Wa?{passive:!1}:!1;var Fy=function(e){return e.tagName==="TEXTAREA"},g0=function(e,t){if(!(e instanceof Element))return!1;var n=window.getComputedStyle(e);return n[t]!=="hidden"&&!(n.overflowY===n.overflowX&&!Fy(e)&&n[t]==="visible")},zy=function(e){return g0(e,"overflowY")},Ly=function(e){return g0(e,"overflowX")},Aa=function(e,t){var n=t.ownerDocument,r=t;do{if(typeof ShadowRoot<"u"&&r instanceof ShadowRoot)r=r.host;var l=v0(e,r);if(l){var i=w0(e,r),u=i[1],o=i[2];if(u>o)return!0}r=r.parentNode}while(r&&r!==n.body);return!1},Ty=function(e){var{scrollTop:t,scrollHeight:n,clientHeight:r}=e;return[t,n,r]},Py=function(e){var{scrollLeft:t,scrollWidth:n,clientWidth:r}=e;return[t,n,r]},v0=function(e,t){return e==="v"?zy(t):Ly(t)},w0=function(e,t){return e==="v"?Ty(t):Py(t)},By=function(e,t){return e==="h"&&t==="rtl"?-1:1},k0=function(e,t,n,r,l){var i=By(e,window.getComputedStyle(t).direction),u=i*r,o=n.target,a=t.contains(o),s=!1,h=u>0,m=0,y=0;do{if(!o)break;var x=w0(e,o),N=x[0],g=x[1],L=x[2],c=g-L-i*N;if(N||c){if(v0(e,o))m+=c,y+=N}var f=o.parentNode;o=f&&f.nodeType===Node.DOCUMENT_FRAGMENT_NODE?f.host:f}while(!a&&o!==document.body||a&&(t.contains(o)||t===o));if(h&&(l&&Math.abs(m)<1||!l&&u>m))s=!0;else if(!h&&(l&&Math.abs(y)<1||!l&&-u>y))s=!0;return s};var Ri=function(e){return"changedTouches"in e?[e.changedTouches[0].clientX,e.changedTouches[0].clientY]:[0,0]},x0=function(e){return[e.deltaX,e.deltaY]},S0=function(e){return e&&"current"in e?e.current:e},My=function(e,t){return e[0]===t[0]&&e[1]===t[1]},$y=function(e){return`
  .block-interactivity-`.concat(e,` {pointer-events: none;}
  .allow-interactivity-`).concat(e,` {pointer-events: all;}
`)},Dy=0,Xn=[];function N0(e){var t=R.useRef([]),n=R.useRef([0,0]),r=R.useRef(),l=R.useState(Dy++)[0],i=R.useState(Jr)[0],u=R.useRef(e);R.useEffect(function(){u.current=e},[e]),R.useEffect(function(){if(e.inert){document.body.classList.add("block-interactivity-".concat(l));var g=o0([e.lockRef.current],(e.shards||[]).map(S0),!0).filter(Boolean);return g.forEach(function(L){return L.classList.add("allow-interactivity-".concat(l))}),function(){document.body.classList.remove("block-interactivity-".concat(l)),g.forEach(function(L){return L.classList.remove("allow-interactivity-".concat(l))})}}return},[e.inert,e.lockRef.current,e.shards]);var o=R.useCallback(function(g,L){if("touches"in g&&g.touches.length===2||g.type==="wheel"&&g.ctrlKey)return!u.current.allowPinchZoom;var c=Ri(g),f=n.current,p="deltaX"in g?g.deltaX:f[0]-c[0],w="deltaY"in g?g.deltaY:f[1]-c[1],C,F=g.target,E=Math.abs(p)>Math.abs(w)?"h":"v";if("touches"in g&&E==="h"&&F.type==="range")return!1;var T=Aa(E,F);if(!T)return!0;if(T)C=E;else C=E==="v"?"h":"v",T=Aa(E,F);if(!T)return!1;if(!r.current&&"changedTouches"in g&&(p||w))r.current=C;if(!C)return!0;var O=r.current||C;return k0(O,L,g,O==="h"?p:w,!0)},[]),a=R.useCallback(function(g){var L=g;if(!Xn.length||Xn[Xn.length-1]!==i)return;var c="deltaY"in L?x0(L):Ri(L),f=t.current.filter(function(C){return C.name===L.type&&(C.target===L.target||L.target===C.shadowParent)&&My(C.delta,c)})[0];if(f&&f.should){if(L.cancelable)L.preventDefault();return}if(!f){var p=(u.current.shards||[]).map(S0).filter(Boolean).filter(function(C){return C.contains(L.target)}),w=p.length>0?o(L,p[0]):!u.current.noIsolation;if(w){if(L.cancelable)L.preventDefault()}}},[]),s=R.useCallback(function(g,L,c,f){var p={name:g,delta:L,target:c,should:f,shadowParent:_y(c)};t.current.push(p),setTimeout(function(){t.current=t.current.filter(function(w){return w!==p})},1)},[]),h=R.useCallback(function(g){n.current=Ri(g),r.current=void 0},[]),m=R.useCallback(function(g){s(g.type,x0(g),g.target,o(g,e.lockRef.current))},[]),y=R.useCallback(function(g){s(g.type,Ri(g),g.target,o(g,e.lockRef.current))},[]);R.useEffect(function(){return Xn.push(i),e.setCallbacks({onScrollCapture:m,onWheelCapture:m,onTouchMoveCapture:y}),document.addEventListener("wheel",a,sn),document.addEventListener("touchmove",a,sn),document.addEventListener("touchstart",h,sn),function(){Xn=Xn.filter(function(g){return g!==i}),document.removeEventListener("wheel",a,sn),document.removeEventListener("touchmove",a,sn),document.removeEventListener("touchstart",h,sn)}},[]);var{removeScrollBar:x,inert:N}=e;return R.createElement(R.Fragment,null,N?R.createElement(i,{styles:$y(l)}):null,x?R.createElement(Qa,{noRelative:e.noRelative,gapMode:e.gapMode}):null)}function _y(e){var t=null;while(e!==null){if(e instanceof ShadowRoot)t=e.host,e=e.host;e=e.parentNode}return t}var E0=$a(Ii,N0);var C0=Vi.forwardRef(function(e,t){return Vi.createElement(Zr,Ce({},e,{ref:t,sideCar:E0}))});C0.classNames=Zr.classNames;var Ia=C0;var Oy=function(e){if(typeof document>"u")return null;var t=Array.isArray(e)?e[0]:e;return t.ownerDocument.body},Zn=new WeakMap,Yi=new WeakMap,Ki={},Ra=0,F0=function(e){return e&&(e.host||F0(e.parentNode))},Uy=function(e,t){return t.map(function(n){if(e.contains(n))return n;var r=F0(n);if(r&&e.contains(r))return r;return console.error("aria-hidden",n,"in not contained inside",e,". Doing nothing"),null}).filter(function(n){return Boolean(n)})},Hy=function(e,t,n,r){var l=Uy(t,Array.isArray(e)?e:[e]);if(!Ki[n])Ki[n]=new WeakMap;var i=Ki[n],u=[],o=new Set,a=new Set(l),s=function(m){if(!m||o.has(m))return;o.add(m),s(m.parentNode)};l.forEach(s);var h=function(m){if(!m||a.has(m))return;Array.prototype.forEach.call(m.children,function(y){if(o.has(y))h(y);else try{var x=y.getAttribute(r),N=x!==null&&x!=="false",g=(Zn.get(y)||0)+1,L=(i.get(y)||0)+1;if(Zn.set(y,g),i.set(y,L),u.push(y),g===1&&N)Yi.set(y,!0);if(L===1)y.setAttribute(n,"true");if(!N)y.setAttribute(r,"true")}catch(c){console.error("aria-hidden: cannot operate on ",y,c)}})};return h(t),o.clear(),Ra++,function(){if(u.forEach(function(m){var y=Zn.get(m)-1,x=i.get(m)-1;if(Zn.set(m,y),i.set(m,x),!y){if(!Yi.has(m))m.removeAttribute(r);Yi.delete(m)}if(!x)m.removeAttribute(n)}),Ra--,!Ra)Zn=new WeakMap,Zn=new WeakMap,Yi=new WeakMap,Ki={}}},z0=function(e,t,n){if(n===void 0)n="data-aria-hidden";var r=Array.from(Array.isArray(e)?e:[e]),l=t||Oy(e);if(!l)return function(){return null};return r.push.apply(r,Array.from(l.querySelectorAll("[aria-live], script"))),Hy(r,l,n,"aria-hidden")};var Zi="Dialog",[L0,gw]=Qd(Zi),[Qy,Ge]=L0(Zi),T0=(e)=>{let{__scopeDialog:t,children:n,open:r,defaultOpen:l,onOpenChange:i,modal:u=!0}=e,o=Q.useRef(null),a=Q.useRef(null),[s,h]=Ad({prop:r,defaultProp:l??!1,onChange:i,caller:Zi});return d(Qy,{scope:t,triggerRef:o,contentRef:a,contentId:Di(),titleId:Di(),descriptionId:Di(),open:s,onOpenChange:h,onOpenToggle:Q.useCallback(()=>h((m)=>!m),[h]),modal:u,children:n})};T0.displayName=Zi;var P0="DialogTrigger",Wy=Q.forwardRef((e,t)=>{let{__scopeDialog:n,...r}=e,l=Ge(P0,n),i=Je(t,l.triggerRef);return d(Pe.button,{type:"button","aria-haspopup":"dialog","aria-expanded":l.open,"aria-controls":l.contentId,"data-state":Ka(l.open),...r,ref:i,onClick:rt(e.onClick,l.onOpenToggle)})});Wy.displayName=P0;var Va="DialogPortal",[Ay,B0]=L0(Va,{forceMount:void 0}),M0=(e)=>{let{__scopeDialog:t,forceMount:n,children:r,container:l}=e,i=Ge(Va,t);return d(Ay,{scope:t,forceMount:n,children:Q.Children.map(r,(u)=>d(Xr,{present:n||i.open,children:d(za,{asChild:!0,container:l,children:u})}))})};M0.displayName=Va;var Xi="DialogOverlay",$0=Q.forwardRef((e,t)=>{let n=B0(Xi,e.__scopeDialog),{forceMount:r=n.forceMount,...l}=e,i=Ge(Xi,e.__scopeDialog);return i.modal?d(Xr,{present:r||i.open,children:d(Ry,{...l,ref:t})}):null});$0.displayName=Xi;var Iy=Oi("DialogOverlay.RemoveScroll"),Ry=Q.forwardRef((e,t)=>{let{__scopeDialog:n,...r}=e,l=Ge(Xi,n);return d(Ia,{as:Iy,allowPinchZoom:!0,shards:[l.contentRef],children:d(Pe.div,{"data-state":Ka(l.open),...r,ref:t,style:{pointerEvents:"auto",...r.style}})})}),fn="DialogContent",D0=Q.forwardRef((e,t)=>{let n=B0(fn,e.__scopeDialog),{forceMount:r=n.forceMount,...l}=e,i=Ge(fn,e.__scopeDialog);return d(Xr,{present:r||i.open,children:i.modal?d(Vy,{...l,ref:t}):d(Yy,{...l,ref:t})})});D0.displayName=fn;var Vy=Q.forwardRef((e,t)=>{let n=Ge(fn,e.__scopeDialog),r=Q.useRef(null),l=Je(t,n.contentRef,r);return Q.useEffect(()=>{let i=r.current;if(i)return z0(i)},[]),d(_0,{...e,ref:l,trapFocus:n.open,disableOutsidePointerEvents:!0,onCloseAutoFocus:rt(e.onCloseAutoFocus,(i)=>{i.preventDefault(),n.triggerRef.current?.focus()}),onPointerDownOutside:rt(e.onPointerDownOutside,(i)=>{let u=i.detail.originalEvent,o=u.button===0&&u.ctrlKey===!0;if(u.button===2||o)i.preventDefault()}),onFocusOutside:rt(e.onFocusOutside,(i)=>i.preventDefault())})}),Yy=Q.forwardRef((e,t)=>{let n=Ge(fn,e.__scopeDialog),r=Q.useRef(!1),l=Q.useRef(!1);return d(_0,{...e,ref:t,trapFocus:!1,disableOutsidePointerEvents:!1,onCloseAutoFocus:(i)=>{if(e.onCloseAutoFocus?.(i),!i.defaultPrevented){if(!r.current)n.triggerRef.current?.focus();i.preventDefault()}r.current=!1,l.current=!1},onInteractOutside:(i)=>{if(e.onInteractOutside?.(i),!i.defaultPrevented){if(r.current=!0,i.detail.originalEvent.type==="pointerdown")l.current=!0}let u=i.target;if(n.triggerRef.current?.contains(u))i.preventDefault();if(i.detail.originalEvent.type==="focusin"&&l.current)i.preventDefault()}})}),_0=Q.forwardRef((e,t)=>{let{__scopeDialog:n,trapFocus:r,onOpenAutoFocus:l,onCloseAutoFocus:i,...u}=e,o=Ge(fn,n),a=Q.useRef(null),s=Je(t,a);return u0(),v(un,{children:[d(Fa,{asChild:!0,loop:!0,trapped:r,onMountAutoFocus:l,onUnmountAutoFocus:i,children:d(Na,{role:"dialog",id:o.contentId,"aria-describedby":o.descriptionId,"aria-labelledby":o.titleId,"data-state":Ka(o.open),...u,ref:s,onDismiss:()=>o.onOpenChange(!1)})}),v(un,{children:[d(Xy,{titleId:o.titleId}),d(Jy,{contentRef:a,descriptionId:o.descriptionId})]})]})}),Ya="DialogTitle",O0=Q.forwardRef((e,t)=>{let{__scopeDialog:n,...r}=e,l=Ge(Ya,n);return d(Pe.h2,{id:l.titleId,...r,ref:t})});O0.displayName=Ya;var U0="DialogDescription",Ky=Q.forwardRef((e,t)=>{let{__scopeDialog:n,...r}=e,l=Ge(U0,n);return d(Pe.p,{id:l.descriptionId,...r,ref:t})});Ky.displayName=U0;var H0="DialogClose",Q0=Q.forwardRef((e,t)=>{let{__scopeDialog:n,...r}=e,l=Ge(H0,n);return d(Pe.button,{type:"button",...r,ref:t,onClick:rt(e.onClick,()=>l.onOpenChange(!1))})});Q0.displayName=H0;function Ka(e){return e?"open":"closed"}var W0="DialogTitleWarning",[vw,A0]=Hd(W0,{contentName:fn,titleName:Ya,docsSlug:"dialog"}),Xy=({titleId:e})=>{let t=A0(W0),n=`\`${t.contentName}\` requires a \`${t.titleName}\` for the component to be accessible for screen reader users.
If you want to hide the \`${t.titleName}\`, you can wrap it with our VisuallyHidden component.
For more information, see https://radix-ui.com/primitives/docs/components/${t.docsSlug}`;return Q.useEffect(()=>{if(e){if(!document.getElementById(e))console.error(n)}},[n,e]),null},Zy="DialogDescriptionWarning",Jy=({contentRef:e,descriptionId:t})=>{let r=`Warning: Missing \`Description\` or \`aria-describedby={undefined}\` for {${A0(Zy).contentName}}.`;return Q.useEffect(()=>{let l=e.current?.getAttribute("aria-describedby");if(t&&l){if(!document.getElementById(t))console.warn(r)}},[r,e,t]),null},I0=T0;var R0=M0,V0=$0,Y0=D0,K0=O0;var X0=Q0;var Gi=B(D(),1);var Z0=(e)=>e.replace(/([a-z0-9])([A-Z])/g,"$1-$2").toLowerCase(),Ji=(...e)=>e.filter((t,n,r)=>{return Boolean(t)&&t.trim()!==""&&r.indexOf(t)===n}).join(" ").trim();var Gr=B(D(),1);var J0={xmlns:"http://www.w3.org/2000/svg",width:24,height:24,viewBox:"0 0 24 24",fill:"none",stroke:"currentColor",strokeWidth:2,strokeLinecap:"round",strokeLinejoin:"round"};var G0=Gr.forwardRef(({color:e="currentColor",size:t=24,strokeWidth:n=2,absoluteStrokeWidth:r,className:l="",children:i,iconNode:u,...o},a)=>{return Gr.createElement("svg",{ref:a,...J0,width:t,height:t,stroke:e,strokeWidth:r?Number(n)*24/Number(t):n,className:Ji("lucide",l),...o},[...u.map(([s,h])=>Gr.createElement(s,h)),...Array.isArray(i)?i:[i]])});var M=(e,t)=>{let n=Gi.forwardRef(({className:r,...l},i)=>Gi.createElement(G0,{ref:i,iconNode:t,className:Ji(`lucide-${Z0(e)}`,r),...l}));return n.displayName=`${e}`,n};var qr=M("ArrowDown",[["path",{d:"M12 5v14",key:"s699le"}],["path",{d:"m19 12-7 7-7-7",key:"1idqje"}]]);var Qt=M("ArrowRight",[["path",{d:"M5 12h14",key:"1ays0h"}],["path",{d:"m12 5 7 7-7 7",key:"xquz4c"}]]);var br=M("Camera",[["path",{d:"M14.5 4h-5L7 7H4a2 2 0 0 0-2 2v9a2 2 0 0 0 2 2h16a2 2 0 0 0 2-2V9a2 2 0 0 0-2-2h-3l-2.5-3z",key:"1tc9qg"}],["circle",{cx:"12",cy:"13",r:"3",key:"1vg3eu"}]]);var Jn=M("Clock",[["circle",{cx:"12",cy:"12",r:"10",key:"1mglay"}],["polyline",{points:"12 6 12 12 16 14",key:"68esgv"}]]);var jr=M("Coffee",[["path",{d:"M10 2v2",key:"7u0qdc"}],["path",{d:"M14 2v2",key:"6buw04"}],["path",{d:"M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1",key:"pwadti"}],["path",{d:"M6 2v2",key:"colzsn"}]]);var el=M("Compass",[["path",{d:"m16.24 7.76-1.804 5.411a2 2 0 0 1-1.265 1.265L7.76 16.24l1.804-5.411a2 2 0 0 1 1.265-1.265z",key:"9ktpf1"}],["circle",{cx:"12",cy:"12",r:"10",key:"1mglay"}]]);var tl=M("Ear",[["path",{d:"M6 8.5a6.5 6.5 0 1 1 13 0c0 6-6 6-6 10a3.5 3.5 0 1 1-7 0",key:"1dfaln"}],["path",{d:"M15 8.5a2.5 2.5 0 0 0-5 0v1a2 2 0 1 1 0 4",key:"1qnva7"}]]);var nl=M("Footprints",[["path",{d:"M4 16v-2.38C4 11.5 2.97 10.5 3 8c.03-2.72 1.49-6 4.5-6C9.37 2 10 3.8 10 5.5c0 3.11-2 5.66-2 8.68V16a2 2 0 1 1-4 0Z",key:"1dudjm"}],["path",{d:"M20 20v-2.38c0-2.12 1.03-3.12 1-5.62-.03-2.72-1.49-6-4.5-6C14.63 6 14 7.8 14 9.5c0 3.11 2 5.66 2 8.68V20a2 2 0 1 0 4 0Z",key:"l2t8xc"}],["path",{d:"M16 17h4",key:"1dejxt"}],["path",{d:"M4 13h4",key:"1bwh8b"}]]);var rl=M("MapPin",[["path",{d:"M20 10c0 4.993-5.539 10.193-7.399 11.799a1 1 0 0 1-1.202 0C9.539 20.193 4 14.993 4 10a8 8 0 0 1 16 0",key:"1r0f0z"}],["circle",{cx:"12",cy:"10",r:"3",key:"ilqhr7"}]]);var ll=M("Moon",[["path",{d:"M12 3a6 6 0 0 0 9 9 9 9 0 1 1-9-9Z",key:"a7tn18"}]]);var il=M("Music2",[["circle",{cx:"8",cy:"18",r:"4",key:"1fc0mg"}],["path",{d:"M12 18V2l7 4",key:"g04rme"}]]);var ul=M("Play",[["polygon",{points:"6 3 20 12 6 21 6 3",key:"1oa8hb"}]]);var Wt=M("Sparkles",[["path",{d:"M9.937 15.5A2 2 0 0 0 8.5 14.063l-6.135-1.582a.5.5 0 0 1 0-.962L8.5 9.936A2 2 0 0 0 9.937 8.5l1.582-6.135a.5.5 0 0 1 .963 0L14.063 8.5A2 2 0 0 0 15.5 9.937l6.135 1.581a.5.5 0 0 1 0 .964L15.5 14.063a2 2 0 0 0-1.437 1.437l-1.582 6.135a.5.5 0 0 1-.963 0z",key:"4pj2yx"}],["path",{d:"M20 3v4",key:"1olli1"}],["path",{d:"M22 5h-4",key:"1gvqau"}],["path",{d:"M4 17v2",key:"vumght"}],["path",{d:"M5 18H3",key:"zchphs"}]]);var ol=M("Star",[["path",{d:"M11.525 2.295a.53.53 0 0 1 .95 0l2.31 4.679a2.123 2.123 0 0 0 1.595 1.16l5.166.756a.53.53 0 0 1 .294.904l-3.736 3.638a2.123 2.123 0 0 0-.611 1.878l.882 5.14a.53.53 0 0 1-.771.56l-4.618-2.428a2.122 2.122 0 0 0-1.973 0L6.396 21.01a.53.53 0 0 1-.77-.56l.881-5.139a2.122 2.122 0 0 0-.611-1.879L2.16 9.795a.53.53 0 0 1 .294-.906l5.165-.755a2.122 2.122 0 0 0 1.597-1.16z",key:"r04s7s"}]]);var al=M("Sun",[["circle",{cx:"12",cy:"12",r:"4",key:"4exip2"}],["path",{d:"M12 2v2",key:"tus03m"}],["path",{d:"M12 20v2",key:"1lh1kg"}],["path",{d:"m4.93 4.93 1.41 1.41",key:"149t6j"}],["path",{d:"m17.66 17.66 1.41 1.41",key:"ptbguv"}],["path",{d:"M2 12h2",key:"1t8f8n"}],["path",{d:"M20 12h2",key:"1q8mjw"}],["path",{d:"m6.34 17.66-1.41 1.41",key:"1m8zz5"}],["path",{d:"m19.07 4.93-1.41 1.41",key:"1shlcs"}]]);var sl=M("Sunset",[["path",{d:"M12 10V2",key:"16sf7g"}],["path",{d:"m4.93 10.93 1.41 1.41",key:"2a7f42"}],["path",{d:"M2 18h2",key:"j10viu"}],["path",{d:"M20 18h2",key:"wocana"}],["path",{d:"m19.07 10.93-1.41 1.41",key:"15zs5n"}],["path",{d:"M22 22H2",key:"19qnx5"}],["path",{d:"m16 6-4 4-4-4",key:"6wukr"}],["path",{d:"M16 18a4 4 0 0 0-8 0",key:"1lzouq"}]]);var At=M("UtensilsCrossed",[["path",{d:"m16 2-2.3 2.3a3 3 0 0 0 0 4.2l1.8 1.8a3 3 0 0 0 4.2 0L22 8",key:"n7qcjb"}],["path",{d:"M15 15 3.3 3.3a4.2 4.2 0 0 0 0 6l7.3 7.3c.7.7 2 .7 2.8 0L15 15Zm0 0 7 7",key:"d0u48b"}],["path",{d:"m2.1 21.8 6.4-6.3",key:"yn04lh"}],["path",{d:"m19 5-7 7",key:"194lzd"}]]);var fl=M("X",[["path",{d:"M18 6 6 18",key:"1bl5f8"}],["path",{d:"m6 6 12 12",key:"d8bk6v"}]]);var q0=[{id:"saddar",title:"Saddar Bazaar at Night",tag:"Neon & Chai",cat:"Night",time:"9:30 PM — past midnight",secret:"The old cinema’s ticket man still keeps a ledger of films that never screened.",snippet:"Neon hums over rain-washed stone. A boy carries chai on a tray like it is treasure. The city stops performing and starts telling the truth.",detail:"After the shops shutter, Saddar belongs to the night walkers. Projector light spills from a doorway older than the road. The chai is sweet, the conversation is sweeter, and every corner seems to remember a film, a first date, a goodbye.",num:"01",color:"#00C896",accent:"linear-gradient(135deg,#00C896,#0B3D2E)",vibe:96,vibeLabel:"Electric · Sleepless",icon:"\uD83C\uDF03"},{id:"raja",title:"Raja Bazaar Food Street",tag:"Nihari & Lassi",cat:"Food",time:"7:00 AM — late",secret:"The best nihari is gone before the first school bell. Locals order it the night before.",snippet:"Smoke, ghee, laughter, and a queue that never apologises. This is not breakfast. This is an inheritance.",detail:"In narrow lanes where the air is thick with spice, cooks work recipes guarded like family names. A steel glass of lassi arrives cold enough to fog. Eat standing up. Ask for extra gravy. You are family now.",num:"02",color:"#FF8A00",accent:"linear-gradient(135deg,#FF8A00,#E63946)",vibe:98,vibeLabel:"Fiery · Joyful",icon:"\uD83C\uDF72"},{id:"liaquat",title:"Liaquat Bagh Stories",tag:"History & Whispers",cat:"History",time:"Golden hour",secret:"Old men on one bench have argued the same story for forty years — and neither will finish it.",snippet:"The ground remembers crowds, speeches, silences. Pigeons cut the sky like punctuation.",detail:"Come at golden hour, when the light turns the trees amber and the park becomes a listening room. People do not just walk here. They pause, remember, and pass the city’s unfinished sentences to the next generation.",num:"03",color:"#FFC24B",accent:"linear-gradient(135deg,#FFC24B,#8A5A00)",vibe:82,vibeLabel:"Reflective · Golden",icon:"\uD83D\uDD4A️"},{id:"ayub",title:"Ayub Park at Dawn",tag:"Mist & Morning Walkers",cat:"Nature",time:"5:40 AM — 7:30 AM",secret:"Follow the walkers who carry no phones. They know where the mist settles longest.",snippet:"Fog sits low over the grass. Sneakers on wet paths. A city waking up in whispers.",detail:"Before Rawalpindi checks its phone, Ayub Park is already awake. Old friends race each other gently. A chai cart opens like a flower. The birds get the first word, and for once, they keep it.",num:"04",color:"#48E7B8",accent:"linear-gradient(135deg,#48E7B8,#0E6B52)",vibe:88,vibeLabel:"Calm · Fresh",icon:"\uD83C\uDF3F"},{id:"murree",title:"Murree Road Sounds",tag:"Horns & Vendors",cat:"Night",time:"5:00 PM rush",secret:"Every rickshaw driver has a shortcut he will only use if you ask about his hometown.",snippet:"A horn, a laugh, a vendor’s call — the road composes itself in real time.",detail:"Murree Road does not move. It pulses. Rickshaws thread through light and noise while vendors sell everything from roasted corn to phone covers. Stand still for one minute and the city will introduce itself.",num:"05",color:"#E63946",accent:"linear-gradient(135deg,#E63946,#FF8A00)",vibe:94,vibeLabel:"Loud · Alive",icon:"\uD83D\uDEFA"},{id:"walls",title:"Old Pindi Walls",tag:"Havelis & Hidden Doors",cat:"History",time:"Late morning",secret:"Knock softly on the blue door near the bend. Someone’s grandmother may show you the courtyard.",snippet:"Behind plain walls: carved wood, courtyards, and doors that have outlived empires.",detail:"The old city hides in plain sight. Look up. A carved balcony. A faded fresco. A door painted blue a hundred years ago and repainted by habit. The walls are not ruins. They are archives with keys.",num:"06",color:"#00C896",accent:"linear-gradient(135deg,#0A120E,#00C896)",vibe:90,vibeLabel:"Secret · Timeless",icon:"\uD83D\uDEAA"}],b0=[{id:"saddar",name:"Saddar Night",urdu:"صدر رات",time:"10:12 PM",emoji:"\uD83C\uDF03",theme:"linear-gradient(135deg,#071B14 0%,#123A2B 48%,#1B0F2A 100%)",accent:"#00C896",chip:"#FFC24B",scene:"Neon reflects in a puddle like a second city. Chai steam rises through the cold. A cinema poster flaps like a flag.",sound:"A kettle, a radio, footsteps that know exactly where they are going.",tip:"Order one chai, then another. The second cup is when the stories start.",mood:"Neon · Cinema · Second cup"},{id:"raja",name:"Raja Bazaar",urdu:"راجہ بازار",time:"7:05 AM",emoji:"\uD83C\uDF72",theme:"linear-gradient(135deg,#261203 0%,#6B2600 48%,#2A0A0A 100%)",accent:"#FF8A00",chip:"#FFC24B",scene:"Shutters roll up like eyelids. The first nihari pot is opened like a ceremony. Lassi glasses clink like small bells.",sound:"Spoons on steel, vendors calling names, a bicycle bell threading the lane.",tip:"Arrive hungry and unhurried. Let someone ahead of you order first — then order what they had.",mood:"Spice · Steel · Morning fire"},{id:"rain",name:"Murree Road Rain",urdu:"مری روڈ بارش",time:"6:40 PM · Monsoon",emoji:"\uD83C\uDF27️",theme:"linear-gradient(135deg,#051A22 0%,#0E4A5A 48%,#0A120E 100%)",accent:"#48E7B8",chip:"#FF8A00",scene:"The sky turns the colour of old film. Rickshaws glow. A vendor flips corn over coal while a bus sighs past.",sound:"Rain on tin, horns in two keys, someone laughing into a phone.",tip:"Walk against the rush, not with it. Stop for corn. Ask the vendor how long he has held this corner.",mood:"Rain · Corn · City glow"}],qy=[{name:"Nihari",where:"Raja Bazaar, before sunrise",price:"Rs 450",rating:5,tag:"Must Try",bg:"#3A1F0E",bg2:"#6B2F0F",text:"#FFF6E8",note:"Slow-cooked, glossy, eaten with naan and a squeeze of lemon. Order extra gravy without shame.",emoji:"\uD83C\uDF72"},{name:"Bun Kabab",where:"Saddar cart, near the chowk",price:"Rs 180",rating:4,tag:"Street Icon",bg:"#FF8A00",bg2:"#E63946",text:"#12100A",note:"Crisp bun, smoky kabab, chutney that wakes you up. Best eaten standing, talking, laughing.",emoji:"\uD83E\uDD59"},{name:"Lassi",where:"Raja Bazaar dairy lane",price:"Rs 220",rating:5,tag:"Must Try",bg:"#FFF6E8",bg2:"#FFD98A",text:"#2A1A08",note:"Served in steel, topped with malai. Drink it slowly. This is not a drink — it is a pause.",emoji:"\uD83E\uDD5B"},{name:"Channa Chaat",where:"College Road carts",price:"Rs 150",rating:4,tag:"Tangy Classic",bg:"#00C896",bg2:"#0E6B52",text:"#07120D",note:"Chickpeas, tamarind, onion, heat. The kind of snack that makes a whole afternoon better.",emoji:"\uD83E\uDD57"},{name:"Samosa + Chai",where:"Any corner, every hour",price:"Rs 120",rating:5,tag:"Perfect Pair",bg:"#E63946",bg2:"#7A1020",text:"#FFF6E8",note:"Crackling samosa, sweet chai, steam on a cold evening. Rawalpindi in two bites.",emoji:"☕"}],by=[{name:"Azan at Dawn",when:"5:10 AM",desc:"The city holds its breath. One voice, then many, moving across rooftops like light.",detail:"From a rooftop to a lane in the old city, the call finds every window. Even the birds wait.",icon:"\uD83C\uDF05"},{name:"Rickshaw Horn",when:"All day, especially rush hour",desc:"Not anger — punctuation. A short beep that means I’m here, watch out, I’m alive.",detail:"Learn the rhythm and you can read the road. Two short beeps: moving. One long: definitely moving.",icon:"\uD83D\uDEFA"},{name:"Chai Wala Call",when:"Morning & midnight",desc:"“Chai! Garam chai!” — the city’s unofficial anthem, carried on a tray.",detail:"He knows who takes sugar, who takes less, and who only pretends to be in a hurry.",icon:"☕"},{name:"Rain on Murree Road",when:"Monsoon, without warning",desc:"The road turns silver. Horns soften. Everyone runs, laughs, and gets soaked together.",detail:"Umbrellas fail. Tea tastes better. The whole city smells like wet earth and possibility.",icon:"\uD83C\uDF27️"}],Gn=[{t:"5:00 AM",title:"First Azan, First Light",text:"Rooftops wake. A kettle whistles somewhere you cannot see.",color:"#FFC24B",icon:"\uD83C\uDF05"},{t:"6:30 AM",title:"Ayub Park Wakes",text:"Walkers, mist, and the first chai cart opening like a flower.",color:"#00C896",icon:"\uD83C\uDF3F"},{t:"8:00 AM",title:"Raja Bazaar Breakfast",text:"Nihari, lassi, and a queue that feels like family.",color:"#FF8A00",icon:"\uD83C\uDF72"},{t:"12:30 PM",title:"Old City Doors",text:"Sun on carved wood. A courtyard opens. Someone offers you tea.",color:"#E63946",icon:"\uD83D\uDEAA"},{t:"5:00 PM",title:"Murree Road Rush",text:"Horns, corn, colour. The city at full volume.",color:"#FF8A00",icon:"\uD83D\uDEFA"},{t:"8:30 PM",title:"Saddar Lights Up",text:"Neon, cinema posters, and the second cup of chai.",color:"#00C896",icon:"\uD83C\uDF03"},{t:"11:00 PM",title:"The City Exhales",text:"Quieter lanes. A last rickshaw. Rawalpindi keeps one light on for you.",color:"#FFC24B",icon:"\uD83C\uDF19"}];async function j0(e,t){let n=await fetch("actions",{method:"POST",headers:{"content-type":"application/json"},body:JSON.stringify({action:e,args:t})}),r=await n.json().catch(()=>({}));if(!n.ok||r.error)throw Error(r.error||`Request failed (${n.status})`);return r.data}function Xa(){let[e,t]=W.useState("Night"),[n,r]=W.useState("All"),[l,i]=W.useState(null),[u,o]=W.useState(b0[0]),[a,s]=W.useState(0),[h,m]=W.useState(2),[y,x]=W.useState([]),[N,g]=W.useState(null),[L,c]=W.useState(""),[f,p]=W.useState(""),[w,C]=W.useState(""),[F,E]=W.useState(""),[T,O]=W.useState(!1),[P,ve]=W.useState(null),[It,np]=W.useState(new Date),rp=W.useRef(null);W.useEffect(()=>{let k=setInterval(()=>np(new Date),1000);return()=>clearInterval(k)},[]),W.useEffect(()=>{let k=document.querySelectorAll(".reveal"),U=new IntersectionObserver((lt)=>lt.forEach((oe)=>{if(oe.isIntersecting)oe.target.classList.add("is-visible")}),{threshold:0.12});return k.forEach((lt)=>U.observe(lt)),()=>U.disconnect()},[n,y.length]);let lp=W.useMemo(()=>{try{return new Intl.DateTimeFormat("en-PK",{timeZone:"Asia/Karachi",hour:"2-digit",minute:"2-digit",second:"2-digit",hour12:!0}).format(It)}catch{return It.toLocaleTimeString()}},[It]),ip=W.useMemo(()=>{try{return new Intl.DateTimeFormat("en-PK",{timeZone:"Asia/Karachi",weekday:"long",day:"numeric",month:"long"}).format(It)}catch{return""}},[It]),Za=W.useMemo(()=>n==="All"?q0:q0.filter((k)=>k.cat===n),[n]),Ja=e==="Morning"?{kicker:"5:40 AM — Ayub Park · Mist on the grass",line:"The city wakes in whispers."}:e==="Afternoon"?{kicker:"2:15 PM — Murree Road · Heat, horns, colour",line:"The city turns up the volume."}:{kicker:"10:12 PM — Saddar · Neon, chai, cinema light",line:"The city stops performing. It starts telling the truth."};async function Ga(){try{let k=await j0("records.list",{collection:"unseen_stories"});x(k.records||[]),g(null)}catch(k){g("Shared stories could not be loaded right now — your story form is still here.")}}W.useEffect(()=>{Ga()},[]);async function up(k){return j0("records.create",{collection:"unseen_stories",value:k,idempotency_key:`story-${Date.now()}-${Math.random().toString(36).slice(2,8)}`})}async function op(){if(!w.trim()||!F.trim()||!f.trim()){ve("Add a title, a place, and your story before sharing.");return}O(!0),ve(null);let k=0,U=null;try{await up({name:L.trim()||"Anonymous wanderer",place:f.trim(),title:w.trim(),story:F.trim()}),k=1}catch(oe){U=oe instanceof Error?oe.message:String(oe)}let lt=null;try{await Ga()}catch(oe){lt=oe instanceof Error?oe.message:String(oe)}if(O(!1),U)ve(`Not saved yet — ${U}. ${k} saved, 1 remaining.`);else if(lt)ve("Saved, but the wall could not refresh. Try reloading the page.");else ve("Saved — thank you. Your unseen story is now on the wall below."),c(""),p(""),C(""),E("")}let cn=(k)=>document.getElementById(k)?.scrollIntoView({behavior:"smooth"});return v("div",{ref:rp,"data-safe-area-inset-root":!0,style:{boxSizing:"border-box",paddingTop:"var(--safe-area-inset-top)",minHeight:"100vh",background:"var(--bg)",color:"var(--ink)"},className:"overflow-x-clip",children:[d("header",{"data-safe-area-sticky-top":!0,style:{top:"var(--safe-area-inset-top)"},className:"sticky z-40 border-b border-[rgba(0,200,150,.22)] bg-[rgba(10,18,14,.92)] backdrop-blur-xl",children:v("div",{className:"mx-auto flex max-w-7xl flex-wrap items-center justify-between gap-3 px-4 py-3 sm:px-6",children:[v("button",{onClick:()=>window.scrollTo({top:0,behavior:"smooth"}),className:"text-left leading-none",children:[d("span",{className:"gradient-text shimmer font-serif-x text-xl font-black tracking-tight sm:text-2xl",children:"PINDI•UNSEEN"}),d("span",{className:"mt-1 block text-[10px] font-black uppercase tracking-[.28em] text-[#C8D6CD]",children:"راولپنڈی · An editorial field guide"})]}),v("nav",{className:"flex flex-wrap items-center gap-1 text-xs font-bold sm:text-sm",children:[[["explore","Explore"],["story","Story Mode"],["food","Eat"],["sounds","Sounds"],["day","One Day"]].map(([k,U])=>d("button",{onClick:()=>cn(k),className:"rounded-full px-2 py-2 hover:bg-white/10 hover:text-[#FFC24B] sm:px-3",children:U},k)),d("button",{onClick:()=>cn("share"),className:"gradient-btn rounded-full px-3 py-2 text-black sm:px-4",children:"Share a Story"})]})]})}),v("section",{className:"grain relative overflow-hidden bg-[#0A120E]",children:[d("div",{className:"blob absolute -left-24 top-10 h-96 w-96 rounded-full bg-[#00C896]"}),d("div",{className:"blob absolute right-[-60px] top-[38%] h-[420px] w-[420px] rounded-full bg-[#FF8A00]",style:{animationDelay:"-4s"}}),d("div",{className:"blob absolute bottom-[-80px] left-[38%] h-80 w-80 rounded-full bg-[#E63946]",style:{animationDelay:"-7s"}}),v("div",{className:"relative mx-auto max-w-7xl px-4 pb-10 pt-10 sm:px-6 sm:pt-14",children:[v("div",{className:"flex flex-wrap items-center justify-between gap-3",children:[d("p",{className:"eyebrow text-[#48E7B8]",children:"Issue No. 07 · Rawalpindi, Pakistan · A city in six secrets"}),v("div",{className:"glass flex items-center gap-3 rounded-full px-4 py-2",children:[v("span",{className:"relative flex h-2.5 w-2.5",children:[d("span",{className:"absolute inline-flex h-full w-full animate-ping rounded-full bg-[#00C896] opacity-75"}),d("span",{className:"relative inline-flex h-2.5 w-2.5 rounded-full bg-[#00C896]"})]}),v("span",{className:"text-xs font-extrabold tracking-wide",children:["LIVE IN PINDI · ",ip]}),d("span",{className:"font-serif-x text-base font-black text-[#FFC24B] tabular-nums",children:lp})]})]}),v("div",{className:"mt-8 text-center sm:mt-10",children:[d("p",{className:"font-urdu text-2xl text-[#FFC24B] sm:text-3xl",children:"راولپنڈی ان سین"}),v("h1",{className:"font-serif-x mt-2 text-[clamp(3.4rem,14vw,10.5rem)] font-black leading-[.82] tracking-[-.04em]",children:[d("span",{className:"text-outline block",children:"RAWALPINDI"}),d("span",{className:"gradient-text shimmer block italic",children:"UNSEEN"})]}),v("p",{className:"mx-auto mt-6 max-w-2xl text-lg leading-relaxed text-[#EAF6EF] sm:text-xl",children:["The city you think you know. ",d("span",{className:"font-bold text-white",children:"The stories you never heard."})]}),d("p",{className:"mx-auto mt-3 max-w-xl text-sm leading-relaxed text-[#C8D6CD]",children:"Not a tourist guide. A love letter written in chai steam, neon, old doors, and the sound of a city that never quite sleeps."})]}),d("div",{className:"mt-7 flex flex-wrap justify-center gap-2",children:["Saddar","Raja Bazaar","Murree Road","Old Pindi","Ayub Park"].map((k,U)=>v("span",{className:"float-y rounded-full border border-white/20 bg-black/30 px-4 py-2 text-xs font-extrabold backdrop-blur sm:text-sm",style:{animationDelay:`${U*0.6}s`},children:[k," ",U===0?"\uD83C\uDF03":U===1?"\uD83C\uDF72":U===2?"\uD83D\uDEFA":U===3?"\uD83D\uDEAA":"\uD83C\uDF3F"]},k))}),v("div",{className:"mt-7 flex flex-wrap items-center justify-center gap-3",children:[v("button",{onClick:()=>cn("explore"),className:"gradient-btn rounded-full px-6 py-3.5 font-extrabold",children:["Explore The Unseen ",d(Qt,{className:"ml-1 inline h-4 w-4"})]}),v("button",{onClick:()=>cn("sounds"),className:"rounded-full border border-white/30 bg-white/5 px-6 py-3.5 font-bold backdrop-blur",children:[d(tl,{className:"mr-1 inline h-4 w-4 text-[#FFC24B]"})," Hear The Streets"]})]}),v("div",{className:"mx-auto mt-8 flex max-w-3xl flex-col items-center gap-3 sm:flex-row sm:justify-between",children:[v("div",{className:"flex flex-wrap justify-center gap-2",children:[d("span",{className:"self-center text-[11px] font-black uppercase tracking-widest text-[#C8D6CD]",children:"Time-based vibe"}),["Morning","Afternoon","Night"].map((k)=>v("button",{onClick:()=>t(k),"aria-pressed":e===k,className:`rounded-full border px-4 py-2 text-sm font-bold ${e===k?"bg-[#FFF6E8] text-black border-[#FFF6E8]":"border-white/25 text-white"}`,children:[k==="Morning"?d(al,{className:"mr-1 inline h-4 w-4"}):k==="Afternoon"?d(sl,{className:"mr-1 inline h-4 w-4"}):d(ll,{className:"mr-1 inline h-4 w-4"}),k]},k))]}),v("p",{className:"text-center text-sm text-[#FFC24B]",children:[d("b",{children:Ja.line}),d("span",{className:"block text-xs text-[#C8D6CD]",children:Ja.kicker})]})]}),d("div",{className:"mt-10 overflow-hidden border-y border-white/10 py-3",children:d("div",{className:"marquee flex w-max gap-8 whitespace-nowrap text-xs font-black uppercase tracking-[.22em] text-[#FFC24B]",children:[0,1].map((k)=>v("span",{className:"flex gap-8",children:[d("span",{children:"Chai steam ✦ Neon ✦ Old doors ✦ Nihari ✦ Rain on Murree Road ✦ Second cup ✦"}),d("span",{children:"Chai steam ✦ Neon ✦ Old doors ✦ Nihari ✦ Rain on Murree Road ✦ Second cup ✦"})]},k))})})]})]}),v("section",{id:"explore",className:"mx-auto max-w-7xl px-4 py-14 sm:px-6 sm:py-20",children:[v("div",{className:"reveal flex flex-wrap items-end justify-between gap-5",children:[v("div",{children:[d("p",{className:"eyebrow text-[#00C896]",children:"02 · Explore"}),v("h2",{className:"font-serif-x mt-3 text-4xl font-black leading-none sm:text-6xl",children:["Six places",d("br",{}),d("span",{className:"gradient-text italic",children:"tourists miss."})]})]}),d("p",{className:"max-w-sm text-sm leading-relaxed text-[#C8D6CD]",children:"Tap a card to open its full story. Filter by what you’re in the mood for — food, history, night, or green quiet."})]}),v("div",{className:"mt-7 flex flex-wrap gap-2",children:[["All","Food","History","Night","Nature"].map((k)=>d("button",{onClick:()=>r(k),"aria-pressed":n===k,className:`rounded-full border px-5 py-2.5 text-sm font-extrabold transition ${n===k?"border-[#00C896] bg-[#00C896] text-black shadow-[0_0_28px_rgba(0,200,150,.55)]":"border-white/20 bg-white/5 hover:border-[#FFC24B]"}`,children:k},k)),v("span",{className:"ml-1 self-center text-xs font-bold text-[#C8D6CD]",children:[Za.length," of 6 places shown"]})]}),d("div",{className:"mt-7 grid gap-5 sm:grid-cols-2 lg:grid-cols-3",children:Za.map((k)=>v("article",{className:"card-lift glass flex flex-col overflow-hidden rounded-[24px]",children:[v("div",{className:"relative overflow-hidden p-5 pb-6",style:{background:k.accent},children:[d("span",{className:"absolute -right-2 -top-4 text-[92px] opacity-25",children:k.icon}),v("div",{className:"relative flex items-start justify-between",children:[d("span",{className:"font-serif-x text-5xl font-black text-white/30",children:k.num}),d("span",{className:"rounded-full bg-black/30 px-3 py-1.5 text-xs font-extrabold text-white backdrop-blur",children:k.tag})]}),d("span",{className:"relative mt-8 inline-block text-5xl",children:k.icon}),d("h3",{className:"relative mt-3 font-serif-x text-[26px] font-black leading-tight text-white",children:k.title}),v("p",{className:"relative mt-1 text-xs font-bold uppercase tracking-widest text-white/80",children:[k.cat," · ",k.vibeLabel]})]}),v("div",{className:"flex flex-1 flex-col p-5",children:[d("p",{className:"text-sm leading-relaxed text-[#DDEBE2]",children:k.snippet}),v("div",{className:"mt-4 space-y-3 text-sm",children:[v("p",{className:"flex gap-2",children:[d(Jn,{className:"mt-0.5 h-4 w-4 shrink-0 text-[#FFC24B]"}),v("span",{children:[d("b",{className:"text-white",children:"Best time:"})," ",k.time]})]}),v("div",{className:"rounded-2xl bg-black/25 p-3",children:[v("p",{className:"text-xs font-black uppercase tracking-widest",style:{color:k.color},children:["Vibe meter · ",k.vibe,"%"]}),d("div",{className:"mt-2 h-2 overflow-hidden rounded-full bg-white/15",children:d("div",{className:"h-full rounded-full",style:{width:`${k.vibe}%`,background:"linear-gradient(90deg,#00C896,#FFC24B,#FF8A00)"}})})]}),v("p",{className:"rounded-xl bg-white/5 p-3 leading-relaxed",children:[d("span",{className:"font-bold text-[#FFC24B]",children:"What locals know — "}),k.secret]})]}),v("button",{onClick:()=>i(k),className:"mt-5 w-fit rounded-full bg-[#FFF6E8] px-4 py-2.5 text-sm font-extrabold text-black hover:bg-[#FFC24B]",children:["Read Story ",d(Qt,{className:"ml-1 inline h-4 w-4"})]})]})]},k.id))})]}),d("section",{id:"story",className:"border-y border-white/10 bg-[#FFF6E8] text-[#0A120E]",children:v("div",{className:"mx-auto max-w-7xl px-4 py-14 sm:px-6 sm:py-20",children:[d("p",{className:"eyebrow text-[#E63946]",children:"03 · Story Mode · Choose a Street"}),v("div",{className:"mt-3 flex flex-wrap items-end justify-between gap-4",children:[v("h2",{className:"font-serif-x text-4xl font-black leading-none sm:text-6xl",children:["A mini film,",d("br",{}),d("span",{className:"gradient-text italic",children:"in one street."})]}),d("p",{className:"max-w-md text-sm leading-relaxed text-[#5E6F65]",children:"Pick a street. We’ll set the scene, turn up the sound, and leave you with a tip only a local would give."})]}),d("div",{className:"mt-8 grid gap-3 sm:grid-cols-3",children:b0.map((k)=>v("button",{onClick:()=>o(k),"aria-pressed":u.id===k.id,className:`rounded-[24px] border-2 p-5 text-left transition ${u.id===k.id?"border-[#0A120E] bg-[#0A120E] text-[#FFF6E8] shadow-xl":"border-[#0A120E]/15 bg-white hover:border-[#FF8A00]"}`,children:[d("span",{className:"text-4xl",children:k.emoji}),d("span",{className:"mt-3 block font-serif-x text-2xl font-black",children:k.name}),d("span",{className:"font-urdu block text-sm text-[#FF8A00]",children:k.urdu}),v("span",{className:"mt-2 block text-xs font-bold uppercase tracking-widest opacity-70",children:[k.time," · ",k.mood]}),u.id===k.id&&d("span",{className:"mt-3 inline-block rounded-full bg-[#00C896] px-3 py-1 text-xs font-black text-black",children:"Now Playing ●"})]},k.id))}),v("div",{className:"fade-up mt-5 grid overflow-hidden rounded-[24px] text-white shadow-2xl lg:grid-cols-[1.2fr_.8fr]",style:{background:u.theme},children:[v("div",{className:"p-6 sm:p-10",children:[v("p",{className:"eyebrow",style:{color:u.chip},children:["Scene · ",u.time," · ",u.name]}),d("p",{className:"font-serif-x mt-5 text-2xl leading-snug sm:text-4xl",children:u.scene}),v("p",{className:"mt-6 border-l-4 pl-4 text-sm leading-relaxed text-white/80",style:{borderColor:u.accent},children:[d("b",{className:"text-white",children:"SOUND ON — "}),u.sound]}),d("p",{className:"mt-6 font-urdu text-xl",style:{color:u.chip},children:u.urdu})]}),v("div",{className:"p-6 sm:p-8",style:{background:u.accent,color:"#07120D"},children:[v("p",{className:"flex items-center gap-2 text-xs font-black uppercase tracking-[.2em]",children:[d(Wt,{className:"h-4 w-4"})," Local tip"]}),d("p",{className:"font-serif-x mt-4 text-2xl font-black leading-snug",children:u.tip}),d("p",{className:"mt-5 text-sm font-bold",children:"Walk it once. Then walk it again — slower."})]})]},u.id)]})}),v("section",{id:"food",className:"mx-auto max-w-7xl px-4 py-14 sm:px-6 sm:py-20",children:[v("div",{className:"flex flex-wrap items-end justify-between gap-4",children:[v("div",{children:[d("p",{className:"eyebrow text-[#FF8A00]",children:"04 · Eat Like A Local"}),v("h2",{className:"font-serif-x mt-3 text-4xl font-black sm:text-6xl",children:["Come hungry.",d("br",{}),d("span",{className:"gradient-text italic",children:"Leave family."})]})]}),v("p",{className:"max-w-xs text-sm text-[#C8D6CD]",children:["Swipe through the five dishes that define a day in Pindi. ",d(Qt,{className:"inline h-4 w-4"})]})]}),d("div",{className:"mt-8 flex snap-x snap-mandatory gap-4 overflow-x-auto pb-4 no-scrollbar",children:qy.map((k,U)=>v("article",{className:"card-lift w-[84vw] max-w-[330px] shrink-0 snap-start overflow-hidden rounded-[24px] border border-white/15 p-5",style:{background:`linear-gradient(160deg,${k.bg},${k.bg2})`,color:k.text},children:[v("div",{className:"flex items-start justify-between",children:[d("span",{className:"text-5xl",children:k.emoji}),d("span",{className:"rounded-full bg-black/25 px-3 py-1 text-xs font-black text-white",children:k.tag})]}),v("span",{className:"font-serif-x mt-4 block text-5xl font-black opacity-40",children:["0",U+1]}),v("div",{className:"mt-2 flex items-center gap-2 text-xs font-bold",children:[d(At,{className:"h-4 w-4"}),k.where]}),d("h3",{className:"font-serif-x mt-2 text-3xl font-black",children:k.name}),d("p",{className:"mt-3 text-sm leading-relaxed opacity-90",children:k.note}),v("div",{className:"mt-5 flex flex-wrap items-center gap-2",children:[d("span",{className:"rounded-full bg-black/25 px-3 py-1.5 text-sm font-black text-white",children:k.price}),v("span",{className:"flex items-center gap-1 rounded-full bg-[#FFF6E8] px-3 py-1.5 text-xs font-black text-black",children:[Array.from({length:5}).map((lt,oe)=>d(ol,{className:`h-3.5 w-3.5 ${oe<k.rating?"fill-[#FF8A00] text-[#FF8A00]":"text-black/25"}`},oe)),k.rating,".0"]})]})]},k.name))}),d("p",{className:"text-xs font-bold text-[#C8D6CD]",children:"Prices are typical street prices — the best meals in Pindi rarely cost more than a rickshaw ride."})]}),d("section",{id:"sounds",className:"border-y border-white/10 bg-[#13251D]",children:v("div",{className:"mx-auto max-w-7xl px-4 py-14 sm:px-6 sm:py-20",children:[d("p",{className:"eyebrow text-[#00C896]",children:"05 · Sounds of Pindi"}),v("h2",{className:"font-serif-x mt-3 max-w-3xl text-4xl font-black leading-none sm:text-6xl",children:["Close your eyes. ",d("span",{className:"gradient-text italic",children:"You’re already there."})]}),d("p",{className:"mt-4 max-w-xl text-sm text-[#C8D6CD]",children:"Tap a sound card to “listen” — read its rhythm, its hour, and the small human story inside it."}),d("div",{className:"mt-8 grid gap-4 sm:grid-cols-2 lg:grid-cols-4",children:by.map((k,U)=>v("button",{onClick:()=>s(U),"aria-pressed":a===U,className:`card-lift rounded-[24px] border p-5 text-left ${a===U?"border-[#00C896] bg-[#0A120E] shadow-[0_0_34px_rgba(0,200,150,.28)]":"border-white/12 bg-white/5"}`,children:[v("div",{className:"flex items-center justify-between",children:[d("span",{className:"text-4xl",children:k.icon}),d("span",{className:`flex h-11 w-11 items-center justify-center rounded-full ${a===U?"gradient-btn":"bg-white/10"}`,children:d(ul,{className:"ml-0.5 h-5 w-5 fill-current"})})]}),d("div",{className:"mt-4 flex h-12 items-end gap-1",children:[10,18,28,16,24,12,30,18,14,22].map((lt,oe)=>d("span",{className:"sound-bar w-1.5 rounded-full bg-[#00C896]",style:{height:lt,animationDelay:`${oe*0.09}s`,animationPlayState:a===U?"running":"paused",opacity:a===U?1:0.35}},oe))}),v("p",{className:"mt-4 flex items-center gap-2 text-xs font-bold text-[#C8D6CD]",children:[d(il,{className:"h-3.5 w-3.5 text-[#FFC24B]"}),k.when]}),d("h3",{className:"font-serif-x mt-2 text-2xl font-black leading-tight",children:k.name}),d("p",{className:"mt-2 text-sm leading-relaxed text-[#C8D6CD]",children:k.desc}),a===U&&d("p",{className:"mt-3 rounded-xl bg-[#00C896]/15 p-3 text-left text-sm leading-relaxed",children:k.detail})]},k.name))})]})}),v("section",{id:"day",className:"mx-auto max-w-7xl px-4 py-14 sm:px-6 sm:py-20",children:[d("p",{className:"eyebrow text-[#FFC24B]",children:"06 · One Day Timeline"}),v("h2",{className:"font-serif-x mt-3 text-4xl font-black sm:text-6xl",children:["One day in Pindi,",d("br",{}),d("span",{className:"gradient-text italic",children:"5 AM to 11 PM."})]}),d("p",{className:"mt-3 max-w-lg text-sm text-[#C8D6CD]",children:"Tap a moment to read it. The day moves whether you are ready or not — best to keep up."}),v("div",{className:"mt-9 grid gap-6 lg:grid-cols-[.4fr_1fr]",children:[v("div",{className:"relative pl-2",children:[d("span",{className:"timeline-line absolute bottom-2 left-[18px] top-2 w-[4px] rounded-full"}),d("div",{className:"relative space-y-2",children:Gn.map((k,U)=>v("button",{onClick:()=>m(U),className:`relative flex w-full items-center gap-3 rounded-2xl px-4 py-3 text-left ${h===U?"bg-[#FFF6E8] text-black":"hover:bg-white/5"}`,children:[d("span",{className:"h-3.5 w-3.5 shrink-0 rounded-full border-2 border-[#0A120E]",style:{background:k.color}}),v("span",{className:"min-w-0",children:[d("span",{className:"text-xs font-black tracking-widest",children:k.t}),v("span",{className:"block truncate font-serif-x text-lg font-bold leading-tight",children:[k.icon," ",k.title]})]})]},k.t))})]}),v("div",{className:"fade-up self-start rounded-[24px] border border-white/12 bg-[#13251D] p-6 sm:p-10",children:[v("p",{className:"eyebrow",style:{color:Gn[h].color},children:[Gn[h].t," · Moment ",h+1," of 7"]}),d("h3",{className:"font-serif-x mt-4 text-3xl font-black leading-tight sm:text-5xl",children:Gn[h].title}),d("p",{className:"mt-4 max-w-xl text-lg leading-relaxed text-[#C8D6CD]",children:Gn[h].text}),v("div",{className:"mt-8 flex flex-wrap gap-2",children:[d("button",{disabled:h===0,onClick:()=>m((k)=>k-1),className:"rounded-full border border-white/20 px-4 py-2 text-sm font-bold disabled:opacity-40",children:"← Earlier"}),d("button",{disabled:h===Gn.length-1,onClick:()=>m((k)=>k+1),className:"gradient-btn rounded-full px-4 py-2 text-sm font-extrabold disabled:opacity-40",children:"Next moment →"})]})]},h)]})]}),d("section",{id:"share",className:"border-t border-white/10 bg-[#FFF6E8] text-[#0A120E]",children:v("div",{className:"mx-auto max-w-7xl px-4 py-14 sm:px-6 sm:py-20",children:[v("div",{className:"grid gap-10 lg:grid-cols-2",children:[v("div",{children:[d("p",{className:"eyebrow text-[#E63946]",children:"07 · Your Turn"}),v("h2",{className:"font-serif-x mt-3 text-4xl font-black leading-none sm:text-6xl",children:["Share your",d("br",{}),d("span",{className:"gradient-text italic",children:"unseen story."})]}),d("p",{className:"mt-5 max-w-md leading-relaxed text-[#5E6F65]",children:"A door, a dish, a voice, a night you never forgot. Tell it in your own words. It will live here for other wanderers to find."}),d("p",{className:"font-urdu mt-4 text-xl text-[#FF8A00]",children:"اپنی کہانی سناؤ"}),d("p",{className:"mt-2 text-xs text-[#5E6F65]",children:"Stories are saved to the shared wall. Please keep them kind and true."})]}),v("form",{onSubmit:(k)=>{k.preventDefault(),op()},className:"space-y-3 rounded-[24px] border border-black/10 bg-white p-5 shadow-xl sm:p-6",children:[v("div",{className:"grid gap-3 sm:grid-cols-2",children:[v("label",{className:"block text-sm font-bold",children:["Your name",d("input",{value:L,onChange:(k)=>c(k.target.value),placeholder:"Anonymous wanderer",className:"mt-1 w-full rounded-xl border border-black/15 bg-[#FFF6E8] px-3 py-2.5 outline-none focus:border-[#00C896]"})]}),v("label",{className:"block text-sm font-bold",children:["Place *",d("input",{value:f,onChange:(k)=>p(k.target.value),placeholder:"e.g. Saddar at night",className:"mt-1 w-full rounded-xl border border-black/15 bg-[#FFF6E8] px-3 py-2.5 outline-none focus:border-[#00C896]"})]})]}),v("label",{className:"block text-sm font-bold",children:["Story title *",d("input",{value:w,onChange:(k)=>C(k.target.value),placeholder:"The chai that changed my evening",className:"mt-1 w-full rounded-xl border border-black/15 bg-[#FFF6E8] px-3 py-2.5 outline-none focus:border-[#00C896]"})]}),v("label",{className:"block text-sm font-bold",children:["Your story *",d("textarea",{value:F,onChange:(k)=>E(k.target.value),rows:5,placeholder:"Tell us what you saw, heard, tasted — and what it meant.",className:"mt-1 w-full resize-y rounded-xl border border-black/15 bg-[#FFF6E8] px-3 py-2.5 outline-none focus:border-[#00C896]"})]}),d("button",{disabled:T,className:"gradient-btn w-full rounded-full py-3 font-extrabold disabled:opacity-50",children:T?"Saving your story…":"Share my unseen story"}),P&&d("p",{role:"status",className:"rounded-xl bg-[#FFF6E8] p-3 text-sm leading-relaxed",children:P})]})]}),v("div",{className:"mt-12",children:[d("h3",{className:"font-serif-x text-2xl font-black",children:"The shared wall"}),N&&d("p",{className:"mt-2 text-sm text-[#5E6F65]",children:N}),y.length===0&&!N&&d("p",{className:"mt-3 max-w-lg text-sm leading-relaxed text-[#5E6F65]",children:"No shared stories yet. Yours could be the first — a door, a dish, a night you never forgot."}),d("div",{className:"mt-5 grid gap-4 md:grid-cols-2 lg:grid-cols-3",children:y.map((k)=>v("article",{className:"rounded-[24px] border border-black/10 bg-white p-5 shadow-sm",children:[v("p",{className:"flex items-center gap-2 text-xs font-bold text-[#E63946]",children:[d(rl,{className:"h-3.5 w-3.5"}),k.value.place]}),d("h4",{className:"font-serif-x mt-2 text-xl font-bold leading-snug",children:k.value.title}),d("p",{className:"mt-2 whitespace-pre-wrap text-sm leading-relaxed text-[#5E6F65]",children:k.value.story}),v("p",{className:"mt-4 text-xs font-bold",children:["— ",k.value.name]})]},k.id))})]})]})}),d("footer",{className:"bg-[#0A120E]",children:v("div",{className:"mx-auto max-w-7xl px-4 py-14 sm:px-6",children:[d("div",{className:"grain relative overflow-hidden rounded-[28px] p-8 text-center sm:p-12",style:{background:"linear-gradient(115deg,#00C896 0%,#FFC24B 48%,#FF8A00 70%,#E63946 100%)"},children:v("div",{className:"relative",children:[d("p",{className:"font-urdu text-2xl text-[#0A120E]",children:"راولپنڈی تم سے پیار ہے"}),v("h2",{className:"font-serif-x mx-auto mt-3 max-w-3xl text-4xl font-black leading-[.95] text-[#0A120E] sm:text-6xl",children:["Share Your",d("br",{}),"Unseen Story"]}),d("p",{className:"mx-auto mt-4 max-w-xl font-semibold text-[#1A1200]",children:"The next great Pindi story is the one only you can tell. Add it to the wall — and come back tomorrow for a new one."}),v("button",{onClick:()=>cn("share"),className:"mt-6 rounded-full bg-[#0A120E] px-6 py-3.5 font-extrabold text-white hover:bg-black",children:["Share your unseen story ",d(Qt,{className:"ml-1 inline h-4 w-4"})]})]})}),v("div",{className:"flex flex-wrap items-center justify-between gap-4 pt-8",children:[v("div",{children:[v("p",{className:"font-serif-x text-2xl font-black",children:["Made with love in ",d("span",{className:"gradient-text",children:"Rawalpindi"})]}),d("p",{className:"mt-1 text-sm text-[#C8D6CD]",children:"Written on foot. Remembered in chai steam."})]}),v("div",{className:"flex gap-2 text-xs font-bold",children:[v("span",{className:"rounded-full bg-white/10 px-3 py-2",children:[d(nl,{className:"mr-1 inline h-3.5 w-3.5"})," Written on foot"]}),v("span",{className:"rounded-full bg-white/10 px-3 py-2",children:[d(jr,{className:"mr-1 inline h-3.5 w-3.5"})," Chai first"]}),v("span",{className:"rounded-full bg-white/10 px-3 py-2",children:[d(br,{className:"mr-1 inline h-3.5 w-3.5"})," Look up"]})]})]}),d("p",{className:"border-t border-white/10 py-4 text-center text-xs text-[#C8D6CD]",children:"RAWALPINDI UNSEEN · Issue 07 · Come back tomorrow — the city will have a new story."})]})}),d(I0,{open:!!l,onOpenChange:(k)=>{if(!k)i(null)},children:v(R0,{children:[d(V0,{className:"fixed inset-0 z-50 bg-black/85 backdrop-blur-sm"}),d(Y0,{className:"fixed left-1/2 top-1/2 z-50 max-h-[88vh] w-[92vw] max-w-2xl -translate-x-1/2 -translate-y-1/2 overflow-y-auto rounded-[24px] border border-[#00C896]/30 bg-[#0D1B15] p-6 sm:p-8",children:l&&v(un,{children:[v("div",{className:"flex items-start justify-between gap-4",children:[v("div",{children:[v("p",{className:"eyebrow text-[#FFC24B]",children:[l.num," · ",l.tag]}),d(K0,{className:"font-serif-x mt-2 text-3xl font-black leading-tight sm:text-4xl",children:l.title})]}),d(X0,{asChild:!0,children:d("button",{"aria-label":"Close story",className:"rounded-full border border-white/20 p-2",children:d(fl,{className:"h-5 w-5"})})})]}),v("div",{className:"mt-4 flex items-center gap-3 rounded-2xl p-4 text-5xl",style:{background:l.accent},children:[d("span",{children:l.icon}),v("span",{className:"text-sm font-bold text-white",children:[l.vibeLabel," · Vibe ",l.vibe,"%"]})]}),v("p",{className:"mt-3 flex gap-2 text-sm",children:[d(Jn,{className:"h-4 w-4 shrink-0 text-[#FFC24B]"}),"Best time: ",l.time]}),d("p",{className:"mt-5 text-base leading-relaxed",children:l.detail}),v("p",{className:"mt-5 rounded-2xl bg-black/25 p-4 text-sm leading-relaxed",children:[d("b",{className:"text-[#FFC24B]",children:"What locals know — "}),l.secret]}),v("p",{className:"mt-5 flex items-center gap-2 text-sm text-[#C8D6CD]",children:[d(el,{className:"h-4 w-4"})," Go slow. Look up. Ask for the second cup."]})]})})]})}),v("button",{onClick:()=>cn("explore"),className:"gradient-btn fixed bottom-4 right-4 z-30 flex items-center gap-1 rounded-full px-4 py-2.5 text-sm font-extrabold shadow-xl",children:[d(qr,{className:"h-4 w-4"})," Explore"]})]})}tp.createRoot(document.getElementById("root")).render(d(ep.default.StrictMode,{children:d(Xa,{})}));
</script>
  <script>(function(){function m(a){var h=a.getAttribute("href");if(!h)return;try{var u=new URL(h,document.baseURI);if(((u.protocol==="http:"||u.protocol==="https:")&&u.host!==location.host)||u.protocol==="mailto:"||u.protocol==="tel:"||u.protocol==="sms:"){a.target="_blank";a.rel="noopener noreferrer";}}catch(e){}}function s(){document.querySelectorAll("a[href]").forEach(m);}if(document.readyState!=="loading"){s();}else{document.addEventListener("DOMContentLoaded",s);}document.addEventListener("click",function(e){var a=e.target&&e.target.closest&&e.target.closest("a[href]");if(a){m(a);}},true);})();</script>
  <script type="application/json" data-clippy-inline-media-manifest="1">{"version":1,"slots":[]}</script>
<script>(function(){var loc=location.href.replace(/#.*$/,"");var ATTR_NAMES=["data-product-id","data-productid","data-product_id","product-id","productid","product_id","data-source-entity-id","source-entity-id","source_entity_id","data-product","data-metadata","data-meta"];var DATASET_KEYS=["productId","productid","product_id","sourceEntityId","sourceentityid","source_entity_id","product","metadata","meta"];function readProductId(value){if(typeof value!=="string"||value.length===0)return null;if(/^[0-9]{6,}$/.test(value))return value;var match=value.match(/(?:product(?:_|-)?id|source(?:_|-)?entity(?:_|-)?id)["'=:\s]+([0-9]{6,})/i);return match?match[1]:null}function extractProductId(start){for(var node=start;node&&node!==document.body;node=node.parentElement){for(var i=0;i<ATTR_NAMES.length;i++){var attrValue=node.getAttribute&&node.getAttribute(ATTR_NAMES[i]);var attrProductId=readProductId(attrValue);if(attrProductId)return attrProductId}var dataset=node.dataset||null;if(dataset){for(var j=0;j<DATASET_KEYS.length;j++){var dataValue=dataset[DATASET_KEYS[j]];var dataProductId=readProductId(dataValue);if(dataProductId)return dataProductId}}}return null}function isInlineMediaSlotElement(node){return !!(node&&node.getAttribute&&node.getAttribute("data-clippy-inline-media-slot")!==null)}function findInlineMediaSlot(start){for(var node=start;node&&node!==document.body;node=node.parentElement){if(isInlineMediaSlotElement(node))return node}return null}function readInlineMediaUrl(node){if(!node)return null;return node.getAttribute&&((node.getAttribute("data-clippy-inline-media-url")||node.getAttribute("data-url")||node.getAttribute("data_url")))||node.href||null}function stripHash(url){return String(url).replace(/#.*$/,"")}function urlsMatch(a,b){if(!a||!b)return false;try{return stripHash(new URL(a,loc).href)===stripHash(new URL(b,loc).href)}catch(_){return stripHash(a)===stripHash(b)}}function isFirstPartyReelUrl(value){try{var url=new URL(value,loc);if(url.protocol!=="https:")return false;var host=url.hostname.toLowerCase();var supported=host==="instagram.com"||host.endsWith(".instagram.com")||host==="facebook.com"||host.endsWith(".facebook.com");return supported&&/\/reels?\//i.test(url.pathname)}catch(_){return false}}function isInlineMediaUrlClick(node,href){var slot=findInlineMediaSlot(node);if(!slot)return false;var slotUrl=readInlineMediaUrl(slot);if(slotUrl)return urlsMatch(href,slotUrl);return isFirstPartyReelUrl(href)}function findDataHref(start){for(var node=start;node&&node!==document.body;node=node.parentElement){if(node.getAttribute){var href=node.getAttribute("data-href")||node.getAttribute("data-url");if(href)return{href:href,node:node}}}return null}var nativeOpen=window.open;window.open=function(url){if(parent!==window&&typeof url==="string"&&/^https?:\/\//.test(url)){parent.postMessage({type:"ecto:usercontent-link-click",href:url},"*");return null}return nativeOpen?nativeOpen.apply(window,arguments):null};document.addEventListener("click",function(e){var target=e.target instanceof Element?e.target:null;if(!target)return;if(parent===window)return;var a=target.closest?target.closest("a[href]"):null;if(a&&a.href&&/^https?:\/\//.test(a.href)&&a.href.replace(/#.*$/,"")!==loc){if(isInlineMediaUrlClick(a,a.href))return;var productId=extractProductId(target)||extractProductId(a);if(productId){e.preventDefault();parent.postMessage({type:"ecto-artifact-link-click",productId:productId},"*");return}e.preventDefault();parent.postMessage({type:"ecto:usercontent-link-click",href:a.href},"*");return}var dataHref=findDataHref(target);if(dataHref&&/^https?:\/\//.test(dataHref.href)&&dataHref.href.replace(/#.*$/,"")!==loc){if(isInlineMediaUrlClick(dataHref.node,dataHref.href))return;e.preventDefault();parent.postMessage({type:"ecto:usercontent-link-click",href:dataHref.href},"*")}},true)})();</script><script>(function(){var FOCUS_TYPE="ecto:artifact-focus-request";var CLOSE_TYPE="ecto:artifact-close-request";function focusArtifactDocument(){var body=document.body;if(!body)return;try{window.focus();}catch(e){}if(!body.hasAttribute("tabindex"))body.setAttribute("tabindex","-1");try{body.focus({preventScroll:true});}catch(e){try{body.focus();}catch(e2){}}}window.addEventListener("message",function(event){if(event.source!==window.parent)return;var data=event.data;if(!data||typeof data!=="object"||data.type!==FOCUS_TYPE)return;if(document.readyState==="loading"){document.addEventListener("DOMContentLoaded",focusArtifactDocument,{once:true});return;}focusArtifactDocument();});window.addEventListener("keydown",function(event){if(event.key!=="Escape")return;window.setTimeout(function(){if(event.defaultPrevented)return;window.parent.postMessage({type:CLOSE_TYPE},"*");},0);});})();</script></body>
</html># rawalpindi-unseen
