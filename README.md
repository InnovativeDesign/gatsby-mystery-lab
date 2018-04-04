# The continued mystery :mag:

In this part of the mystery, you'll learn how to use those React component-building skills to display data from an external source. After that, you'll be creating the source yourself and deploying your site online.

Throughout this part, you'll get a feel for the following important Gatsby and React topics:

1. How to pull in external data as GraphQL queries (which you tried out in the last part) _from components_
2. How to deploy your Gatsby website
3. How to connect your external data provider with the deployment process (answering the question: _when I change something on an admin dashboard, how does it update the site?_)

## Part 1: You've got data! :mailbox:

> :warning: In this part of the mystery, we'll be using some credentials you've received in the previous part. **Make sure you have them on hand before you begin this part of the mystery!**


**Installing a source plugin**

For this walkthrough, we will be using [**Contentful**](https://www.contentful.com/) as our data provider. Contentful is a service that gives you a nice _content management system_<sup>1</sup> and an _open API_<sup>2</sup>.

Let's start by running `npm install --save gatsby-source-contentful`. This package will allow us to source our assets directly as GraphQL nodes.

<sub>[1] A dashboard that allows you to easily edit and create content, like WordPress</sub>
<sub>[2] API: application programming interface - a way for other applications (like your website) to get authorized access to a service's data (in our case, Contentful's)</sub>

**Adding the source plugin**

Next, open up `gatsby-config.js` in your project root to add another plugin (you previously added one for Google Sheets!) - and **add to the array** with this object for Contentful:

```jsx
{
  resolve: `gatsby-source-contentful`,
  options: {
    spaceId: `<SPACE>`,
    accessToken: `<KEY>`,
  }
}

```

**Hint:** You've received `SPACE` and `KEY`... from some other place.

**Check out the schema**

Previously, we sourced Google Sheets data as `googleSheetMysteryRow` nodes (which is how you got here!). Now, our nodes are going to be of the type `contentfulBlogPost`. Here's how it's structured:

```jsx
{
  contentfulBlogPost {
    title
    author
    date
    coverPhoto {
      file {
        url
        fileName
        contentType
      }
    }
    article {
      article // this is the actual article text! (in Markdown format)
    }
  }
}
```

> **Note:** The model for this node was defined by myself beforehand, but in the next Part, you'll be creating it yourself.

Notice that the article text itself is in `contentfulBlogPost.article.article` - make sure you keep this in mind when you are writing a component to render an article.

**Creating pages at build time**

Now that we know what's a part of our model for an article, let's do two things to get them to show up on our blog:

1. Write a GraphQL query to get _all_ of the `contentfulBlogPost` nodes, not just an individual one.
2. Use Gatsby's `createPage` function inside of `gatsby-node.js` to dynamically create pages during the _build_ step of our static site generation.

**_As a reminder_**, here's what Gatsby does when you run `gatsby build` / `gatsby develop`:

- Loads plugins and data from `gatsby-config.js`
- Allows plugins to do work and `-source` packages to fetch data from external sources
- Builds GraphQL schema from the resulting work of these sources
- Runs `exports.createLayouts` from `gatsby-node.js` (optional)
- **Runs `exports.createPages` from `gatsby-node.js` (optional)**
  -  Here's the step we'll be using. `createPages` allows us to dynamically create pages that aren't already inside of our `src/pages/` folder.
- Extracts queries exported by components (looks for `graphql` strings)
- Runs the queries, writes page data into static HTML/CSS/JavaScript

> :loudspeaker: **Now, think about why `createPages` is useful for our situation.**
> 
> We don't want to create a`new-article-2018-03-16.js` in `src/pages/` every time we need to post a new article. There would be tons of repeated code if we took this approach - the only change would be the existing content. If we wanted to make a small class name change for our article pages, we'd have to go back _and change each [some-article]-yyyy-mm-dd.js file_. 
> 
> Instead, we'll create a **template** for a single article view, and run `createPages` on each of our `contentfulBlogPost` models.

Let's begin by addressing the first task: **"Writing a GraphQL query to get _all_ of the `contentfulBlogPost` nodes, not just an individual one."**

Start by entering this query fragment into GraphiQL:

:globe_with_meridians:  `localhost:8000/___graphql`

```jsx
{
  allContentfulBlogPost {
    edges {
      node {
        // what goes in here?
      }
    }
  }
}
```

This query looks a little bit different, since we're now querying for _all_ of our `contentfulBlogPost` nodes. If we know that `node` represents a single `contentfulBlogPost`, how would you write this query to get all of the parameters of _each_ of our blog posts?

Try writing this out yourself, but if get stuck, go ahead and expand the solution below.

---
<details>
  <summary><b>Reveal solution</b></summary>
<p>
  
```jsx

{
  allContentfulBlogPost {
    edges {
      node {
        title
        author
        date
        coverPhoto {
          file {
            url
            fileName
            contentType
          }
        }
        article {
          article
        }
      }
    }
  }
}

```

</p>
</details>

---

Does your solution match the above? If not, take some time to understand why your understanding might have differed. Notice that the contents of `node` are the same properties we wrote above inside of `contentfulBlogPost`. This is because each `node` _is_ a `contentfulBlogPost`, as mentioned before.

If you understand why your solution works, run the query and see that the output has an array called `data.allContentfulBlogPost.edges`, which is a list of all the `contentfulBlogPost` nodes with all the fields that we queried for.

We'll now use this inside of our Gatsby project to start dynamically creating pages.

Our second task was to: **"Use Gatsby's `createPage` function inside of `gatsby-node.js` to dynamically create pages during the build process of our static site."**

First, let's add the following to `gatsby-node.js` (which should already exist in your project folder) to tell Gatsby that we want to run a function during the `createPages` step.

:page_facing_up: `gatsby-node.js`

```jsx
const path = require('path'); // We'll use this later

// The first parameter is an object with the following properties:
// 
//  - boundActionCreators: another object that 
//    contains the `createPage` function that we want to use.
//    
//  - graphql: a function that runs GraphQL queries  
//  
//  We destructure them with the syntax ({ boundActionCreators, graphql })
//  
exports.createPages = ({ boundActionCreators, graphql }) => {
  const { createPage } = boundActionCreators;
};
```

> <sub>**Optional sub-note**: If the `{}` stuff is looking like _not_ JavaScript to you, read up on [destructuring assignment here](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment). Specifically, the sections titled _Basic assignment_ and _Unpacking fields from objects passed as function parameter_. **Make sure you're in the Object destructuring super-section as well!**</sub>
> 

Gatsby will call now call this function when it's building pages for our website. We have access to both `createPage` and `graphql` functions through the parameters we received, so let's place our GraphQL query written above inside:

**THIS IS IMPORTANT: Please note the one change from the query above: we add an `id` field to the `node` for reasons explained later.**

:page_facing_up: `gatsby-node.js`

```jsx
exports.createPages = (...) => {
  ...
  graphql(`
  {
    allContentfulBlogPost {
      edges {
        node {
          id
          title
          author
          date
          coverPhoto {
            file {
              url
              fileName
              contentType
            }
          }
          article {
            article
          }
        }
      }
    }
  }
  `).then((queryResult) => {
    // How do we get the information return out of the "queryResult" parameter?
  });
}

```

Now that a GraphQL query runs when we attempt to build pages, we receive the result of the query inside of the `queryResult` parameter, which gets passed to us _asynchronously_ (meaning, not instantly).

Try writing the body of the `.then` callback to get the information we want (an array of `contentfulBlogPost` nodes) out of the `queryResult` parameter! Store the result inside of a variable called `blogPosts`.

_Bonus points for using object destructuring._
**Hint:** `queryResult` will be the exact same object we received back when running this in GraphQL.

---
<details>
  <summary><b>Reveal solution</b></summary>
<p>
  
```jsx

graphql(`
  ...
`).then((queryResult) => {
  let blogPosts = queryResult.data.edges;
});

```

</p>
</details>


---
<details>
  <summary><b>Reveal solution with bonus points</b></summary>
<p>

**It's okay to not use destructuring.** This is just a new way to access nested properties from function parameters, and you don't need to use or know about it.

```jsx

graphql(`
  ...
`).then(({
  data: {
    allContentfulBlogPost: { edges: blogPosts }
  }
}) => {
  // blogPosts already in scope from destructuring
});


```

</p>
</details>

---

We now have a query that returns us all the blog posts from Contentful. Next, we will use this information to create pages using the `createPage` function!

To do this, we can iterate over `blogPosts` and call `createPage` for each of the blog posts.

:page_facing_up: `gatsby-node.js`

```jsx
...
.then((...) => {
  ... // blogPosts defined before this
  
  // == { node: post } destructuring explanation ==
  // 
  // Remember: each of our objects inside of the edges array
  // (which we called blogPosts) has this structure:
  // 
  // { node: { <the rest of the Blog Post object> }  }
  // 
  // What we're doing is assigning the name "post" to the
  // "node" field inside of the object that is passed in
  // by our forEach callback (the object, which has the
  // structure defined above).
  // 
  blogPosts.forEach(({ node: post }, index) => {
    // 
    // Note that index, the second parameter of forEach,
    // is the place within the blogPosts array we are on:
    // 
    // It will become 0, 1, 2, ... on subsequent runs of the
    // forEach callback (this function).
    // 
    createPage({
      path: `/article/${index}`,
      component: path.resolve('src/templates/article.js'),
      context: { id: post.id }
    });
  });
});

  // An equivalent function that does not use destructuring
  // is in a section immediately below.

```
---
<details>
  <summary><b>Non-destructuring version (optional viewing)</b></summary>
<p>
  
```jsx

...
.then((...) => {
  ... // blogPosts defined before this
  
  blogPosts.forEach((dataEntry, index) => {
    // If this makes more sense to you,
    // by all means: use it!

    let post = dataEntry.node; 
    
    createPage({
      path: `/article/${index}`,
      component: path.resolve('src/templates/article.js'),
      context: { id: post.id }
    });
  });
});
```

</p>
</details>


---


Whew! **Let's break down what's happening now:**

- We created a `forEach` loop over our `blogPosts`, which calls the function we passed in as the first parameter on _each_ of the items in `blogPosts`.
- The function we passed in calls `createPage` with the following parameters:
  - `path`: The relative URL for this generated page. In our case, this will be `/article/0`, `/article/1`, etc - depending on where we are in the array (the `index` parameter).
  - `component`: The location of the React component to render this page. 
     > <sub>:warning:`path.resolve` **_is not_** the same as the `path` defined above - this `path` is a [Node.js](https://nodejs.org) module that we imported at the top of the file. The `path.resolve` function takes a relative file path (like `src/templates/article.js`) and returns the absolute file path (like `/Users/ethan/Documents/mystery-lab-part-2/src/templates/article.js`).</sub>
  - `context`: Extra data to be provided to the GraphQL query exported by the component rendering this page. You'll see why this comes in handy pretty soon!

If you run `gatsby develop` now, you'll notice an error occurs while attempting to `createPages`. This is because our `src/templates/article.js` component we provided to the `createPage` function doesn't actually exist yet!

We will create that in the next part.

**Creating page templates**

Start by creating a folder inside of `src/` called `templates`. This will be used to hold React components that won't immediately be given pages/page URL's, but we want to use them to create other pages dynamically.

Inside of your `src/templates/` folder, create the `article.js` file that we're missing. It's time to get back to writing components! First, some basics:

:page_facing_up: `src/templates/article.js`

```jsx
import React from 'react';

class ArticleTemplate extends React.Component {
  render() {
    return <p>I'm an article!</p>
  }
}

export default ArticleTemplate;
```

**Checkpoint:** Try re-running `gatsby develop` now, and navigate to `localhost:8000/articles/0` and `localhost:8000/articles/1`. You should see the same `ArticleTemplate` components rendered!

Now, we'll export a GraphQL query from `ArticleTemplate` to query for _specific_ information about this (singular) article.

We've already written a query to do this for us (_way back, when we were just poking around to see the schema_), but it will look slightly different to accept a new _query parameter_:

:eyeglasses: `Just for reference`

```jsx
query ArticleQuery($id: String!) {
  contentfulBlogPost(id: { eq: $id }) {
    title
    author
    date
    coverPhoto {
      file {
        url
        fileName
        contentType
      }
    }
    article {
      article
    }
  }
}
```

Let's break down the new components to this query:

- We _named_ this query explicitly (not required) `ArticleQuery`.
- This query accepts an `$id` parameter, of a String type.
- `contentfulBlogPost` looks for a node that has an `id` field that matches the `$id` parameter passed into the query.

Where is this _mysterious_ `$id` coming from? **It's coming from the `context` property we set inside of the object we passed into `createPage`!**

Since we loop over all of the `contentfulBlogPost` nodes to call `createPage`, this `ArticleTemplate` React component will be rendered with each `$id` (a Contentful unique ID) that we received. **This means that each post's `ArticleQuery` will be different and specific to that `contentfulBlogPost` ID!**

> If you do not understand this part, no worries - this is tough! Please ping `#development` in our Slack or reach out to Ethan. It's important that you understand why this query works before we move forward!

Now, we'll add this query to an exported constant of our `ArticleTemplate` component:

:page_facing_up: `src/templates/article.js`

```jsx
...
export default ArticleTemplate;
// This should currently be the end of the file. Add:

export const pageQuery = graphql`
// Type out the GraphQL query here!
`;
```

> <sub>**Optional sub-note:** Wondering why we're not using parentheses around this `graphql` statement? It's because this is actually _not_ the `graphql` function we saw earlier; it's a **[tagged template](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals#Tagged_templates)** of a template literal (string with backticks).</sub>

Now try visiting `localhost:8000/articles/0` and open up your handy React Developer Tools (`Ctrl or Command-Shift-J`) to inspect the `props` of your `ArticleTemplate`.

**Hint:** Use the eyedropper tool and click on your "I'm an article" message!

Expand the `data.contentfulBlogPost` prop to see that your data is now being loaded into your component!
![](https://i.imgur.com/pvT5YDU.png)

We now have all the pieces we need to display our article in the template. We'll do that next!

**Rendering our `this.props.data`**

Now having our externally-sourced data in props, we can write the `render` method of `ArticleTemplate` to actually show article contents!

I encourage you to take some time to think about the relation between your props given and the elements you need on the page.

Here's a rough wireframe of what we want on each page:

```
BREAKING: HEADLINE
 -------------------------
|                         |
|      <cover photo>      |
|                         |
 -------------------------
By: Author Name    3/15/18

(New York, NY) Reported in ...
 
```

This should mainly be a matter of getting the data you want from props and writing HTML elements to display them. _Do try putting this together yourself!_ 

However, if you're stuck - or you just want to check your answer against something that works:

---
<details>
  <summary><b>Reveal (one possible) solution</b></summary>
<p>
  
```jsx

import React from 'react';

// I use the Moment module to parse and format the date!
// 
// If you'd like to use this, run `npm install --save moment`
// and include this line:
const moment = require('moment');

class ArticleTemplate extends React.Component {
  render() {
    const {
      title,
      author,
      date,
      article: { article },
      coverPhoto: {
        file: { url: coverPhoto }
      }
    } = this.props.data.contentfulBlogPost;

    return (<div>
      <h1>{title}</h1>
      <img src={coverPhoto} />
      <sub>By {author} on {moment(date).format('MM/DD/YYYY')}</sub>
      <p>{article}</p>
    </div>);
  }
}
```

</p>
</details>

---

**Changing page locations**

Let's change our blog post URL's from `/articles/[some index number]` to `/articles/[the date and time]`.

**First: which file did we set where an _article's_ page URL would be?**

:one: `gatsby-config.js` :two: `gatsby-node.js` :three: `ArticleTemplate.js`

The answer:

`gatsby-node.js`! Remember when we called the `createPage` function, it looked something like:

```jsx
createPage({
  path: `/article/${index}`,
  component: path.resolve('src/templates/article.js'),
  context: { id: post.id }
});
```

The **`path`** property is where we told Gatsby to create the "route," or page location for the new page being created. Instead of just `index`, which doesn't mean anything to our user, let's create a URL with the date and time.

First, if you haven't already, install the [**`moment`** library](https://momentjs.com/), which is a nice way of parsing datetime strings and formatting them in a straightforward way. Run this command within your project directory:

:computer: `Your terminal, within your project root`

```
npm install --save moment
```

Now, we can import this dependency with the following line at the top of `gatsby-node.js`:

:page_facing_up: `gatsby-node.js`

```jsx
const moment = require('moment');
```

`moment` is now available to you as a call-able function: you pass in a datetime string like: `moment('2018-03-15')` and the return result is a "Moment object," which can be formatted. Here's some examples of using formatting:

```jsx
moment('2013-02-04T22:44:30.652Z').format('YYYY-MM-DD') // 2013-02-04
moment('2013-02-04T22:44:30.652Z').format('hh:mm') // 2:44
moment('2013-02-04T22:44:30.652Z').format('MMM Do') // Feb 4th
moment('2013-02-04T22:44:30.652Z').format('ddd, hA') // Mon, 2PM
```

See the full table of options you can pass into the `format` function of a [Moment object here](https://momentjs.com/docs/#/displaying/format/).

Let's now use `moment` to create formatted dates as parts of our page URL! Find this section of your `gatsby-node.js` file and make this modification:

:page_facing_up: `gatsby-node.js`

```diff
+ // Should be close to the top of your file:
+ const moment = require('moment');
...
blogPosts.forEach(({ node: post }, index) => {
    createPage({
-     path: `/article/${index}`,
+     path: `/article/${moment(post.date).format('YYYY-MM-DD-hh-mm')}`, 
      component: path.resolve('src/templates/article.js'),
      context: { id: post.id }
    });
  });
```

Restart your Gatsby development server to see your changes!

> :triangular_flag_on_post: **To verify that your path changes are working correctly:** try visiting `localhost:8000/article/2018-03-09-09-03`. You should end up at an article titled _Longer Subway Trains. High-Tech Signals. Communications Robots. Subway 'Genius' Ideas Announced._

:mag: **Mystery:** Find the post from the future. As a hint, this post will be authored on our next cancelled meeting date and time. _Finding this post is required to solve the mystery._

**Bonus part (optional):**

Make your page locations represent a part of the title, but separated by dashes - something like `/articles/longer-subway-trains-high-tech-signals`. This dash-separated string is sometimes called a "slug," and can be generated from the title of your article.

Find (or create!) a function to generate slugs from strings, and generate your articles at these slug-ified paths.

---
<details>
  <summary><b>Reveal solution</b></summary>
<p>
  
:page_facing_up: `gatsby-node.js`
  
```jsx

// Defined at top:
// https://gist.github.com/mathewbyrne/1280286
function slugify(text) {
  return text.toString().toLowerCase()
    .replace(/\s+/g, '-')           // Replace spaces with -
    .replace(/[^\w\-]+/g, '')       // Remove all non-word chars
    .replace(/\-\-+/g, '-')         // Replace multiple - with single -
    .replace(/^-+/, '')             // Trim - from start of text
    .replace(/-+$/, '');            // Trim - from end of text
}

[...]

blogPosts.forEach(({ node: post }, index) => {
  createPage({
    path: `/article/${slugify(post.title)}`,
    component: path.resolve('src/templates/article.js'),
    context: { id: post.id }
  });
});

```

</p>
</details>

---

## Part 2: Deploying your site with Netlify :earth_africa:

**But first, a precautionary measure**

In this step, we will be committing our work to a public GitHub repository. This means that we _don't_ want to leave our `KEY` and `SPACE` ID or the "InnoD Mystery.json" file floating around in our project!

The way we will avoid this for our Contentful information is by replacing your keys with the following, inside of `gatsby-config.js`:

:page_facing_up: `gatsby-config.js`

```javascript
...

{
  resolve: `gatsby-source-contentful`,
  options: {
    spaceId: process.env.SPACE,
    accessToken: process.env.KEY,
  },
},
  
...
```

> **Note:** What we've done here is tell Gatsby to source these authentication tokens from _environment variables_ in our currently running shell (terminal window). If you try running `gatsby develop` again, you'll get an error, because these are undefined!
> 
> To use environment variables in development, create a file called `.env` (name doesn't matter, but make sure it is in a line of the `.gitignore` file), and format it as follows:
> ```
> export SPACE=<paste the space ID here>
> export KEY=<paste the token here>
> ```
> Now, run `source .env` inside of your terminal window. Your environment variables are defined, and you can verify this by running `echo $SPACE`.
> 
> You only have to run `source .env` _once_ per open terminal window to get your environment variables loaded in!

Next, add the following to the `.gitignore` file, which is in your project root:

:page_facing_up: `.gitignore`

```diff
.env
InnoD\ Mystery-975d82e0c439.json
```

Now that we're safe from other people using our API keys, let's get committing and deploying!

**GitHub :heart: Netlify**

Let's get our project site online now - with [**Netlify**](https://www.netlify.com/), a free static site host and continuous deployment service. 

Netlify sources your sites from Git, so begin by initializing your project directory as a Git repository:

:computer: `Your terminal, within your project root`

```
git init
```

Next, add your project files to the working tree with:

```
git add .
```

> :warning: **BEFORE YOU CONTINUE:** Run `git status` and **_make sure_** that it does _NOT_ show `.env` or your `InnoD Mystery.json` file! If it is showing in your added files, look at the steps above again!


and track your changes as a "commit," with:

```
git commit -m "<insert a message of joy here>"
```

Verify that you have committed by running `git status` - you should see:

```
 On branch master
 nothing to commit, working tree clean
```

We're now ready to send a Git repository to a remote source.
Create a new repository on GitHub by [following this link](https://github.com/new) (if you don't already have GitHub, sign up for an account).

<sub>**Note:** You can leave out the README, LICENSE, and .gitignore parts of repository creation. </sub>

You should end up with a blank repository screen - follow the commands below (_they have a remote Git location specific to your repository name_):

![](https://i.imgur.com/lxGwvzb.png)

**Don't follow these commands above, look for this section on your repository page and use those!** Refresh your GitHub repository page, and you should see your entire project shown in the web view.

Next, sign into [Netlify with GitHub here](https://app.netlify.com/signup). You should be taken to a screen that looks like:

![](https://i.imgur.com/QHekHhB.png)

Click **New site from Git** and follow the instructions to connect your GitHub account's repositories to Netlify. Select your project repository that you created earlier. Once you get to **Step 3: Build options, and deploy!**, make sure to set your environment variables (under **Advanced build settings**):

![](https://i.imgur.com/4p4W7lZ.png)


Finally, click the Deploy button!
Your website will immediately start building: Netlify will download your project and run `gatsby build`. You can follow its build progress under the "Deploys" tab, which looks like:

![](https://i.imgur.com/UJRRind.png)

Once this is finished, visit the URL that Netlify has created for you (it should be at `[something-something-numbers].netlify.com`!) and verify that your website is displaying as expected.

## Part 3: The End

Our deployed Netlify site is now set up to re-build every time a new change is made to our GitHub repository. That's great, but we _also_ want the site to re-build when new content has been added from our CMS (Contentful, in our case).

In this final step of the mystery, you will set up your own Contentful and connect your Netlify deploy to this Contentful space to cause content updates to update your site.

Firstly, you'll want to [sign up for Contentful](https://www.contentful.com/sign-up/#dev) (you can sign up with GitHub, again!). For the _Organization_ field, you can just put your name.

When you login for the first time, click **Start building** to be taken straight into the dashboard. We will be working within this example project that Contentful generates for you.

**Importing the schema**

Normally, you'd start creating the schema ("Content model") for any kind of data using their nice web interface, but you've already mysteriously received a `mystery-content.json` file to import the `blogPost` content model. 

Begin by installing the `contentful-import` command line tool with NPM:

:computer: `Your terminal, anywhere`
```
npm install -g contentful-import
```

Next, you'll need a **management token** and your **space ID** to run the `contentful-import` tool. Create a management token by going to the API keys page (shown below):

![](https://i.imgur.com/m1AiIVg.png)

and click the **Content management tokens** tab. Click the **Generate personal access token** button and name this "Import Token," or anything you'd like. 

**_Keep the value that shows up next safe! Contentful will not show it to you again._**

Get your space ID from the URL you are at - it should be immediately after `/spaces/`.

![](https://i.imgur.com/syOauQS.png)

In the example above, my Space ID is `ml9wc867q9og`.

Now that we have a management token, our space ID, and `mystery-content.json`, you're ready to run the command below:


```
contentful-import --management-token <your token> \
  --space-id <your space ID>  --content-model-only true \
  --content-file <path to mystery-content.json>
```

If all goes well, you should see this at the end:

<img src="https://i.imgur.com/OqYWupW.png" width="300" />


> **Checkpoint:** Check in your **Content model** tab within Contentful to verify that "Blog Post" is now a kind of model that you can create! 

You don't have any Blog Post content yet, so head into the **Content** tab and create a couple by clicking **Add entry** > **Blog Post**.

**Using your Contentful space**

Now that we have the Blog Post model and some entries for it, we will update your Netlify site to use _your_ Contentful space.

First, we'll need to get a _Content delivery token_ (different, confusingly, than a Content _management_ token). In the API Keys page of the Contentful dashboard we were at before, click the **Content delivery/preview tokens** and click the automatically-generated **Example space token 1**. Copy the **Content Delivery Token** value on this page, and save it for the next part of this step.

Now go back to your Netlify dashboard and go to the **Settings** tab of your deployed site. Click the **Build and Deploy** link in the sidebar (shown below) and scroll to the **Build environment variables** section.

<img src="https://i.imgur.com/rKNeFJW.png" width="300" />

Click _Edit variables_ to change your `SPACE` to the Space ID you used earlier, and your `KEY` to the _Content delivery_ token we copied earlier.

> **Checkpoint:** After re-setting your environment variables, go back to the Deploys tab and click _Trigger deploy_. In a couple minutes, your updated site should contain the new content _you_ created (navigate to the `path` you used for article pages).

Now, let's connect it to your Netlify site so that when you publish a new Contentful entry, your site re-builds to show those changes!

**Connecting build hooks**

We will now set up something called a _build hook_. A build hook is a URL provided by Netlify that you can request to trigger a re-build of your site. In our case, we will set this build hook up between Contentful's entry publishing events and Netlify's build action.

In the Netlify dashboard, under the **Build and deploy** section of settings we were at earlier, scroll down to **Build Hooks**. Create a new Build hook. Call this something like "Contentful Deploy," and keep the branch at `master`. It will generate a URL that begins with `api.netlify.com` - copy this to your clipboard and head back into Contentful!

In Contentful, go to **Space settings** > **Webhooks** (just under the API keys section we visited last). Click **Add webhook** and name it what you'd like - the important thing is that the URL is the `api.netlify.com` URL that you generated in the last step.

Lastly, set this webhook to trigger on _Only selected events_ and select "Publish" and "Unpublish" for _Entry_ types (shown below):

![](https://i.imgur.com/MKTtCLA.png)

Setting this webhook will cause our Netlify site to re-deploy with new content whenever it is published (or removed) from Contentful!

> **Checkpoint:** Verify that creating and publishing a new blog post entry on Contentful causes a re-deploy on Netlify. Make sure that your new content is there, too!
 
:mag: **Finishing the mystery**

Go back to your contact form on your website, and fill out the form as follows:

```
Name: <your Space ID>
Email: build@innovativedesign.club
Messsage: <your content MANAGEMENT token>
```

Good luck with the end of the mystery!
