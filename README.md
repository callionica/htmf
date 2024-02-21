# htmf - Shared state for multipage web applications

`htmf` is a micro-framework for allowing multipage web applications to preserve elements (such as `audio` and `video`) across navigations using the powerful technique of *self-framing*.

## Self-framing
A self-framing web page is a page that detects whether it is hosted in a frame. If it is not in a frame, it alters the structure of its own page to create an `iframe` and load itself again within the `iframe`.

The page understands which parts of itself are common to the web application and should be preserved (such as a `video` element or navigation controls) and which parts of itself are page-specific content, to be replaced when the user navigates to another page.

The common parts of the page live in the top-level, outer document and the unique parts of the page live in the `iframe` in the inner document.

## The Code
Add the `script` to the bottom of the `body` element to turn a normal web page into a self-framing page.

A self-framing page rehosts itself in an `iframe`. Links in that page automatically target the iframe, so all navigation now happens in the `iframe`.

The first self-framing page in a session essentially loads twice: once as an outer page and once as an inner page. Make sure that the page has appropriate caching so that the second load into the `iframe` hits the already cached content and doesn't need to go back to the server. Other self-framing pages only load once: as an inner page in the `iframe`.

A small amount of script makes the outer page look the same as the inner page by updating the title and the URL that the user sees to match the title and URL of the inner page. This happens once during load so if your inner pages rely on changing the title during use, you might need to make sure that you're updating the title where users can see it.

The frame replaces the `#htmf` element or the `body` of the page if no such element exists.

If you have parts of the page that should be shown when it's the outer page, but not the inner page (or vice versa), you can use CSS like

```CSS
body[framed='true'] .htmf-outer {
    display: none;
}
body[framed='false'] .htmf-inner {
    display: none;
}
```

Here's the script:

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
            const iframe = body.querySelector("iframe#htmf");
            iframe.src = document.location;

            function enableIFrameEvents(iframe) {

                function onLocationChanged(oldURL, newURL) {
                    const inner = iframe.contentDocument;
                    document.title = inner.title;
                    // TODO - copy other items from the inner document to the outer document (eg canonical links)
                    // TODO - adjust the URL if necessary
                    history.replaceState({}, document.title, newURL);

                    element.dispatchEvent(new CustomEvent("location-changed", { bubbles: true, cancelable: true, detail: { oldURL, newURL } }));
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
