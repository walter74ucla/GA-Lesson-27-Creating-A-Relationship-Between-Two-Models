# https-git.generalassemb.ly-WebDev-Connected-Classroom-two-model-relationship-build-blob-master-REA

# Creating A Relationship Between Two Models

## Lesson Objectives

1. Add Articles Array to Author Model
1. Display Authors on New Article Page
1. Creating a new Article Pushes a Copy Onto Author's Articles Array
1. Display Author With Link on Article Show Page
1. Display Author's Articles With Links On Author Show Page
1. Deleting an Article Updates An Author's Articles List
1. Updating an Article Updates An Author's Articles List
1. Deleting an Author Deletes The Associated Articles
1. Change Author When Editing an Article

### Relevant Documentation

1. [Populate](https://mongoosejs.com/docs/populate.html)

## Add Articles Array to Author Model

models/authors.js

```javascript
const Article = require('./models/articles.js')

const authorSchema = new mongoose.Schema({
  name: String,
  articles: [{
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Article'
  }]
});
```

- `[]` lets the author schema know that each authors's `articles` attribute will hold an array.
- The object inside the `[]` describes what kind of elements the array will hold.
- Giving `type: Schema.Types.ObjectId` tells the schema the `articles` array will hold ObjectIds. That's the type of that unique `_id` that Mongo automatically generates for us (something like `55e4ce4ae83df339ba2478c6`).
- `ref: Article` tells the schema we will only be putting ObjectIds of  `Article` documents inside the `articles` array.

## Display Authors on New Article Page

Require the Author model in controllers/articles.js:

```javascript
const Author = require('../models/authors.js')
```

Find all Authors When Rendering New Page:

```javascript
router.get('/new', (req, res)=>{
    Author.find({}, (err, allAuthors)=>{
        res.render('articles/new.ejs', {
            authors: allAuthors
        });
    });
});
```

Create A Select Element in views/articles/new.ejs:

```html
<form action="/articles" method="post">
    <select name="authorId">
        <% for(let i = 0; i < authors.length; i++) { %>
            <option value="<%=authors[i]._id%>"><%=authors[i].name%></option>
        <% } %>
    </select><br/>
    <input type="text" name="title" /><br/>
    <textarea name="body"></textarea><br/>
    <input type="submit" value="Publish Article"/>
</form>
```

## Creating a new Article Pushes a Copy Onto Author's Articles Array

controllers/articles.js:

```javascript
Articles.create(req.body, (err, createdArticle) => {
        if(err){
          res.send(err);
        } else {
          Author.findById(req.body.authorId, (error, foundAuthor) => {
            console.log(foundAuthor, 'foundAuthor');
            foundAuthor.articles.push(createdArticle);
            foundAuthor.save((err, savedAuthor) => {
              console.log(savedAuthor, 'savedNewAuthor')
              res.redirect('/articles');
            })
          })
        }
      })
```

**NOTE: req.body.authorId is ignored when creating Article due to Article Schema**

## Display Author With Link on Article Show Page

controllers/articles.js:

```javascript
// articles show route
// :id is going to be the articles id
router.get('/:id', (req, res)=>{
  // req.params.id is an article id
  Author.findOne({'articles': req.params.id})
    .populate(
        {
        path: 'articles',
        match: {_id: req.params.id}
        }) // this pulls in the articles document
    .exec((err, foundAuthor) => { //exec executes the query
      console.log(foundAuthor, ' this is found AUthorrrr')
      if(err){
        res.send(err);
      } else {
        res.render('articles/show.ejs', {
          author: foundAuthor,
          article: foundAuthor.articles[0]
        });
      }
    })
});
```

1. Line 1: We call a method to find only **one** `Author` document that matches the id: `req.params.id` (Which is the article id.

1. Line 2: We ask the articles array within that `Author` document to fetch the actual `Article` document instead of just  its `ObjectId`.

1. Line 3: When we use `find` without a callback, then `populate`, like here, we can put a callback inside an `.exec()` method call. Technically we have made a query with `find`, but only executed it when we call `.exec()`.

views/articles/show.ejs:

```html
<h1><%=article.title%></h1>
<small>by: <a href="/authors/<%=author._id%>"><%=author.name%></a></small>
```

## Display Author's Articles With Links On Author Show Page

authors/show route
```javascript
router.get('/:id', (req, res) => {

  Authors.findById(req.params.id)
  .populate({path: 'articles'})
  .exec((err, foundAuthor) => {
    if(err) console.log(err);
      console.log(foundAuthor)

      res.render('authors/show.ejs', {
        author: foundAuthor
      });
  })
});


```

views/authors/show.ejs:

```html
<section>
    <h2>Articles Written By This Author:</h2>
    <ul>
        <% for(let i = 0; i < author.articles.length; i++){ %>
            <li><a href="/articles/<%=author.articles[i]._id%>"><%=author.articles[i].title%></a></li>
        <% } %>
    </ul>
</section>
```

## Deleting an Article Updates An Author's Articles List

controllers/articles.js

```javascript
router.delete('/:id', (req, res)=>{
  Article.findByIdAndRemove(req.params.id, (err, deletedArticle)=>{
    Author.findOne({'articles': req.params.id}, (err, foundAuthor) => {
         if(err){
            res.send(err);
          } else {
            foundAuthor.articles.remove(req.params.id);
            foundAuthor.save((err, updatedAuthor) => {
              console.log(updatedAuthor);
              res.redirect('/articles');
            })
          }
    })
  });
});
```


## Deleting an Author Deletes The Associated Articles

controllers/authors.js

```javascript
const Article = require('../models/articles.js');

//...farther down the file
router.delete('/:id', (req, res)=>{
	Authors.findByIdAndRemove(req.params.id, (err, deletedAuthor) => {
    if(err) {
      console.error(err);
      res.send("it didn't work check the console")
    }
    else {
      // we want to get all the article id's associated

      Articles.remove({
        _id: {
          $in: deletedAuthor.articles
        }
      }, (err, data) => {
        console.log(data, ' dat')
        res.redirect('/authors')
      });

    }
  })
});
```

## Change Author When Editing an Article

controllers/articles.js

```javascript
router.get('/:id/edit', (req, res)=>{
  Author.find({}, (err, allAuthors) => {
    Author.findOne({'articles': req.params.id})
    .populate({path: 'articles', match: {_id: req.params.id}})
    .exec((err, foundArticleAuthor) => {
        if(err){
          res.send(err);
        } else {
          res.render('articles/edit.ejs', {
            article: foundArticleAuthor.articles[0],
            authors: allAuthors,
            articleAuthor: foundArticleAuthor
          });
        }
    })
  })
});
```

views/articles/edit.ejs

```html
<form action="/articles/<%=article._id%>?_method=PUT" method="post">
    <select name="authorId">
        <% for(let i = 0; i < authors.length; i++) { %>
            <option
                value="<%=authors[i]._id%>"
                <% if(authors[i]._id.toString() === articleAuthor._id.toString()){ %>
                    selected
                <% } %>
                >
                <%=authors[i].name%>
            </option>
        <% } %>
    </select><br/>
    <input type="text" name="title" value="<%=article.title%>"/><br/>
    <textarea name="body"><%=article.body%></textarea><br/>
    <input type="submit" value="Update Article"/>
</form>
```

**NOTE: to compare ObjectIds, which are objects, you must first convert them to Strings (e.g. articleAuthor._id.toString())**

Update the PUT route in controllers/articles.js

```javascript
router.put('/:id', (req, res)=>{
    Article.findByIdAndUpdate(req.params.id, req.body, { new: true }, (err, updatedArticle)=>{
        Author.findOne({ 'articles._id' : req.params.id }, (err, foundAuthor)=>{
		if(foundAuthor._id.toString() !== req.body.authorId){
			foundAuthor.articles.remove(req.params.id);
			foundAuthor.save((err, savedFoundAuthor)=>{
			
				Author.findById(req.body.authorId, (err, newAuthor)=>{
					newAuthor.articles.push(updatedArticle);
					newAuthor.save((err, savedNewAuthor)=>{
			        	        res.redirect('/articles/'+req.params.id);
					});
				});
			});
		} else {
			res.redirect('/articles/' + req.params.id)
		}
        });
    });
});
```
