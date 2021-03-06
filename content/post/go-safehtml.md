---
title: "Go Safe HTML"
date: 2020-06-30T13:03:31+02:00
categories: ["Software engineering", "Security", "Go"]
tags: ["go", "golang", "security"]
authors: ["Rob"]
draft: true
---

> Disclaimer: this is not an official Google post or communication, it is just me commenting on something that is now available.

The Google infosec team just released [the Go "safehtml" package](https://github.com/google/safehtml). If you are interested in making your application resilient to server-side XSS you might want to adopt it instead of "html/template". Migration to safehtml should be quite straightforward as it is just a hardened fork of the original html/template standard package. If you don't have major flaws in your app it should not be too complicated to convert it to use the safe version.

This is what Google uses internally to protect products from XSS.

If you just want to use it without reading through the explanation you can jump to [the checklist](#the-checklist).

# Issues with "html/template"
"html/template" has no concept of "tainting" and does not keep track on how dangerous types like [`template.HTML`](https://golang.org/pkg/html/template/#HTML) are constructed. There is just a line in the documentation stating:

> Use of this type presents a security risk: the encapsulated content should come from a trusted source, as it will be included verbatim in the template output.

This not only is lacking explanation on the "why" and the "how" to use the type, but it is also very commonly used, making it a very dangerous pitfall together with all the other types that have that line in the doc (seven in total).

Moreover "html/template" has some standing issues that don't have a good way to be properly fixed without breaking backward compatibility. The tradeoff between breakage potential and security benefit there is not clear, so if you want to **opt in** to be more secure, you should probably abandon "html/template". 

Note that I am maintaining "html/template" (I'm empijei [here](https://dev.golang.org/owners)), so I am telling you this with a bit of background of being bitten by that package. If I could magically migrate all users to the safe version I would.

# The structure
Safehtml is composed of several packages. Depending on your build system or your company toolchain there are some constraint that you should enforce. Your company's security team or security-aware folks should be setting this up for everyone.

### safehtml
This is the root package and it is just providing types that are [safe by construction](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/42934.pdf). Here is how it works in a nutshell:

* There are only 3 ways to construct the [HTML type](https://godoc.org/github.com/google/safehtml#HTML):
  * [by escaping an untrusted string](https://godoc.org/github.com/google/safehtml#HTMLEscaped), making it safe to use
  * [by rendering a safe template](https://godoc.org/github.com/google/safehtml/template#Template.ExecuteToHTML), which uses contextual auto-escaping, so it is safe (you can read more about this [in a previous post of mine](https://blogtitle.github.io/robn-go-security-pearls-cross-site-scripting-xss/))
  * [by concatenating already trusted HTML](https://godoc.org/github.com/google/safehtml#HTMLConcat), which is mostly safe and practically requires bad intention to get wrong.

This guarantees that every single instance of the HTML type is known to be safe, because there is no unsafe way to construct it. The [Script type](https://godoc.org/github.com/google/safehtml#Script) behaves similarly but instead of having templates it can only be built from constants or data. To express the concept of "compile time constant" it has [constructors that take unexported string types](https://godoc.org/github.com/google/safehtml#ScriptFromConstant), so the only way to call them is with a [string literal](https://golang.org/ref/spec#String_literals) (I find this to be a very neat trick).

All other types in this package follow a similar pattern.

### safehtml/legacyconversions
This package should only be used to transition to the safe API. It blesses any arbitrary string to be a safe type so that transition to safehtml can be very quick and all new code will be safe. <ins>**Once the migration has happened the use of this package should be prevented**</ins>. As the name states: this is just for legacy code, no new code should be using it, and **all usages of this packages should gradually be refactored to use safe constructors instead**.

### safehtml/raw
Importing this package should be prevented. Anything outside of the "safehtml/" directory tree should not have visibility of this package.

### safehtml/safehtmlutil
Yes, I know, not a good name. Consider that this package, like the previous one, should also not be imported outside of safehtml and it was just created to reduce code duplication and avoid cyclic dependencies. I agree this could have been named or structured differently, but since you're never going to interact with this package it should not bother you too much.

### safehtml/template
This is your real "html/template" replacement and it is the package everyone should be using. If "legacyconversions" and "uncheckedconversions" are not used and **all your HTML responses are generated by this package** you have the guarantee there is not going to be any server-side XSS in your products.

We are working on tools that ensure this last condition is true, but it will take a bit of time. [Stay tuned](https://blogtitle.github.io/index.xml) for updates.

### safehtml/testconversions
This package should only be used in test targets and only when necessary. You should set up some [linters](https://godoc.org/golang.org/x/tools/go/analysis) that make sure of it.

### safehtml/uncheckedconversions
This is the most nuanced matter. Sometimes the safehtml API is too inconvenient or even impossible to use. Sometimes you *have to* use unsafe patterns because you want to do something that cannot be proven to be safe (e.g. take some HTML that you trust from a database and serve it to the client).

For these **very rare** situations you can use this package. Importing it should be restricted to a hand-picked set of dependants and every new import should require some security-aware folks to review it.

Usages of this package are your single point of failure, so make sure you only use it when necessary. (This sentence assumes you will eventually get rid of legacy conversions)

One example of a correct use of this package would be for the output of a sanitizer. If you need to have some HTML that your users provide to be embedded in a response (e.g. because you render markdown or you have a webmail) you will sanitize that HTML. Once it is sanitized (if your sanitizer is correctly implemented) it should be okay to use an unchecked conversion to promote it to the HTML type.

# How to do the refactor
### `Printf` and nested templates
One example of code that you might have is
```go
var theLink template.HTML = fmt.Sprintf("<a href=%q>the link</a>", myLink)
myTemplate.Execute(httpResponseWriter, theLink)
```
To refactor this you have multiple options: you either build the string with another template (note that here "template" is "safehtml/template")
```go
myLinkTpl := template.Must(template.New("myUrl").Parse("<a href={{.}}>the link</a>"))
theLink, err := myLinkTpl.ExecuteToHtml(myLink)
// handle err
myTemplate.Execute(httpResponseWriter, theLink)
```
or, for more complex cases, you can use nested templates:
```go
const outer = `<h1> This is a title <h2> {{ template "inner" .URL }}`
const inner = `<a href="{{.}}">the link</a>`
t := template.Must(template.New("outer").Parse(outer))
t = template.Must(t.New("inner").Parse(inner))
t.ExecuteTemplate(os.Stdout, "outer", map[string]string{"URL": myLink})
```
### Constants
If you have an HTML `const` in your code, you can just use it as a template and execute it to html. This will check that all tags are balanced and other things and return an instance of the HTML type.

So this
```go
var myHtml template.HTML := `<h1> This is a title </h1>`
```
becomes this
```go
myHtml := template.MustParseAndExecuteToHTML(`<h1> This is a title </h1>`)
```

# The checklist
1. Block access to some packages:
 * Prevent packages outside of the "safehtml" directory from importing "raw", "uncheckedconversions" and "safehtmlutil"
 * Only allow test builds to import the "testconversions" package.
2. Migrate away from "html/template" and replace it with "safehtml/template"
 * For every breakage or every issue, use a "legacyconversions" call. Some manual refactoring might be needed, but migration should be fairly straightforward
 * RUN ALL YOUR INTEGRATION AND E2E TESTS. This is *important*, so important I used SHIFT and not CAPS to type it
 * Block down the list of legacy conversions: from this moment on new imports of the "legacyconversions" package are forbidden
 * Ban use of "html/template", so that all new code is safe
3. Refactor legacy conversions to use safe patterns
 * Wherever possible construct HTML in a safe way and remove the legacy conversions
 * Where it is not possible use unchecked conversions. Every new import of the "uncheckedconversions" package should be reviewed.

# Conclusions
If you want to make sure you don't have server-side XSS in your Go code this probably is the best way to do so. If you have any questions or need more refactoring examples please let me know, you can contact me [on twitter](https://twitter.com/empijei) (direct messages are open) or [via email](mailto:empijei@gmail.com).

