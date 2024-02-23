# htmf (HyperTextMicroFramework v0.7) provides shared state for multipage web applications

`htmf` is a micro-framework for allowing multipage web applications to preserve elements (such as `audio` and `video`) across navigations using the powerful technique of *self framing*.

## Self Framing
A self-framing web page is a page that detects whether it is hosted in a frame. If it is not in a frame, it alters the structure of its own page to create an `iframe` and load itself again within the `iframe`.

The page understands which parts of itself are common to the web application and should be preserved across navigations (such as a `video` element or navigation controls) and which parts of itself are page-specific content, to be replaced when the user navigates to another page.

Marking up the page is as simple as applying `id="htmf"` to the element that contains all the page-specific content. Everything outside of that element is common content.

When the page is loaded in the browser, the common parts of the page live in the top-level, outer document and the unique parts of the page live in the `iframe` in the inner document.

In the outer page, the unique parts are removed: the micro-framework replaces the `#htmf` element (which contains the unique parts of the page) with the `iframe` that loads the inner document.

In the inner page, the shared parts are removed: the micro-framework removes all elements outside of the `#htmf` element.

Your code and CSS can detect if you are in the outer or inner document by looking for the `htmf-document` attribute on the document's `body` (`htmf-document="outer"` for the outer document and `htmf-document="inner"` for the inner document) or by looking for the presence or absence of common elements or by using a standard technique to tell if the page is framed: `if (window.frameElement !== null)` or by comparing `document === outerDocument`.

## Self Preservation
Normally dividing a web page into an outer parent and an inner child within an `iframe` could change its behavior, so the `htmf` micro-framework takes steps to preserve the behavior of the page.

First, `htmf` creates a `base` element in the `head` of the outer document that retargets links from the current document to the inner document: `<base target="htmf">` where `htmf` is the `name` given to the automatically created `iframe`. That means that if you click a link in the outer document, the outer page won't be replaced by the new document; instead, the inner iframe will load the new content.

Next, the micro-framework hooks various events on the `iframe` which tell it when a new document has been loaded. When it detects that a new document has been loaded in to the `iframe`, it copies some of the information from the inner document into the outer document, so that the user feels as though a normal page navigation has occurred. These items are copied from the inner document to the outer document:

1. The URL
2. The title
3. The description
4. The canonical URL link
5. Alternate representation links (RSS, Atom, etc)
6. rel=me links

In this way, the document preserves the appearance of a single page instead of an outer and inner page.

## Quick Start
Mark up the page-specific content by applying `id="htmf"` to the single element that contains it. All other content in your page is shared content.

Add this `script` to the bottom of the `body` element to turn your normal web page into a self-framing page.

```HTML
    <script id="htmf-script">
        const body = document.body;
        body.querySelector("script#htmf-script").remove();

        if (window.frameElement !== null) {
            globalThis.innerDocument = document;
            globalThis.outerDocument = window.frameElement.ownerDocument;

            body.setAttribute("htmf-document", "inner");
            [...body.querySelectorAll("body[htmf-document='inner']:has(#htmf) :not(:has(#htmf)):not(:is(#htmf)):not(:is(#htmf *))")].map(e => e.remove());
        } else {
            Object.defineProperty(globalThis, "innerDocument", {
                get() {
                    return document.querySelector("#htmf").contentDocument;
                },
                enumerable: true,
                configurable: true,
            });

            globalThis.outerDocument = document;

            body.setAttribute("htmf-document", "outer");

            const self = body.querySelector("#htmf");
            if (self != undefined) {
                self.outerHTML = `<iframe name="htmf" id="htmf"></iframe>`;
            } else {
                body.innerHTML = `<iframe name="htmf" id="htmf"></iframe>`;
            }

            const head = document.head;
            const base = document.createElement("base");
            base.target = "htmf";
            head.append(base);

            const iframe = body.querySelector("iframe#htmf");
            iframe.src = document.location;

            function enableIFrameEvents(iframe) {

                function onLocationChanged(oldURL, newURL) {
                    const inner = iframe.contentDocument;
                    document.title = inner.title;

                    const selectors = ["link[rel='canonical']", "link[rel='alternate']", "link[rel='me']", "meta[name='description']", "meta[name='robots']"];
                    for (const selector of selectors) {
                        [...document.head.querySelectorAll(selector)].map(e => e.remove());
                        const items = [...inner.head.querySelectorAll(selector)];
                        for (const item of items) {
                            document.head.append(item.cloneNode());
                        }
                    }

                    const visibleURL = new URL(newURL);
                    const hiddenParameters = (document.body.getAttribute("htmf-hidden-parameters") ?? "").split(" ");
                    for (const hiddenParameter of hiddenParameters) {
                        visibleURL.searchParams.delete(hiddenParameter);
                    }

                    history.replaceState({}, document.title, visibleURL);

                    iframe.dispatchEvent(new CustomEvent("location-changed", { bubbles: true, cancelable: true, detail: { oldURL, newURL } }));
                }

                function onHashChange(e) {
                    if (iframe.contentWindow == undefined) {
                        return;
                    }

                    const { oldURL, newURL } = e;
                    console.log(oldURL.toString(), newURL.toString());
                    onLocationChanged(oldURL, newURL);
                }

                function onUnload() {
                    const w = iframe.contentWindow;
                    if (w == undefined) {
                        return;
                    }

                    const oldURL = new URL(w.location.href);
                    // Timeout of 0 because the URL changes immediately _after_ the `unload` event fires.
                    setTimeout(function () {
                        if (w) {
                            const newURL = new URL(w.location.href);
                            console.log(oldURL.toString(), newURL.toString());
                            onLocationChanged(oldURL, newURL);
                        }
                    }, 0);
                }

                function attach() {
                    const w = iframe.contentWindow;
                    if (w == undefined) {
                        return;
                    }

                    w.removeEventListener("unload", onUnload);
                    w.removeEventListener("hashchange", onHashChange);

                    w.addEventListener("unload", onUnload);
                    w.addEventListener("hashchange", onHashChange);
                }

                iframe.addEventListener("load", attach);
                attach();
            }

            enableIFrameEvents(iframe);
        }
    </script>
```

## Background
A self-framing page rehosts itself in an `iframe`. Links in that page automatically target the iframe, so all navigation now happens in the `iframe`.

The first self-framing page in a session essentially loads twice: once as an outer page and once as an inner page. Make sure that the page has appropriate caching so that the second load into the `iframe` hits the already cached content and doesn't need to go back to the server. Other self-framing pages only load once: as an inner page in the `iframe`.

A small amount of script makes the outer page look the same as the inner page by updating the title, the URL, and some metadata to match those of the inner page. This happens once during load so if your inner pages rely on changing the title during use, you might need to make sure that you're updating the title where users can see it.

The iframe replaces the `#htmf` element or the `body` of the page if no such element exists.

The `htmf` micro-framework makes two globals available to you: `innerDocument` and `outerDocument`. These give you access to the inner & outer documents regardless of whether they are being used from the inner document or outer document.

## Uses
### The `video` element

If you are developing a website where video is important to users, you know that you want to preserve the `video` element even as the user navigates across pages. This is important to allow smooth, continuous play, and it's also important because the `video` element isn't scriptable until the first user interaction. You can't provide a good experience unless you share a `video` element across the different pages of your web site, but you want to retain the benefits of a multipage architecture. `htmf` is the solution.

### User interface state

If you have a multipage website where navigation elements are common across pages, you may start to see cases where the user's position in a tree or menu is being lost as they navigate from page to page, or you may be doing work to pass UI state between your pages to avoid these problems, but the complexity of your code is increasing. Rather than losing state that's important to the user or doing extra work to pass state between pages, you can instead use `htmf` to put the common UI in the outer page. The user never loses their place in the UI because the elements are not destroyed and created on each navigation, and you get to keep all the benefits of your multipage architecture.

### URL simplification

When you use `htmf`, your primary page is loaded in an `iframe`. By default, the user sees the URL of that page at the top level in the browser because `htmf` uses `history.replaceState` to surface it to the user. Once you're using `htmf`, you can decide to customise exactly what URL the user sees by adding URL search parameters that you'd like to hide to the `htmf-hidden-parameters` attribute on the document's `body`. Maybe you like to use `cachebuster` or `version` parameters in the URL, but you don't need the user to see or share URLs with those parameters. These parameters can be easily hidden when you use the `htmf-hidden-parameters` attribute, which is simply a space-separated list of URL search parameter names that you want to be hidden.

You can use URL simplification without dividing your page in two parts. If you don't have an element with `id="htmf"`, your entire page will end up in the `iframe` and you can enjoy URL simplification without having part of your page at the top level and part of it within the iframe. 


