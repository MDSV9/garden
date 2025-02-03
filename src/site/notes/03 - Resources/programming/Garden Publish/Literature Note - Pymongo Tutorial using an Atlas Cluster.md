---
{"dg-publish":true,"permalink":"/03-resources/programming/garden-publish/literature-note-pymongo-tutorial-using-an-atlas-cluster/","tags":["mongodb","fastapi","Tutorial"]}
---


# What is a CRUD application

CRUD in different terms:

| CRUD   | SQL    | HTTPS                           |
| ------ | ------ | ------------------------------- |
| Create | INSERT | POST                            |
| Read   | SELECT | GET                             |
| Update | UPDATE | PUT to replace, PATCH to modify |
| Delete | DELETE | DELETE                          |

### In Concept

Data can put in storage

- Such storage must allow its data to be _readable_ and _updatable_
- Before such actions can be taken the storage location must _allocate_ and _initialize_ the content
- And at some later point the storage can be _deleted_ and _deallocated_

These make the basic operations of storage management **CRUD**

## Learning MongoDB with pymongo

This project focuses on learning by doing, we will follow the tutorial found in mongodb's website.
For now we will use the cloud solution MongoDB.Atlas but in future projects we will use docker and self hosting solutions

> [!note] Sources
> https://www.mongodb.com/resources/languages/pymongo-tutorial

## Requirements

- Python
- FastAPI
- python-dotenv
- pydantic

## Project Setup

Created a conda virtual env with the required packets, and created a mongodb account to use Atlas. For future projects I will try to use native MongoDB soulutions.
Using the `python-dotenv` we establish our variables to connect to the Atlas Cluster

- ATLAS_URI -> the connection string used by `pymongo` to connect to the cluster
- DB_NAME -> The name of the database we are going to create the CRUD actions

## models.py

Using `pydantic`'s `BaseModel` we create data classes for data validation.
Inside each model we include another class _Config_ which is acceced by `fastapi` for the documentation using `RestApi`.

We create 2 models:

1. Book

- This model validates the book data when creating a new book, takes in title, author, and synopsis.

2. BookUpdate

- Used to validate data before updating a book inside the database

## routes.py

This file creates all the CRUD functions we need for the application, we use FastAPI to do most of the heavy lifting for url routing and page formating(docs)
While pydantic takes care of the data validation from the imported models from the models.py file

### CREATE

`@router.post("/", response_description = ..., status_code = status.HTTP_201_CREATED, response_model = Book)`
Here we define our post request at the root path, if the function below is executed correctly the API return a 201 code.

```python
def create_book(request: Request, book: Book = body(...)):
  book = jsonable_encoder(book)
  new_book = request.app.database["books"].insert_one(book)
  created_book = request.app.database["books"].find_one(
    {"_id":new_book.inserted_id}
  )
```

First we define the function to create a new Book entry in the database, using the Request clas from FastAPI we obtain the HTTP response, from which we extract the body using the Body function from the fast api package.
After pydantic verifies the data for the book created, we then change it to a format that mongodb is able to handle(JSON).
We then acces the database named _books_ and use the `insert_one()` function from pymongo to add the new book to the collection.

> [!note] new_book = ?
> Bad naming -> new_book does not contain the book we inserted into the database, instead it contains the ruturn type InsertOneResult from the insert_one() funtion.
> This type of object has 2 attributes:
>
> - inserted_id: the inserted document's id
> - acknowledged: the response to the write operation

Finally we query to find the inserted book in the database, and return its id. (My guess is as a check that it was inserted correctly)

### READ

`@router.get("/", response_description="...", response_model=List[Book])`
Here we tell fastapi that calling the function will handle get requests to the root path, and we specify the type of object we will return in the body of the request.

```python
def list_books(request: Request):
    books = list(request.app.database["books"].find(limit=100))
    return books
```

Our read function, which lists the first 100 books of our database. But what if we want a specific book?

```python
@router.get("/{id}", response_description="Get a single book by id", response_model=Book)
def find_book(id: str, request: Request):
    if (book := request.app.database["books"].find_one({"_id": id})) is not None:
        return book
    raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail=f"Book with ID {id} not found")
```

Let's break it down line by line.
First we define the route, which will call this function while at the same time sets the value of _id_
Then we define our function and pass it the _id_ of the book we are trying to find.
Within our if statement we make use of the `:=` operator, which assigns and checks the validity of our statement at the same time. If our `find_one` query return something we then return the book. Otherwise we return an HTTP exception showing that the id for this book does not match with any within our database.

### UPDATE

`@router.put("/{id}", response_description="...", response_model=Book)`
Same as before we specify that the following funcition will make a _PUT_ request to the indicated path, and assign the value to _id_

```python
def update_book(id: str, request: Request, book: BookUpdate = Body(...)):
    book = {k: v for k, v in book.dict().items() if v is not None}
    if len(book) >= 1:
        update_result = request.app.database["books"].update_one(
            {"_id": id}, {"$set": book}
        )

        if update_result.modified_count == 0:
            raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail=f"Book with ID {id} not found")

    if (
        existing_book := request.app.database["books"].find_one({"_id": id})
    ) is not None:
        return existing_book

    raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail=f"Book with ID {id} not found")
```

For this function we use pymongo's `update_one(filter,update)` function and the `$set` operator, this allows us to simploify the process.
Firs we generate a new dictionary using the data validated from the BookUpdate pydantic model, then if the new dictionary is not empty we call the function, filtering by the _id_ assigned before.
We then use the operator `$set` which changes the values of each key found in the BookUpdate model passed into the function.

> [!note] $set
> This operation will only update the given fields of the book passed into it. That is why we filtered None values before.

Finally we return the updated entry of the book, if it still exists in the database. The reason for the final _raise_ is for the fringe case where the updated entry is deleted in between the update and find call.

### DELETE

```python
@router.delete("/{id}", response_description="Delete a book")
def delete_book(id: str, request: Request, response: Response):
    delete_result = request.app.database["books"].delete_one({"_id": id})

    if delete_result.deleted_count == 1:
        response.status_code = status.HTTP_204_NO_CONTENT
        return response

    raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail=f"Book with ID {id} not found")
```

One of the biggest differences between this function and the rest is the use of the _Response_ class, which helps us to convey that the function was executed correctly to the client.
We also use of the delete_one returned object attributes to check if our operation was succesful.

## main.py

Here we put everything together.
`from routes import router as book_router`
We import the router we created in _routes.py_ which contains all of the CRUD functions we created earlier.
`config = dotenv_values(".env")`
We read the environmental variables that we need to connect to the mongodb cluster.

```python
#Deprecated on_event
@app.on_event("startup")
def startup_db_client():
    app.mongodb_client = MongoClient(config["ATLAS_URI"])
    app.database = app.mongodb_client[config["DB_NAME"]]
    print("Connected to the MongoDB database!")
```

Our function for initializing our site, we start by createing MongoClient (connecting to the cluster). Then we specify which of the collections we are going to connect to, and finally we show that we are connected.

> [!info] app.mongodb_client
> We store the client instance as part of the app object(FastAPI) which makes the app available from the application.
> This allows us to put the CRUD functions in the _router.py_ file instead

```python
#Deprecated on_event
@app.on_event("shutdown")
def shutdown_db_client():
    app.mongodb_client.close()
```

Simple, close the client connection when shutting down the application.

```python
app.include_router(book_router, tags = ["books"], prefix="/book")
```

Since we created the routing operations in a different file we need to indicate as souch witht he _include_router_ function.
For better readablitiy of our API we tag it under _books_ and we add a prefix to the roting operations such that if in the _routes.py_ file they are on path `/` they will be instead be in path `/book/`
