# **HOW TO:**

# _Route-based middleware to handle default population query logic_

## Example content structure:

> For this example, I will use a simple blog post type consisting of a **title, body, hero image, slug and authors**, which is a one-to-many relationship with a user. _Each blog post can have many authors_.

![example content structure](https://res.cloudinary.com/dxghtqpao/image/upload/v1660071065/Screen_Shot_2022-08-09_at_2.50.08_PM_uvrojg.png)

Before getting into how to implement custom middleware, let's look at why you might want to add middleware for this use case in the first place.

> Prefer video? [Watch here](https://www.loom.com/share/9f443a97c8b343b5b0a194f05e070d94)

---

## The problem:

- By default, population structure needs to be defined and sent on each `client-side` request, or the request will return only top-level parent content.

A `GET` request to `localhost:1337/api/blog-posts` returns the following:

```json
// localhost:1337/api/blog-posts
{
  "data": [
    {
      "id": 1,
      "attributes": {
        "title": "Test Blog Post",
        "body": "Test blog content",
        "slug": "test-blog-post",
        "createdAt": "2022-08-09T18:45:19.012Z",
        "updatedAt": "2022-08-09T18:45:21.710Z",
        "publishedAt": "2022-08-09T18:45:21.707Z"
      }
    }
  ],
  "meta": {
    "pagination": {
      "page": 1,
      "pageSize": 25,
      "pageCount": 1,
      "total": 1
    }
  }
}
```

This is not ideal, as important data has been excluded, such as the `heroImage` and `authors`' information.

## Populate = \*

An easy solution to the above problem involves adding `populate=*` to the initial query.

`localhost:1337/api/blog-posts?populate=*` which returns the following:

```json
// localhost:1337/api/blog-posts?populate=*
{
  "data": [
    {
      "id": 1,
      "attributes": {
        "title": "Test Blog Post",
        "body": "Test blog content",
        "slug": "test-blog-post",
        "createdAt": "2022-08-09T18:45:19.012Z",
        "updatedAt": "2022-08-09T19:22:39.637Z",
        "publishedAt": "2022-08-09T18:45:21.707Z",
        "heroImage": {
          "data": {
            "id": 1,
            "attributes": {
              "name": "test_cat.jpeg",
              "alternativeText": "test_cat.jpeg",
              "caption": "test_cat.jpeg",
              "width": 500,
              "height": 500,
              "formats": {
                "thumbnail": {
                  "name": "thumbnail_test_cat.jpeg",
                  "hash": "thumbnail_test_cat_2bdaa9fbe9",
                  "ext": ".jpeg",
                  "mime": "image/jpeg",
                  "path": null,
                  "width": 156,
                  "height": 156,
                  "size": 5.01,
                  "url": "/uploads/thumbnail_test_cat_2bdaa9fbe9.jpeg"
                }
              },
              "hash": "test_cat_2bdaa9fbe9",
              "ext": ".jpeg",
              "mime": "image/jpeg",
              "size": 21.78,
              "url": "/uploads/test_cat_2bdaa9fbe9.jpeg",
              "previewUrl": null,
              "provider": "local",
              "provider_metadata": null,
              "createdAt": "2022-08-09T19:06:25.220Z",
              "updatedAt": "2022-08-09T19:06:25.220Z"
            }
          }
        },
        "authors": {
          "data": {
            "id": 1,
            "attributes": {
              "username": "testUser",
              "email": "test@test.com",
              "provider": "local",
              "confirmed": true,
              "blocked": false,
              "createdAt": "2022-08-09T19:07:03.325Z",
              "updatedAt": "2022-08-09T19:07:03.325Z"
            }
          }
        }
      }
    }
  ],
  "meta": {
    "pagination": {
      "page": 1,
      "pageSize": 25,
      "pageCount": 1,
      "total": 1
    }
  }
}
```

While this does return _more_ data, the main flaw with this approach is that we don't have control over _what_ data is returned. We are still not receiving valuable information, such as the author's `role` while also receiving data we might not care about.

## Getting granular:

Instead of using `populate=*` we have the ability to filter our query using [LHS Bracket syntax](https://docs.strapi.io/developer-docs/latest/developer-resources/database-apis-reference/rest/api-parameters.html).

Our query might look something like this:

> `localhost:1337/api/blog-posts?populate[heroImage][fields][0]=name&populate[heroImage][fields][1]=alternativeText&populate[heroImage][fields][2]=caption&populate[heroImage][fields][3]=url&populate[authors][fields][0]=username&populate[authors][populate][role][fields][0]=name`

While this _does_ work; correctly returning the data specified, it is not really feasible to use. This query is quite unruly and certainly not something you'd want to consistently use throughout your application.

## Enter... Query-String

Using [query-string](https://www.npmjs.com/package/qs), we can implement the exact same query as above, in a much more readable and re-usable manner. The query can easily be used directly in the front-end of our application.

For example:

```javascript
const qs = require('qs')
const query = qs.stringify(
  {
    populate: {
      heroImage: {
        fields: ['name', 'alternativeText', 'caption', 'url'],
      },
      authors: {
        fields: ['username'],
        populate: {
          role: {
            fields: ['name'],
          },
        },
      },
    },
  },
  {
    encodeValuesOnly: true, // prettify URL
  }
)
// `localhost:1337/api/blog-posts?${query}`
```

Successfully returns the exact same result as the above query where we used bracket syntax:

```json
{
  "data": [
    {
      "id": 1,
      "attributes": {
        "title": "Test Blog Post",
        "body": "Test blog content",
        "slug": "test-blog-post",
        "createdAt": "2022-08-09T18:45:19.012Z",
        "updatedAt": "2022-08-09T19:22:39.637Z",
        "publishedAt": "2022-08-09T18:45:21.707Z",
        "heroImage": {
          "data": {
            "id": 1,
            "attributes": {
              "name": "test_cat.jpeg",
              "alternativeText": "test_cat.jpeg",
              "caption": "test_cat.jpeg",
              "url": "/uploads/test_cat_2bdaa9fbe9.jpeg"
            }
          }
        },
        "authors": {
          "data": {
            "id": 1,
            "attributes": {
              "username": "testUser"
            }
          }
        }
      }
    }
  ],
  "meta": {
    "pagination": {
      "page": 1,
      "pageSize": 25,
      "pageCount": 1,
      "total": 1
    }
  }
}
```

Now we are getting somewhere!

> For many use cases, this will be the logical end. However, if you find that you are re-using the same query over and over again, read on.

# **Query logic in middleware**

Now that we know how to build useful queries, we can look at optimizing the process further by adding a query directly into route-based middleware.

### Initializing the new middleware

In Strapi, we can generate boilerplate code directly from the CLI. In your terminal, run the command:

```bash
yarn strapi generate
```

From there, arrow down to `middleware`
![middleware](https://res.cloudinary.com/dxghtqpao/image/upload/v1660160852/Screen_Shot_2022-08-10_at_3.44.53_PM_auhydo.png)

You will be prompted to name the middleware.

Then you will need to select where you want to add this middleware.
![run middleware](https://res.cloudinary.com/dxghtqpao/image/upload/v1660161253/Screen_Shot_2022-08-10_at_3.54.00_PM_dg8yd4.png)

In our example, we will choose `Add middleware to an existing API` since we only want it to run on the `blog-post` route.

![select route](https://res.cloudinary.com/dxghtqpao/image/upload/v1660161362/Screen_Shot_2022-08-10_at_3.55.52_PM_ignbcb.png)

Now in our Strapi project if we navigate to `src > api > blog-post > middlewares > test.js` we will see the following boilerplate:

```javascript
'use strict'

/**
 * `test` middleware.
 */

module.exports = (config, { strapi }) => {
  // Add your own logic here.
  return async (ctx, next) => {
    strapi.log.info('In test middleware.')

    await next()
  }
}
```

### Enable middleware on route

Before utilizing the middleware, we first need to enable it on the route.

If we head to `src > api > blog-post > routes > blog-post.js` we will see the default route configuration:

```javascript
'use strict'

/**
 * blog-post router.
 */

const { createCoreRouter } = require('@strapi/strapi').factories

module.exports = createCoreRouter('api::blog-post.blog-post')
```

We will need to edit this file as follows:

```javascript
'use strict'

/**
 * blog-post router.
 */

const { createCoreRouter } = require('@strapi/strapi').factories

module.exports = createCoreRouter('api::blog-post.blog-post', {
  config: {
    find: {
      middlewares: ['api::blog-post.test'],
    },
  },
})
```

> Pro Tip :
> If you can't remember the internal UIDs of the middleware, in this case `api::blog-post.test`. From your terminal, you can run:

```bash
yarn strapi middlewares:list
```

> This will give you a list of all internal middleware UIDs in your project

![list of middlewares](https://res.cloudinary.com/dxghtqpao/image/upload/v1660163150/Screen_Shot_2022-08-10_at_4.17.29_PM_l0mhro.png)

> To see all of the available customizations for core routes, [check out the docs](https://docs.strapi.io/developer-docs/latest/development/backend-customization/routes.html#configuring-core-routers)

## Adding logic to our middleware

Now that our middleware has been initialized in our project and added to the `blog-post`route, it's time to add some logic.

The purpose of this middleware is so we do not need to build our query on the front-end to return the data we are looking to fetch.

By adding our logic directly to the middleware, all of the querying will happen automatically, when we head to the `localhost:1337/api/blog-post` route.

Instead of writing our query on the front-end, we will add it directly to the middleware, as such:

```javascript
// src > api > blog-post > middlewares > test.js

module.exports = (config, { strapi }) => {
  // This is where we add our middleware logic
  return async (ctx, next) => {
    ctx.query.populate = {
      heroImage: {
        fields: ['name', 'alternativeText', 'caption', 'url'],
      },
      authors: {
        fields: ['username'],
        populate: {
          role: {
            fields: ['name'],
          },
        },
      },
    }

    await next()
  }
}
```

Now we can stop the Strapi server and run `yarn strapi build` to rebuild our Strapi instance. Once the build is complete we can run `yarn develop` to restart the Strapi server.

Now it's time to see all of our hard work pay off.

If we go to the route `localhost:1337/api/blog-posts` the response returns:

```json
{
  "data": [
    {
      "id": 1,
      "attributes": {
        "title": "Test Blog Post",
        "body": "Test blog content",
        "slug": "test-blog-post",
        "createdAt": "2022-08-09T18:45:19.012Z",
        "updatedAt": "2022-08-09T19:22:39.637Z",
        "publishedAt": "2022-08-09T18:45:21.707Z",
        "heroImage": {
          "data": {
            "id": 1,
            "attributes": {
              "name": "test_cat.jpeg",
              "alternativeText": "test_cat.jpeg",
              "caption": "test_cat.jpeg",
              "url": "/uploads/test_cat_2bdaa9fbe9.jpeg"
            }
          }
        },
        "authors": {
          "data": {
            "id": 1,
            "attributes": {
              "username": "testUser"
            }
          }
        }
      }
    }
  ],
  "meta": {
    "pagination": {
      "page": 1,
      "pageSize": 25,
      "pageCount": 1,
      "total": 1
    }
  }
}
```

No ugly query string needed!

## Congrats for making it to the end!

A recap of what you've just learned:

- How to use `populate=*`
- How to query and filter using [LHS Bracket syntax](https://docs.strapi.io/developer-docs/latest/developer-resources/database-apis-reference/rest/api-parameters.html)
- How to use [query-string](https://www.npmjs.com/package/qs) to build custom client-side queries for better re-use
- How to add custom middlewares to your project
- How to implement middlewares on an api route
