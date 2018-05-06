# gatsby-source-prismic

Source plugin for pulling data into [Gatsby][gatsby] from [prismic.io][prismic] repositories.

## Features

- Supports Rich Text fields, slices, and content relation fields

## Install

```sh
npm install --save gatsby-source-prismic
```

## How to use

```js
// In your gatsby-config.js
plugins: [
  /*
   * Gatsby's data processing layer begins with “source”
   * plugins. Here the site sources its data from prismic.io.
   */
  {
    resolve: "gatsby-source-prismic",
    options: {
      // The name of your prismic.io repository. This is required.
      // Example: 'gatsby-source-prismic-test-site' if your prismic.io address
      // is 'gatsby-source-prismic-test-site.prismic.io'.
      repository: "gatsby-source-prismic-test-site",

      // An API access token to your prismic.io repository. This is required.
      // You can generate an access token in the "API & Security" section of
      // your repository settings. Setting a "Callback URL" is not necessary.
      // The token will be listed under "Permanent access tokens".
      accessToken: "example-wou7evoh0eexuf6chooz2jai2qui9pae4tieph1sei4deiboj",

      // Set a link resolver function used to process links in your content.
      // Fields with rich text formatting or links to internal content use this
      // function to generate the correct link URL.
      // The document node, field key (i.e. API ID), and field value are
      // provided to the function, as seen below. This allows you to use
      // different link resolver logic for each field if necessary.
      // See: https://prismic.io/docs/javascript/query-the-api/link-resolving
      linkResolver: ({ node, key, value }) => doc => {
        // Your link resolver
      },

      // Set an HTML serializer function used to process formatted content.
      // Fields with rich text formatting use this function to generate the
      // correct HTML.
      // The document node, field key (i.e. API ID), and field value are
      // provided to the function, as seen below. This allows you to use
      // different HTML serializer logic for each field if necessary.
      // See: https://prismic.io/docs/nodejs/beyond-the-api/html-serializer
      htmlSerializer: ({ node, key, value }) => (
        (type, element, content, children) => {
          // Your HTML serializer
        }
      )
    }
  }
]
```

## How to query

You can query nodes created from Prismic using GraphQL like the following:

**Note**: Learn to use the GraphQL tool and Ctrl+Spacebar at
<http://localhost:3000/___graphql> to discover the types and properties of your
GraphQL model.

```graphql
{
  allPrismicPage {
    edges {
      node {
        id
        uid
        first_publication_date
        last_publication_date
        data {
          title {
            text
          }
          content {
            html
          }
        }
      }
    }
  }
}
```

All documents are pulled from your repository and created as
`prismic${contentTypeName}` and `allPrismic${contentTypeName}`, where
`${contentTypeName}` is the API ID of your document's content type.

For example, if you have `Product` as one of your content types, you will be
able to query it like the following:

```graphql
{
  allPrismicProduct {
    edges {
      node {
        id
        data {
          name
          price
          description {
            html
          }
        }
      }
    }
  }
}
```

### Query Rich Text fields

Data from fields with rich text formatting (e.g. headings, bold, italic) is
transformed to provide HTML and text versions. This uses the official
[prismic-dom][prismic-dom] library and the `linkResolver` and `htmlSerializer`
functions from your site's `gatsby-node.js` to create the HTML and text fields.

**Note**: If you need to access the raw data, the original data is accessible
using the `raw` field, though use of this field is discouraged.

```graphql
{
  allPrismicPage {
    edges {
      node {
        id
        data {
          title {
            text
          }
          content {
            html
          }
        }
      }
    }
  }
}
```

### Query Link fields

Link fields are processed using the official [prismic-dom][prismic-dom] library
and the `linkResolver` function from your site's `gatsby-node.js`. The resolved
URL is provided at the `url` field.

If the link type is a web link (i.e. a URL external from your site), the URL is
provided without additional processing.

**Note**: If you need to access the raw data, the original data is accessible
using the `raw` field, though use of this field is discouraged.

```graphql
{
  allPrismicPage {
    edges {
      node {
        id
        data {
          featured_post {
            url
          }
        }
      }
    }
  }
}
```

### Query Content Relation fields

Content Relation fields relate a field to another document. Since fields from
within the related document is often needed, the document data is provided at
the `document` field. The resolved URL to the document using the official
[prismic-dom][prismic-dom] library and the `linkResolver` function from your
site's `gatsby-node.js` is also provided at the `url` field.

**Note**: Data within the `document` field is wrapped in an array. Due to the
method in which Gatsby processes one-to-one node relationships, this
work-around is necessary to ensure the field can accomidate different content
types. This may be fixed in a later Gatsby relase.

**Note**: If you need to access the raw data, the original data is accessible
using the `raw` field, though use of this field is discouraged.

```graphql
{
  allPrismicPage {
    edges {
      node {
        id
        data {
          featured_post {
            url
            document {
              uid
              data {
                title {
                  text
                }
              }
            }
          }
        }
      }
    }
  }
}
```

### Query slices

Prismic slices allow you to build a flexible series of content blocks. Since
the content structure is dynamic, querying the content is handled differently
than other fields.

To access slice fields, you need to use GraphQL [inline
fragments][graphql-inline-fragments]. This requires you to know types of nodes.
The easiest way to get the type of nodes is to use the `/___graphql` debugger
and run the below query (adjust the document type and field name).

```graphql
{
  allPrismicPage {
    edges {
      node {
        id
        data {
          body {
            __typename
          }
        }
      }
    }
  }
}
```

When you have node type names, you can use them to create inline fragments.

Full example:

```graphql
{
  allPrismicPage {
    edges {
      node {
        id
        data {
          body {
            __typename
            ... on PrismicPageBodyRichText {
              text {
                html
              }
            }
            ... on PrismicPageBodyQuote {
              quote {
                html
              }
              credit {
                text
              }
            }
            ... on PrismicPageBodyFootnote {
              content {
                html
              }
            }
          }
        }
      }
    }
  }
}
```

### Image processing

Coming soon!

To learn more about image processing, check the documentation of
[gatsby-plugin-sharp][gatsby-plugin-sharp].

## Site's `gatsby-node.js` example

```js
const path = require('path')

exports.createPages = async ({ graphql, boundActionCreators }) => {
  const { createPage } = boundActionCreators

  const pages = await graphql(`
    {
      allPrismicPage {
        edges {
          node {
            id
            uid
            template
          }
        }
      }
    }
  `)

  const pageTemplates = {
    'Light': path.resolve('./src/templates/light.js'),
    'Dark': path.resolve('./src/templates/dark.js'),
  }

  pages.data.allPrismicPage.edges.forEach(edge => {
    createPage({
      path: `/${edge.node.uid}`,
      component: pageTemplates[edge.node.template],
      context: {
        id: edge.node.id,
      },
    })
  })
}
```

[gatsby]: https://www.gatsbyjs.org/
[prismic]: https://prismic.io/
[graphql-inline-fragments]: http://graphql.org/learn/queries/#inline-fragments
[gatsby-plugin-sharp]: https://github.com/gatsbyjs/gatsby/blob/master/packages/gatsby-plugin-sharp