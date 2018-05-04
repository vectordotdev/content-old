[Next.js](https://github.com/zeit/next.js/tree/v3-beta/) is an awesome new framework for building universal React applications. In simple terms, that means you can use React to render templates on the server, as well frontend components the way you're most likely used to. The advantages of this are numerous (shared components, faster rendering, great tooling), but getting it all to work properly is generally a pain.

Next.js makes this process easy, and with the release of V3 I thought I'd whip up a blog to learn and demonstrate how it works. In this tutorial we'll be using the following:

- `next` (3.X)
- `styled-components` (phenomenal css-in-js solution)
- `next-routes` (middleware for expressive routes in next)
- `express` (for serving our pages, although you can also do static exports)

I'd highly recommend you follow along with the github repo here [https://github.com/timberio/next-go/](https://github.com/timberio/next-go/), as some components are left out for the sake of brevity.

### 1. Getting Started

Getting set up is simple, you can follow along in the docs but the gist of it is to install next, react, and react-dom, add a simple build script and create your index file.

`yarn add next@beta react react-dom --save`

Add the following scripts to your `package.json`

```json
{
  "scripts": {
    "dev": "next",
    "build": "next build",
    "start": "next start"
  }
}
```

Then create an `index.js` file inside of a `pages` folder in the root

```js
// ./pages/index.js

export default () => (
  <div>Welcome to next.js!</div>
)
```

Then you can just run `yarn dev` and you should be up and running on `localhost:3000`. Hot reloading is baked in by default, which you can peek at if you inspect the `.next` folder in your root directory.

### 2. Adding Some Style

Next up we'll configure [styled-components](https://github.com/styled-components/styled-components) to style our blog.

First run `yarn add styled-components`.

Then create a custom `_document.js` file in the root with the following:

```js
import Document, { Head, Main, NextScript } from 'next/document'
import { ServerStyleSheet } from 'styled-components'
import 'styles/global-styles';

export default class SiteDocument extends Document {
  render () {
    const sheet = new ServerStyleSheet()
    const main = sheet.collectStyles(<Main />)
    const styleTags = sheet.getStyleElement()
    return (
      <html>
        <Head>
          <meta charSet="utf-8" />
          <meta name="viewport" content="initial-scale=1.0, width=device-width" />
          {styleTags}
        </Head>
        <body>
          <div className="root">
            {main}
          </div>
          <NextScript />
        </body>
      </html>
    )
  }
}

```

The custom `_document.js` allows us to override the default page layout and inject our own styles and markup surrouding our react app.

### 3. Creating a Layout

Now let's create a main layout which all of our blog views will use, put the following in `layouts/Main.js`:

```js
/* layouts/Main.js */

import Head from 'next/head'
import Wrapper from './Wrapper'
import Nav from 'components/Nav'
import Footer from 'components/Footer'

export default ({ children, title = 'This is the default title' }) => (
  <Wrapper>
    <Head>
      <title>{ title }</title>
    </Head>
    <header>
      <Nav />
    </header>

    <main>
      { children }
    </main>

    <Footer>
      Footer
    </Footer>
  </Wrapper>
)
```

We'll use this layout to wrap our pages, which can override the `<Head>` tags and render content into the `{ children }` block.

### 4. Rendering Posts

Now that we have our layout set up, let's modify our `index.js` page to take advantage of it, and also render some posts.

Update `pages/index.js` with the following:

```js
import React from 'react'
import Layout from 'layouts/Main';
import { getPosts } from 'api/posts'
import { Link } from 'routes'

import Post from 'components/Post'

const IndexPage = ({ posts }) => (
  <Layout>
    <ul>
      {posts.map(p => (
        <Post key={p.title} post={p} />
      ))}
    </ul>
  </Layout>
)

IndexPage.getInitialProps = async ({ req }) => {
  const res = await getPosts()
  const json = await res.json()
  return { posts: json }
}

export default IndexPage
```

The key here is the `getInitialProps` on our `IndexPage` component, which fetches all of the required data needed for this page to render. When this page is accessed directly at `localhost:3000`, Next will take care of fetching the data before the page is rendered. If we're navigating to this page from another one, no additional page reloads will happen, Next's clientside routing will take over and fetch the data for us before rendering the component thanks to the `Link` component. You can even add the `prefetch` [property](https://github.com/zeit/next.js/tree/v3-beta/#prefetching-pages) to tell Next to prefetch that page for blazing fast page loads.

Now we'll use some sample json and place the api in `api/posts/index.js`:

```js
import fetch from 'isomorphic-fetch'

export function getPosts() {
  return fetch('https://jsonplaceholder.typicode.com/posts')
}

export function getPost(slug) {
  return fetch(`https://jsonplaceholder.typicode.com/posts?title=${slug}`)
}

```

And add our `Post` component in `components/Post/index.js`:

```js
import React from 'react'
import { Link } from 'routes'
import Wrapper from './Wrapper'

const PostItem = ({ post }) => (
  <Wrapper>
    <Link route='post' params={{ slug: post.title }}>
      <a>
        <h3>{post.title}</h3>
        <p>{post.body}</p>
      </a>
    </Link>
  </Wrapper>
)

export default PostItem
```

When you reload the page you should see a list of posts getting rendered by our index page like so (you can see the styles in the github repo [https://github.com/timberio/next-go/](https://github.com/timberio/next-go/)).

![Next.js Blog Preview](//images.contentful.com/h6vh38q7qvzk/4ZnPINC8piYgC6cwaueWso/aaa5a31ed35217f0595270e14b3566c1/Screenshot_2017-06-05_18.04.59.png)

### 5. Post Pages

Now that we have a list of posts, lets add a route to view each individual post. Create a new page in `pages/Post.js` like so:

```js
import React from 'react'
import Link from 'next/link'
import styled from 'styled-components'
import Layout from 'layouts/Main';
import { getPost } from 'api/posts'

const PostPage = ({ post }) => (
  <Layout>
  	<h1>{post.title}</h1>
  	<p>{post.body}</p>
  </Layout>
)

PostPage.getInitialProps = async ({ query }) => {
  const res = await getPost(query.slug)
  const json = await res.json()
  return { post: json[0] }
}

export default PostPage
```

This page is reponsible for fetching and rendering individual posts, so let's add a route to show it. We'll be using `next-routes` for some nice expressive route definitions, so simply run:

`yarn add next-routes`

and add a `routes.js` folder in the root with the following:

```js
const nextRoutes = require('next-routes')
const routes = module.exports = nextRoutes()

routes.add('index', '/')
routes.add('post', '/blog/:slug')
```

Then make sure to add this middleware in `./server.js`

```js
const express = require('express')
const next = require('next')
const routes = require('./routes')

const dev = process.env.NODE_ENV !== 'production'
const app = next({ dev })
const handle = app.getRequestHandler()
const handler = routes.getRequestHandler(app)

app.prepare()
.then(() => {
  const server = express()
  server.use(handler)

  server.get('*', (req, res) => {
    return handle(req, res)
  })

  server.listen(3000, (err) => {
    if (err) throw err
    console.log('> Ready on http://localhost:3000')
  })
})
```

Now our `<Link route='post' params={{ slug: post.title }}>` components in `pages/index.js` will map to this page with the proper params and if you click on one you should see something like this:

![Next.js Post](//images.contentful.com/h6vh38q7qvzk/74mFamDazCuewqAeYY8qYy/f16103179b5390a43be2ebc6d569f95c/Screenshot_2017-06-05_18.24.56.png)

That's it! You can easily sub in your own endpoints in `api/posts/index.js` to fetch from your API or CMS of choice.

You can see a working demo at [https://next-go.now.sh/](https://next-go.now.sh/) and view the code at [https://github.com/timberio/next-go](https://github.com/timberio/next-go). You can also learn more about Next at [https://learnnextjs.com/](https://learnnextjs.com/).