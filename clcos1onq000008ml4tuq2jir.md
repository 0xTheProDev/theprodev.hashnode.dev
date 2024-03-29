---
title: "I had to integrate few React components in a Django View using jQuery: A Journey"
seoTitle: "Integrating React Components into an Existing UI"
seoDescription: "How did I load my React Components into existing jQuery based UI rendered by Django on Server at Interview Kickstart"
datePublished: Mon Jan 09 2023 12:25:30 GMT+0000 (Coordinated Universal Time)
cuid: clcos1onq000008ml4tuq2jir
slug: rendering-react-component-in-django-jquery-ui
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1673245430375/a73ea775-7afe-4a92-a1c9-fb01f1ccc488.jpeg
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1673267073681/6282d170-154e-406d-b1bf-3d8a8c874ffa.png
tags: technology, django, reactjs, software-engineering, frontend-development

---

A small journey into my Tech World filled with bizarre experiences. One such instance that comes to mind is when I had to make some enhancements to an existing UI that was written earlier. The older Tech Stack was Django-based SSR (using View Templating) with jQuery in the browser. However, the enhancement planned was based on React, as the library, we will be composing on top of was built in React and we were already in the middle of a Tech Stack migration for the front-end. On the other hand, the whole UI system of the existing one was quite heavy and would require significant *person-hour* to completely migrate to React. So we made a decision that we will build all enhancements in React while keeping the older UI intact and just extensible.

*TLDR;* Experienced React developers can skip directly to the [Let's Get Started](https://theprodev.hashnode.dev/rendering-react-component-in-django-jquery-ui?showSharer=true#heading-lets-get-started) chapter.

## How React Works in Browser

Before we get started with the nitty-gritty implementation details of the topic. Let us take a moment and understand how the React libraries work under the hood to build the UI we declared in our codebase.

In our codebase, a React component might look like this:

```javascript
import React from "react";

export const HelloWorldComponent = () => <h1>Hello World</h1>;
```

And we might be rendering this into UI as follows, assuming we have an empty *div* with the id *"app"*:

```javascript
import React from "react";
import ReactDOMClient from "react-dom/client";

import { HelloWorldComponent } from "./HelloWorld";

const appContainer = document.getElementById("app");
const appRoot = ReactDOMClient.createRoot(appContainer);

appRoot.render(<HelloWorldComponent />);
```

Now underneath the hood, React will create the required element tree and append it to the aforesaid *div*. The purpose of *ReactDOM* here is to reflect all our declarative UI mapping into actual DOM elements.

## Module Bundlers

It would not be an overstatement to say front-end state of art applications as we see them today might not have existed if not for high-end module bundlers like Webpack, Rollup, Turbopack, Vite etc. The core purpose of these tools is to convert a set of source files from the developer workspace into a couple of executable files for the staging/production environment, e.g., a browser.

![Module Bundlers](https://res.cloudinary.com/practicaldev/image/fetch/s--0dDhpCqD--/c_imagga_scale,f_auto,fl_progressive,h_420,q_auto,w_1000/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nwmf2clu7cew1lgdsltf.png align="left")

![Webpack Documentation - Image: https://webpack.js.org](https://miro.medium.com/max/1400/1*dbLSMmG_9YNaVRyn30YKBQ.png align="left")

One of the key file these module bundlers generate is the Manifest file. This file is responsible to provide metadata to the runtime so that the runtime can load the entry point and dependencies as they are needed. Webpack generates a file called *asset-manifest.json* which holds information on the mapping between source files and runtime assets and a list of entry points.

## Let's Get Started

We start our journey by creating a factory function that will be exposed to the outside world. For the sake of simplicity, let us assume we want to make this function available in the global scope. This pattern can be improved using custom loaders, but we can skip it for the time being.

```javascript
import { Component, ComponentProps } from "components";
import * as ReactDOMClient from "react-dom/client";

/**
 * Factory Method for React Component.
 * @param container DOM Container to render React Component.
 * @param props Incoming Props for React Component.
 * @returns Object with a handler to unmount React Component from the DOM.
 */
export const ComponentFactory = (container: Element, props: ComponentProps): VoidFunction => {
  const root = ReactDOMClient.createRoot(container, { identifierPrefix: "react" });
  root.render(<Component {...props} />);

  return {
    destroy: () => root.unmount(),
  };
};
```

Once went through the build pipeline, this yields `main_[contenthash].js` file which would be the entry point of our application, considering everything inside the *Component* is valid.

## The Middleware House

Now we have two problems at hand, one is to tell Django that we have to load runtime assets generated by our module bundler, Webpack in this case; and invoke the factory function from the current system with proper parameters to create the UI. Let's take the first one for a spin.

```python
# Standard library imports.
import json
import re
from typing import Any, Callable, Dict, List, Union

# Valid Keys in Asset Manifest JSON Type Alias.
AssetManifestKeys = Literal["entrypoints", "files"]


class AssetManifest:
    """
    Asset Manifest Class
    """

    files: Dict[str, str]
    """
    Mapping between Original Filenames and their Artifact Name (appended by content-hash for caching purposes).
    """
    entrypoints: List[str]
    """
    List of files that needs to loaded by the Page to start Client application.
    """

    def __init__(self, decoded_json: Dict[AssetManifestKeys, Any]) -> None:
        """
        Constructor method for Asset Manifest Class.
        """
        self.entrypoints = decoded_json["entrypoints"]
        self.files = decoded_json["files"]


# Default Callbale to parse JSON String.
WHITESPACE = re.compile(r"[ \t\n\r]*", re.VERBOSE | re.MULTILINE | re.DOTALL)


class AssetManifestDecoder(json.JSONDecoder):
    """
    JSON Decoder Class to Parse JSON String into Asset Manifest Class instance.
    """

    def decode(self, s: str, _w: Callable[..., Any] = WHITESPACE.match) -> AssetManifest:
        """
        Return the Asset Manifest class instance of ``s`` (a ``str`` instance
        containing a JSON document).
        """
        decoded_json: Dict[AssetManifestKeys, Any] = super().decode(s, _w)
        asset_manifest = AssetManifest(decoded_json)
        return asset_manifest


# Valid File Name/Descriptor Type Alias.
StrOrBytes = Union[str, bytes]
StrOrBytesPath = Union[StrOrBytes, PathLike[str], PathLike[bytes]] OpenFile = Union[StrOrBytesPath, int]  # noqa: Y026


def load_asset_manifest(filepath: OpenFile) -> AssetManifest:
    """
    Helper Function to load Module Bundler (like Webpack, Rollup etc.) generated Asset Manifest.
    """
    with open(filepath, "r") as fp:
        return json.load(fp, cls=AssetManifestDecoder)


def parse_asset_manifest(strOrBytes: StrOrBytes) -> AssetManifest:
    """
    Helper Function to parse Module Bundler (like Webpack, Rollup etc.) generated Asset Manifest.
    """
    return json.loads(strOrBytes, cls=AssetManifestDecoder)
```

Here we have developed our custom *JSON* Decoder that serializes a *JSON* string into an `AssetManifest` object. The function `parse_asset_manifest` can be tested in isolation for validation, however, we would be using the `load_asset_manifest` function more often. Provided we can configure the destination directory for Webpack to emit the manifest file, the actual file path can be given through environment variables.

Now let's create a mixin that composes this middleware and simplifies the integration process.

```python
class ReactViewMixin(TemplateView):
    """
    Mixin that helps loading React assets into Server-side HTML.
    This should add `react_assets` property to context data which then to be added into `<script>` tag.
    """

    def get_context_data(self, **kwargs: Any) -> Dict[str, Any]:
        """
        Overwrite Context Data to hold static assets required to load React Component in the UI.
        """
        context_data = super().get_context_data(**kwargs)
        asset_manifest = load_asset_manifest(settings.REACT_ASSET_MANIFEST_PATH)
        context_data["react_assets"] = asset_manifest.entrypoints
        return context_data
```

Now we can simply inherit any Django View from here and it will work out of the box.

## Bootstrapping in Browser

First of all, we need to add `<script>` tags for React resources into our view templates.

```php-template
{% for asset in react_assets %}
  <script type="application/javascript" defer="" src="/{{ asset }}"></script>
{% endfor %}
```

I believe here on, figuring out the rest is quite easy. We will have an empty *div* and will pick that element once the document is ready and invoke the factory function with proper parameters.

```xml
<div id="id_react_app"/>
<script>
$(document).ready(function() {
  var $container = $("#id_react_app");
  var container  = $container.get(0);

  var ref = ComponentFactory(container, {
    onMount: () => console.log("React Component Mounted!"),
  });

  $(window).on("beforeunload", ref.destroy());
</script>
```

## Conclusion

So it turns out, despite sounding unorthodox, it is possible to integrate a React UI system into an existing one and could be a potential choice that saves some $$ for the business.

![Abhijeet Dancin Abhijeet Cid GIF - Abhijeet Dancin Abhijeet Cid Smile -  Discover & Share GIFs](https://media.tenor.com/-CSZYL3DkmYAAAAd/abhijeet-dancin-abhijeet-cid.gif align="left")