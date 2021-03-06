- Feature Name: incremental-compilation
- Start Date: 2020-03-15
- RFC PR:
  [christopherbiscardi/toast/pull/1](https://github.com/ChristopherBiscardi/toast/pull/1)
- RFC Issue:
  [christopherbiscardi/toast#3](https://github.com/christopherbiscardi/toast/issues/3)

# Summary

Make Toast an incremental compiler of HTML files and assets.

# Motivation

Why are we doing this? What use cases does it support? What is the expected
outcome?

One of the largest problems with JAMStack sites at scale is the time it takes to
build and deploy changes. This is both a production and a development time
problem, yielding both

An [Incremental Compiler](https://en.wikipedia.org/wiki/Incremental_compiler) is
a compiler that takes only the changes to a set of source files and updates the
corresponding output files. Toast should function as an incremental compiler as
much as possible.

> It can be said that an incremental compiler reduces the granularity of a
> language's traditional compiling units while maintaining the language's
> semantics, such that the compiler can append and replace smaller parts. --
> [wikipedia](https://en.wikipedia.org/wiki/Incremental_compiler)

# Guide-level explanation

An incremental compiler has a number of effects for users developing locally as
well as build performance in CI. There are roughly two scenarios: new data
changes and new code changes. New data coming in triggers builds for pages that
it shows up on. While new code does the same.

## Users, when building a site locally

When building a site locally, we can take advantage of incremental compilations
when in --watch mode and also when _not_ in --watch mode.

When _not_ in --watch mode, builds are faster because they do less work.

When in --watch mode, we can track changes and new files to do the minimum
amount of work necessary.

## Caches in CI

When working with caches in CI, builds should be faster because much of the work
is already done. One could imagine making webhooks for data updates that trigger
builds skip extra data fetching and code commit updates skip fetching entirely.
Code updates for a specific page would also not require the entire site to be
built.

# Reference-level explanation

Because of the way builds work, in that we have a single .js component that is
used to render a given .html file, we can track the changes through the system
at each step, invalidating the rest of the steps. We _don't_ have a congestion
point from webpack bundling all .js files.

- The HTML file depends on it's `page-data.json`, `page-component.js`, as well
  as a `page-wrapper.js`.

  - `page-data.json` files depend on the data pipeline that created them
  - `page-component.js` files depend on
    - corresponding code files in `src/pages` (.js, .mdx, etc)
    - or the data pipelines they are associated with (Sector mdxast -> .js file)
    - and the file's own imports
  - `page-wrapper.js` depends on itself as well as anything it imports

- Pipelines define the behavior of data as it flows through the system. The
  pipelines RFC would have to accommodate incremental compilation in it's own
  RFC. Data flowing into the system can result in:

  - node.js and browser components used to render page html
  - JSON files used to populate props

* The processing step for any files in `src/` are embarassingly parallel. We can
  have babel operate on every file twice, once for the browser and once for node
  (for pre-rendering).
* The processing step for any files in `static/` are embarassingly parallel as
  well, since we can copy/paste into the `public/` distribution directory.
* Once files in `src/` are compiled, the node component files can be used to
  pre-render HTML. We must either finish data processing or know that a
  particular page's JSON file is created/not going to be created by this time.
* The user's `page-wrapper` affects every single HTML file

in visual terms, we can imagine building a single HTML file for a theoretical
"blog post a". The HTML file is generated by `page-renderer-ssr`, which is a
Toast primitive and does not change unless Toast is upgraded. The
`page-renderer-ssr` requires the _actual_ node page component, the _actual_ page
data (optional), the _actual_ node-compatible page-wrapper (optional), and the
_location_ of the browser component (optional).

We will leave "shipping individual HTML files to the CDN" for another RFC, but
as a mention that is when the browser-page-component.js needs to exist and not
before. It is not needed for the generation of the page HTML itself, only the
location of the file.

```sh
.
└── blog-post-a.html
    └── page-renderer-ssr
        ├── node-page-component.js
        │   └── location-of-node-component
        ├── page-data.json
        │   └── data-pipeline
        ├── page-wrapper.js
        │   └── es6-import-map-dependency-tree (header.js, etc)
        └── the-location-of-browser-page-component
```

We could theoretically hash things at different points in the pipeline,
resulting in a merkle tree.

```
.
└── blog-post-a.html
    └── a1b1c1d
        ├── a1
        │   └── a
        ├── b1
        │   └── b
        ├── c1
        │   └── c
        └── d
```

The result of which is that we have multiple dependency trees of hashes and a
change in a leaf invalidates the HTML file and causes re-generation.

We could also skip the merkle tree (combination of hashes on parents) and
attempt to process until hitting a point at which the hash for the current step
matches the old one.

Ex 1: we get to the point of generating node-page-component, which ends up
having a file content hash of a1 (maybe there was an insignificant whitespace
change). We can then stop processing that chain immediately because the HTML
file won't change.

Ex 2: the `page-data.json` changes to be a hash of `b2`, we then scoop up the
files that are still the same and rendering the HTML file with the new props.

## Drawbacks

Why should we not do this?

Determining dependency graphs for dynamic imports is tough and on the level of
impossible in the extreme case. for example, if a pre-rendered component
included code that looked like this, we would have to always build it. It's
possible that this becomes a norm for lazy-loading because it is so well
supported and we would have to offer some kind of API to determine which files
this code depended on.

```js
let somevar = Math.rand();
// other code
const Component = import(`${somevar}.js`);
```

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

Pre-rendering based alternatives involve bundling and batch-data fetching
always. Gatsby is one system that does this as it relies on Webpack, which is a
serialization point for code compilation that happens before HTML can be
rendered. While this is partially ok in watch mode in development, webpack and
the surrounding code (such as, but not specifically) to do type-inference in
GraphQL, etc can result in long startup times.

### What other designs have been considered and what is the rationale for not choosing them?

- We could batch-render everything, every time. This was not chosen as the
  intent is to enable larger-scale systems to ship faster and batch-rendering,
  while fast in Toast, will never beat incremental compilation with a cache.

### What is the impact of not doing this?

We have to render everything, every time, leading to lost developer time as well
as longer production cycles.

## Prior Art

Since we are thinking of Toast as a compiler itself, I've included prior art for
incremental compilation in other compilers.

- Rust implements incremental compilation:
  https://blog.rust-lang.org/2016/09/08/incremental.html. This is what we're
  aiming for.
- [Sapper](https://sapper.svelte.technology/) is built on
  [Svelte](https://svelte.dev/), which is pioneering the
  ["compiler-as-framework" approach](https://svelte.dev/blog/sapper-towards-the-ideal-web-app-framework#The_compiler-as-framework_paradigm_shift).

Slightly related:

- Gatsby has something called
  ["Incremental Builds"](https://github.com/gatsbyjs/gatsby/issues/5002) that
  has been worked on for two years. This does not seem to be an incremental
  compiler but is relevant in how Gatsby is pursuing performance improvements.
  Notably there is a SaaS feature they offer called "Incremental Builds" that
  seems only partially related to incremental compilation. Perhaps best served
  as an "if we don't think about this now, it will be _really hard_ later"
  warning.
- Next.js relies on runtime SSR and has recently added static features that rely
  on runtime SSR for previews, etc.

## Unresolved Questions

- What parts of the design do you expect to resolve through the implementation
  of this feature before stabilization?

The idea of "hash thing and check has to continue" has merit. The acutal
implementation and further edge cases are TBD.

- What related issues do you consider out of scope for this RFC that could be
  addressed in the future independently of the solution that comes out of this
  RFC?

* User changes a blog post in a CMS, wants to see a "preview" without needing to
  run the site locally
* Large site (10K+ pages) or very large site (100K+ pages) can ship subsets of
  the site rather than full rebuilds

## Future Possibilities

- We could position the incremental compilation as something that can happen in
  production, shortening the lifecycle from code or data commit to seeing
  changes in production or a preview environment. Notably an incremental
  compiler could take only the data it needs and render a single html page,
  greatly reducing the turnaround time and infrastructure needed to support
  features like Preview.
- We could do incremental compilation on a sub-file level, noticing changes in
  specific exports, etc.
