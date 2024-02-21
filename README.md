# htmf - Shared state for multipage web applications

`htmf` is a micro-framework for allowing multipage web applications to preserve elements (such as `audio` and `video`) across navigations using the powerful technique of *self framing*.

## Self Framing
A self-framing web page is a page that detects whether it is hosted in a frame. If it is not in a frame, it alters the structure of its own page to create an `iframe` and load itself again within the `iframe`.

The page understands which parts of itself are common to the web application and should be preserved across navigations (such as a `video` element or navigation controls) and which parts of itself are page-specific content, to be replaced when the user navigates to another page.

Marking up the page is as simple as applying `id="htmf"` to the element that contains all the page-specific content. Everything outside of that element is common content.

When the page is loaded in the browser, the common parts of the page live in the top-level, outer document and the unique parts of the page live in the `iframe` in the inner document.

In the outer page, the common parts are removed by default: the micro-framework replaces the `#htmf` element which contains the common parts with the `iframe` that loads the inner document.

In the inner page, the shared parts are hidden by default: the micro-framework uses CSS to set elements outside of the `#htmf` element to `display: none`.

Your code and CSS can detect if you are in the outer or inner document by looking for the `framed` attribute on the document's `body`: `framed="false"` for the outer document and `framed="true"` for the inner document.

## Self Preservation
Normally dividing a web page into an outer parent and an inner child within an `iframe` could change its behavior, so the `htmf` micro-framework takes steps to preserve the behavior of the page.

First, `htmf` creates a `base` element in the `head` of the outer document that retargets links from the current document to the inner document: `<base target="htmf">`. That means that if you click a link in the outer document, the outer page won't be replaced by the new document; instead, the inner iframe will load the new content.

Next, the micro-framework hooks various events on the `iframe` which tell it when a new document has been loaded. When it detects that a new document has been loaded in to the `iframe`, it copies some of the information from the inner document into the outer document, so that the user feels as though a normal page navigation has occurred. These items are copied from the inner document to the outer document:

1. The URL
2. The title
3. The canonical URL link
4. Alternate representation links (RSS, Atom, etc)

In this way, the document preserves the appearance of a single page instead of an outer and inner page.

## Quick Start
Mark up the page-specific content by applying `id="htmf"` to the single element that contains it. All other content in your page is shared content.

Add this CSS to your documents' CSS:

```CSS
body[framed='true']:has(#htmf) :not(:is(#htmf)):not(:is(#htmf *)) {
    display: none;
}
```

This ensures that all the shared content like navigation controls, media elements, etc will be correctly hidden when the document is loaded in the `iframe`.

Add this `script` to the bottom of the `body` element to turn your normal web page into a self-framing page.

```JS
<script id="self-framer">
    const body = document.body;
    if (window.frameElement !== null) {
        body.setAttribute("framed", "true");
        body.querySelector("script#self-framer").remove();
    } else {
        body.setAttribute("framed", "false");

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

                const selectors = ["link[rel='canonical']", "link[rel='alternate']"];
                for (const selector of selectors) {
                    [...document.head.querySelectorAll(selector)].map(i => i.remove());
                    const items = [...inner.head.querySelectorAll(selector)];
                    for (const item of items) {
                        document.head.append(item.cloneNode());
                    }
                }

                history.replaceState({}, document.title, newURL);

                iframe.dispatchEvent(new CustomEvent("location-changed", { bubbles: true, cancelable: true, detail: { oldURL, newURL } }));
            }

            function dispatchEventNow(e) {
                if (iframe.contentWindow == undefined) {
                    return;
                }

                const { oldURL, newURL } = e;
                console.log(oldURL.toString(), newURL.toString());
                onLocationChanged(oldURL, newURL);
            }

            function dispatchEventLater() {
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

                w.removeEventListener("unload", dispatchEventLater);
                w.removeEventListener("hashchange", dispatchEventNow);

                w.addEventListener("unload", dispatchEventLater);
                w.addEventListener("hashchange", dispatchEventNow);
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

A small amount of script makes the outer page look the same as the inner page by updating the title and the URL that the user sees to match the title and URL of the inner page. This happens once during load so if your inner pages rely on changing the title during use, you might need to make sure that you're updating the title where users can see it.

The frame replaces the `#htmf` element or the `body` of the page if no such element exists.

Typically your page will contain elements outside of the `#htmf` element for your navigation controls, your media elements, or other shared state. To hide them when the page is framed, you can use CSS like the following which hides all elements that aren't descendants of the `#htmf` element when the page is framed.


